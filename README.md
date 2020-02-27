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
 * Installation Borg
 * Script Backup
 * Crontab
 * Restore Backup
4. FREEIPA
 * Installation freeipa
 * Création de Users et Group pour NextCloud
5. NEXTCLOUD
* Install Docker
* Install NextCloud
* Install Plugin LDAP sur NextCloud
***

### INTRODUCTION

***Guide d’installation pas à pas d'un gestionnaire de media (NextCloud) héberger chez sois sur une Raspberry pi 3B+ avec deux disques dur monté en Raid1.Mise en place d'une authentification via FreeIPA héberger sur un VPS OVH. Ainsi qu'une solution de sauvegarde avec herbergement des données sur la Raspberry (Borg)***

***
Matériels utilisées:
Software / Hardware:
* Rasberry 3B+. (With Raspbian latest)
* 2 disques dur 2TO Chacun.
* VPS héberger chez OVH(With Centos 7)
* Un nom de domaine avec DNS sur OVH
* Un client (With ArchLinux)
* Borg 1.1.9
* Docker version 19.03.6
* Image NextCloud Latest pour Docker
* Fail2Ban v0.10.2
* Cronie 1.5.5
* Freeipa 4.6.5
***
## OS INSTALL
### 2.1 Raspbian
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
### 2.2 Configuration Utilisateur
1. Allumer votre Raspberry puis repérer son IP pour vous y connecter en SSH.
 ```
ssh pi@192.168.X.X
 ```
Mdp par default: ``raspberry`` 

2. Mettez vôtre system à jour:
```
sudo apt update
```
```
sudo apt upgrade
```

3. Créer un nouvel utilisateur (pour ma part ça sera `idk`):
```
sudo adduser idk
```

4.  Ajouter vôtre nouvel utilisateur aux groupes suivant:
``` 
sudo usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users, input,netdev,gpio,i2c,spi idk 
```

5. Changer d'utilisateur:
```
`sudo su - idk 
```


6. Arrêtez les process utilisé par ``pi``:
```
sudo pkill -u pi
```

7. Reconnecter vous en ssh avec vôtre nouvel utilisateur:
```
ssh idk@192.168.X.X
```

8. Supprimer ``pi`` et son home dir:
```
sudo deluser -remove-home pi
```

9. Modifier le fichier en remplacent ``pi`` par vôtre nouvel utilisateur:
```
sudo nano /etc/sudoers.d/010_pi-nopasswd 
``` 

10. Télécharger ``openssh-server``:
```
apt install openssh-server
```

11. Vous pouvez vous déconnecter.
***
### 2.3 Configurer la redirection de port sur vôtre box:
1. Pour cela aller sur le site officielle de vôtre fournisseur accès internet et regarder la doc concernant la redirection de port.

2. Il nous faut trois redirection de port.
* Port 80 -> Sur le port 80 de vôtre Rasberry. (Pour le HTTP)
* Port 443 -> Sur le port 443 de vôtre Rasberry. (Pour le HTTPS)
* Port 22 -> Sur le port 22 de vôtre Rasberry. (Pour le SSH)
***
### 2.4 SSH config
1. On va activer la connexion SSH par clés.
3. Sur vôtre host générer une nouvel clé:
```
ssh-keygen
```

4. Envoyez cette nouvel clé sur vôtre Raspberry:
```
ssh-copy-id idk@192.168.X.X
```

5. Éditer le fichier de config ssh de vôtre hots:
```
nano ~/.ssh/config
```

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
```
ssh Raspberry
```

7. Rajouter à la fin du fichier ``sshd_config`` les lignes suivantes:
```
sudo nano /etc/ssh/sshd_config
```
```
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
```
8. Recharger le service SSH:
```
sudo service ssh reload
```
***
### 2.5 Fail2ban
1. Télécharger fail2ban:
```
sudo apt install fail2ban
```
2. Copier le fichier ``jail.conf`` en ``jail.local``:
```
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
3. Éditer le fichier ``jail.local`` puis rajouter les lignes suivantes:
```
sudo nano /etc/fail2ban/jail.local
```
```
[ssh]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
```
4. Redémarrer le service fail2ban: 
```
sudo systemctl restart fail2ban.service
```

***
### 2.6 Partition LVM
1. Télécharger ``lvm2``:
```
apt install lvm2
```

2. Brancher vos deux disques dur.
3. Création d'un volume physique:
```
sudo pvcreate /dev/sda1 /dev/sdb1
```

