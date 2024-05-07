# Théorie

Ce cours présente :
- le RAID et ses différents niveaux
  - Le RAID
  - Les niveaux de RAID
  - Différence entre RAID et backup
- les différents modes de stockage réseau
  - le NAS
  - le SAN
    - Fiber Channel
    - iSCSI
  - les modes d'accès
    - NFS
    - SMB/CIFS
  - Les systèmes de fichiers répartis
 
# Le RAID

## Qu'est-ce que le RAID ?

Le RAID permet d'isoler le stockage matériel (les disques physiques) du stockage vu par le système d'exploitation. Il est possible de cumuler plusieurs disques physiques en un seul gros disque virtuel, et/ou de répliquer les écritures entre différents disques pour apporter une tolérance aux pannes.

Le RAID peut être soit matériel (géré par un contrôleur matériel dédié) ou logiciel.

Dans les deux cas, le système d'exploitation verra seulement les volumes que le système de gestion du RAID lui présente.


## Les niveaux de RAID

Il existe plusieurs niveaux de RAID, en fonction des fonctionnalités voulues :

### Le Raid 0

Le RAID0, aussi appelé "stripping", permet de cumuler plusieurs disques physiques en un seul disque visible par le système d'exploitation.

Si j'ai 2 disques de 128Go et 1 disque de 1To, alors mon OS verra 1 disque de 1,25To.

Avantages : 
- l'OS ne gère qu'une seule "grosse" partition, au lieu de devoir gérer plusieurs partitions de taille différente (sous Windows, D:, E:, etc)
- les disques peuvent avoir des tailles différentes (on peut cumuler des disques de 120Go avec des disques de 256Go ou 512Go par exemple)
- les écritures sont plus rapides car elles peuvent être réparties sur plusieurs disques simultanément

Inconvénient :
- en cas de panne d'un des disques, *toutes* les données du RAID sont perdues

Le RAID0 apporte donc une facilité d'administration de plusieurs disques, au risque de tout perdre si un seul disque tombe en panne.

### Le RAID 1

Le RAID1, aussi appelé "mirroring", nécessite au moins deux disques exactement identiques. Il permet de répliquer les écritures sur plusieurs disques physiques.

Si j'ai 3 disques d'1To, alors mon système verra 1 disque de 1To.

Avantages :
- tant qu'au moins un disque fonctionne, alors les données ne sont *normalement* pas perdues
- comme les données sont répliquées sur l'ensemble des disques, les taux de lecture sont plus rapides car l'OS pourra lire des blocs en parallèle sur chacun des disques

Inconvénient :
- la taille d'un seul disque est visible par l'OS, quel que soit le nombre de disques physiques : on gaspille énormement d'espace

Le RAID1 apporte donc une très forte tolérance aux pannes, puisque chaque disque physique dispose de sa propre copie des données.

### Le RAID 5 et le RAID 6

Le RAID 5 nécessite au moins 3 disques de taille identique, tandis que le RAID 6 en nécessite au moins 4.

En RAID 5, chaque disque est découpé en blocs d'une certaine taille (figée lors de la configuration du RAID), par exemple 1Mo.  
Les écritures sont ensuite découpées en blocs d'1Mo, puis réparties séquentiellement sur chacun des disques, avec une petite subtilité : après deux blocs de données, le système rajoute un bloc dit *de parité* qui sert à contrôler la validité des données (un peu comme le checksum).  
Le bloc de parité est positionné différemment pour chaque ensemble de blocs de données afin que chaque disque physique ait à la fois des blocs de données et des blocs de parité.

Par exemple, pour un fichier de 2Mo : 
- le disque 1 aura les données de 0 à 1Mo
- le disque 2 aura les données de 1Mo à 2Mo
- le disque 3 aura les données de parité

Pour un fichier de 6Mo :
- le disque 1 aura les données de 0 à 1Mo
- le disque 2 aura les données de 1 à 2Mo
- le disque 3 aura le premier bloc de parité
- le disque 1 aura les données de 2 à 3Mo
- le disque 2 aura le deuxième bloc de parité
- le disque 3 aura les données de 3 à 4Mo
- le disque 1 aura le troisième bloc de parité
- le disque 2 aura les données de 4 à 5Mo
- le disque 3 aura les données de 5 à 6Mo
Chaque disque a donc 2 blocs de données et 1 bloc de parité

Le RAID 6 est l'équivalent du RAID 5, mais avec 2 blocs de parité.

Avantages :
- contrairement au RAID 1, on ne perd "qu'un" disque en RAID5 et "que deux" disques en RAID 6
- le système continue de fonctionner en cas de panne d'un disque en RAID5 et de deux disques en RAID 6
- il est possible de reconstruire les blocs manquants à partir des données stockées sur les autres disques (on parle de *reconstruire* le RAID)

