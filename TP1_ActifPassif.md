# TP1 - Cluster Actif/PAssif


Dans ce TP, nous allons mettre en place un cluster actif/passif de serveurs web, à l'aide des technologies corosync et pacemaker sous Debian.


## Pré-requis :


- 2 machines debian 12 avec apache d'installé et qui écoute sur le port 80

### Configuration commune d'une IP statique sur les VM

On va mettre une IP fixe sur chacune des VM. Pour cela, on va regarder l'adresse actuelle de la machine, et modifier le fichier de configuration du réseau sous Debian pour choisir une adresse dans le même réseau

1. noter l'adresse IP de la machine : `ip a`
2.  noter l'adresse de la passerelle par défaut : `ip route show | grep default` 

3. Éditer le fichier de configuration :

`nano /etc/network/interfaces`

Remplacer la ligne

```
iface ens18 inet dhcp
```

par

```
iface ens18 inet static  
    address VOTRE_ADDRESSE_STATIQUE/24  
    gateway <votre passerelle par défaut>
```

par exemple, si vous aviez 192.168.138.124 comme adresse, remplacez VOTRE_ADDRESSE_STATIQUE par 192.168.138.11 pour votre machine 1, et par 192.168.138.12 pour la machine 2


4. Appliquez la configuration

`systemctl restart networking.service`

5. Vérifiez la configuration

Vérifiez votre IP avec `ip a` puis essayez de pinguer les machines entre elles.

### Paramétrage du fichier hosts pour le DNS manuel

Sur chacune des machines, éditez le fichier /etc/hosts poiur y ajouter les lignes suivantes :

```
192.168.138.11 hote1
192.168.138.12 hote2
```

Remplacez les IP par celles que vous avez choisies précédemment.

### Configuration de la connexion sans mot de passe

1. Connectez-vous en root sur la machine 1
2. Lancez la commande `cd .ssh` (créez le dossier s'il n'existe pas)
3. Générez une clé SSH SANS METTRE DE PASSPHRASE avec `ssh-keygen -t ed25519 -f ~/.ssh/cle_root_1`
4. Créez le fichier `/root/.ssh/config` avec le contenu suivant :
```
Host hote2
    IdentityFile /root/.ssh/cle_root_1
```
Ce fichier sert à indiquer à SSH que lorsque root essaie de se connecter à hote2, il faut utiliser la clé privée /root/.ssh/cle_root_1
5. Affichez la clé *publique* avec `cat /root/.ssh/cle_root_1.pub`
6. Copiez cette clé vers le fichier /root/.ssh/authoriezd_keys sur la machine 2
7. Répétez les commandes sur la machine 2
8. Testez la connexion : `ssh root@hote2` depuis la machine 1 : il doit vous demander d'accepter la clé du serveur (répondez `yes`) mais pas de mot de passe !
9. Testez la connexion dans l'autre sens, avec le même résultat :-)

### Désactivation d'Apache

sur les deux serveurs, désactiver apache pour que le cluster le gère et pas le système Linux :

`systemctl disable apache2`

### Sur la première machine

mettre le contenu

> Serveur 1

dans le fichier /var/www/html/index.html

### Sur la deuxième machine

mettre le contenu

> Serveur 2

dans le fichier /var/www/html/index.html

## Installation des outils

Nous allons avoir besoin d'installer :

- corosync
- pacemaker
- crmsh, outil de configuration pour pacemaker

` apt install corosync pacemaker crmsh`

## Configuration de corosync

### Mise en cluster des deux machines

Ouvrez le fichier /etc/corosync/corosync.conf

1. Dans le bloc `totem`, changez le nom du cluster pour `ClusterWebEpsi`

2. Dans le bloc totem > interface, changez l'adresse pour mettre l'adresse de *votre réseau* (pas l'IP de la VM, l'adresse DU RÉSEAU)

