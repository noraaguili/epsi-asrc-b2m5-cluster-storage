# TP Stockage

Le but de ce TP est de manipuler :
- le RAID0 et le RAID1 via l'outil lvm2 sous Linux
- un NAS avec accès SMB sous Windows et NFS sous Linux

## Préparation du TP

- Forkez ce dépôt sur votre compte personnel (vous pouvez le mettre en privé)
- Créez un projet "EPSI - B2M5 - TP Stockage" sur GitHub
- Lisez tout ce que vous aurez à faire pendant le TP, et *créez les tâches sur votre projet !* Je vous ai prévenus, à partir de maintenant, manipulation technique = gestion des tâches sur un board ;-D
- Clonez votre dépôt via Codium
- Créez une branche "TP"
- Ajoutez vos notes dans ce fichier au fur et à mesure de votre avancement !
- Vous pouvez vous connecter en SSH en utilisant le Terminal intégré à Codium, de préférence le git bash
- Bonus points : créez une paire de clés "b2m5", et mettez la clé publique directement dans le authorized_keys de root pour pouvoir vous connecter en root directement ;-)



## Partie 1 : RAID avec LVM

LVM2 est l'outil standard sous Linux pour faire du RAID Logiciel. Son principal concurrent est mdadm.

### Pré-requis

- une machine Debian 12
- configuration matérielle :
  - 2vCPU
  - 4Go de RAM
  - un disque principal de 12Go
  - le réseau en NAT

### Installation et configuration de la VM

- Nom : EPSI-debian-b2m5.asr2024.epsi
- Réseau : DHCP
- mot de passe root : tyty123
- créer un user epsi, mot de passe tyty
- tout dans une seule partition
- installer seulement "Serveur SSH" et "utilitaires usuels du système"

Une fois l'installation terminée, *éteindre la VM*

### Ajout des disques supplémentaires

Dans la configuration de la VM, ajoutez **4** disques de 10Go chacun.  
Ces disques devraient apparaître comme sdb, sdc, sdd, et sde.

Une fois les disques ajoutés, *démarrez la VM*

#### Formatage des disques

- Connectez-vous en SSH sur votre VM avec l'utilisateur epsi
- Passez root avec la commande `su -` (ajoutez bien le - à la fin de la commande !)
- Utilisez l'utilitaire fdisk pour partitionner vos disques (appuyez simplement sur Enter lorsque fdisk vous demande le numéro de la partition, le secteur de départ, et la taille)

Exemple : 
```
fdisk /dev/sdb
p
n
p
<Enter>
<Enter>
<Enter>
w
```


### Création des volumes RAID

Documentation officielle de LVM pour Debian : https://wiki.debian.org/fr/LVM (elle est *un peu* obsolète)  
Documentation officielle Ubuntu (un peu plus claire) : https://doc.ubuntu-fr.org/lvm

- suivez ce tuto pour créer les deux raids : https://linuxfr.org/news/gestion-de-volumes-raid-avec-lvm
  - pour le raid0, créez le vg 'vgepsi0' et le lv 'lvepsi0' => vous devriez avoir un volume /dev/vgepsi0/lvepsi0
  - pour le raid1, créez le vg 'vgepsi1' et le lv 'lvepsi1' => vous devriez avoir un volume /dev/vgepsi1/lvepsi1
  - **attention** pour les 2 LV, ils doivent prendre *toute* la place disponible : il faut trouver le bon paramètre à donner à lvcreate (hint: https://www.linuxquestions.org/questions/linux-hardware-18/lvcreate-with-max-size-available-749253/)
  - utilisez l'utilitaire mkfs.ext4 pour formater vos lv

### Monter les volumes

- créez les dossier /opt/raid0 et /opt/raid1

Comme Linux affecte les lettres de disques (sda, sdb, sdc, etc) au démarrage, on n'a aucune garantie que les lettres seront préservées d'un redémarrage à l'autre.  
On va donc configurer nos points de montage en fonction de l'ID unique de chacune des partitions pour ne pas avoir de surprise :-)

Pour obtenir les ID des partitions, on va utiliser l'utilitaire `blkid`, et pour monter automatiquement les disques, on va modifier le /etc/fstab

- Créez dans le fstab les 2 lignes nécessaires pour monter automatiquement les deux volumes :-)
- Utilisez ensuite la commande `mount /opt/raid0` puis la commande `mount raid1` pour monter les 2 lecteurs

