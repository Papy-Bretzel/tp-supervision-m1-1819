# Projet Supervision Ingésup M1 (2018-2019)
Dans ce document, nous, Victor Mignot et Clément Ruiz, décrirons dans un premier temps nos plateformes de développement et
leurs services, puis nous exposerons les solutions de surveillance et d'alerting choisies et mises en place,
et enfin présenterons quelques exemples de visualisation des données de supervision


## Présentation de l'infrastructure
La plateforme supervisée aujourd'hui est celle du projet libre Radio Bretzel (https://www.radiobretzel.org).
Elle est hébergée sur un seul serveur physique et comporte quelques services web et entièrement virtualisés.
Le réseau est géré par une instance OpnSense virtuelle placée en frontal. (cf Schéma Archi 1)
Afin de monitorer cette infra, nous utilisons la plateforme personnelle de Victor Mignot. Composée entre autre d'un pfSense virtuel en frontal,
comme pour Radio Bretzel, cette plateforme sera destinée à héberger les services de monitoring dans le cadre de ce TP.


#### Description des réseaux :
  * Perceval (victor-mignot.me):
    * 192.168.10.0/24 -> LAN. Réseau dédié au monitoring de l'infra de Radio Bretzel
  * Bretzelix-4000 (radiobretzel.org):
    * 10.12.10.0/24 -> ADMIN. Réseau de management interne de la plateforme Radio Bretzel
    * 10.12.20.0/24 -> DMZ_INT. DMZ interne hébergeant les différents services de Radio Bretzel
    * 10.12.30.0/24 -> DMZ_PUB. DMZ externe, hébergeant le reverse proxy (pas de WAF pour l'instant =( )
    * 10.12.40.0/24 -> DOCKER. DMZ dédiée à faire tourner des hôtes dockers contrôlés par des gitlab-runners
    * 10.12.12.0/24 -> VPN_SSL. VPN SSL OpenVPN hébergé directement sur l'OpnSense pour les accès de maintenance.

Des tunnels IPSec sont mis en place entre les 2 firewalls afin d'encapsuler le trafic entre les 2 plateformes à travers internet.

Tunnels IPSec :
  * ADMIN     <-->  LAN
  * DMZ_INT   <-->  LAN
  * DMZ_PUB   <-->  LAN
  * DOCKER    <-->  LAN
  * VPN_SSL    -->  LAN (dans le but de travailler depuis )

#### Description des services
Les machines de la plateforme Radio Bretzel ont les rôles suivants :
* jobmaster.admin.radiobretzel.org : Machine de pilotage, accessoirement hôte responsable des processus de backup et de déploiement de configuration (Ansible)
* console.admin.radiobretzel.org : Accès à la console de Proxmox (hyperviseur) depuis le réseau interne.
* fw.admin.radiobretzel.org : Accès à l'interface d'administration du OpnSense de Radio Bretzel.
* ns1.dmz.radiobretzel.org : Serveur DNS interne
* www.dmz.radiobretzel.org :  Serveur web classique. Site statique (HTML/CSS only)
* gitlab.dmz.radiobretzel.org : Instance gitlab installée via omnibus. Accessible depuis le net via https://source.radiobretzel.org. Héberge également un registre docker (https://registry.radiobretzel.org)
* chat.dmz.radiobretzel.org : Instance Mattermost câblée via oAuth à Gitlab. Installée via le gitlab omnibus installer. (https://chat.radiobretzel.org)
* proxy-01.public.radiobretzel.org : HTTP Reverse Proxy utilisant nginx. Projet de Naxsi / Modsecurity à l'étude, pas encore implémenté. Terminaison SSL pour les services accessibles depuis l'extérieur (Let's Encrypt)


## Présentation de la solution de supervision
Pour ce TP, nous avons voulu essayer la stack Prometheus / Grafana. Elle se compose de 3 éléments :
* Les `node_exporter`, agents de collecte sur les hôtes à surveiller. Il expose les métriques via HTTP (ou HTTPS)
* Prometheus Server, un ordonanceur de