3. Dans le bloc `quorum`, ajoutez une ligne avec la valeur `two_node: 1 ` car notre cluster ne contiendra que 2 noeuds

4. À la fin du fichier, ajoutez le bloc suivant :
```
nodelist {
        node {
                ring0_addr: 192.168.138.11
        }
        node {
                ring0_addr: 192.168.138.12
        }
}
```
Évidemment, pensez à remplacer les IP si vous n'avez pas les mêmes !

Une fois cette configuration terminée, copiez le fichier vers la deuxième machine (les deux fichiers doivent être identiques).

### Démarrage du cluster

Lancez le cluster avec

`systemctl start corosync && systemctl start pacemaker`

### Remarques

Attention, on n'a configuré absolument aucun chiffrement, aucune sécurité, on est en train de mettre en place un cluster de base !

En entreprise ou pendant votre MSPR, il faudra chiffrer la communication :-)

## Paramétrage de pacemaker

Corosync gère la bascule d'un côté à l'autre du cluster, pacemaker va gérer quels sont les services disponibles sur le cluster.

### Vérificaton du fonctionnement de pacemaker

`crm status`

Vous devriez obtenir l'affichage suivante : 

```
Last updated: Tue Mar 29 10:02:59 2016
Last change: Tue Mar 29 10:02:50 2016 by root via crm_attribute on hote1
Stack: corosync
Current DC: hote1 (version 1.1.14-70404b0) - partition with quorum
2 nodes and 0 resources configured

Online: [ hote1 hote2 ]
```

L'affichage doit être équivalent sur les deux serveurs (le Current DC et la liste des noeurs Online doivent être identiques).

### Ajout d'une VIP

On va configurer un service mis à disposition par le cluste : la VIP (c'st-à-dire l'IP qui va basculer d'un côté à l'autre).

`crm resource create ClusterIP ocf:heartbeat:IPaddr2 ip=192.168.138.20 cidr_netmask=24 op monitor interval=30s`

On demande ici à pacemaker de créer un service qui s'appelle ClusterIP, en lui fournissant 192.168.138.20/24 comme adresse, et on lui demande de la surveiller toutes les 30 secondes.

Plus d'infos : https://github.com/ClusterLabs/pacemaker/blob/main/doc/sphinx/Clusters_from_Scratch/active-passive.rst

REMPLACEZ l'IP par l'adresse en .20 qui se trouve sur le *même* réseau que vos VM.

### Ajout du service Apache

On ajoute un nouveau service au cluster : 
```
crm resource create LeSiteWebDuCluster ocf:heartbeat:apache  \
      configfile=/etc/apache2/apache2.conf \
      statusurl="http://localhost/server-status" \
      op monitor interval=1min
```

Attention, il s'agit d'une seule commande !

On crée donc un service qui s'appelle LeSiteWebDuCluster, de type apache, avec le fichier de config d'apache en paramètre, et en lui demandant de vérifier toutes les minutes que tout va bien (il faut obligatoirement un temps supérieur à celui du lien).

Il existe beaucoup d'autres services paramétrables via pacemaker (notamment proxmox par exemple).

Dans le cas d'un cluster actif/actif, il est possible de créer des groupes de services qui doivent tourner sur le même hôte (par exemple, apache et la VIP), et de gérer un ordre de démarrage des ressources (pour apache avant la VIP par exemple).

Plus d'informations : https://github.com/ClusterLabs/pacemaker/blob/main/doc/sphinx/Clusters_from_Scratch/apache.rst

### Vérification que tout est OK

`crm status`

Qu'est-ce qui est différent depuis la dernière fois ?

### Vérification par navigateur

Lancez votre navigateur, et accédez à l'URL http://192.168.138.20/

Quelle valeur s'affiche à l'écran ?

Éteignez le serveur correspondant, attendez deux minutes, et actualisez la page dans votre navigateur.

Que se passe-t-il ?

Rallumez la machine. Que se passe-t-il ?


# Fin du TP !
