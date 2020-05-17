# NTP HOWTO
Article original pour la certification.

## Résumé
Cet article vise à expliquer comment déployer le service NTP dans un environnement multi-tiers.
Avoir des horloges correctement configurées dans le système d'information est important pour notamment (1) la corrélation des logs et (2) pour la synchronisation d'équipements en fail-over.
Le protocole NTP permet la synchronisation des horloges à partir de sources locales (antenne GPS par exemple) ou de sources de confiance sur Internet, ou d'une source du réseau locale. NTP est un service de base d'infrastructe d'un système d'information.

## Constats
On peut parfois observer en entreprise, même dans les grands groupes, un manque de soin quant à avoir une base de temps fiable.
- *Des assets ne sont pas à l’heure*\
    Naturellement les équipements et serveurs ont des horloges qui dérivent dans le temps. Seul un service de temps permet une synchronisation. Autant dans un environnement Windows, ce service est "de base" pour peut que l'équippement fasse partie du domaine. Mais dans un environnement hétérogène (appliances, serveurs AIX ou Redhat) la synchronisation est loin d'être évidente.
- *Des serveurs ont le service ntp mal configurés*\
    Certains assets ont un service NTP mais non configuré (configuration par défaut) ou mal configuré (mauvais serveur NTP cible). Cela a pour conséquence de mettre le service NTP en erreur.
- *Règles de pare-feu manquantes pour de nombreux VLAN*\
    Le service NTP est bien configuré, mais cependant la règle de pare feu permettant le protocole de fonctionner (port 123 UDP) n'est pas paramétré. La conséquence est la même : service en erreur. Pourtant, le service NTP étant un service de base d'infrastructure, la règle sur NTP devrait être écrite de base lors de la création d'un VLAN par exemple, et porter sur tout le scope du VLAN.
- *manque de standard sur le paramétrage NTP*\
    Le paramétrage service NTP doit être spécifié par l'architecte du SI, avec validation du RSSI. Les questions à se poser sont par exemple : quels serveurs de confiance considérer ?, combien de strates NTP spécifier ?, quels sont les risques résiduels ?
    
Tout cela a pour conséquence d'avoir des timestamps de logs faux, entrainant des problèmes de corrélation de logs et d'imputabilité des actions d'administration. Et aussi de polluer les pare-feux avec des DROP indus, issus des tentatives de synchronisation des différents services NTP du SI.

## Architecture
L’architecture cible utilise les serveurs existants, et se base sur un découpage en 3 zones :
- Zone exposée à des requêtes venant d’internet ou devant les servir («DMZ ENTRANTE»)
- Zone dédiée à certains équipements d’infrastructure et aux serveurs générant des flux sortant de COMPANY («DMZ SORTANTE»)
- Zone interne des différents VLAN non exposés

Ce découpage est fonctionnel : chaque zone contient plusieurs VLAN et il y a du firewalling entre ces VLANs.
Le but est que les systèmes (serveurs, équipement réseau, ESX, baies de stockage…) de chaque zone soient à l’heure et utiliser des serveurs NTP propres à chaque zone pour synchroniser leur horloge.
L'intérêt d'avoir plusieurs serveurs NTP permet de respecter l'étanchéité entre les zones. Les assets en DMZ entrante, très exposés à des attaques venant internet (directement ou via un déplacement latéral de l'attaquant), ne doivent pas pouvoir contacter n'importe quel serveur en DMZ sortante par exemple.

- En zone «DMZ Sortante», les serveurs (nommons-les ntp-out-1 et ntp-out-2) sont paramétrés pour se synchroniser avec des serveurs NTP sur internet. Ce seront les serveurs de référence pour l'Entreprise et les serveurs NTP de tous les assets de cette zone.
- En zone «DMZ Entrante», les serveurs NTP (nommons-les ntp-in-1 et ntp-in-2) se synchronisent avec ceux de «DMZ Sortante». Les serveurs de cette ne contacterons que ces serveurs.
- En zone non exposée, les serveurs NTP sont les DC (Contrôleurs de Domaine Windows) les plus proches. Tous les DC sont synchronisés et se synchronisent eux-mêmes avec les serveurs NTP de «DMZ Sortante».

