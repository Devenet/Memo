# Installation et configuration d’un serveur sur Debian

Ce guide a été effectué et mis à jour pour une installation sur une Dedibox. Cependant, à part la première partie à adapter, le reste est valable quelque soit le serveur.


* [Serveur](#serveur)
* [Première connexion](#première-connexion)
  * [Mots de passe](#mots-de-passe)
  * [Service SSH](#service-ssh)
  * [Terminal](#terminal)
* [Configuration et installation de paquets](#configuration-et-installation-de-paquets)
  * [Hostname](#hostname)
  * [SSH](#ssh) : [Notification de connexion](#notification-de-connexion), [Message of the Day](#message-of-the-day)
  * [Utilitaires](#utilitaires) : _[msmtp](#msmtp), [fail2ban](#fail2ban), [logwatch](#logwatch), [apticron](#apticron), [unattended-upgrades](#unattended-upgrades)_
  * [Apache2 et PHP 8](#apache-2-et-php-8) (&rarr; [guide](https://github.com/Devenet/Memo/blob/master/apache.md))
  * [Git](#git)
  * [Munin](#munin): _[Munin node](#munin-node), [Munin server](#munin-server)_
  * [Nextcloud](#nextcloud)
* [Mise en place de sauvegardes](#mise-en-place-de-sauvegardes)
  * [Sauvegardes incrémentales locales](#sauvegardes-incrémentales-locales)
  * [Sauvegardes externes](#sauvegardes-externes)
  * [Vérification/récupération d'une sauvegarde](#vérificationrécupération-dune-sauvegarde)


***

# Serveur

La première chose à faire est de choisir la distribution ; on prendra du Debian 64 bits, la version la plus récente.

Pour le partitionnement, j’ai fait le choix suivant :

* __Boot__ : 512 Mo
* __Swap__ : 4096 Mo (~ RAM)
* __/__ : 80000 Mo (~ 80 Go pour le système)
* __/data__ : ce qui reste

(Si votre serveur sera pour du web avec gestion de fichiers d’utilisateurs, il est conseillé d’avoir une partition `tmp` dédiée.)

On choisit ensuite le nom de la machine, les mots de passe (que l’on changera à notre première connexion !), et on attend que les opérations soient terminées pour la suite.

_On peut en profiter pour activer le backup FTP que l’on configurera plus tard._

# Première connexion

Une fois que le serveur est prêt, on va pouvoir s’y connecter en SSH, sur le port 22.  

## Mots de passe

On change les mots de passe !

	passwd username

## Service SSH

On se rend dans le dossier `/etc/ssh`, et on va créer le fichier `sshd_config./servername.conf` qui va permettre de configurer et sécuriser le service SSH, avec les options suivantes :

	Port 1024

	LoginGraceTime 90
	PermitRootLogin no
	StrictModes yes
	MaxAuthTries 3
	
	AllowUsers username

	X11Forwarding no
	PrintMotd no
	PrintLastLog yes

On supprime ensuite les clefs SSH par défaut :

	rm /etc/ssh/ssh_host_*

On en régénère de nouvelles :

	dpkg-reconfigure openssh-server

Pour que les paramètres soient pris en compte :

	service ssh restart

On peut se déconnecter pour se reconnecter avec les nouveaux paramètres, et on est prêt pour la suite.

## Terminal

On peut ajouter dans le fichier `~/.bashrc` de l'utilisateur les alias suivants pour les commandes usuelles :

	alias ls='ls $LS_OPTIONS -F --color=auto'
	alias ll='ls $LS_OPTIONS -AlFh --color=auto'

	alias sizeh='du -hs *'
	alias sizes='du -ms * | sort -g'

	alias version="cat /etc/issue /etc/debian_version /etc/os-release"

        alias mail='mail -a "From: Server <server@domain.tld>"'
	

Il faudra se reconnecter pour que les modifications soient prises en compte.

***

# Configuration et installation de paquets

Avant l’installation d’un paquet, on vérifie toujours que son système est à jour :

	apt update && apt upgrade

Comme on a créé la partition `/data` pour stocker nos données, c’est dedans qu’on va mettre les données voire certains fichiers de configuration partagés.  
Au niveau de l’arborescence, j’ai fait le choix suivant :

	/data
		/apache-credentials
		/backups
		/cloud
		/db
			/munin
		/git
			/some-repository
		/www
			/nextcloud
			/some-vhost
			/munin

## Hostname

On va modifier le fichier `/etc/hosts` pour y ajouter notre nom de domaine complet :

	127.0.0.1       localhost
	127.0.1.1       servername.domain.tld name

On peut vérifier que c’est correct avec `hostname` puis `hostname -f`.

_Si l’on voulait obtenir une IP statique au lieu de celle obtenue par DHPC, il faudrait modifier le fichier `/etc/network/interfaces`, voir [Attribution d’une IP fixe](https://github.com/Devenet/Memo/blob/master/raspberrypi.md#attribution-ip-fixe)._

### IP externe dynamique

Si votre serveur est hebergé par un professionnel ou que vous avez une IP fixe, il suffit de modifier vos entrées DNS pour que `server.domain.tld` pointe vers l’IP publique de votre serveur, et vous pouvez passer à la suite.  

Dans le cas où l’on ne possède qu’une adresse IP dynamique, il va falloir ruser en mettant à jour notre enregistrement DNS à chaque fois que l’IP change.  
On a différentes manières de le faire, comme utiliser le service externe DynHost d’OVH.  

On installe `ddclient` qui va permettre de mettre à jour automatiquement notre IP sur l’entrée DNS correspondante si elle a changé depuis la dernière fois :

	apt install ddclient

Au moment de l’installation, on peut déjà préconfigurer certains paramètres :

* DNS service provider : `other`
* Dynamic DNS server : `www.ovh.com`
* DNS update protocol : `dyndns2`
* Username : _votre login_
* Password : _votre mot de passe (stocké en clair)_
* Network interface : `eth0`
* DynDns fully qualified domain names : `servername.domain.tld`

On peut maintenant modifier le fichier de configuration :

	cp /etc/ddclient.conf /etc/ddclient.conf.default
	nano /etc/ddclient.conf

et on va changer et ajouter certains paramètres :

	use=web, web=checkip.dyndns.com, web-skip='IP Address'

	server=www.ovh.com
	ssl=yes

On redémarre le service avec `service ddclient restart`.

Pour vérifier que la mise à jour s’est bien effectuée, on peut visualiser le fichier `/var/cache/ddclient/ddclient.cache` et s’assurer que notre domaine pointe bien vers notre dernière IP.  
Sinon, on peut aussi lancer le service en mode debug avec `ddclient -daemon=0 -debug -verbose -noquiet`.

## SSH

Normalement on a déjà fait une première configuration pour sécuriser le service.  
Sinon on effectue les [changements](#première-connexion) !

### Notification de connexion

On peut ajouter une alerte lors de chaque connexion SSH.
Pour cela, il suffit d’ajouter le fichier `/etc/ssh/sshrc` et d’y ajouter les actions souhaitées :

	#!/bin/bash
	(
	IP=`echo $SSH_CONNECTION | awk '{print $1}'`

	REVERSE=`dig -x $IP +short`
	if [ -z ${REVERSE} ]
	  then REVERSE="unknow"
	fi

	echo "Hello,

	User “$USER” connected on `hostname -f` from $IP ($REVERSE)." | mail -a "From: Servername <servername@domain.tld>" -s "SSH connection for $USER" you@domain.tld
	) &


Le fait de déterminer le reverse de l’IP peut prendre plus de temps au moment de la connexion, c’est pour ça qu’on effectue l’opération dans un processus fils. Si c’est cependant encore trop gênant, il suffit de le désactiver.  
Le paquet `dnsutils` est nécessaire pour que la commande `dig` fonctionne.

_Pour que l’envoi d’e-mail fonctionne, on configurera `msmtp` dans [la suite](#msmtp)._

### Message of the Day

Lors de la connexion (locale ou SSH) un message de bienvenue accueille l’utilisateur lors d’une connexion en ligne de commande.

Pour personnaliser ce message, il suffit de modifier ou créer le fichier `/etc/motd`.



## Utilitaires

### msmtp

Pour envoyer des e-mails, on installe :

	apt install mailutils msmtp msmtp-mta

On peut ensuite configurer msmtp via le fichier `/etc/msmtprc` à créer. Ici on suppose que l’on a un compte Gmail dédié pour le serveur ; à adapter selon votre fournisseur.

	defaults
	logfile /var/log/msmtp.log
	aliases /etc/aliases
	auth on
	tls on
	
	account servername@gmail.com
	host smtp.gmail.com
	port 587
	tls_starttls on
	user servername@gmail.com
	password complex_password
	from servername@gmail.com
	domain servername
	
	account default : server@gmail.com

Pour vérifier que la configuration est bonne, il suffit de s’envoyer un e-mail :

	echo "Test OK" | mail -s "Hello world" you@domain.tld
	# pour avoir plus d’information en cas d’erreur :
	echo "Test ?" | msmtprc you@domain.tld

Penser à modifier le fichier `/etc/passwd` pour mettre à jour le nom des utilisateurs avec quelque chose de plus friendly :

	root:x:0:0:Servername:/root:/bin/bash

Et ajouter un alias dans `/root/.bashrc` pour forcer le nom de l’expéditeur :

	alias mail='mail -a "From: Servername <servername@domain.tld>"'


#### Réponse automatique

J’ai choisi que la boîte e-mail du serveur ne servirait qu’à envoyer des e-mails : j’ai mis en place une réponse automatique (côté fournisseur de mon adresse e-mail) :

	Hello

	This is an unmonitored mailbox.

	We are sorry but the server is unable to respond to your request.
	Anyway he can neither speak nor respond to messages.

	Please forward your message to a valid e-mail address, such as you@domain.tld.

	Have a nice day!

	--
	Servername


### fail2ban

Notre serveur étant connecté au web, on installe fail2ban qui permet de bannir une IP pour une durée en fonction de règles prédéfinies (tentatives infructeuses de connexion SSH, …) :

	 apt install fail2ban

Pour le configurer, on copie le fichier `/etc/fail2ban/jail.conf` en `/etc/fail2ban/jail.local` et c’est ce dernier qu’on va modifier :

	ignoreip = 127.0.0.1/8 ::1
	bantime  = 7200
	findtime = 3600
	maxretry = 3

	destemail = you@domain.tld
	sendername = Servername
	sender = servername@domain.tld
	
	mta = mail
	action = %(action_mwl)s

Ensuite, activer ou modifier les jails selon vos préférences. D’une manière générale n’hésitez pas à abaisser le nombre d’essais avant bannissement.  
Pour SSH, n’oubliez pas d’ajouter le nouveau port `port = ssh,XYZ`.

On relance pour prendre en compte les modifications :

	fail2ban-client reload

_Pour voir les statuts, utiliser la commande `fail2ban-client status`_.

### logwatch

Pour recevoir par e-mail un état journalier de notre serveur, on peut installer logwatch :

	apt install logwatch
	mkdir /var/cache/logwatch
	cp /usr/share/logwatch/default.conf/logwatch.conf /etc/logwatch/conf/

On peut maintenant le modifier :

	Output = mail
	MailTo = you@domain.tld
	MailFrom = Servername <servername@domain.tld>

Pour tester et recevoir le premier rapport

	logwatch --mail you@domain.tld

Pour modifier le format de la rubrique HTTP et afficher les _vhosts_ Apache, il est nécessaire de suivre ce [tutorial](http://romain.novalan.fr/wiki/LogWatch_Apache_/_HTTP_avec_Virtual_Host).

### apticron

Pour recevoir un e-mail dès que des mises à jour sont disponibles :

	apt install apticron

On configure le service grâce au fichier `/etc/apticron/apticron.conf` :

	EMAIL="you@domain.tld"
	CUSTOM_FROM="server@domain.tld"

### unattended-upgrades

Pour que les mises à jour critiques soient automatiquement faites (voir [cet article](https://wiki.debian.org/UnattendedUpgrades)), on installe :

	apt install unattended-upgrades apt-listchanges

On modifie le fichier `/etc/apt/apt.conf.d/50unattended-upgrades` de configuration (décommanter et compléter) :

	Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
	Unattended-Upgrade::Remove-Unused-Dependencies "true";
	Unattended-Upgrade::Mail "you@domain.tld";

On lance la configuration avec 

	dpkg-reconfigure -plow unattended-upgrades

Et on termine par modifier le fichier généré `/etc/apt/apt.conf.d/20auto-upgrades` en ajoutant :

	APT::Periodic::AutocleanInterval "31";

## Apache 2 et PHP 8

Reportez-vous au document [Installation et configuration d’Apache 2 et PHP 8](https://github.com/Devenet/Memo/blob/master/apache.md) pour installer et configurer votre serveur web.  

## Git

On installe juste de quoi cloner et mettre à jour un dépôt :

	apt install git-core

Sauf exception, les dépôts git seront clonés dans `/data/git`, et on fera des liens symboliques  vers les dépôts si besoin (permet, sauf exception, de rationnaliser).

	ln -s /source/folder-source /target/other-folder/folder-target

permet de faire une lien depuis la source `folder-source` réelle vers le lien virtuel `folder-target`.


Comme on a installé Apache, on va faire en sorte que le répertoire `.git` ne soit pas accessible en ajoutant dans `/etc/apache2/conf.d/security` les directives suivantes, si ce n’est pas déjà fait :

	<DirectoryMatch "/\.git">
		Deny from all
		Satisfy all
	</DirectoryMatch>


## Munin

Munin "server" récupère les infos à partir des Munin "nodes". Notre Dedibox sera donc forcément un nœud, mais sera aussi le serveur pour les autres nœuds de notre réseau.

On va donc installer simplement :

	apt install munin

Si seul le nœud nous intéresse, il suffit de faire un `apt install munin-node`.

### Munin node

#### Paramètres

Il faut regarder du côté de `/etc/munin/munin-node.conf`. On vérifie qu’on s’autorise à écouter le nœud pour récupérer les informations :

	allow ^127\.0\.0\.1$
	allow ^::1$

Pour le cas d’un nœud de notre réseau, il faudra insérer l’IP du serveur :

	allow ^172\.16\.0\.42$

_Si, malheureusement, le serveur (qui héberge Munin serveur) a une adresse IP dynamique, il faudra ruser en autorisant n’importe quelle IP avec :_

	allow ^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$

_Néanmoins, celle solution permet à n’importe qui connaissant votre IP de demander et recevoir les informations, ce qui est donc à éviter !_

#### Plugins

La documentation disponible sur le site francophone d’Ubuntu est assez complète : [doc.ubuntu-fr.org/munin](http://doc.ubuntu-fr.org/munin#munin-node_le_demon_sur_les_noeuds).

	munin-node-configure --suggest --shell
	munin-run processes --debug
	…
	service munin-node restart

### Munin server

#### Configuration

La configuration se fait dans `/etc/munin/munin.conf` :

	dbdir /data/db/munin
	htmldir /data/www/munin
	contact.you.command mail -s "[Munin] Alert on ${var:host}" you@domain.tld

	[server.domain.tld]
    	address 127.0.0.1
    	use_node_name yes
    	contacts you
	[server2.domain.tld]
    	address server2.domain.tld
    	use_node_name yes
    	contacts you
	[server3.domain.tld]
    	address server3.domain.tld
    	use_node_name yes
    	contacts you

On supprime le lien symbolique que Munin a ajouté dans `/etc/apache2/conf-enabled` et `/etc/apache2/conf-available` qui a pour conséquence que chaque vhost Apache suivi de `/munin` affiche Munin :/

	rm /etc/apache2/conf-enabled/munin.conf
	rm /etc/apache2/conf-available/munin.conf

Et on crée un vhost spécifique pour Munin !

#### Templates

Le template par défaut étant assez… moche, on va le changer. On va utiliser [MunStrap](https://github.com/Devenet/MunStrap).

On commence par récupérer le dépôt Git :

	git clone https://github.com/Devenet/MunStrap /data/git/munstrap

Puis on change les chemins dans le fichier de configuration de Munin `/etc/munin/munin.conf` :

	tmpldir /data/git/munstrap/templates
	staticdir /data/git/munstrap/static


Vous verrez le changement lors de la prochaine itération de Munin.

#### Tâche CRON

Par défaut Munin server s’exécute toutes les 5 minutes pour récupérer les données et générer le HTML et les graphes. Cela peut utiliser beaucoup de ressources (sur un Rapsberry Pi par exemple) ; on peut donc générer le HTML et les graphes moins régulièrement.

Il suffit de modifier le fichier `/usr/bin/munin-cron` en :

	MINUTE=`date +"%M"`
	if [ `expr $MINUTE % 30` -eq 0  ] ; then

	# We always launch munin-html.
	# It is a noop if html_strategy is "cgi"
	nice /usr/share/munin/munin-html $@ || exit 1

	# The result of munin-html is needed for munin-graph.
	# It is a noop if graph_strategy is "cgi"
	nice /usr/share/munin/munin-graph --cron $@ || exit 1

	fi

Ici, on ne génère le HTML et les graphes que toutes les `30` minutes.

#### Graphes récapitulatifs

Si vous monitorez plusieurs hosts, en plus des pages de comparaison, vous souhaitez peut-être avoir sur le même graphe des courbes de comparaison de vos nœuds, ou simplement avoir un graphe totalisant les valeurs d’un même type de graphe pour tous vos nœuds.

Pour cela, il est nécessaire de modifier le fichier de configuration du serveur Munin `/etc/munin/munin.conf` et d’y ajouter un nœud virtuel.

	[domain.tld;global]
        update no

        # loads
        load.graph_title Load average
        load.graph_category system
        load.graph_order myhost=myhost.domain.tld:load.load other-host=other-host.domain.tld:load.load

        # uptime
        uptime.graph_title Uptime
        uptime.graph_category system
        uptime.graph_order myhost=myhost.domain.tld:uptime.uptime other-host=other-host.domain.tld:up

        # bandwidth
        bandwidth.graph_args --base 1000 -l 0
        bandwidth.cdef 0
        bandwidth.graph_category network
        bandwidth.graph_title Bandwidth
        bandwidth.graph_vlabel bits/sec
        bandwidth.upload.label upload
        bandwidth.total.graph yes
        bandwidth.upload.sum \
            myhost.domain.tld:if_eth0.up \
            other-host.domain.tld:if_eth0.up
        bandwidth.upload.type COUNTER
        bandwidth.download.type COUNTER
        bandwidth.download.label download
        bandwidth.graph_order upload download
        bandwidth.total.graph no
        bandwidth.download.sum \
            myhost.domain.tld:if_eth0.down \
            other-host.domain.tld:if_eth0.down

    [domain.tld;]
        node_order myhost.domain.tld other-host.domain.tld global

La dernière directive permet de changer l’ordre d’affichage (par défaut alphabétique).  
En fonction des graphes globaux que vous souhaitez, n’hésitez pas à adapter la configuration.

## Nextcloud

Je ne préfère pas installer Nextcloud depuis le paquet Debian ; on va donc télécharger l’archive depuis le site.

On va installer les fichiers web dans  `/data/www/nextcloud` et les données propres dans `/data/cloud`.

	cd /data/www
	wget https://download.nextcloud.com/server/releases/nextcloud-XYZ.tar.bz2
	tar -vxjf nextcloud-XYZ.tar.bz2

On installe aussi le support de PHP GD :

	apt install php-gd

On créé ensuite un _vhost_ dans Apache et on peut accéder à l’URL souhaitée pour la configuration.  

Si vous ne disposez que de peu d’utilisateurs qui s’y connecteront (ou pour de petites machines), il suffit de choisir SQLite comme base de données. Sinon on prendra MariaDB / MySQL.

Pour activer les tâches CRON nécessaires, on lance l’éditeur CRON pour l’utilisateur `www-data` d’Apache :

	crontab -u www-data -e

Et on peut ajouter la ligne suivante :

	*/15  *  *  *  * php -f /data/www/nextcloud/cron.php

Après avoir configurer les utilisateurs et la configuration via l’application web, on va aller modifier le fichier `/data/www/nextcloud/config/config.php` :

	'asset-pipeline.enabled' => true,
	'knowledgebaseenabled' => false,
	'default_language' => 'fr',
	'enable_previews' => false,
	'loglevel' => '4',
	'mail_domain' => 'domain.tld',
	'mail_from_address' => 'cloud',
	'mail_smtpmode' => 'php',
	'maintenance' => false


Si on a besoin de lancer une commande Nextcloud depuis un autre utilisateur, on peut utiliser :

        su - www-data -s /bin/bash -c '/usr/bin/php /data/www/nextcloud/occ maintenance:repair --include-expensive'


***

# Mise en place de sauvegardes

## Sauvegardes incrémentales locales

On va sauvegarder localement à intervalles réguliers l’état de nos données à différents moments.


Ce paquet [n’étant plus distribué](https://github.com/rsnapshot/rsnapshot/issues/279#issuecomment-944907068) depuis Debian 11 (Bullseye), on ne peut plus faire `apt install rsnapshot`. Heureusement il est encore possible de l’installer manuellement :

	wget https://github.com/rsnapshot/rsnapshot/releases/download/1.4.4/rsnapshot_1.4.4-1_all.deb
	chown _apt ./rsnapshot_1.4.4-1_all.deb
	apt install ./rsnapshot_1.4.4-1_all.deb


### Configuration

On copie le fichier modèle `cp /etc/rsnapshot.conf.default /etc/rsnapshot.conf` pour ensuite modifier ce dernier avec notre configuration :

	snapshot_root	/data/backups/
	
	cmd_cp		/usr/bin/cp

	retain		a_hourly		12
	retain		b_daily			7
	retain		c_weekly		4

	# Server
	backup          /home/                          server/
	backup		/root/				server/
	backup		/etc/				server/
	backup		/usr/local/			server/
	
	backup		/data/apache-credentials/	server/
	backup		/data/cloud/			server/
	backup		/data/db/			server/
	backup		/data/git/			server/
	backup		/data/www/			server/
	
	backup		/var/www/			server/
	backup		/var/spool/cron/crontabs/	server/


* Les intervalles sont à choisir en fonction de ce que vous voulez et de la sécurité des sauvegardes vous souhaitez.
* Les répertoires à sauvegarder aussi ; dans mon cas je sauvegarde les données présentes sur `/data` et les fichiers de configuration intéressants de `/etc`.
* Il est possible de faire exécuter un script avant et après l’exécution de rsnapshot avec `cmd_preexec` ou `cmd_postexec`.
* Vérifiez bien que ce sont de vraies tabulations qui séparent les données.

On peut ensuite tester notre fichier de configuration et simuler une première itération pour voir les actions qui seraient effectuées :

	rsnapshot configtest

	rsnapshot -t a_hourly > /tmp/rsnap_test
	cat /tmp/rsnap_test | less

### Automatisation

Il suffit d’ajouter dans le fichier crontab les lignes suivantes (en fonction des intervalles de sauvegarde que vous avez choisi !)

	crontab -u root -e

	0 */2 * * *       /usr/bin/rsnapshot a_hourly
	30 23 * * *       /usr/bin/rsnapshot b_daily
	00 23 * * 7       /usr/bin/rsnapshot c_weekly


Pour effectuer une sauvergarde locale non programmée, la commande suivante suffit :

	 rsnapshot a_hourly

Ainsi, on a une sauvegarde locale de l’état de nos données à différents moments. On peut donc récupérer un document dans son état la veille, etc.  
Seulement ces backups sont stockés localement, on va donc devoir en faire une sauvegarde autre part.

## Sauvegardes externes

Pour sécuriser nos sauvergardes locales incrémentales, on va utiliser l’espace de stockage FTP fourni dans l’offre Dedibox.  
Le paquet `rsync` ne permet pas directement de sauvegarder sur un FTP, on va donc utiliser `backup-manager` :

	apt install backup-manager
	cp backup-manager.conf backup-manager.conf.default

### Configuration

On peut maintenant modifier la configuration du fichier `/etc/backup-manager.conf` pour coller avec nos souhaits :

	export BM_ARCHIVE_TTL="1"
	export BM_ARCHIVE_NICE_LEVEL="0"

	export BM_TARBALL_DIRECTORIES="/data/backups"
	

	export BM_MYSQL_SEPARATELY="false"
	export BM_MYSQL_DBEXCLUDE="information_schema performance_schema"

	export BM_UPLOAD_METHOD="ftp"
	export BM_UPLOAD_DESTINATION="/backups"
	
	export BM_UPLOAD_FTP_USER="ftp-user"
	export BM_UPLOAD_FTP_PASSWORD="ftp-password"
	export BM_UPLOAD_FTP_HOSTS="ftp-host.domain.tld"
	export BM_UPLOAD_FTP_TTL="8"

	export BM_BURNING_METHOD="none"


_Se reporter à la [documentation Dedibox](http://documentation.online.net/fr/serveur-dedie/sauvegarde/sauvegarde-dedibackup) pour les identifiants Dedibox à utiliser._


Si on veut sauvegarder sa base de données SQL (en local + export FTP) :

    # On modifie le paramètre pour ajouter l’option mysql
    export BM_ARCHIVE_METHOD="tarball mysql"
    
    # On peut aussi n’indiquer que les tables à sauvegarder plutôt que tout comme par défaut
    export BM_MYSQL_DATABASES="__ALL__"
    
    # Utilisateur créé avec GRANT SHOW DATABASES,SHOW VIEW,SELECT,LOCK TABLES ON *.* TO 'backup-manager'@'localhost' IDENTIFIED BY 'mot de passe'
    export BM_MYSQL_ADMINLOGIN="backup-manager"
    export BM_MYSQL_ADMINPASS="mot de passe"
    
    # Pour exporter le fichier au format gzip plutôt que bzip2
    export BM_MYSQL_FILETYPE="gzip"
	
    # Une seule archive SQL, sinon autant de fichiers que de bases sauvegardées
    export BM_MYSQL_SEPARATELY="false"
    
    # On retire les tables SQL qui sont des vues (erreur 1044 sinon) ou celles qu’on veut exclure
    export BM_MYSQL_DBEXCLUDE="information_schema performance_schema"



On peut ensuite lancer manuellement la copie pour s’assurer que tout se passe bien :

	backup-manager -v

Pour vérifier que notre archive a bien été déposée, on se connecte au FTP

	ftp -n ftp-host.domain.tld
	ftp> user ftp-user ftp-password
	ftp> passive
	ftp> ls
	ftp> exit

On peut maintenant automatiser ce backup.

### Automatisation

Comme pour les sauvegardes locales, on utilise les tâches CRON :

	crontab -u root -e

	30 1 * * *	/usr/sbin/backup-manager -v | mail -a "From: Servername <servername@domain.tld" -s "[Backup] Synchronization performed" you@domain.tld

Ici, les sauvergades seront envoyées sur le serveur FTP tous les jours à 1 h 30 du matin.

## Vérification/récupération d’une sauvegarde

C’est bien on a mis en place une sauvegarde locale incrémentale et une sauvergade externe de ces sauvegardes incrémentales.  
Mettons nous dans le cas où nous aurions besoin de récupérer une sauvegarde !

### Sauvegerde locale

S’il s’agit d’un ou plusieurs fichiers modifiés il y a peu de temps, on peut aller les récupérer grâce à notre sauvegarde locale dans `/data/backups`.

Il suffit juste de savoir quelle version nous intéresse (il y a 2 heures, il y 2 jours, il y a 1 semaine ?).

### Sauvegarde externe

Si notre disque a crashé, ou si le dossier des sauvegardes locales est inutilisable, on peut respirer et se tourner vers le backup sur le serveur FTP.

On s’y connecte et on liste les fichiers disponibles :

	ftp -n ftp-host.domain.tld
	ftp> user ftp-user ftp-password
	ftp> passive
	ftp> ls backups

Ensuite, il faut rapatrier l’archive qui nous intéresse localement avec

	ftp> cd backups
	ftp> get nom_du_backup.date.tar.gz
	ftp> exit

On a maintenant le fichier en local, qu’on extrait et que l’on peut parcourir pour récupérer le ou les fichiers souhaités :-)

	tar -xvzf nom_du_backup.date.tar.gz

Même si cette manipulation ne serait à faire qu’en cas de pépin, je vous conseille de la faire au moins une fois au moment de la mise en de la sauvegarde pour vérifier qu’elle fonctionne bien, et si vous pouvez de temps en temps après sa mise en place, pour vérifier que tout fonctionne bien, ou que vous n’avez pas oublié des fichiers à sauvegarder ;-)