- Utilisez la commande `df -h` pour vérifier la taille de chacune de vos partitions. Que constatez-vous ?

- **n'éteignez PAS la VM à l'issue de la partie 1**


## Partie 2 : NAS


### Pré-requis

- avoir l'ISO de TrueNAS Core
- Une VM Linux qui servira de client NFS
- Un poste Windows qui servira de client (votre PC ira très bien)

### Création de la VM TrueNAS

Créez une VM :
- configuration matérielle :
  - 2vCPU
  - 4Go de RAM
  - un disque principal de 12Go
  - le réseau en NAT
 
Insérez l'ISO de TrueNAS dans le lecteur, et procédez à l'installation (il suffit de suivre les instructions à l'écran). Choisissez "tyty" comme mot de passe root (vous êtes en clavier QWERTY pendant l'install, on met toujours un mot de passe simple)

À la fin de l'installation, la VM va redémarrer. Si elle souhaite relancer le processus d'installation, *arrêtez-là*.  Sinon, utilisez l'option du menu pour l'éteindre proprement.

### Préparation du NAS

- Rajoutez un disque de 10Go à la VM, puis redémarrez-là
- Quand la VM démarre, elle vous indique une URL pour accéder à l'interface graphique ; ouvrez votre navigateur (Firefox) et accédez à cette URL
- connectez-vous avec le user root et le mot de passe tyty
- Allez tout de suite dans Accounts > Users, sélectionnez le compte root (petite flèche bleue à droite), cliquez sur EDIT, et changez son mot de passe pour un mot de passe plus sécurisé
- Allez dans Storage > Pools, puis cliquez sur ADD pour créer un nouveau pool de stockage que vous appelerez "PartageEPSI"
- Sélectionnez le seul disque qui vous est présenté (celui de 10Go), ajoutez-le au pool, validez (forcez le stripping sur un seul disque)

Le pool de partage est prêt !

### Création d'un user pour les partages

- Retournez dans Accounts > Users, et créez un user "Partage EPSI", "username "pepsi" avec un mot de passe assez simple
- Placez son home directory dans /mnt/PartageEPSI

### Partage SMB pour Windows

- Allez dans Sharing > Windows Shares (SMB)
- Cliquez sur ADD, puis créez le partage PartageEPSI en sélectionnant le dossier /mnt/PartageEPSI
- Sur votre poste Windows, ouvrez l'Explorateur de fichiers, et accédez à \\<ip de votre VM>\
- Que se passe-t-il ?
- Renseignez les credentials de pepsi, puis validez
- Que se passe-t-il ?
- Montez le dossier PartageEPSI dans le lecteur P: ; comment faites-vous ? (attention, ne *cochez pas* la case "se reconnecter automatiquement à l'ouverture de session)
- Créez le fichier P:\Readme.md, et ajoutez votre nom + le label "Windows" dans le fichier

### Partage NFS sous Linux

- Sur l'interface d'administration de TrueNAS, allez dans Sharing > Unix Shares (NFS)
- Sélectionnez le Path /mnt/PartageEPSI, puis cliquez sur "ADVANCED"
- Mappez le User root sur le user pepsi, et le groupe root sur le groupe pepsi
- Autorisez tout votre réseau NAT à accéder au partage

Le partage NFS est prêt !

Sur votre VM Linux : 
- Créez le dossier /opt/nas
- Suivez le tuto suivant pour monter votre dossier partagé sous Linux : https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-debian-11
- **attention**, ne suivez bien que la partie *client* du tuto ! Le serveur sera la VM TrueNAS :-)
- montez le dossier partagé dans le dossier /opt/truenas en tant que root (on ne va pas rentrer dans la gestion des Users dans ce TP)
- affichez le contenu du fichier /opt/nas/Readme.md
- ouvrez le fichier avec nano, puis rajoutez la ligne "et ici sous Linux !"
- sur votre poste windows, rouvrez le fichier P:\Readme.md. Que constatez-vous ?

# Fin du TP !
