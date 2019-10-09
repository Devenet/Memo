# Installation et configuration d'un Raspberry Pi

Ce guide a été effectué et mis à jour pour une installation de Rapbian sur un Rapsberry Pi, dans l'optique de l'utiliser comme serveur.

* [Graver Raspbian sur une carte SD](#graver-raspbian-sur-une-carte-sd)
* [Premier démarrage](#premier-démarrage)
* [Configuration initiale](#configuration-initiale)
	* [Nom du serveur](#nom-du-serveur)
	* [Attribution IP fixe](#attribution-ip-fixe)
	* [Comptes utilisateurs](#comptes-utilisateurs)
	* [Package sudo](#package-sudo)
	* [Sécurisation de SSH](#sécurisation-de-ssh)
	* [Prise en compte](#prise-en-compte)
* [Configuration pour serveur](#configuration-pour-serveur)
	* [Suppression des paquets](#suppression-des-paquets)
	* [Mises à jour](#mises-à-jour)
	* [Paquet rpi-update](#paquet-rpi-update)
	* [Wake On Lan](#wake-on-lan)
* [Installation de services](#installation-de-services)
	* [Plugins Munin](#plugins-munin)


***

# Graver Raspbian sur une carte SD

Mon seul lecteur de carte étant sur un Mac pour le moment, voici les différentes étapes à effectuer pour rendre bootable sa carte SD avec une image de Raspbian.  
La démarche peut-être adaptée pour n'importe quelle distribution (OpenElec, …).

On télécharge d'abord la dernière version de Raspbian, par exemple depuis le site [raspberrypi.org](http://www.raspberrypi.org/downloads/).

Ensuite, dans un terminal après avoir obtenu les super-pouvoirs (`sudo su`), on récupère le point de montage de notre carte SD :

	diskutil list

On peut ainsi trouver que la carte est montée sur `/dev/disk1` (dans mon cas).

Ensuite, on démonte la carte avec :

	diskutil unmountDisk /dev/disk2

Et on peut enfin graver l'image avec :

	dd bs=1m if=/Users/You/Downloads/raspbian.img of=/dev/disk2

Une fois l'opération terminée, on est prêt pour la suite.

***

# Premier démarrage

Penser à brancher un clavier USB ainsi qu'avoir un écran (ou une TV) avec un port HDMI.

Pour se connecter, le nom d’utilisateur est `pi` et le mot de passe par défaut est `raspberry`.  
Attention, le clavier étant par défaut en Qwerty, il faut taper la lettre `q` pour faire un `a`.

On lance l'écran de configuration avec :

	sudo su
	raspi-config

On va d'abord étendre la partition sur toute la carte SD :

	Advenced Options
	A1 Expand Filesystem > Enter

Le mot de passe pour l'utilisateur `pi` est par défaut `raspberry`. On pourrait le modifier sur cet écran, mais pour des raisons de sécurité, comme on va le supprimer par la suite, on peut se passer de cette étape.

On vérifie par contre que le RBPi boote bien “from Scratch” :

	Boot options
	B1 Desktop / CLI > Enter
	B1 Console (Text console, requiring user to login) > Enter

Pour l'internationalisation, on va laisser l'anglais comme langue du système, mais on va changer la configuration du clavier et la zone de temps (après chaque modification, on revient sur l'écran d'accueil) :

	Localisation Options > Enter
	I2 Change Timezone > Enter
	Europe -> Paris

	Localisation Options > Enter
	I3 Change Keyboard Layout > Enter
	Genereic 104-key PC > Enter
	Other -> French -> French -> Default -> No compose key -> Ok

Si vous souhaitez overclocker votre RBPi, il suffit de suivre les insctructions de `7 Overclock`.

Grâce aux options avancées, on va pouvoir entre autre régler la taille de la mémoire dédiée à la vidéo (pour un serveur, au plus petit), activer le SSH, …

	Network Options > Enter
	N1 Hostname > Enter

	Advanced Options > Enter
	A3 Memory Split > Enter
	16 > Enter

	Interfacing Options > Enter
	P2 SSH > Enter
	Enable > Yes

En cliquant sur `Finish` on arrive sur le bash, et l'on peut redémarrer le RBPi avec `sudo reboot`.

Si on a besoin de revenir sur cet écran, la commande `raspi-config` est là.

# Configuration initiale

La première chose à faire est de connaître l'adresse IP que le serveur DHCP (box ou autre) lui a attribué.  
On peut ensuite s'y connecter en SSH :

	ssh pi@192.168.1.10 -p 22

On passe en `root` avec la commande `sudo su`.

## Nom du serveur

On met à jour complètement son hostname (pour avoir un FQDN) ; d'abord dans le fichier `/etc/hosts` à la ligne suivante :

	127.0.1.1 server.domain.tld server

et dans le fichier `/etc/hostname` :

	server

_Il se peut que le hostname soit plus ou moins correct si vous avez lors du premier démarrage utilisé la fonction pour renommer le RBPi._  
Pour vérifier que c'est correct, on utilise les commandes `hostname` puis `hostname -f`.

## Attribution IP fixe

Ensuite on lui attribue une adresse IP fixe, qui va nous faciliter la tâche. On modifie le fichier `/etc/network/interfaces` pour remplacer la ligne `eth0 inet dhcp` par

	iface eth0 inet static
		address 192.168.1.100
		netmask 255.255.255.0
		#network 192.168.1.0
		#broadcast 192.168.1.255
		gateway 192.168.1.1

en adaptant selon la configuration de votre réseau.

On peut en profiter pour ouvrir les (présents et futurs) ports utilisés dans la page de configuration de votre box ou de votre routeur.  
Vous pouvez aussi mettre votre RBPi en DMZ si vous souhaitez que toutes les requêtes provenant de l'extérieur lui soient soumises.

## Comptes utilisateurs

On commence par modifier (ou attribuer) un mot de passe pour le compte `root` :

	passwd root

On créé son compte utilisateur

	adduser you

On se déloggue (`exit`) et on se reconnecte avec le compte utilisateur nouvellement créé.

Sur Raspbian, l'utilisateur `pi` a été créé. Comme on a créé notre propre compte et qu'on est loggué avec, on va pouvoir le supprimer (après être passé en `root`) :

	deluser --remove-home pi

## Package sudo

Le package `sudo` est installé par défaut. Pour des raisons de sécurité, on le supprimer avec

	apt-get --purge remove sudo

Pour effectuer des commandes nécessitant les super-pouvoirs on passera en `root` avec la commande `su -` depuis son compte local.

## Sécurisation de SSH

Dans la configuration par défaut, SSH est configuré sur le port 22, n'importe quel utilisateur (dont `root` !) peut s'y connecter.

Se reporter à la partie SSH du paragraphe « [Première connexion](https://github.com/Devenet/Memo/blob/master/debian.md#première-connexion) » du document sur [Debian](https://github.com/Devenet/Memo/blob/master/debian.md) pour changer ses paramètres.  
Le paragraphe « [SSH](https://github.com/Devenet/Memo/blob/master/debian.md#ssh) » permet de recevoir un email à chaque connexion d'un utilisateur SSH.

On se connectera donc maintenant grâce à :

	ssh you@192.168.1.100 -p XXXXX

_Ces modifications sont vraiment importantes à faire, notamment si votre RBPi est accessible depuis les Internets._

## Prise en compte

On peut maintenant redémarrer le RBPi pour que les nouveaux paramètres soient pris en compte avec

	reboot

# Configuration pour serveur

Maintenant que notre RBPi a une IP fixe avec des comptes locaux à différents niveaux et un service SSH plus sécurisé, on va continuer notre configuration pour désinstaller les paquets initules pour notre serveur.

## Suppression des paquets

On commencer par mettre à jour le dépôt des paquets avec

	apt-get update

On supprimer les paquets inutiles (ça en fait un paquet…) :

	aptitude purge xserver-xorg xserver-xorg-core xserver-xorg-input-all xserver-xorg-input-evdev xserver-xorg-input-synaptics xserver-xorg-video-fbdev xserver-common xpdf xinit x11-common x11-utils x11-xkb-utils xarchiver screen pcmanfm penguinspuzzle lxde-common lxappearance lxde-icon-theme lxinput lxmenu-data lxpanel lxpolkit lxrandr lxsession lxsession-edit lxshortcut lxtask lxterminal leafpad dillo galculator gnome-icon-theme gnome-themes-standard gnome-themes-standard-data gpicview hicolor-icon-theme
	aptitude purge ~c

## Mises à jour

On peut maintenant lancer les mises à jour (à faire régulièrement) :

	apt-get update && apt-get upgrade -y

Si vous souhaitez mettre à jour votre distribution pour la version suppérieure, il faut en plus faire

	apt-get dist-upgrade

## Paquet rpi-update

Pour mettre à jour automatiquement le firmware du RBPi, on va installer le paquet `rpi-update` (s'il n'a pas déjà été installé) :

	apt-get install rpi-update

Ainsi, il suffira de lancer la commande `rpi-update` pour mettre à jour le firmware avec la dernière version disponible (nécessite un redémarrage pour la prise en compte).

## Wake On Lan

Pour réveiller d'autres serveurs ou PC depuis votre RBPi grâce au WoL, je vous conseille d'installer `etherwake`.

Il faut ensuite créer le fichier `etc/ethers` dans lequel vous mettez l'adresse MAC associée à un nom

	00:00:00:00:00:00	gentil_nom

Pour réveiller un ordinateur, il suffit de faire :

	etherwake gentil_nom



# Installation de services

Pour stocker les données, je vous conseille de créer le répertoire racine `data` pour les y stocker

	mkdir /data

Si vous avez une clef USB ou disque dur (à ne pas garder au format NTFS pour des raisons de performances) qui y est monté, placer y plutôt les données dessus pour éviter toute perte de données si le RBPi crachait pour une raison inconnue…

Pour la suite, il suffit de vous reporter au document [debian.md](https://github.com/Devenet/Memo/blob/master/debian.md) pour installer et configurer les différents services que vous souhaitez pour votre serveur RBPi :-)

## Plugins Munin

Une fois le nœud Munin installé, il existe des plugins propres pour les Raspberry Pi, dont un intéressant : celui de la température du CPU.  
Pour cela, on récupère le script (basé sur [pisense](https://github.com/perception101/pisense)) qu'on enregistre et qu'on rend exécutable :

	wget https://raw.githubusercontent.com/Devenet/Memo/master/ressources/munin_pisense -O /usr/share/munin/plugins/pisense_
 	chmod +x /usr/share/munin/plugins/pisense_

On peut ensuite l'ajouter dans le nœud Munin :

	ln -s /usr/share/munin/plugins/pisense_ /etc/munin/plugins/pisense_temp

en présicant dans `/etc/munin/plugin-conf.d/munin-node`

	[pisense_*]
	user_root

On n'oublie pas de redémarrer les services

	service munin-node restart
