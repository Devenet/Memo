# Raspberry Pi

Ce guide a été effectué et mis à jour pour une installation de Rapbian sur un Rapsberry Pi, dans l'optique de l'utiliser comme serveur.

***

# Graver Raspbian sur une carte SD

Mon seul lecteur de carte étant sur un Mac pour le moment, voici les différentes étapes à effectuer pour rendre bootable sa carte SD avec une image de Raspbian.  
La démarche peut-être adapter pour n'importe quelle distribution (OpenElec, …).

On télécharge d'abord la dernière version de Raspbian, par exemple depuis le site [raspberrypi.org](http://www.raspberrypi.org/downloads/).

Ensuite, dans un terminal après avoir obtenu les super-pouvoirs (`sudo su`), on récupère le point de montage de notre carte SD :

	diskutil list
	
On peut ainsi trouver que la carte est montée sur `/dev/disk1` (dans mon cas).

Ensuite, on démonte la carte avec :

	diskutil unmountDisk /dev/disk2
	
Et on peut enfin graver l'image avec :

	dd bs=1m if=/Users/You/Downloads/raspbian.img of=/dev/disk2
	
Une fois l'opération terminée, on est prêt pour la suite.