Inconvénients :
- le calcul et la vérification du bloc de parité ne sont pas neutres en ressources
- pour reconstruire le RAID, il est nécessaire de déconnecter les volumes (déconnecter la lettre de lecteur sous Windows, et démonter le disque sous Linux) : les données peuvent être innacessibles plusieurs heures/jours suivant la taille du RAID

Le RAID 5 et le RAID 6 sont très utilisés pour les serveurs de fichiers en entreprise, pour leur tolérance à la panne et leur "faible" espace perdu.

### Le Raid X+Y

Les RAID X+Y, aussi appelés *nested RAIDs*, permettent de cumuler deux niveaux de RAID X et Y (par exemple 0+1, 5+0, etc).

Le X correspond au niveau de RAID appliqué sur les disques physiques, afin de créer des disques intermédiaires.  
Le Y correspond au niveau de RAID appliqué sur ces disques intermédiaires.

Par exemple, sur un RAID 0+1 de 4 disques de 512Go : 

- on crée deux disques intermédiaires de 1To en appliquant du RAID0 (disques 1 et 2 d'un côté, disques 3 et 4 de l'autre)
- on crée ensuite un RAID 1 sur sur ces deux disques intermédiaires de 1To.
Bilan : l'OS voit un disque d'1To, les données seront répliquées sur les deux disques virtuels intermédiaires.

Pour un RAID 1+5, il faut 6 disques identiques :
- on crée 3 disques virtuels en appliquant du RAID 1 à chaque groupe de 2 disques physiques
- on applique le RAID 5 sur ces 3 disques virtuels

Dans ce cas, on peut perdre 1 disque physique par disque virtuel RAID1, et un disque virtuel RAID1 sur la grappe RAID5, sans perdre de données.

## Différence entre RAID et Backup

Définition d'un backup : disposer d'une copie de sauvegarde des fichiers pour pouvoir récupérer un fichier corrompu ou supprimé par erreur de l'environnement de travail principal.

Dans le cadre du RAID, les modifications sont effectuées instantanément sur les disques physiques :
- si je supprime un fichier d'un des disques en RAID1, il est supprimé sur *tous* les disques de la grappe
- si je modifie un fichier sur un des disques, la modification est reportée partout

Donc un RAID (autre que le RAID 0) apporte une tolérance à la panne d'un ou plusieurs disques physiques, mais il ne protège absolument pas des erreurs de manipulation !

# Les différents modes de stockage réseau

Lorsqu'on parle de "stockage réseau", les données traitées par la machine sont en réalité hébergées sur une autre machine du réseau (local ou WAN).

## Le NAS

Le stockage permet d'accéder au niveau *fichier* en tant qu'*utilisateur* à des données partagées sur le réseau.  
Lorsqu'on ouvre un fichier situé sur un NAS, le contenu du fichier transite sur le réseau pour être lu ou écrit.

Sous Windows, l'utilisateur affectera une lettre de lecteur (généralement N:, R:, S:, ou Z:) au lecteur réseau ; sous Linux, le dossier sera monté comme un disque local.

### Les modes d'accès

#### NFS

Le protocole NFS est le protocole standard sous Linux ; il permet de mapper les droits via UID/GID.

Plus d'informations ici :
- https://fr.wikipedia.org/wiki/Network_File_System
- https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-debian-11
- https://aws.amazon.com/fr/compare/the-difference-between-nfs-and-cifs/

#### SMB/CIFS

SMB et CIFS sont les protocoles développés pour permettre à des clients Windows d'accéder à des ressources sur le réseau. Le nom CIFS existe toujours bien que le protocole ait été rendu obsolète avec l'apparition de SMBv2 en 2008.

Ils permettent une authentification par domaine, et de partager autre chose que des répertoires : des imprimantes, etc.

Plus d'informations : 
- https://en.wikipedia.org/wiki/Server_Message_Block
- https://www.it-connect.fr/serveur-de-fichiers-debian-installer-et-configurer-samba-4/

## Le SAN

Contrairement au NAS qui travaille en mode "fichier", le SAN travaille en mode "bloc", c'est-à-dire en mode "disque" au niveau OS : le système d'exploitation voit un système SAN comme s'il s'agissait d'un disque dur physique directement relié à la machine.

### Fibre Channel

Fibre Channel est le mode standard pour monter des SAN en grande entreprise. Il nécessite une architecture complexe dédiée et coûteuse.

Plus d'information : 
- https://en.wikipedia.org/wiki/Fibre_Channel
- https://www.juniper.net/documentation/fr/fr/software/junos/storage/topics/concept/fibre-channel-fc-understanding.html


### iSCSI

iSCSI est un protocole permettant à un client d'émettre des commandes SCSI via le réseau.

Plus d'informations : 
- https://en.wikipedia.org/wiki/ISCSI
- https://www.ibm.com/docs/fr/flashsystem-5x00/8.5.x?topic=pc-iscsi-overview
