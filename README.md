## Deploiement Vapormap

- Lancement de la pile swarm.yml sur Openstack
- ssh -i {{cle-prive}}.pem ubuntu@ip_adressePublic_node01
- Le noeud docker n'appartient pas à un cluster, il faut donc initialiser swarm :
 ``` docker swarm init ```

- Ajouter "Workers Nodes" au cluster:
   - ouvrir les autres noeuds: node02, node03, node04 avec ssh et ip_adressePublic
     Pour ajouter chaque noeud au cluster, on utilise la commande docker swarm join token qui est fournis par la commande docker    swarm init dans chaque noeud
- Creation des services:
  Dans le manager node01:

```bash
cd Vapormap
Docker stack deploy -c docker-compose.yml vapormap
docker service ls # pour verifier.
cd ..
```

## Monitoring avec Prometheus

```bash
ADMIN_USER='admin' ADMIN_PASSWORD='admin' ADMIN_PASSWORD_HASH='$2a$14$1l.IozJx7xQRVmlkEQ32OeEEfP5mRxTpbDTCTcXRqn19gXD8YK1pO' docker-compose up -d
```

Containers:

* Prometheus `http://<host-ip>:9090`
* Prometheus-Pushgateway  `http://<host-ip>:9091`
* AlertManager  `http://<host-ip>:9093`
* Grafana `http://<host-ip>:3000`
* NodeExporter 
* cAdvisor `http://<host-ip>:8080`
* Caddy

## Setup Visualizer

"Visualizer" est un outil permettant de visualiser graphiquement les noeuds d'un clusters Swarm, ainsi que les conteneurs déployés.

Pour le déployer dans un cluster Swarm, créer un service :
```bash
docker service create \
  --detach \
  --name=visualizer \
  --publish=8060:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer

```

## Setup Mysql_exporter

/ Création de service mysql exporter avec la commande :

```bash
docker service create --name mysql80-exporter --network vapormap_default -p 9104 --constraint "node.role==manager" –e DATA_SOURCE_NAME="user_vapormap:vapormap@(container_name:3306)/" prom/mysqld-exporter:latest --collect.info_schema.processlist --collect.info_schema.innodb_metrics --collect.info_schema.tablestats --collect.info_schema.tables --collect.info_schema.userstats --collect.engine_innodb_status

```

Pour granatir l'accés a la base de donnée on a ajouté a la commande {DATA_SOURCE_NAME="user_vapormap:vapormap@(container_name:3306)} qui définit l'utilisateur et le mot de pass de la base donnée db_vapormap et le conteneur de la base de donnée.

Et on definit les metriques --collect.info_schema.processlist --collect.info_schema.innodb_metrics --collect.info_schema.tablestats --collect.info_schema.tables --collect.info_schema.userstats --collect.engine_innodb_status

aprés la création de la service , on prend le port du service et on le remplace dans le fichier /prometheus/prometheus.yml

```yaml
  - job_name: 'Mariadb'
    static_configs:
      - targets: ['host-ip:{{port-service}}']

```
- Voir les metriques sur http://host-ip:port-service/metrics


## Setup Nginx_exporter

/ Création de service nginx exporter avec la commande :

Il est nécessaire d'utiliser un exportateur pour récupérer les métriques du service nginx, car Prometheus ne peut pas communiquer directement avec le service NGINX. Cela explique la présence d'un NginxExporter dans la liste des cibles de Prometheus.

Le service NginxExporter est un élément important pour récupérer les métriques d'un serveur web NGINX. En effet, cet exportateur jouera le rôle d'intermédiaire entre le serveur web et le service Prometheus, en récupérant les métriques NGINX, puis en les transmettant à Prometheus.

```bash
docker run -p 9114:9114 nginx/nginx-prometheus-exporter:0.10.0 -nginx.scrape-uri=http://10.29.247.93:8000/metrics

```
Ce service ne nécessite pas de fichier de configuration spécifique, juste une petite modification dans le fichier de configuration NGINX, pour autoriser une adresse IP autre que localhost afin que l'exportateur puisse accéder aux métriques.

Pour le faire, on accéde à l'intérieur du conteneur frontend à l'aide du commande ci-dessous:

```bash
docker exec -it frontend sh

```
Et après, on ajoute au fichier du configuration /etc/nginx/conf.d/default.conf le code ci-dessous:

```bash
 location /metrics {
   stub_status;
   # Security: Only allow access from the IP below.
   10.29.247.93
   # Deny anyone else
   deny all;
 }

```
Enfin, on redémarre le serveur nginx:

```bash
docker exec frontend  nginx -s reload

```
***Docker Host Dashboard***

Le tableau de bord de l'hôte Docker affiche des métriques clés pour surveiller l'utilisation des ressources de votre serveur :

