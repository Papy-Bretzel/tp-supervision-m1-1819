# Projet Supervision Ingésup M1 (2018-2019)

Auteurs :

- Clément RUIZ
- Victor MIGNOT

---

Dans ce document, nous décrirons dans un premier temps nos plateformes de développement et leurs services, puis nous exposerons les solutions de surveillance et d'alerting choisies et mises en place, et enfin présenterons quelques exemples de visualisation des données de supervision

## Infrastructure monitorée

### Présentation de Radio Bretzel

L'infrastructure que nous avons choisi d'utiliser pour cet exercice est celle de la plateforme de développement de [Radio Bretzel](https://www.radiobretzel.org), un projet libre d'application de webradio collaboratives.

Pour commencer, toute la plateforme est accessible depuis internet via le nom de domaine **radiobretzel.org** ([lien whois](https://www.whois.com/whois/radiobretzel.org)).
Si la disponibilité de sa plateforme est appréciable, **l'essentiel du projet Radio Bretzel repose sur l'intégrité de son code et de sa documentation.** Ces éléments, ainsi qu'une partie de la configuration de la plateforme est versionné et géré à travers une instance Gitlab, accessible via l'URL <https://source.radiobretzel.org.> Ce service se constitue de l'application `gitlab`, son serveur de base de données `postgresql`, et il est placé derrière un reverse proxy qui s'occupe de la terminaison SSL. Également, Gitlab fait appel à un `gitlab-runner` comme outil de CI CD, qui permet d'automatiser une partie du cycle de vie de l'application `radiobretzel`. Il fournit également le registre docker pour les projets qu'il héberge.

Gitlab est accompagné d'une instance `mattermost` pour qui il sert de serveur d'authentification, et dans lequel il publie des messages relatifs aux pipelines de CI-CD et à la création d'issues. Elle est accessible via <https://chat.radiobretzel.org> et se constitue du service `mattermost`, de sa base de données `postgresql`, et il est lui aussi placé derrière un reverse proxy.

D'autre part, un site web vitrine sert de page d'accueil pour les robots et les éventuels touristes qui découvrent le projet. Il est accessible via <https://www.radiobretzel.org> et n'est constitué que d'une seule page html statique hébergée sur un serveur `nginx`, toujours derrière notre reverse proxy.

Pour s'assurer de l'intégrité des données, un mécanisme de sauvegarde est mis en place et lancé depuis une machine d'administration. Il réalise un export des données des différents services, et les exporte vers une machine tierce, hors plateforme. Cette machine est une raspberry, utilisant Raspbian et accessible en SSH via le nom de domaine **papybretzel.ddns.net**.

Enfin, un serveur OpenVPN destiné à l'administration de la plateforme est accessible depuis internet via le nom de domaine **vpn.radiobretzel.org**.

### Description technique de l'infrastructure

La plateforme de Radio Bretzel est hébergée sur un seul serveur physique loué chez **Online.net**. La gestion du réseau n'est pas assurée par Proxmox, la solution d'orchestration de VM installée sur la machine hôte. Elle est déléguée à une instance OpnSense (fork de pfSense), qui assure le lien avec internet, ainsi que le routage et filtrage L3/L4 entre les différents sous-réseaux :

- **10.12.10.0/24** -> ADMIN. Réseau de management interne de la plateforme Radio Bretzel. Accessible en interne via le sous-domaine **admin.radiobretzel.org**
- **10.12.20.0/24** -> DMZ_INT. DMZ interne hébergeant les différents services de Radio Bretzel. Accessible en interne via le sous-domaine **dmz.radiobretzel.org**
- **10.12.30.0/24** -> DMZ_PUB. DMZ externe, hébergeant le reverse proxy. Accessible en interne via le sous-domaine **public.radiobretzel.org**
- **10.12.40.0/24** -> DOCKER. DMZ dédiée à faire tourner des hôtes dockers contrôlés par des gitlab-runners. Accessible en interne par le sous-domaine **docker.radiobretzel.org**
- **10.12.12.0/24** -> VPN_SSL. Réseau de clients VPN SSL OpenVPN. Le serveur est hébergé directement sur l'OpnSense et permet l'administration à distance de la plateforme.

Ces réseaux comprennent un ensemble de machines virtuelles :

- **jobmaster.admin.radiobretzel.org** : Machine de pilotage, responsable des processus de backup et de déploiement de configuration (Ansible)
- **console.admin.radiobretzel.org** : Accès à Proxmox (hyperviseur), soit la machine physique, depuis le réseau interne.
- **fw.admin.radiobretzel.org** : le firewall OpnSense en charge du routage et du filtrage réseau de la plateforme, ainsi que de l'accès VPN
- **ns1.dmz.radiobretzel.org** : Serveur DNS interne
- **www.dmz.radiobretzel.org** : Serveur web. Site statique (HTML/CSS only)
- **gitlab.dmz.radiobretzel.org** : Instance gitlab installée via omnibus. Accessible depuis le net <https://source.radiobretzel.org>. Héberge également un registre docker (<https://registry.radiobretzel.org>)
- **chat.dmz.radiobretzel.org** : Instance Mattermost câblée via oAuth à Gitlab. Installée via le gitlab omnibus installer. (<https://chat.radiobretzel.org)>
- **proxy-01.public.radiobretzel.org** : HTTP Reverse Proxy utilisant nginx. S'occupe de la terminaison SSL pour les services accessibles depuis le web (Let's Encrypt)

## Plateforme de surveillance

### Présentation technique

La plateforme de monitoring est externe, et a été montée pour l'occasion. Elle est, comme pour Radio Bretzel, entièrement virtualisée et hébergée sur un seul serveur physique dans le Cloud. Elle ne contient qu'un seul réseau, **192.168.10.0/24**, et qu'un seul hôte, hébergeant le service de surveillance, de visualisation et d'alerte, accessible à l'IP `192.168.10.20`. Une instance de `pfSense` permet de d'encapsuler le trafic entre les 2 plateformes à travers internet via des tunnels IPSec :

- ADMIN     <-->  LAN
- DMZ_INT   <-->  LAN
- DMZ_PUB   <-->  LAN
- DOCKER    <-->  LAN
- VPN_SSL    -->  LAN (sens unique)

Aucun filtrage réseau particulier n'est effectué en vue de l'aspect temporaire de la solution et nos besoins de flexibilité pour ce TP.

### Solution de supervision

Pour ce TP, nous avons voulu essayer la stack `Prometheus / Grafana`. Elle se compose de 3 éléments indépendants :

- Les "**exporters**" sont des agents de collecte sur les hôtes à surveiller. Ces agents sont spécifiques au types de métriques recueillies. Par exemple, les données relatives au système de la machine seront traités pas le module officiel fourni par Prometheus, `node_exporter`. Cependant, d'autres modules non-officiels existent pour des applications, par exemple `mysqld_exporter` pour des métriques relatives à une instance `mysql`. Une fois ces métriques collectées, elles sont exposées via HTTP (ou HTTPS) sur  [un port "réservé"](https://github.com/prometheus/prometheus/wiki/Default-port-allocations) en fonction du type d'exporter.

- **Prometheus Server**, l'outil de supervision en lui même. Son rôle est d'agréger les données en requêtant les exporters renseignés dans sa configuration. Il intègre un service d'alerte, qui peut aussi être externalisé, et offre une API pour d'éventuels outils de visualisation.  
Il fournit  également un interpréteur de son propre langage de requête dynamique, `PromQL` et des opérations de traitements sont également possibles afin de générer des métriques dynamiques. (Ex : On récupère différentes informations sur la RAM, et on  calcule à la volée un pourcentage d'utilisation pour faciliter la lisibilité).  
Cependant, il ne permet pas directement de traiter les alertes chaînées et la gestion des tickets, comme pourrait le faire un Zabbix. Ces fonctionnalités peuvent néanmoins être externalisées facilement via l'utilisation du plugins, assez répandus en vue de la popularité croissante de cette solution.

- **Grafana**, l'outil de visualisation et dashboarding via interface web. Son principe est simple, il agrège des données via différents **data sources**, et les mets en forme via différents types de graphiques définis par l'utilisateur. Nous utiliserons un plugin officiel de data source pour Grafana qui utilise `PromQL` pour récupérer les métriques que l'on souhaite mettre en forme sur un serveur `prometheus`. Le site officiel [grafana.com](https://grafana.com/dashboards) recense des centaines de dashboards créés par la communauté pour la plupart des systèmes, applications, et exporters officiels ou non-officiels.

![Diagramme Prometheus/Grafana](diapo/img/prom_grafana_system.jpg "Diagramme Prometheus/Grafana")

## Implémentation de la solution de surveillance

### Choix des métriques

Pour toute machine, virtuelle comme physique, il est essentiel de surveiller les différentes métriques systèmes et matérielles comme :

- La charge CPU
- Le taux d'utilisation de la RAM
- L'espace disque disponible sur toutes les partitions
- Le taux d'occupation de la bande passante des cartes réseaux
- Le nombre de mises à jour système disponibles (de sécurité / logicielles)

Aussi, il est aussi intéressant de monitorer l'état, le nombre de requête par minute et d'erreurs levées, pour les différents services de la plateforme :

- `sshd` sur tous les serveurs
- `nginx` sur les serveurs web
- `postgresql` sur les serveurs `gitlab` et `mattermost`
- `named` sur `ns1.dmz.radiobretzel.org`
- `openVPN` sur l'instance `OpnSense`

D'autre part, nous avons plus haut exprimé l'importance de l'intégrité du code le Radio Bretzel, et donc de la sensibilité de son processus de sauvegarde. Le processus de sauvegarde est assuré par un playbook Ansible joué chaque nuit via `cron`. Il récupère les éléments de configuration et de données des différents services de la plateforme, et en fait une archive, qu'il garde en local et qu'il exporte sur un troisième noeud, externe à la plateforme. Ainsi, il peut être intéressant de surveiller le code de sortie du job `cron` et l'output du playbook.

Enfin, nous devons également surveiller notre réseau, entre autres les différentes métriques relatives au filtrage du trafic et à l'occupation de la bande passante.

### Outils de collecte utilisés

Les métriques systèmes communes seront collectées par le service `node_exporter`, fournit par prometheus. Il est possible de configurer le node_exporter en lui passant différents flags en arguments afin de lui préciser quelles métriques doivent être collectées, et lesquelles peuvent être ignorées.
Un de ces flags correspond à la surveillance des différents services de  `systemd`, et nous permettra de faire un monitoring basique des différents services cités plus haut (`sshd`, `nginx`, `named`, etc...)

Pour monitorer nos services de manière plus précise, nous avons utilisé des exporters spécifiques :

- `nginx_exporter` pour les serveurs web et proxy, rendant compte du nombre de requêtes et de connexions actives.
- `postgresql_exporter` pour les services de base de données présents sur gitlab.dmz.radiobretzel.org et chat.dmz.radiobretzel.org.

Il est également possible de monitorer les jobs `cron` en utilisant une autre fonctionnalité de Prometheus qui est la _Push Gateway_. Elle constitue un intermédiaire entre les outils de collectes et Prometheus. Par exemple, nous exécutons un script, nous envoyons l'output vers la Push Gateway afin de rendre son contenu disponible au serveur prometheus. Contrairement au node_exporter, la Push Gateway est entièrement passive, et ne sert que de tampon dans des cas où les fonctionnalités des exporters sont limitées, comme par exemple dans la surveillance applicative.  
Cependant, nous n'avons pas eu le temps de mettre cette fonctionnalité en place via prometheus, à savoir le déploiement d'une Push Gateway. Nous utilisons à la place un [module Ansible](https://docs.ansible.com/ansible/latest/plugins/callback/mail.html) permettant d'envoyer un mail avec un résumé du playbook en cas d'erreur.

Pour finir, nous utiliserons un plugin Prometheus Exporter d'OpnSense qui permet de simuler un node_exporter sur OpnSense. Ce plugin nous renseigne sur l'état des interfaces, et du système en général, mais il lui manque quelques métriques qui nous auraient intéressées, comme l'état du serveur VPN  OpenVPN ou des passerelles IPSec.

### Alerting

Une fois les mécanismes d'agrégation de métriques en place, il nous faut définir des seuils à partir desquels nous souhaitons déclencher des actions.  
Dans le cas de Radio Bretzel, peu d'aspects de l'administration de la plateforme sont automatisés. Ainsi, les seules actions que nous choisirons de mettre en place sont des alertes par mail.

#### Seuils systèmes communs

Si certaines métriques sont systématiquement surveillées, leurs valeurs seuil sont pour la plupart conventionnelles, et ne nécessitent pas d'étalonnage préalables. Par exemple :

- RAM :
  - Si % utilisation > 85 % --> Avertissement
  - Si % utilisation > 95 % --> Alerte critique
- CPU :
  - Si % utilisation > 85 % --> Avertissement
  - Si % utilisation > 95 % --> Alerte critique
- Stockage (pour chaque partition) :
  - Si % d'espace libre < 15 % --> Avertissement
  - Si % d'espace libre < 5 % --> Alerte critique

#### Métriques liées aux services

Une alerte sera levée dès lors que l'un des services énoncés plus haut sera en erreur, ou arrêté. Les métriques liées au nombre de requêtes sont intéressantes mais ne constituent pas un facteur alarmant pour l'intégrité des données. Cependant, un grand nombre de services sont critiques, comme évidemment `gitlab`, mais aussi le service de DNS interne assuré par `named` sur ns1.dmz.radiobretzel.org. Sans DNS, l'intégralité de la plateforme et le processus de sauvegarde son mis en échec.

#### Seuils liés à surveillance du réseau

La détermination des seuils d'alertes liés au monitoring du réseau est plus délicate, car s'il peut être intéressant de faire des stress tests de l'architecture pour en tester la résistance à la charge, la taille du projet et le nombre de visites par an ne justifient pas de politique d'alerte particulière.

#### Implémentation de l'alerting

Dans les faits, l'alerting est assez simple à mettre en place, mais ne comporte que très peu de fonctionnalités. Les alertes se programment **par graphique**, à l'aide d'une requête à Prometheus, sur laquelle on fixe des conditions. Si ces conditions ne sont pas respectées, la notification est déclenchée.  

Le principal point négatif de ce système est **l'impossibilité de programmer plusieurs alertes par graphique**. Par exemple, si on souhaite placer un warning à 85% de charge CPU et une alerte à 95%, il faut deux graphiques. C'est très peu pratique et efficace.  
Un autre problème lié à ce système est à noter : **il est nécessaire d'écrire une requête sans variables liées au dashboard**, ce qui est logique mais embêtant car cela oblige à réécrire la requête du graphique pour enlever ces variables et à fixer en dur les hôtes concernés par l'alerte.

## Visualisation des données

Dans Grafana, les dashboards mettent en relation des données de différentes _Data Sources_, et les représentent graphiquement en fonction de certaines variables que nous définissons à l'avance. Le travail lié à la confection d'un dashboard peut rapidement devenir (très) laborieux, et la maîtrise du langage de requête de Prometheus n'est pas des plus intuitifs au premier abord, et nécessite quelques heures de pratiques avant de pouvoir être plus ou moins confortablement utilisé.

### Exemples de dashboards

![Dashboard système](diapo/img/explosion.png "Dashboard système")

![Dashboard PostgreSQL](diapo/img/postgres.png "Dashboard PostgreSQL")

![Dashboard NGINX](diapo/img/nginx.png "Dashboard NGINX")

## Limites observées et expérience personnelle

### Les limites de la stack Prometheus / Grafana
