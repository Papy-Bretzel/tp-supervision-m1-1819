# Projet Supervision Ingésup M1 (2018-2019)

Auteurs :

- Clément RUIZ
- Victor MIGNOT

---

Dans ce document, nous décrirons dans un premier temps nos plateformes de développement et leurs services, puis nous exposerons les solutions de surveillance et d'alerting choisies et mises en place, et enfin présenterons quelques exemples de visualisation des données de supervision

## Présentation de l'infrastructure

L'infrastructure supervisée aujourd'hui est celle de la plateforme de développement de [Radio Bretzel](https://www.radiobretzel.org), un projet libre d'application web.
Entièrement virtualisée, elle est hébergée sur un seul serveur physique et propose quelques services web, tels qu'un outil de gestion de version, un registre Docker et un outil de chat.
Afin de monitorer cette infra, nous utiliserons la plateforme personnelle de Victor Mignot. Elle sera destinée à héberger les services de monitoring dans le cadre de ce TP.

### Définition des actifs et priorités

Dans un système d'information, certains éléments sont forcément plus sensibles que d'autres. Il est donc important de définir des priorités et des degrés de criticité en fonction de l'activité du SI et du métier ; une entreprise dont le métier est l'hébergement web sera un peu plus sensible à une chute du service `apache2` sur leurs serveurs de production qu'un service FTP sur l'environnement de développement.  

#### Principaux actifs

Pour commencer, toute la plateforme est accessible depuis internet via le nom de domaine [radiobretzel.org](https://www.whois.com/whois/radiobretzel.org).
Si la disponibilité de sa plateforme est appréciable, **l'essentiel du projet Radio Bretzel repose sur l'intégrité de son code et de sa documentation.** Ces éléments, ainsi qu'une partie de la configuration de la plateforme est versionné et géré à travers une instance Gitlab, accessible via l'URL <https://source.radiobretzel.org.> Ce service se constitue de l'application `Gitlab`, son serveur de base de données `mariadb` (hébergé sur la même VM), et il est placé derrière un reverse proxy qui s'occupe de la terminaison SSL. Également, Gitlab fait appel à un `gitlab-runner` comme outil de CI CD, qui permet de lancer des builds automatisés sur une instance Docker isolée (**docker-01.docker.radiobretzel.org**). Il fournit au passage un registre docker pour les projets qu'il héberge.

Gitlab est accompagné d'une instance `Mattermost` pour qui il sert de serveur d'authentification, et dans lequel il publie des messages relatifs aux pipelines de CI-CD et à la création d'issues. Elle est accessible via <https://chat.radiobretzel.org> et se constitue du service `mattermost`, de sa base de données `PostgreSQL`, et il est lui aussi placé derrière un reverse proxy.

D'autre part, un site web vitrine sert de page d'accueil pour les robots et les éventuels touristes qui découvrent le projet. Il est accessible via <https://www.radiobretzel.org> et n'est constitué que d'une seule page html statique hébergée sur un serveur `apache2`, toujours derrière notre reverse proxy.

Pour s'assurer de l'intégrité des données, un mécanisme de sauvegarde est mis en place et lancé depuis une machine d'administration. Il réalise un export des données des différents services, et les exporte vers une machine tierce, hors plateforme. Cette machine est une raspberry, utilisant Raspbian et accessible en SSH via le nom de domaine **papybretzel.ddns.net**.

Enfin, un serveur OpenVPN destiné à l'administration de la plateforme est accessible depuis internet via le nom de domaine **vpn.radiobretzel.org**.

#### Présentation technique

##### Plateforme monitorée

La plateforme de Radio Bretzel est hébergée sur un seul serveur physique loué chez **Online.net**. Elle est divisée en plusieurs réseaux pour isoler les différents services hébergés :

- **10.12.10.0/24** -> ADMIN. Réseau de management interne de la plateforme Radio Bretzel. Accessible en interne via le sous-domaine **admin.radiobretzel.org**
- **10.12.20.0/24** -> DMZ_INT. DMZ interne hébergeant les différents services de Radio Bretzel. Accessible en interne via le sous-domaine **dmz.radiobretzel.org**
- **10.12.30.0/24** -> DMZ_PUB. DMZ externe, hébergeant le reverse proxy (pas de WAF pour l'instant =( ). Accessible en interne via le sous-domaine **public.radiobretzel.org**
- **10.12.40.0/24** -> DOCKER. DMZ dédiée à faire tourner des hôtes dockers contrôlés par des gitlab-runners. Accessible en interne par le nom de domaine **docker.radiobretzel.org**
- **10.12.12.0/24** -> VPN_SSL. VPN SSL OpenVPN hébergé directement sur l'OpnSense pour les accès de maintenance.

Ces réseaux comprennent un ensemble de machines virtuelles :

- **jobmaster.admin.radiobretzel.org** : Machine de pilotage, accessoirement hôte responsable des processus de backup et de déploiement de configuration (Ansible)
- **console.admin.radiobretzel.org** : Accès à Proxmox (hyperviseur), soit la machine physique, depuis le réseau interne.
- **fw.admin.radiobretzel.org** : Firewall OpnSense en charge du routage et du filtrage réseau de la plateforme. Héberge également l'instance VPN via la prise en charge de la terminaison OpenVPN
- **ns1.dmz.radiobretzel.org** : Serveur DNS interne
- **www.dmz.radiobretzel.org** : Serveur web. Site statique (HTML/CSS only)
- **gitlab.dmz.radiobretzel.org** : Instance gitlab installée via omnibus. Accessible depuis le net <https://source.radiobretzel.org>. Héberge également un registre docker (<https://registry.radiobretzel.org>)
- **proxy-01.public.radiobretzel.org** : HTTP Reverse Proxy utilisant nginx. Projet de Naxsi / Modsecurity à l'étude, pas encore implémenté. Terminaison SSL pour les services accessibles depuis l'extérieur (Let's Encrypt)
- **chat.dmz.radiobretzel.org** : Instance Mattermost câblée via oAuth à Gitlab. Installée via le gitlab omnibus installer. (<https://chat.radiobretzel.org)>

##### Plateforme de surveillance

La plateforme de monitoring est externe, et a été montée pour l'occasion. Elle est hébergée sur un autre serveur physique, dans le cloud aussi.

- Perceval (**victor-mignot.me**):
  - 192.168.10.0/24 -> LAN. Réseau dédié au monitoring de l'infra de Radio Bretzel
Des tunnels `IPSec` sont mis en place entre les 2 firewalls afin d'encapsuler le trafic entre les 2 plateformes à travers internet. L'un est un `pfSense`, l'autre un `OpnSense`

Tunnels IPSec :

- ADMIN     <-->  LAN
- DMZ_INT   <-->  LAN
- DMZ_PUB   <-->  LAN
- DOCKER    <-->  LAN
- VPN_SSL    -->  LAN (sens unique)

La disponibilité des services énoncés précédemment dépend directement du service de DNS interne. Chacune des interactions des interactions entre les différentes application de la plateforme utilisent un serveur DNS

## Présentation de la solution de supervision

Pour ce TP, nous avons voulu essayer la stack `Prometheus / Grafana`. Elle se compose de 3 éléments indépendants :

- Les "**exporters**" sont des agents de collecte sur les hôtes à surveiller. Ces agents sont spécifiques au types de métriques recueillies. Par exemple, les données relatives au système de la machine seront traités pas le module officiel fourni par Prometheus, `node_exporter`. Cependant, d'autres modules non-officiels existent pour des applications, par exemple `mysqld_exporter` pour des métriques relatives à une instance mysql.
- **Prometheus Server**, l'outil de supervision en lui même. Il agrège les données qu'il récolte sur les `node_exporter` présents dans sa configuration. Il permet également de créer des politiques d'alerting, et offre une API pour d'éventuels outils de visualisation. Des opérations de traitements sont également possibles afin de générer des métriques dynamiques. (Ex : On récupère différentes informations sur la RAM, et on  calcule à la volée un pourcentage d'utilisation pour faciliter la lisibilité). Enfin, il fournit un interpréteur de son propre langage de requête dynamique, `PromQL`.
- **Grafana**, l'outil de visualisation et dashboarding via interface web. Il utilise `PromQL` pour générer dynamiquement des dashboard. Le site officiel [grafana.com](https://grafana.com/dashboards) recense des centaines de dashboards créés par et pour la communauté.

### Choix des métriques collectées

Le choix des métriques collectées découle directement de la priorisation des actifs et de la qualité de service visée.

Pour toute machine, virtuelle comme physique, le minimum est de surveiller:

- La charge CPU
- Le taux d'utilisation de la RAM
- L'espace disque disponible sur toutes les partitions
- Le taux d'occupation de la bande passante des cartes réseaux
- Le nombre de mises à jour système disponibles (de sécurité / logicielles)
- Certains services d'administration comme `sshd`

D'autre part, il est aussi intéressant de surveiller certains services propres au rôle de chaque machine. Par exemple :

- Surveiller `apache2` / `nginx` sur les serveurs web
- Surveiller `mysql` ou `pgsql` sur les serveurs `gitlab` et `mattermost`
- Surveiller `named` sur `ns1.dmz.radiobretzel.org`
- Surveiller `proxmox` sur les machines physiques
Pour cela il nous suffit de nous assurer que le service est actif et activé au démarrage.

Nous surveillerons aussi différents éléments propres à nos équipements réseaux, `OpnSense` et `pfSense` :

- Service `openVPN` hébergé sur `OpnSense`
- État du VPN IPSec reliant les deux infrastructures
- Les différentes métriques relatives au filtrage de trafic, comme par exemple le nombre de paquets rejetés par minute.

### Définition des seuils

Une fois les mécanismes d'agrégation de métriques en place, il nous faut définir des seuils à partir desquels nous souhaitons déclencher des actions.  
Dans le cas de Radio Bretzel, peu d'aspects de l'administration de la plateforme sont automatisés. Ainsi, les seules actions que nous choisirons de mettre en place sont des alertes par mail, ou par chat Mattermost.

#### Métriques système communes

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

## Mise en place de la solution de surveillance
