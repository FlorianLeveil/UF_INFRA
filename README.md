# PROJET UF INFRA
## SOMMAIRE

1. INTRODUCTION
* Matériels utilisées
2. OS INSTALL
* Raspbian
* Configuration utilisateur 
* Faite un redirection de port sur vôtre box
* SSH Config
* Fail2ban
* Partition LVM
3. BORG
4. LDAP
5. NEXTCLOUD
***

### INTRODUCTION

***Guide d’installation pas à pas d'un gestionnaire de media (NextCloud) avec une authentification LDAP,  héberger chez sois sur une Raspberry pi 3B+ avec deux disques dur monté en Raid1. Ainsi qu'une solution de sauvegarde avec herbergement des données sur la Raspberry (Borg)***

***
Matériels utilisées:
* Rasberry 3B+.
* 2 disques dur 2TO Chacun.
***
### OS INSTALL
#### 2.1 Raspbian
1. Télécharger l'iso Raspbian Buster Lite:
https://www.raspberrypi.org/downloads/raspbian/

**Si vous avez Windows:**

2.  Télécharger Etcher:
https://www.balena.io/etcher/

1. Prener vôtre carte SD et formater la au format FAT32
2. Lancer Etcher, selectionner vôtre carte SD puis votre ISO pour installer Raspbian dessus.
3. Une fois l'installation terminer aller sur vôtre carte SD, dans la section BOOT créer un fichier `ssh` sans extension afin d'activer le ssh.

**Si vous avez Linux**

1. Utiliser la commande DD:
 ```
 sudo dd bs=4M if=path/to/input.iso of=/dev/sdX conv=fdatasync  status=progress
```
2.  Une fois l'installation terminer aller sur vôtre carte SD, dans la section BOOT créer un fichier `ssh` sans extension afin d'activer le ssh.
***
##### 2.2 Configuration Utilisateur
1. Allumer votre Raspberry puis repérer son IP pour vous y connecter en SSH.
``ssh pi@192.168.X.X``
Mdp par default: ``raspberry`` 

2. Mettez vôtre system à jour:
``sudo apt update``
``sudo apt upgrade``

3. Créer un nouvel utilisateur (pour ma part ça sera `idk`):
``sudo adduser idk``

4.  Ajouter vôtre nouvel utilisateur aux groupes suivant:
``` sudo usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users, input,netdev,gpio,i2c,spi idk ```

5. Changer d'utilisateur:
``sudo su - idk``

6. Arrêtez les process utilisé par ``pi``:
``sudo pkill -u pi``

7. Reconnecter vous en ssh avec vôtre nouvel utilisateur:
``ssh idk@192.168.X.X``

8. Supprimer ``pi`` et son home dir:
``sudo deluser -remove-home pi``

9. Modifier le fichier en remplacent ``pi`` par vôtre nouvel utilisateur:
``sudo nano /etc/sudoers.d/010_pi-nopasswd `` 

10. Télécharger ``openssh-server``:
``apt install openssh-server``

11. Vous pouvez vous déconnecter.
***
#### 2.3 Configurer la redirection de port sur vôtre box:
1. Pour cela aller sur le site officielle de vôtre fournisseur accès internet et regarder la doc concernant la redirection de port.

2. Il nous faut trois redirection de port.
* Port 80 -> Sur le port 80 de vôtre Rasberry. (Pour le HTTP)
* Port 443 -> Sur le port 443 de vôtre Rasberry. (Pour le HTTPS)
* Port 22 -> Sur le port 22 de vôtre Rasberry. (Pour le SSH)
***
#### 2.4 SSH config
1. On va activer la connexion SSH par clés.
3. Sur vôtre host générer une nouvel clé:
``ssh-keygen``

4. Envoyez cette nouvel clé sur vôtre Raspberry:
``ssh-copy-id idk@192.168.X.X``

5. Éditer le fichier de config ssh de vôtre hots:
``nano ~/.ssh/config``

* Rajouter ceci dans vôtre fichier:
```
Host Raspberry
    HostName IP_PUBLIC_DE_VOTRE_BOX
    User idk
    IdentityFile /home/idk/.ssh/NOM_DE_VOTRE_CLE
    Port 22
    IdentitiesOnly yes
```
6. Une fois cette connecter vous sur la Raspberry avec la commande suivante :
``ssh Raspberry``

7. Rajouter à la fin du fichier ``sshd_config`` les lignes suivantes:
``sudo nano /etc/ssh/sshd_config``
```
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
```
8. Recharger le service SSH:
``sudo service ssh reload``
***
#### 2.5 Fail2ban
1. Télécharger fail2ban:
``sudo apt install fail2ban``
2. Copier le fichier ``jail.conf`` en ``jail.local``:
``sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local``
3. Éditer le fichier ``jail.local`` puis rajouter les lignes suivantes:
``sudo nano /etc/fail2ban/jail.local``
```
[ssh]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
```
4. Redémarrer le service fail2ban: 
``sudo systemctl restart fail2ban.service``

***
#### 2.6 Partition LVM
1. Télécharger ``lvm2``:
``apt install lvm2``

2. Brancher vos deux disques dur.
3. Création d'un volume physique:
``sudo pvcreate /dev/sda1 /dev/sdb1``

4. Création d'un volume groupe qui s'appellera ``vg-camembert``:
`` sudo vgcreate vg-camembert /dev/sda1 /dev/sdb1``

5. Création d'un volume logique en raid1 faisant 50% de vôtre espace disponible qui servira d'espace pour NextCloud:
   ``sudo lvcreate --mirrors 1 --type raid1 -l 50%FREE --nosync -n lvm_raid1 vg-camembert``
   
 6. Création d'un deuxième volume logique en raid1 faisant 100% de l'espace disponible restant qui servira pour Borg:
   ``sudo lvcreate --mirrors 1 --type raid1 -l 100%FREE --nosync -n lvm_raid2 vg-camembert``
   
 7. Formater le premier volume logique en ext4:
   ``sudo mkfs.ext4 /dev/vg-camembert/lvm_raid1``
   
  8. Formater le deuxieme volume logique en ext4:
   ``sudo mkfs.ext4 /dev/vg-camembert/lvm_raid2``
   
  9. Création d'un dossier dans ``/media`` pour monter le premier volume:
  ``sudo mkdir /media/DATA_BACKUP``
  
  10. Création d'un second dossier dans ``/media`` pour monter le deuxième volume:
  ``sudo mkdir /media/DATA_NEXTCLOUD``
  
 11. Monter le premier volume sur le premier dossier:
``sudo mount /dev/vg-camembert/lvm_raid1 /media/DATA_BACKUP/``

12. Monter le deuxième volume sur le deuxième dossier:
``sudo mount /dev/vg-camembert/lvm_raid2 /media/DATA_NEXTCLOUD/``
***