Le cas particulier des équipements "hors bande" c'est à dire les équipement d'administration d'infrastructures (consoles d'ordonnances, consoles firexall, ports d'administrations), plusieurs solutrions:
1. Ajouter une paire de serveurs NTP, et les synchroniser avec les serveur NTP de référence d'une zone non exposée
2. Considérer les serveurs NTP les plus proches. Si un ESX ne motorise que des VM de DMZ sortante, autant que ce serveurs ESX soit synchronisé avec le serveur NTP de la DMZ sortante.

## Paramétrage
### Principes du choix du serveur NTP
Il y a plusieurs manières de parvenir à ses fins pour avoir un système à l’heure :
- Si un système est virtuel, l’idéal est que l’ordonnanceur qui le motorise soit à l’heure via NTP et que la VM ait un driver qui permet à l’ordonnanceur de fournir directement l’heure système. Dans ce cas le serveur NTP de la VM n’a pas de valeur ajoutée ;
- Pour les systèmes Windows, physiques ou virtuels, les GPO forcent la synchronisation de l’horloge système avec les DC. Il n’y a pas de débat, vu que les DC sont directement ou indirectement synchronisés avec les serveurs NTP sortant ;
- Il reste les systèmes Unix physiques, les systèmes sans les drivers précités et les équipements (réseau ou autres) pour lesquels un paramétrage NTP est possible et donc nécessaire.

Au niveau du firewalling, il faut s’assurer que le flux udp/123 est autorisé entre VLAN où se trouve l’asset et le VLAN des serveurs de temps.
Pour les appliances, les options de paramétrages se limitent sauvent aux IP des serveurs de temps. Sur Redhat ou AIX, le fichier de paramétrage est un peu plus complet et nous allons en décrire la partie de fichier de configuration qui traite des connexions NTP.
À noter que les serveurs de temps précités ont un paramétrage NTP particulier (qu’il convient de ne pas modifier) et ne sont pas concernés par ce qui suit.

### Configuration de /etc/ntp.conf:
Ici c'est la description de la configuration du service ntpd d'un système quelconque (redhat, aix, …) souhaitant se synchroniser avec les serveurs NTP.
À noter que pour RedHat par exemple, il existe d’autres services NTP : celui de systemd (/etc/systemd/timesyncd.conf) et le service « chrony » qui remplace ntpd ; cependant ces cas ne seront pas abordés ici.
Dans ce fichier de configuration, seuls les paramètres 'restrict' et 'server' sont décrits, il faut vérifier les autres paramètres du fichier de configuration si nécessaire (driftfile, tinker…) mais ce n’est pas non plus l’objet du document.
 
Dans le cas par exemple de la «DMZ SORTANTE», les serveurs NTP sont ntp-out-1 et ntp-out-2.
La configuration est alors la suivante pour ces serveurs de temps :
``` 
restrict 127.0.0.1    nomodify notrap nopeer
restrict ::1          nomodify notrap nopeer
restrict ntp-out-1 limited kod nomodify notrap nopeer noquery
restrict ntp-out-2 limited kod nomodify notrap nopeer noquery
restrict -6 default   ignore
restrict default      ignore
server   ntp-out-1 iburst
server   ntp-out-2 iburst
```

Dans le cas «DMZ ENTRANTE», il faut remplacer l’IP des serveurs NTP par ntp-in-1 et ntp-in-2.
Dans le cas des DMZ non exposées, il faut se synchroniser avec 2 serveurs AD les plus proches autorisés à se faire contacter en UDP/123. 

