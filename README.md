# bkp_from_nas_without_network

## Objectif
Sauvegarde différentielle fichiers entre pools ZFS RAID1 2 disques distants sans utiliser le réseau

Voir aussi : [ZFS - ajouter un pool de 1 disque pour du RAID1](https://www.commentcamarche.net/forum/affich-36406170-zfs-ajouter-un-pool-de-1-disque-pour-du-raid1?gF-4hFHlxvdlq69e5GwmSAzhIdyMIVCDBAsuKJhgBdY)

## Le contexte :
Je voulais mettre en place un serveur de sauvegarde différentielle de fichiers **BKP** sous Linux (rsnapshot) sur un RAID1 (miroir) ZFS de 2 disques qui sauvegarderait les données d'un serveur de fichier OpenMediaVault **NAS** sous Linux, lui aussi sur un RAID1 ZFS de 2 disques. Les serveurs NAS et BKP seront dans des lieux différents. 

Pourquoi des miroirs de 2 disques ? Pour que chaque disque contienne les mêmes données et qu'en cas de sinistre, on puisse éjecter physiquement 1 des 2 disques dans des racks SATA amovibles. Ainsi 1 des 4 disques est suffisant pour tout reconstruire et si on a pu sauver 1 disque du BKP on a en plus les sauvegardes différentielles d'1 an.