* Disponibilité du serveur, pourcentage d'inactivité du processeur, nombre de cœurs de processeur, mémoire disponible, échange et stockage
* Graphique de la charge moyenne du système, en cours d'exécution et bloqué par le graphique des processus IO, graphique des interruptions
* Graphique d'utilisation du CPU par mode (guest, idle, iowait, irq, nice, softirq, steal, system, user)
* Graphique d'utilisation de la mémoire par distribution (utilisé, libre, tampons, mis en cache)
* Graphique d'utilisation IO (lire Bps, lire Bps et temps IO)
* Graphique d'utilisation du réseau par appareil (Bps entrants, Bps sortants)
* Permuter les graphiques d'utilisation et d'activité


***Docker Containers Dashboard***

Le tableau de bord des conteneurs Docker affiche des métriques clés pour surveiller les conteneurs en cours d'exécution :

* Charge CPU totale des conteneurs, utilisation de la mémoire et du stockage
* Graphique des conteneurs en cours d'exécution, graphique de charge du système, graphique d'utilisation des E/S
* Graphique d'utilisation du processeur du conteneur
* Graphique d'utilisation de la mémoire du conteneur
* Graphique d'utilisation de la mémoire cache du conteneur
* Graphique d'utilisation entrante du réseau de conteneurs
* Graphique d'utilisation sortante du réseau de conteneurs


## Gestions des alerts

***Monitoring services alerts***

Déclenchez une alerte si l'une des cibles de surveillance (node-exporter et cAdvisor) est indisponible pendant plus de 30 secondes :


***Docker Host alerts***

Déclenchez une alerte si le processeur de l'hôte Docker est soumis à une charge élevée pendant plus de 30 secondes :

```yaml
- alert: high_cpu_load
    expr: node_load1 > 1.5
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server under high load"
      description: "Docker host is under high load, the avg load 1m is at {{ $value}}. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

***Docker Containers alerts***

Déclenchez une alerte si un conteneur est indisponible plus de 30 secondes :

```yaml
- alert: mariadb_down
    expr: absent(container_memory_usage_bytes{name="container_name"})
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "mariadb down"
      description: "mariadb container is down for more than 30 seconds."