4. Création d'un volume groupe qui s'appellera ``vg-camembert``:
```
sudo vgcreate vg-camembert /dev/sda1 /dev/sdb1
```

5. Création d'un volume logique en raid1 faisant 50% de vôtre espace disponible qui servira d'espace pour NextCloud:
```
sudo lvcreate --mirrors 1 --type raid1 -l 50%FREE --nosync -n lvm_raid1 vg-camembert
```
   
 6. Création d'un deuxième volume logique en raid1 faisant 100% de l'espace disponible restant qui servira pour Borg:
 ```
 sudo lvcreate --mirrors 1 --type raid1 -l 100%FREE --nosync -n lvm_raid2 vg-camembert
 ```
   
 7. Formater le premier volume logique en ext4:
 ```
 sudo mkfs.ext4 /dev/vg-camembert/lvm_raid1
 ```
   
  8. Formater le deuxieme volume logique en ext4:
 ```
 sudo mkfs.ext4 /dev/vg-camembert/lvm_raid2
 ```
   
  9. Création d'un dossier dans ``/media`` pour monter le premier volume:
 ```
 sudo mkdir /media/DATA_BACKUP
 ```
  
  10. Création d'un second dossier dans ``/media`` pour monter le deuxième volume:
 ```
 sudo mkdir /media/DATA_NEXTCLOUD
 ```
  
 11. Monter le premier volume sur le premier dossier:
```
sudo mount /dev/vg-camembert/lvm_raid1 /media/DATA_BACKUP/
```

12. Monter le deuxième volume sur le deuxième dossier:
```
sudo mount /dev/vg-camembert/lvm_raid2 /media/DATA_NEXTCLOUD/
```
***
## BORG
### INSTALL BORG:
1. Sur la Raspberry créer un dossier `BACKUP` dans `/media/DATA_BACKUP/`
```
sudo mkdir /media/DATA_BACKUP/BACKUP
```
```
sudo chmod 777 /media/DATA_BACKUP/BACKUP
```
2. Sur la Raspberry Installer Borg:
```
sudo apt install borgbackup
```
3. Sur votre host installer Borg:
```
sudo pacman -S borgbackup
```
4. Depuis vôtre host Initialiser Borg sur vôtre Raspberry:
```
borg init --encryption=repokey ssh://idk@IP_PUBLIC_DE_VOTRE_BOX:22/media/DATA_BACKUP/BACKUP/
```
***
### SCRIPT BACKUP:
1. Faire une sauvegarde:
```
borg create ssh://IP_PUBLIC_DE_VOTRE_BOX:22/media/DATA_BACKUP/BACKUP/::idk_{now:%d.%m.%Y} ~/VOTRE_DOSSIER_A_SAUVEGARDER
```
2.  Créer un fichier dans le dossier que vous voulez sauvegarder:
```
vim ~/VOTRE_DOSSIER_A_SAUVEGARDER/script_backup.sh
```
3. Éditer vôtre fichier et ajouter ceci:
```
export BORG_PASSPHRASE='VOTRE_MDP_BORG'
borg create ssh://IP_PUBLIC_DE_VOTRE_BOX:22/media/DATA_BACKUP/BACKUP/::idk_{now:%d.%m.%Y} ~/VOTRE_DOSSIER_A_SAUVEGARDER
```
4. Pour lancer vôtre script:
```
bash ~/VOTRE_DOSSIER_A_SAUVEGARDER/script_backup.sh
```
***
### CRONTAB:
1. Installer la CronTab:
```
sudo pacman -S cronie
```
2. Activer la crontab au démarrage et la lancer:
```
sudo systemctl start cronie
sudo systemctl enable cronie
```
3. Ajouter vôtre script à la crontab:
```
crontab -e
```
4. Et rajouter ceci pour que vôtre script de backup s'éxécute tout les jours à 4H00:
```
0 4 * * * /home/idk/VOTRE_DOSSIER_A_SAUVEGARDER/script_backup.sh> /dev/null 2>&1
```
***
### RESTORE BACKUP:
1. Création d'un script récupérant la list des backups:
```
vim ~/VOTRE_DOSSIER_A_SAUVEGARDER/script_list_backup.sh
```
2. Éditer vôtre fichier et ajouter ceci:
```
export BORG_PASSPHRASE='VOTRE_MDP_BORG'
borg list ssh://idk@109.21.215.30:3333/media/DATA_BACKUP/BACKUP/
```
3. Pour lancer vôtre script:
```
bash ~/VOTRE_DOSSIER_A_SAUVEGARDER/script_list_backup.sh
```
4. Une fois les backups lister placez-vous à la racine:
```
cd /
```
5. Choisissez la backup qui vous intéresse et exécuter la commande suivante:
```
borg extract ssh://idk@109.21.215.30:3333/media/DATA_BACKUP/BACKUP/::LE_NOM_DE_VOTRE_BACKUP
```
***
## Freeipa
Liens pouvant vous être utilse:

https://www.golinuxcloud.com/configure-setup-freeipa-server-client-linux/
https://www.worteks.com/fr/2018/03/29/freeipa-part1/
https://www.howtoforge.com/tutorial/how-to-install-freeipa-server-on-centos-7/
***
### Installation Freeipa:
1. Connectez vous en SSH sur vôtre VPS
2. Set le hostname:
```
hostnamectl set-hostname ldap.VOTRE.DOMAIN
```
3. Éditer le hosts:
```
vim /etc/hosts
```
4. Rajouter ceci dans les `/etc/hosts`:
```
VOTRE_IP       ldap.VOTRE.DOMAIN
```
5. Télécharger Freeipa:
```
sudo yum install ipa-server bind-dyndb-ldap ipa-server-dns -y
```
6. Ajouter des ports au firewall:
```
firewall-cmd --permanent --add-port={80/tcp,443/tcp,389/tcp,636/tcp,88/tcp,464/tcp,53/tcp,88/udp,464/udp,53/udp,123/udp}
```
7. Recharger vôtre firewall:
```
firewall-cmd --reload
```
8. Instalation de Freeipa avec le DNS de OVH:
* Si vous avec besoin de setup vôtre propre DNS regarder les liens qui vous sont proposer au début de l'article.
```
ipa-server-install
``` 
* Cela peut prendre pas mal de temps

9. Une fois fini, copier le fichier ``/tmp/ipa.system.records.XXXXX.db`` en supprimant les ports(dans le fichier) et ajouter le dans les paramètre de vôtre DNS.
 
11. Recharger vôtre firewall:
```
firewall-cmd --reload
```
***
### Création de Users et Group pour NextCloud:
* Création d'un user:
	 * Aller sur l'onglet Identité >Utilisateurs > Utilisateurs Actif > Ajouter.
	 * Vous pouvez mettre seulement un nom et un prénom.
	 
* Création d'un Group Next_Cloud:
	*  Aller sur l'onglet Identité > Groupes > Groupes Utilisateurs > Ajouter.
	* Type de groupe, sélectionner `POSIX`.

* Mettre un User dans le groupe Next_Cloud:
	* Aller sur l'onglet Identité > Groupes > Groupes Utilisateurs.
	* Cliquer sur le User concerner.
	* Cliquer sur Groupe Utilisateur > Ajouter.
	* Cocher le groupe `Next_Cloud`, cliquer sur la flèche pour le basculer à droite puis cliquer sur` Ajouter`.
*** 
## NEXTCLOUD
### Install Docker:
1. Installer des paquets indispensable pour Docker:
```
sudo apt-get install apt-transport-https ca-certificates software-properties-common -y
```
2. Récupération de Docker:
```
curl -fsSL get.docker.com -o get-docker.sh && sh get-docker.sh
```
3. Ajout de l'utilisateur `idk` dans le groupe `docker`:
```
sudo usermod -aG docker idk
```

### Install NextCloud:
1. Installer du container NextCloud:
```
sudo docker run --restart always --name NC_IDK -v/media/DATA_NEXTCLOUD/NEXTCLOUD:/var/www/html/data -d -p 443:443 nextcloud
```
2. Taper l'ip public de vôtre box dans l'url et vous devriez arriver sur la page d'acceuil NextCloud.
3. Ajouter un utilisateur Admin.
### Install du plugin LDAP
* Pour télécharger le Plugin suivez ce tuto:
	* http://www.techspacekh.com/integrating-nextcloud-user-authentication-with-ldapactive-directory-ad/
* Pour parametrer le Plugin suivez ce tuto:
	* https://www.freeipa.org/page/Owncloud_Authentication_against_FreeIPA
	
**ATTENTION**
Par défaut vôtre login est: *"prénom nom"*