## Forçage de la synchronisation 
Le protocole NTP est pénible dans le fait que si la différence d’horloge entre le système local et les serveurs de temps est trop importante, alors le service NTP refuse de remettre l'horloge à l’heure pour éviter certains plantage et attaques basées sur NTP. Comme il faut bien régler l’horloge au moins une fois, autant le faire au démarrage du système ou au démarrage du service. Les configurations suivantes décrivent la marche à suivre pour FORCER l’heure système en supposant que le fichier de configuration a été correctement paramétré. 

### Au démarrage du service ntpd de RedHat :
```
[root@myhost29 ~]# systemctl edit ntpd 
```
*Un éditeur vi apparait…, ajouter :*
```
[Service]
ExecStartPre=-/bin/bash –c '/sbin/ntpd -g –q'
Restart=always
RestartSec=30
```
* Sauvegarder. Si vous voulez synchroniser dans la foulée : *
```
[root@myhost29 ~]# systemctl restart ntpd
```

### Au démarrage d'un système AIX :
```
[root@myhost77 ~]# crontab –e
```
* Un éditeur vi apparait. vérifier les paths de chaque exécutable (ntpd, service) et adapter la ligne crontab pour que les paths soient complets *
```
@reboot /sbin/service ntpd stop ; /sbin/ntpd -g -q ; /sbin/service ntpd restart
```
* Sauvegarder *

### Manuellement : 
```
[root@myhost77 ~]# /sbin/service ntpd stop;/sbin/ntpd -g -q;/sbin/service ntpd restart
```
 
## Vérification de la synchronisation

C’est bien beau tout ça mais comment contrôler le bel ouvrage ? Pour cela il faut faire une requête sur le service ntpd local et connaitre son statut. C’est possible sur RedHat, AIX et certains appliances.

### NTPQ
La commande `ntpq` permet d'avoir le statut. Prenons par exemple un serveur de DMZ entrante. Ce serveur est synchronisé avec les serveurs NTP de DMZ entrante, qui sont eux-mêmes synchronisés avec les serveurs NTP de la DMZ sortante.

```
[root@myhost001 ~]# ntpq -pn
remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
ntp-in-1      ntp-out-2     4 u  386 1024  377    0.544   -0.688   0.384
ntp-in-2      ntp-out-2     4 u  430 1024  377    1.087   -0.967   0.654
```
- La colonne 'remote' décrit les serveurs de temps configurés dans /etc/ntp.conf
- Si le statut (colonne 'refid') vaut ".INIT." alors le client NTP tente de contacter le serveur 'remote', sinon il affiche l'IP du serveur 2 strates au-dessous (le serveur avec lequel le serveur 'remote' est synchronisé). 
- Les temps affichés à droite sont en millisecondes. La justesse de l'horloge système (colonne 'offset') est de l'ordre de la milliseconde, en général c'est suffisant.
 
Si après 10 secondes, 'st' ne contient que des ‘16’ et ‘refid’ que des ‘.INIT.’ alors il y a des problèmes de communication avec les serveurs et l'horloge locale reste désynchronisée. Vérifier les logs, la configuration et regarder les traces de pare-feux à la recherche de drops des paquets échangés en udp/123 entre le client (le service ntpd) et les serveurs de temps. Le cas échéant, faire une demande d’ouverture de route concernant tout le VLAN.
 
### « Stratum » NTP
Si le stratum (colonne "st" du tableau précédent) vaut 16 alors ntp n'est pas synchronisé avec le serveur de temps. 
Pour information, il y a 15 niveaux de strates dans le chainage des serveurs NTP. La strate 1 sont les serveurs Internet directement câblés sur une horloge atomique. 
- Si les serveurs NTP de «DMZ Sortante» arrivent à contacter ceux d’Internet, ils ont le  « stratum » de ces serveurs +1, soit 3 ou 4. Les autres serveurs de temps et assets auront alors un stratum inférieur ou égal à 7 (selon la DMZ)
- Dans le cas où les serveurs se trouvent coupés d’internet, l’horloge locale de ntp-out-1 sera la référence et ce serveur se donnera le stratum 10. Les autres serveurs de temps et assets auront alors un stratum entre 11 et 14 selon le chainage des serveurs.