```

## Monitoring Avec Zabbix

On a utilise Zabbix pour monitorer le reseau et la gestion des logs.

** Monitoring du Reseau**

•	Utilisation de la machine zabbix-server installé dans le TP de zabbix.

•	Installez l'agent Zabbix sur votre hôte Docker : l'agent Zabbix est un processus léger qui s'exécute sur la même machine que votre hôte Docker et collecte des métriques à partir de votre application et de vos conteneurs. 

•	Commandes pour installer l’agent zabbix sur ubuntu 20.04 :
```bash
wget https://repo.zabbix.com/zabbix/5.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_5.0-1+focal_all.deb 
sudo dpkg -i zabbix-release_5.0-1+focal_all.deb
sudo apt update
sudo apt install zabbix-agent
```
•	Configuration de l'agent sur un hôte Ubuntu. Les fichiers de configuration se trouvent dans /etc/zabbix.

•	Réaliser la configuration de l'agent * Server : serveur autorisé à démander des métriques * ServerActive : l'adresse du serveur a qui envoyer les métriques (trapping) * Hostname : le nom de l'hôte.

•	Puis, redémarrer l'agent avec la commande ```sudo systemctl restart zabbix-agent.service```

•	Après avoir installer et configurer l’agent zabbix, il est temps pour créer un host dans l’interface web de zabbix avec le même nom donné avant dans le fichier de configuration de zabbix.

•	Il faut choisir un groupe pour le host et l’adresse IP privé de la machine dans laquelle l’agent zabbix a été configurer.

•	Il va falloir maintenant ajouter un Template qui exploite l'agent.

•	Pour cela, nous pouvons nous baser sur le template "Linux by Zabbix agent" prédéfini. 

•	Pour surveiller le service SSH, on ajoute un autre template prédéfini qui est "Template APP SSH Service"

•	On crée un trigger pour l’item utilisé dans le template SSH, qui va déclancher une alerte lorsque le service est down, l’expression utilisée dans ce trigger est la suivante :

``` {Hostname:net.tcp.service[ssh].last( )}=0 ```
avec Hostname c’est le nom du host qui a été créé.


•	Pour surveiller le service ICMP, on crée un nouvel Item en lui donnant un nom, un type : simple check, la clé : « icmpping » et le type d’information : numeric vu qu’il peut prendre deux états actifs ou non c-à-d 1 ou 0.

•	Maintenant il faut créer un trigger pour le même item qui va déclencher une alerte lorsque la valeur de l’item est égale à 0. Pour le créer il faut lui donné un nom est l’expression suivante : ``` {Hostname:icmpping.last( )}=0 ``` avec Hostname c’est le nom du host qui a été créé.

•	Pour surveiller le traffic réseau entrant et sortant sur l’interface réseau ens3, on crée deux nouvaux Item un pour le traffic entrant et l’autre pour le traffic sortant.
Les clès à mettre dans chaque item sont les suivants :
``` traffic entrant : net.if.in[ens3] ```
``` traffic sortant : net.if.out[ens3] ```

•	Pour surveiller l’état d’exécution des containers dans les hosts de la stack swarm, on configurer l’agent zabbix pour collecter des métriques en modifiant le fichier de configuration /etc/zabbix/*.conf et en ajoutant les lignes suivantes :
```bash
UserParameter=docker.discovery[*],/usr/bin/python3 /usr/local/bin/zabbix-docker.py discover "$1”
UserParameter=docker.stats[*],/usr/bin/python3 /usr/local/bin/zabbix-docker.py stats "$1" "$2"
```
Ces lignes configurent l'agent Zabbix pour utiliser un script Python appelé "zabbix-docker.py" pour découvrir et collecter des métriques à partir des conteneurs Docker.

•	Dans l'interface Web Zabbix, créer un nouvel élément pour surveiller la santé de votre application.

•	Dans le formulaire de configuration de l'article, vous pouvez spécifier les éléments suivants :

Nom : un nom descriptif pour votre élément, tel que "Application Health".

Type : sélectionnez "Agent Zabbix (actif)" comme type d'élément.

Clé : spécifiez la clé qui correspond au point de terminaison de vérification de l'état de l’application. La clé utilisé est : ``` net.tcp.service[http,IP publique de l’hote,8000]```

Intervalle de mise à jour : spécifiez l'intervalle auquel Zabbix doit collecter les données de votre application. Par exemple, vous pouvez définir l'intervalle sur 30 secondes.

•	Créer un déclencheur Zabbix pour alerter en cas d'état malsain : Enfin, vous pouvez créer un déclencheur Zabbix pour vous alerter lorsque la vérification de l'état de votre application échoue. Pour cela, rendez-vous dans l'onglet "Configuration", cliquez sur "Triggers", puis cliquez sur "Create Trigger". Dans le formulaire de configuration du déclencheur, vous pouvez spécifier les éléments suivants :
•	Nom : un nom descriptif pour votre déclencheur, tel que "Application défectueuse".
•	Expression : une expression logique qui prend la valeur true lorsque la vérification de l'état de votre application échoue. L’expression utilisée pour declancher l’alerte si la valeur de l’item = 0 est la suivante :

```{Hostname:net.tcp.service[http,IP publique de l’hote,8000].last( )}=0```


** Gestion des logs**

Nous avons supervisé un fichier logs généere par la command ```docker compose log```, Ce log contient des informations sur l’execution de l'application vapormap qui peuvent nous aider à réagir,aux problèmes potentiels, nous avons utilisé Zabbix qui permet la surveillance centralisé et l’analyse des logs. 
 un element de surveillance de log avec le mode actif de l’agent Zabbix et en utlisant  le chemin complet du fichier log qu’on souhaite surveiller. 

Nous avons créé un élément qui lit le fichier log. Pour ce faire, nous allons dans l'onglet "Configuration" de l'interface web de Zabbix, sélectionnez "Hosts", puis on sélectionne l'hôte. Cliquez sur l'onglet "Items", puis sur le bouton "Create item".

Dans le formulaire "Create item", on indique le type d'élément par "Zabbix agent (active)" et on définisse la clé "log[file,chemin vers le fichier journal]".

Puis on crée un trigger basé sur la sortie du log de l'élément. Pour ce faire, nous allon dans l'onglet "Configuration" de l'interface web de Zabbix, sélectionnez "Hosts", puis on sélectionne l'hôte que vous souhaitez surveiller. on cliquz sur l'onglet "Triggers", puis sur le bouton "Create trigger".

Dans le formulaire "Créer un trigger", on définit l'expression du trigger comme suit :

``` {<nom d'hôte>:log[file,<chemin vers le fichier journal>].str("Error")} ```

On remplace <nom d'hôte> par le nom de l'hôte qu'on surveille dans notre cas c'était "test", <chemin du fichier log> par le chemin d'accès au fichier log qu'on surveille.
Nous avons mis une condition sur le mot cle qui “error”, on recoit donc une alerte de haute niveau dans le daoshborad de zabbix lorsque le mot "Error" apparaît dans le fichier log

Et pour génère une alerte ou une notification lorsque le trigger est activé, on crée une action en spécifiant les destinataires du courriel, le sujet et le contenu du message dans la configuration de l'action.

