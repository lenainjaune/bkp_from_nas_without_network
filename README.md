# bkp_from_nas_without_network

# Objectif
Sauvegarde différentielle fichiers entre pools ZFS RAID1 2 disques distants sans utiliser le réseau

Voir aussi : [ZFS - ajouter un pool de 1 disque pour du RAID1](https://www.commentcamarche.net/forum/affich-36406170-zfs-ajouter-un-pool-de-1-disque-pour-du-raid1?gF-4hFHlxvdlq69e5GwmSAzhIdyMIVCDBAsuKJhgBdY)

# Le contexte
Je voulais mettre en place un serveur de sauvegarde différentielle de fichiers **BKP** sous Linux ([rsnapshot](https://rsnapshot.org/download.html)) sur un RAID1 (miroir) ZFS de 2 disques qui sauvegarderait les données d'un serveur de fichier OpenMediaVault **NAS** sous Linux, lui aussi sur un RAID1 ZFS de 2 disques. Les serveurs NAS et BKP seront dans des lieux différents.

Pourquoi des miroirs de 2 disques ?

Pour que chaque disque contienne les mêmes données et qu'en cas de sinistre, on puisse éjecter physiquement 1 des 2 disques dans des racks SATA amovibles. Ainsi 1 des 4 disques est suffisant pour tout reconstruire et si on a pu sauver 1 disque du BKP on a en plus les sauvegardes différentielles d'1 an. 

# Un mot sur ZFS on Linux
J'ai découvert la technologie ZFS (et surtout [ZFS on Linux](https://zfsonlinux.org/index.html)) alors que je cherchais une solution pour déterminer l'emplacement d'un disque en défaut d'un RAID1 mdadm de 2 disques sur un NAS [OpenMediaVault](https://www.openmediavault.org/) (voir le fil [ici](https://forum.openmediavault.org/index.php/Thread/26559-A-way-to-know-which-disk-replace-on-a-bay/#post200280)). Cette technologie Open Source ne cesse de m'étonner et j'en fais volontier sa promotion : système de fichier haute capacité (2^48 fichiers), fiabilité, RAID, compression, ACLs, ...

Un petit bémol cependant au moment où je rédige (13/02/20), actuellement ZFS a une incompatibilité de licence, car sa licence Open-Source CDDL (Common Development and Distribution License) est incompatible avec la licence Open-Source GNU GPL (General Public License) du noyau Linux qui empêche (pour le moment) d'être incluse dans le noyau Linux et fait craindre des procès éventuels de la part d'Oracle, propriétaire de ZFS (voir [cet article](https://linux.developpez.com/actu/290040/Linus-Torvalds-n-utilisez-pas-ZFS-jusqu-a-ce-que-j-aie-une-lettre-officielle-d-Oracle-signee-par-son-conseil-ou-par-Larry-Ellison-qui-nous-y-autorise-et-precise-que-le-produit-final-est-sous-GPL/) pour plus d'explications et cet [article \[EN\]](https://lwn.net/Articles/687550/) qui explique l'incompatibilité des 2 licences).

On a par ailleurs un avertissement lors de l'installation
<pre>
 ┌───────────────────────┤ Configuration de zfs-dkms ├───────────────────────┐  
 │                                                                           │  
 │ Les licences de ZFS et de Linux ne sont pas compatibles    
 │
 │ ZFS dispose d'une licence « Common Development and Distribution              
 │ License » (CDDL), et le noyau Linux d'une licence GNU « General Public       
 │ License » Version 2 (GPL-2). Bien qu'elles soient toutes les deux des        
 │ licences libres pour logiciels ouverts, elles restent restrictives. La       
 │ combinaison des deux pose des problèmes car cela empêche l'utilisation       
 │ de parties de code disponibles exclusivement sous une licence avec           
 │ d'autres parties de code disponibles exclusivement sous l'autre, dans le     
 │ même fichier.                                                                
 │                                                                              
 │ Vous êtes sur le point de construire ZFS en utilisant DKMS d'une manière     
 │ qui fera qu'ils ne seront pas intégrés dans un binaire unique. Veuillez      
 │ prendre en considération que la <b>DISTRIBUTION</b> de ces deux <b>BINAIRES</b> au         
 │ sein d'un <b>MEME</b> média (images de disque, machines virtuelles, etc) <b>PEUT</b>       
 │ mener à une infraction.  
</pre>
Dans cet avertissement, l'infraction citée ne serait que dans le cas d'une distribution des 2 binaires ; comme je le comprends, pour être sûr de rester dans les clous, il faut passer par l'étape de construction lors de l'installation.

Nota : souhaitons que dans les temps à venir cette situation s'apaise et que Larry Ellison autorise la GPL pour ZFS, car pour moi c'est vraiment une belle technologie ;)

# Environnement de test
Avant de le mettre réellement en place j'ai voulu tester cette solution en environnement virtuel sur un PC nommé **HOST**. J'ai donc créé 2 VMs Linux Debian Buster **NAS** et **BKP** avec chacune 1 disque dur virtuel système et chacune 2 disques dur virtuels (ici de 5 GB) pour le RAID1 ZFS (nommés **nas1**, **nas2** pour NAS et **bkp1**, **bkp2** pour BKP). 

Au final, on a donc 6 disques dur virtuels (vHD). Puisqu'on va cloner nas2 sur bkp2, il faut que la capacité de bkp1 == bkp2 >= nas1 == nas2.

## Visuel (http://asciiflow.com/)
<pre style="font-size:50px">
   +==========[NAS]==========+
   |             +--------+  |
   |             |        |  |
   | nas1\       |  nas1  |  |
   |      -(1)---->       |  |
   | nas2/ zpool |  nas2 -------+
   |       create|        |  |  |
   |             +-R1 ZFS-+  |  |
   +=========================+  |
                                |
   +==========[BKP]==========+ (2) dd
   |  +--------+ zpool       |  |
   |  |        | create      |  |
+------> bkp1 <--(3)-- bkp1  |  |
|  |  |        |             |  |
|  |  |  bkp2 <--(5)-- bkp2 <---+
|  |  |        | zpool  |    |
|  |  +-R1 ZFS-+ attach |    |
|  +====================|====+
|                       |
+---------(4)-----------+
       rsnapshot
</pre>

Attention : ici j'ai utilisé l'hyperviseur QEMU/KVM avec libvirt + virt-manager pour gérer, avec des vHDs au format QCOW2 ajoutés en tant que périphériques SATA (voir [ici](https://wiki.debian.org/KVM) pour plus d'infos) : les commandes sont donc à adapter à VOTRE hyperviseur

Astuce : ne pas hésiter à utiliser les snapshots pour tester des points critiques et éviter de tout refaire (de mémoire, il me semble que les snapshots sont possibles depuis virt-manager que si le format des disques est en QCOW2)

Rappel Linux : pour identifier un disque on peut le faire par :
- son "device" (ex : /dev/sda)
- son identifiant unique constructeur (ex : /dev/disk/by-id/ata-TOSHIBA_MK5055GSXN_Z9PLS4HBS)
- son emplacement sur le bus PCI (ex : /dev/disk/by-path/pci-0000:00:1f.2-ata-2) ; j'utilise ici ce mode car il permet de repérer physiquement quel rack contient le disque défaillant (remontée par mail)
- son emplacement sur un autre type de bus ou par un autre type d'identifiant

Nota d'écriture pour la lisibilité :
<pre>
- commande très très très très très très très très très très très très très très très très très\
   <b>très longue</b> (\ final de continuation bash, on peut copier/coller la commande complète)
- affichage très très très très très très très très très très très très très très très très ...
   <b># très long</b> (reste coupé et mis en commentaire derrière #)
</pre>

# Installer ZFS sur NAS (Debian Buster)
```sh
# Nota : sur NAS il n'est pas nécessaire d'installer OpenMediaVault,
# puisqu'on veut juste des données à sauvegarder depuis un pool ZFS. Si vous voulez l'installer
# pour faire une maquette plus complète, c'est encore mieux :)

# Sur NAS ajouter les disques nas1 et nas2

# Depuis HOST déterminer quels sont les fichiers de stockage du NAS
root@HOST:~# virsh domblklist NAS | awk '! /^$/ && NR > 2 { print ( NR - 2) ":" $2 }'
1:/mnt/3tb/buster-nas.qcow2
2:-
3:/data/vhd/zfs_nas1.qcow2
4:/data/vhd/zfs_nas2.qcow2

# Démarrer NAS

# Déterminer quels sont les emplacements des "devices" du NAS
# TODO : nota : comme indiqué précédemment les devices ont été ajoutés sur le bus SATA (en IDE ça ne marche pas) + idem plus bas
root@NAS:~# find /dev/disk/by-path -type l -not -iname "*part*"\
 -exec bash -c "l=\$( readlink -f {} ) ; echo {} \"->\" \$l" \; | sort | awk '{ print NR ":" $0 }'
1:/dev/disk/by-path/pci-0000:00:08.0-ata-1 -> /dev/sda
2:/dev/disk/by-path/pci-0000:00:08.0-ata-2 -> /dev/sr0
3:/dev/disk/by-path/pci-0000:00:08.0-ata-3 -> /dev/sdb
4:/dev/disk/by-path/pci-0000:00:08.0-ata-4 -> /dev/sdc
# Attention : il n y a pas forcément de correspondance entre le numéro ata et le n° de périphérique

# Correspondances résumées : 
# n: device sur NAS -> emplacement sur NAS -> fichier source de stockage de HOST
# 1: /dev/sr0 -> /dev/disk/by-path/pci-0000:00:1f.2-ata-1 -> rien (cdrom vide)
# 2: /dev/sda -> /dev/disk/by-path/pci-0000:00:1f.2-ata-2 -> /data/vhd/NAS.qcow2
# 3: /dev/sdb -> /dev/disk/by-path/pci-0000:00:1f.2-ata-3 -> /data/vhd/zfs_nas1.qcow2
# 4: /dev/sdc -> /dev/disk/by-path/pci-0000:00:1f.2-ata-4 -> /data/vhd/zfs_nas2.qcow2

# Depuis le NAS installer ZFS + rsync (obligatoire pour être accédé par rsnapshot)
# Guide installation ZFS On Linux : https://github.com/zfsonlinux/zfs/wiki/Debian
root@NAS:~# cat /etc/apt/sources.list.d/buster-backports.list
deb http://deb.debian.org/debian buster-backports main contrib
deb-src http://deb.debian.org/debian buster-backports main contrib
root@NAS:~# cat /etc/apt/preferences.d/90_zfs
Package: libnvpair1linux libuutil1linux libzfs2linux libzpool2linux spl-dkms zfs-dkms\
 zfs-test zfsutils-linux zfsutils-linux-dev zfs-zed
Pin: release n=buster-backports
Pin-Priority: 990
root@NAS:~# apt update
root@NAS:~# apt install -y dpkg-dev linux-headers-$(uname -r) linux-image-amd64
# attention la phase suivante prend du temps car il y a compilation ; de plus le message
# d'avertissement sur l'incompatibilité de licence Open Source (voir plus haut) 
# doit être pris en compte
root@NAS:~# apt-get install -y zfs-dkms zfsutils-linux
root@NAS:~# apt install -y rsync
```

# Créer pool NAS
```sh
# créer le pool miroir sur les 2 disques vides (nas1 = sdb et nas2 = sdc)
# nota : ashift=12 permet d optimiser le stockage 
# (https://www.reddit.com/r/zfs/comments/ax7u1l/usage_of_ashift_on_wd_red_pool/)
root@NAS:~# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk
├─sda1   8:1    0   16G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0    4G  0 part [SWAP]
sdb      8:16   0    5G  0 disk
sdc      8:32   0    5G  0 disk
sr0     11:0    1 1024M  0 rom  
root@NAS:~# zpool create -o ashift=12 pool_nas mirror\
 /dev/disk/by-path/pci-0000:00:1f.2-ata-3 /dev/disk/by-path/pci-0000:00:1f.2-ata-4
The ZFS modules are not loaded.
Try running '/sbin/modprobe zfs' as root to load them.
root@NAS:~# /sbin/modprobe zfs
root@NAS:~# zpool create -o ashift=12 pool_nas mirror\
 /dev/disk/by-path/pci-0000:00:1f.2-ata-3 /dev/disk/by-path/pci-0000:00:1f.2-ata-4
# nota : au cas où s affiche un message tel que "invalid vdev specification use '-f' to override
# the following errors: xxx is part of potentially active pool 'yyy'", 
# ce qui indique le disque n est pas "clean" (données présentes), 
# si on est sur de ce qu on fait (ATTENTION perte de données) on peut utliser zpool create -f ...

# Par défaut le point de montage est à la racine / (voir BONUS pour modifier le point de montage)
root@NAS:~# zfs mount
pool_nas                        /pool_nas

# On peut ajouter les options importantes du stockage ZFS (voir BONUS)

root@NAS:~# zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool_nas  4,50G   110K  4,50G        -         -     0%     0%  1.00x    ONLINE  -
root@NAS:~# zpool status
  pool: pool_nas
 state: ONLINE
  scan: none requested
config:

	NAME                        STATE     READ WRITE CKSUM
	pool_nas                    ONLINE       0     0     0
	  mirror-0                  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-3  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-4  ONLINE       0     0     0

errors: No known data errors
root@NAS:~# lsblk -f
NAME FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
...
sdb                                                                   
├─sdb1
│                                                                     
└─sdb9
                                                                      
sdc                                                                   
├─sdc1
│                                                                     
└─sdc9
# => le pool_nas formaté en ZFS a bien été créé, il est opérationnel (ONLINE)
# MAIS lsblk ne montre pas encore que les disques font parti d un pool ZFS ;
# je n ai pas trouvé d autre solution pour le moment que de rebooter
root@NAS:~# reboot

# Après reboot

root@NAS:~# lsblk -f | grep "zfs_member"
├─sdb1 zfs_member pool_nas 5980358820798495578                                 
├─sdc1 zfs_member pool_nas 5980358820798495578
# => OK lsblk voit le pool_nas 

root@NAS:~# echo ok > /pool_nas/test
root@NAS:~# ls -l /pool_nas/test
-rw-r--r-- 1 root root 3 févr.  6 15:42 /pool_nas/test
# => on peut écrire dessus
```

# Test en dégradé + reconstruction
```sh
# On éteint, on enlève le disque nas2 puis on redémarre
root@NAS:~# zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool_nas  4,50G   134K  4,50G        -         -     0%     0%  1.00x  DEGRADED  -
root@NAS:~# zpool status
  pool: pool_nas
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-4J
  scan: none requested
config:

	NAME                        STATE     READ WRITE CKSUM
	pool_nas                    DEGRADED     0     0     0
	  mirror-0                  DEGRADED     0     0     0
	    pci-0000:00:1f.2-ata-3  ONLINE       0     0     0
	    15556620838095572052    UNAVAIL      0     0     0  was ...
# was /dev/disk/by-path/pci-0000:00:1f.2-ata-4-part1

errors: No known data errors
root@NAS:~# cat /pool_nas/test
ok
# => Le fichier est toujours disponible même si on fonctionne en mode dégradé (DEGRADED)
# et le disque à l'emplacement ata-4 est absent

# On éteint, on remet le disque nas2 puis on redémarre

root@NAS:~# zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool_nas  4,50G   166K  4,50G        -         -     0%     0%  1.00x    ONLINE  -
root@NAS:~# zpool status
  pool: pool_nas
 state: ONLINE
  scan: resilvered 18K in 0 days 00:00:00 with 0 errors on Fri Feb  7 15:52:31 2020
config:

	NAME                        STATE     READ WRITE CKSUM
	pool_nas                    ONLINE       0     0     0
	  mirror-0                  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-3  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-4  ONLINE       0     0     0

errors: No known data errors
root@NAS:~# cat /pool_nas/test
ok
# => tout est revenu à la normale après une opération de reconstruction (resilvering)
```

# Clone nas2 vers bkp2
```sh
# On éteint NAS, on ajoute le disque bkp2 puis on redémarre
root@HOST:~# virsh domblklist NAS | awk '! /^$/ && NR > 2 { print ( NR - 2) ":" $2 }'
...
5:/data/vhd/zfs_bkp2.qcow2

root@NAS:~# find /dev/disk/by-path -type l -not -iname "*part*"\
 -exec bash -c "l=\$( readlink -f {} ) ; echo {} \"->\" \$l" \; | sort | awk '{ print NR ":" $0 }'
...
5:/dev/disk/by-path/pci-0000:00:1f.2-ata-5 -> /dev/sdd
# Attention : il n y a pas forcément de correspondance entre le numéro ata et le n° de périphérique
# => nouveau périphérique SATA en 5 : /dev/sdd

# Correspondances résumées : 
# n: device sur NAS -> emplacement sur NAS -> fichier source de stockage de HOST
# ...
# 5:/dev/sdd -> /dev/disk/by-path/pci-0000:00:1f.2-ata-5 -> /data/vhd/zfs_bkp2.qcow2

root@NAS:~# lsblk -f
NAME   FSTYPE     LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda                                                                            
├─sda1 ext4                266ae74a-594a-41be-bb36-19011c48b626    4,7G    35% /
├─sda2                                                                         
└─sda5 swap                4baad2bb-0be1-4055-932a-2d22e8e076b7                [SWAP]
sdb                                                                            
├─sdb1 zfs_member pool_nas 5980358820798495578                                 
└─sdb9                                                                         
sdc                                                                            
├─sdc1 zfs_member pool_nas 5980358820798495578                                 
└─sdc9                                                                         
sdd                                                                            
sr0 
# => le nouveau disque n est pas formaté

# On clone le disque sdc du pool_nas (nas2) vers ce nouveau disque (bkp2)
# nota : suivant la taille des disques et les performances du système,
# cette étape peut être longue...
root@NAS:~# dd if=/dev/sdc of=/dev/sdd status=progress
5363024384 octets (5,4 GB, 5,0 GiB) copiés, 741 s, 7,2 MB/s 
10485760+0 enregistrements lus
10485760+0 enregistrements écrits
5368709120 octets (5,4 GB, 5,0 GiB) copiés, 750,516 s, 7,2 MB/s
root@NAS:~# lsblk -f
NAME   FSTYPE     LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINT
...
sdc                                                                            
├─sdc1 zfs_member pool_nas 5980358820798495578                                 
└─sdc9                                                                         
sdd                                                                            
├─sdd1 zfs_member pool_nas 5980358820798495578                                 
└─sdd9                                                                         
sr0
root@NAS:~# blkid | grep pool_nas
/dev/sdb1: LABEL="pool_nas" UUID="5980358820798495578" UUID_SUB="11821135121542305994" 
 TYPE="zfs_member" PARTLABEL="zfs-3afb623cd3a14d93" PARTUUID="9b7fc0fe-0fb2-4249-b4c2-73d4a4fd80ee"
/dev/sdc1: LABEL="pool_nas" UUID="5980358820798495578" UUID_SUB="15556620838095572052" 
 TYPE="zfs_member" PARTLABEL="zfs-129eba65b8877744" PARTUUID="722dc293-49e1-6f4b-9811-0b1bbd80ffb6"
/dev/sdd1: LABEL="pool_nas" UUID="5980358820798495578" UUID_SUB="15556620838095572052" 
 TYPE="zfs_member" PARTLABEL="zfs-129eba65b8877744" PARTUUID="722dc293-49e1-6f4b-9811-0b1bbd80ffb6"
# => on voit que sdd est une copie exacte de sdc du pool_nas

# éteindre NAS
root@NAS:~# poweroff

# enlever le disque bkp2 de NAS
```

# ZFS sur BKP + pool RAID1 de 1 disque unique (Debian Buster)
```sh
# depuis BKP ajouter les disques bkp1 et bkp2

# depuis HOST déterminer quels sont les fichiers de stockage du NAS
root@HOST:~$ virsh domblklist BKP | awk '! /^$/ && NR > 2 { print ( NR - 2) ":" $2 }'
1:-
2:/data/vhd/BKP.qcow2
3:/data/vhd/zfs_bkp1.qcow2
4:/data/vhd/zfs_bkp2.qcow2

# Démarrer BKP

# Installer ZFS sur BKP (voir Installer ZFS sur NAS)

root@BKP:~$ lsblk -f
NAME   FSTYPE     LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda                                                                            
├─sda1 ext4                266ae74a-594a-41be-bb36-19011c48b626    4,7G    35% /
├─sda2                                                                         
└─sda5 swap                4baad2bb-0be1-4055-932a-2d22e8e076b7                [SWAP]
sdb                                                                            
sdc                                                                            
├─sdc1 zfs_member pool_nas 5980358820798495578                                 
└─sdc9                                                                         
sr0
# => on voit bkp2 sur sdc (clone de nas2 du pool_nas) et bkp1 sur sdb non formaté
root@BKP:~$ find /dev/disk/by-path -type l -not -iname "*part*"\
 -exec bash -c "l=\$( readlink -f {} ) ; echo {} \"->\" \$l" \; | sort | awk '{ print NR ":" $0 }'
1:/dev/disk/by-path/pci-0000:00:1f.2-ata-1 -> /dev/sr0
2:/dev/disk/by-path/pci-0000:00:1f.2-ata-2 -> /dev/sda
3:/dev/disk/by-path/pci-0000:00:1f.2-ata-3 -> /dev/sdb
4:/dev/disk/by-path/pci-0000:00:1f.2-ata-4 -> /dev/sdc
# Attention : il n y a pas obligatoirement de correspondance entre le numéro ata
# et le n° de périphérique

# Correspondances résumées : 
# device sur NAS -> emplacement sur NAS -> fichier source de stockage de HOST
# /dev/sr0 -> /dev/disk/by-path/pci-0000:00:1f.2-ata-1 -> rien (cdrom vide)
# /dev/sda -> /dev/disk/by-path/pci-0000:00:1f.2-ata-2 -> /data/vhd/BKP.qcow2
# /dev/sdb -> /dev/disk/by-path/pci-0000:00:1f.2-ata-3 -> /data/vhd/zfs_bkp1.qcow2
# /dev/sdc -> /dev/disk/by-path/pci-0000:00:1f.2-ata-4 -> /data/vhd/zfs_bkp2.qcow2

# On créée un pool miroir de 1 disque (l autre sera ajouté après) sur le disque non formaté sdb
# https://unix.stackexchange.com/questions/525950#comment973250_525959
# nota : voir plus haut pour ashift=12 et pour message d erreur "invalid vdev specification ..."
root@BKP:~# zpool create -o ashift=12 pool_bkp /dev/disk/by-path/pci-0000:00:1f.2-ata-3
The ZFS modules are not loaded.
Try running '/sbin/modprobe zfs' as root to load them.
root@BKP:~# /sbin/modprobe zfs
root@BKP:~# zpool create -o ashift=12 pool_bkp /dev/disk/by-path/pci-0000:00:1f.2-ata-3

# Par défaut le point de montage est à la racine / (voir BONUS pour modifier le point de montage)
root@BKP:~# zfs mount
pool_bkp                        /pool_bkp

root@BKP:~# zpool status pool_bkp
  pool: pool_bkp
 state: ONLINE
  scan: none requested
config:

	NAME                      STATE     READ WRITE CKSUM
	pool_bkp                  ONLINE       0     0     0
	  pci-0000:00:1f.2-ata-3  ONLINE       0     0     0

errors: No known data errors
# => remarque : ici le pool N est PAS détecté comme étant un miroir,
# il contient un unique disque et de plus il est marqué ONLINE (et non DEGRADED)

# On peut ajouter les options importantes du stockage ZFS (voir BONUS)
```

# Importer le pool NAS en lecture seule
```sh
# On importe le pool_nas en RO (non permanent) et on tente de lire et écrire
# nota1 : si l'import montre un message tel que "cannot import 'xxxx': pool was previously in use
#  from another system ... The pool can be imported, use 'zpool import -f' to import the pool.",
#  on peut forcer avec zpool import -f ...
# nota2 : on peut modifier la RACINE du montage zpool import -R /chemin/montage ...
#  (par défaut celui du NAS : /pool_nas) donc si je monte en /chemin/montage
#  le pool_nas sera accessible par /chemin/montage/pool_nas
root@BKP:~# zpool import -o readonly=on pool_nas
root@BKP:~# zfs mount | grep pool_nas
pool_nas                        /pool_nas
root@BKP:~# zpool list pool_nas
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool_nas  4,50G  96,5K  4,50G        -         -     0%     0%  1.00x  DEGRADED  -
root@BKP:~# zpool status pool_nas
  pool: pool_nas
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-4J
  scan: none requested
config:

	NAME                        STATE     READ WRITE CKSUM
	pool_nas                    DEGRADED     0     0     0
	  mirror-0                  DEGRADED     0     0     0
	    9707397900334357862     FAULTED      0     0     0  was ...
	    pci-0000:00:1f.2-ata-4  ONLINE       0     0     0
# was /dev/disk/by-path/pci-0000:00:1f.2-ata-3-part1

errors: No known data errors
# => on voit que le pool_nas fonctionne en mode dégradé et que l'autre disque est manquant
root@BKP:~# cat /pool_nas/test
ok
root@BKP:~# echo ko > /pool_nas/test
-bash: /pool_nas/test: Système de fichiers accessible en lecture seulement
# => on peut bien lire les données du pool_nas mais PAS écrire
```

# Mise en place de RSNAPSHOT
```sh
# En Résumé
root@BKP:~# zfs mount
pool_bkp                        /pool_bkp
pool_nas                        /pool_nas
root@BKP:~# zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool_bkp  4,50G   408K  4,50G        -         -     0%     0%  1.00x    ONLINE  -
pool_nas  4,50G  96,5K  4,50G        -         -     0%     0%  1.00x  DEGRADED  -
# => à ce stade les 2 pools sont activées et montés :
# - pool_bkp en lecture/écriture est opérationnel avec 1 seul disque pour l'instant
# - pool_nas en lecture seule fonctionne en dégradé car il manque un disque

# Rendre BKP accessible par SSH depuis lui même sans mot de passe
# https://unix.stackexchange.com/questions/337465/username-and-password-in-command-line-with-sshfs
# nota : j'utilise le nom de serveur BKP.local (par "avahi" non détaillé ici)
#        mais on peut aussi utiliser l'adresse IP de BKP
root@BKP:~# REMOTE_SYSTEM="root@BKP.local"
root@BKP:~# [ ! -f "$HOME/.ssh/id_rsa.pub" ]\
 && ( cat /dev/zero | ssh-keygen -t rsa -q -N "" > /dev/null )
root@BKP:~# ssh $REMOTE_SYSTEM "mkdir -p .ssh" ; 
root@BKP:~# cat ~/.ssh/id_rsa.pub | ssh $REMOTE_SYSTEM "cat >> ~/.ssh/authorized_keys"
# Saisir 2 fois le mot de passe et on peut tester la connexion : 
root@BKP:~# ssh $REMOTE_SYSTEM
# => on doit pouvoir se connecter sans mot de passe

# Configurer rsnapshot pour sauvegarder depuis le pool_nas local contenant bkp2 (clone de nas2)
# Nota : ici les commentaires et lignes vides sont occultées
# Attention à ne pas oublier :
# 1 - de désactiver les backup localhost
# 2 - de désactiver --relative de la commande rsync
# 3 - séparer par des tabulations
root@BKP:~# cat /etc/rsnapshot.conf | grep -Ev "^[[:space:]]*(#.*|)$"
config_version	1.2
snapshot_root	/pool_bkp/
cmd_cp		/bin/cp
cmd_rm		/bin/rm
cmd_rsync	/usr/bin/rsync
cmd_ssh	/usr/bin/ssh
cmd_logger	/usr/bin/logger
retain	hourly	6
verbose		2
loglevel	3
logfile		/pool_bkp/rsnapshot.log
lockfile	/pool_bkp/rsnapshot.pid
rsync_long_args	--delete --numeric-ids --delete-excluded
backup	root@BKP.local:/pool_nas/	nas/
root@BKP:~# rsnapshot configtest
Syntax OK
```

# Premières sauvegardes sur BKP
```sh
# rsnapshot nécessite rsync sur les cibles sauvegardées
root@BKP:~# apt-get install -y rsnapshot

# simulation (test)
root@BKP:~# rsnapshot -t hourly
echo 1226 > /pool_bkp/rsnapshot.pid 
mkdir -m 0755 -p /pool_bkp/hourly.0/ 
/usr/bin/rsync -a --delete --numeric-ids --delete-excluded \
    --rsh=/usr/bin/ssh root@BKP.local:/pool_nas/ \
    /pool_bkp/hourly.0/nas/ 
touch /pool_bkp/hourly.0/ 
=> OK

# 1ère sauvegarde "heure" (attention : les sauvegardes sont soumises à rotation
#  => dès qu'il y a 6 sauvegardes (configuré dans /etc/rsnapshot.conf)
#  la plus ancienne "hourly" est supprimée
root@BKP:~# rsnapshot hourly
root@BKP:~# find /pool_bkp/ -type f | sort
/pool_bkp/hourly.0/nas/test
/pool_bkp/rsnapshot.log

# Modifier la 1ère sauvegarde pour l'identifier
root@BKP:~# echo debut > /pool_bkp/hourly.0/nas/test
root@BKP:~# cat /pool_bkp/hourly.0/nas/test
debut

# 2ème sauvegarde "heure"
root@BKP:~# rsnapshot hourly
root@BKP:~# find /pool_bkp/ -type f | sort
/pool_bkp/hourly.0/nas/test
/pool_bkp/hourly.1/nas/test
/pool_bkp/rsnapshot.log
# Vérification
root@BKP:~# cat /pool_bkp/hourly.0/nas/test
ok
root@BKP:~# cat /pool_bkp/hourly.1/nas/test
debut
# => la plus ancienne sauvegarde est bien celle du début
```

# Attacher bkp2 (clone de nas2) à pool_bkp
```sh
# On abandonne le pool_nas sur le disque bkp2
root@BKP:~# zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool_bkp  4,50G   568K  4,50G        -         -     0%     0%  1.00x    ONLINE  -
pool_nas  4,50G   152K  4,50G        -         -     0%     0%  1.00x    ONLINE  -
root@BKP:~# zpool export pool_nas
root@BKP:~# zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool_bkp  4,50G   568K  4,50G        -         -     0%     0%  1.00x    ONLINE  -
root@BKP:~# zpool status pool_bkp
  pool: pool_bkp
 state: ONLINE
  scan: none requested
config:

	NAME                      STATE     READ WRITE CKSUM
	pool_bkp                  ONLINE       0     0     0
	  pci-0000:00:1f.2-ata-3  ONLINE       0     0     0

errors: No known data errors
# => rappel : le pool_bkp N est PAS vu comme un miroir et il est néanmoins opérationnel (ONLINE)
root@BKP:~$ find /dev/disk/by-path -type l -not -iname "*part*"\
 -exec bash -c "l=\$( readlink -f {} ) ; echo {} \"->\" \$l" \; | sort | awk '{ print NR ":" $0 }'
1:/dev/disk/by-path/pci-0000:00:1f.2-ata-1 -> /dev/sr0
2:/dev/disk/by-path/pci-0000:00:1f.2-ata-2 -> /dev/sda
3:/dev/disk/by-path/pci-0000:00:1f.2-ata-3 -> /dev/sdb
4:/dev/disk/by-path/pci-0000:00:1f.2-ata-4 -> /dev/sdc
# Attention : il n y a pas obligatoirement de correspondance entre le numéro ata
# et le n° de périphérique

root@BKP:~# zpool attach pool_bkp\
 /dev/disk/by-path/pci-0000:00:1f.2-ata-3 /dev/disk/by-path/pci-0000:00:1f.2-ata-4
invalid vdev specification
use '-f' to override the following errors:
/dev/disk/by-path/pci-0000:00:1f.2-ata-4-part1 is part of exported pool 'pool_nas'
# => indique que le disque n'est pas "clean" (données présentes),
# si on est sur de ce qu'on fait (ATTENTION perte de données) on peut utliser zpool attach -f ...
root@BKP:~# zpool attach -f pool_bkp\
 /dev/disk/by-path/pci-0000:00:1f.2-ata-3 /dev/disk/by-path/pci-0000:00:1f.2-ata-4
root@BKP:~# zpool status pool_bkp
  pool: pool_bkp
 state: ONLINE
  scan: resilvered 1,34M in 0 days 00:00:01 with 0 errors on Mon Feb 10 16:55:42 2020
config:

	NAME                        STATE     READ WRITE CKSUM
	pool_bkp                    ONLINE       0     0     0
	  mirror-0                  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-3  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-4  ONLINE       0     0     0

errors: No known data errors
# => bkp1 et bkp2 sont attachés à un raid miroir sur pool_bkp et il est opérationnel (ONLINE) ;
#  de plus ata-4 a été reconstruit (resilvered)
root@BKP:~# cat /pool_bkp/hourly.0/nas/test
ok
root@BKP:~# cat /pool_bkp/hourly.1/nas/test
debut
# => tout va bien on a intégré ata-4 (bkp2 ancien clone de nas2) dans un miroir à 2 disques
#  et les données n'ont pas été altérées
```

# Finaliser sauvegarde depuis NAS
```sh
# Démarrer NAS

# Rendre NAS accessible par SSH depuis BKP sans mot de passe puis tester
# (voir Mise en place de RSNAPSHOT)

# Modifier le fichier de test
root@NAS:~# echo depuis_nas > /pool_nas/test

# Configurer BKP pour sauvegarder NAS
root@BKP:~# nano /etc/rsnapshot.conf
# remplacer la ligne :
# backup root@BKP.local:/pool_nas/        nas/
# par :
backup	root@NAS.local:/pool_nas/	nas/
root@BKP:~# rsnapshot configtest
Syntax OK

# 3ème sauvegarde "heure"
root@BKP:~# rsnapshot hourly
root@BKP:~# find /pool_bkp/ -type f -name test -print0 | sort -z\ | xargs -0 -I{}\
 bash -c "echo {} ; cat {}"
/pool_bkp/hourly.0/nas/test
depuis_nas
/pool_bkp/hourly.1/nas/test
ok
/pool_bkp/hourly.2/nas/test
debut
# => la plus ancienne sauvegarde est bien celle du début
# => la plus récente est bien celle depuis NAS
```

Voilou ;)

# BONUS
## Changer l'identification d'un disque dans un pool (Path <-> ID <-> device ...)
```sh
# A mon premier essai, j'ai créé le RAID en utilisant les IDs uniques au lieu des emplacements
#  physique (path). Comment corriger ça ?
# J'ai créé comme ceci :
#  zpool create pool_nas mirror\
#   /dev/disk/by-id/ata-QEMU_HARDDISK_QM00005 /dev/disk/by-id/ata-QEMU_HARDDISK_QM00007
root@BKP:~# zpool status pool_bkp
  pool: pool_bkp
 state: ONLINE
  scan: resilvered 1,34M in 0 days 00:00:01 with 0 errors on Mon Feb 10 16:55:42 2020
config:

	NAME                           STATE     READ WRITE CKSUM
	pool_bkp                       ONLINE       0     0     0
	  mirror-0                     ONLINE       0     0     0
	    ata-QEMU_HARDDISK_QM00005  ONLINE       0     0     0
	    ata-QEMU_HARDDISK_QM00007  ONLINE       0     0     0

errors: No known data errors
root@BKP:~# zpool export -f pool_bkp
root@BKP:~$ find /dev/disk/by-path -type l -not -iname "*part*"\
 -exec bash -c "l=\$( readlink -f {} ) ; echo {} \"->\" \$l" \; | sort | awk '{ print NR ":" $0 }'
...
3:/dev/disk/by-path/pci-0000:00:1f.2-ata-3 -> /dev/sdb
4:/dev/disk/by-path/pci-0000:00:1f.2-ata-4 -> /dev/sdc
root@BKP:~# find /dev/disk/by-id -type l -not -iname "*part*"\
 -exec bash -c "l=\$( readlink -f {} ) ; echo {} \"->\" \$l" \; | sort | awk '{ print NR ":" $0 }'
...
3:/dev/disk/by-id/ata-QEMU_HARDDISK_QM00005 -> /dev/sdb
4:/dev/disk/by-id/ata-QEMU_HARDDISK_QM00007 -> /dev/sdc
root@BKP:~# zpool import\
 -d /dev/disk/by-path/pci-0000\:00\:1f.2-ata-3-part1\
 -d /dev/disk/by-path/pci-0000\:00\:1f.2-ata-4-part1 pool_bkp
root@BKP:~# zpool status pool_bkp
  pool: pool_bkp
 state: ONLINE
  scan: resilvered 1,34M in 0 days 00:00:01 with 0 errors on Mon Feb 10 16:55:42 2020
config:

	NAME                        STATE     READ WRITE CKSUM
	pool_bkp                    ONLINE       0     0     0
	  mirror-0                  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-3  ONLINE       0     0     0
	    pci-0000:00:1f.2-ata-4  ONLINE       0     0     0

errors: No known data errors
# => après une reconstruction (resilvering), on se réfère de manière permanente au disque sdc
#  par son emplacement (path)
```

## Ajouter les options ZFS importantes
```sh
# On peut ajouter des options importantes du stockage ZFS (par défaut elle ne sont pas activées)
# compression=lz4 : https://www.servethehome.com/the-case-for-using-zfs-compression/
# xattr=sa acltype=posixacl : https://blog.alt255.com/post/posix-acls/
# Nota : ces options m ont été indispensables pour la création du NAS OpenMediaVault ;
#  la compression m a permis de stocker 1,5TB sur seulement 1TB, c est colossal)
root@srv:~# zfs get all pool_bkp | egrep "compression|xattr|acltype"
pool_bkp  compression           off                       local
pool_bkp  xattr                 on                        local
pool_bkp  acltype               off	                  local
root@srv:~# zfs set compression=lz4 pool_bkp
root@srv:~# zfs set xattr=sa pool_bkp
root@bsrv:~# zfs set acltype=posixacl pool_bkp
root@bsrv:~# zfs get all pool_bkp | egrep "compression|xattr|acltype"
pool_bkp  compression           lz4                       local
pool_bkp  xattr                 sa                        local
pool_bkp  acltype               posixacl                  local
```

## Changer le montage du pool ZFS
```sh
# Changer le montage du pool ZFS (par défaut le pool_xxx est monté en /pool_xxx)
root@srv:~# mkdir /data
root@srv:~# zfs set mountpoint=/data/pool_nas/ pool_nas
root@srv:~# zfs mount
pool_bkp                        /pool_bkp
```
