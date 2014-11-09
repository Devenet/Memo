# Installation et configuration d'un serveur sur Debian

Ce guide a été effectué et mis à jour dans pour une installation sur une Dedibox. Cependant, à part la première partie à adapter, le reste est valable quelque soit le serveur ;-)

***

# Serveur

La première chose à faire est de choisir la distribution ; on prendra du Debian 7 64 bits.

Pour le partitionnement, j'ai fait le choix suivant :

* __Boot__ : 200 Mo
* __Swap__ : 2048 Mo (~ RAM)
* __/__ : 51200 Mo (~ 50 Go pour le système)
* __/data__ : ce qui reste

On choisit ensuite le nom de la machine, les mots de passe (que l'on changera à notre première connexion !), et on attend que les opérations soient terminées pour la suite.

_On peut en profiter pour activer le backup FTP que l'on configurera plus tard._

# Première connexion

Une fois que le serveur est prêt, on va pouvoir s'y connecter en SSH, sur le port 22.  

On change les mots de passe !

	passwd username

Puis on continue par configurer et sécuriser le service SSH (une fois en root) grâce au fichier `/etc/ssh/sshd_config` :

	Port XYZ
	
	LoginGraceTime 120
	PermitRootLogin no
	StrictModes yes
	AllowUsers username
	
	X11Forwarding no

On supprime ensuite les clefs SSH par défault :

	rm /etc/ssh/ssh_host_*

On en régénère de nouvelles :

	dpkg-reconfigure openssh-server

Pour que les paramètres soient pris en compte :

	service ssh restart

On peut se déconnecter pour se reconnecter avec les nouveaux paramètres, et on est prêt pour la suite.

# Configuration et installation de paquets

Avant l'installation d'un paquet, on vérifie toujours que son système est à jour :

	apt-get update && apt-get upgrade

Comme on a créé la partition `/data` pour stocker nos données, c'est dedans qu'on mettre les données voire certains fichiers de configuration partagés.  
Au niveau de l'arborescence, j'ai fait le choix suivant :

	/data
		/apache
			/conf
			/alias
			/error
		/cloud
		/git
			/some-repository
		/script
			/shell
			/cron
		/www
			/owncloud
			/some-vhost
			/munin

## Hostname

On va modifier le fichier `/etc/hosts` pour y ajouter notre nom de domaine complet :

	127.0.0.1       localhost
	127.0.0.1       XYZ.dedibox.fr
	127.0.1.1       name.domain.tld name

On peut vérifier que c'est correct avec `hostname` puis `hostname -f`.

_Si l'on voulait obtenir une IP statique au lieu de celle obtenue par DHPC, il faudrait modifier le fichier `/etc/network/interfaces`._

## SSH

Normalement on a déjà fait une première configuration pour sécuriser le service.

On peut aussi ajouter une alerte lors de chaque connexion SSH.
Pour cela, il suffit d'ajouter le fichier `/etc/ssh/sshrc` et d'y ajouter les actions souhaitées :

	IP=`echo $SSH_CONNECTION | awk '{print $1}'`
	REVERSE=`dig -x $IP +short`
	if [ -z ${REVERSE} ]
		then REVERSE="unknow"
	fi
	
	echo "$USER connected on `hostname -f` from $IP ($REVERSE)" | mail -s "SSH connection" you@domain.tld

Le fait de déterminer le reverse de l'IP pour prendre plus de temps au moment de la connexion. Si c'est trop gênant, il suffit de le désactiver.

_Pour que l'envoie d'email fonctionne, on configurera `ssmtp` dans la suite._

## Utilitaires

### ssmtp

Pour l'envoyer d'emails, on installe : 

	apt-get install mailutils ssmtp

On peut ensuite configurer ssmtp via le fichier `/etc/ssmtp/ssmtp.conf`. Ici on suppose que l'on a un compte Gmail dédié pour le serveur ; à adapter selon votre fournisseur.

	root=you@domain.tld
	mailhub=smtp.gmail.com:587
	rewriteDomain=domain.tld
	hostname=name.domain.tld
	FromLineOverride=YES
	
	UseSTARTTLS=YES
	AuthUser=server@domain.tld
	AuthPass=password

Pour vérifier que la configuration est bonne, il suffit de s'envoyer un email :

	echo "Test OK" | mail -s "Hello world!" you@domain.tld

Penser à modifier le fichier `/etc/passwd` pour mettre à jour le nom des utilisateurs avec quelque chose de plus friendly :

	root:x:0:0:Server Name:/root:/bin/bash

#### Réponse automatique

J'ai choisi que la boîte email du serveur ne servirait qu'à envoyer des emails, j'ai donc mis en place une réponse automatique :

	Hello

	This is an unmonitored mailbox.
	
	We are sorry but the server is too busy at the moment to respond to your request.
	Anyway he can neither speak (yes it is a he!) nor respond to messages.
	
	Please forward your message to a valid e-mail address, such as you@domain.tld.
	
	Have a nice day!
	
	--
	The busy mailserver


### fail2ban

Notre serveur étant connecté au web, on installe fail2ban qui permet de bannir une IP pour une durée en fonction de règles prédéfinies (tentatives infructeuses de connexion SSH, ...) :

	 apt-get install fail2ban

Pour le configurer, on copie le fichier `/etc/fail2ban/jail.conf` en `/etc/fail2ban/jail.local` et c'est ce dernier qu'on va modifier :

	ignoreip = 127.0.0.1/8
	bantime  = 3600
	findtime = 1800
	maxretry = 3
	
	destemail = you@domain.tld
	action = %(action_mwl)s

Ensuite, activer ou modifier les jails selon vos préférences. D'une manière générale n'hésitez pas à abaisser le nombre d'essais avant un bannissement.  
Pour SSH, n'oubliez pas d'ajouter le nouveau port `port = ssh,XYZ`. 

On relance pour prendre en compte les modifications :

	fail2ban-client reload

_Pour voir les statuts, utiliser la commande `fail2ban-client status`_.

### logwatch

Pour recevoir par email un état journalier de notre serveur, on installe logwatch :

	apt-get install logwatch
	mkdir /var/cache/logwatch
	cp /usr/share/logwatch/default.conf/logwatch.conf /etc/logwatch/conf/

On peut maintenant le modifier :

	Output = mail
	MailTo = you@domain.tld
	MailFrom = Name <server@domain.tld>

Pour tester et recevoir le premier rapport

	logwatch --mail you@domain.tld

Pour modifier le format de la rubrique HTTP et afficher les vhosts Apache, il est nécessaire de suivre ce [tutorial](http://romain.novalan.fr/wiki/LogWatch_Apache_/_HTTP_avec_Virtual_Host) de Romain.

### apticron

Pour recevoir un email dès que des mises à jour sont disponibles sur votre server :

	apt-get install apticron

On configure le deamon grâce au fichier `/etc/apticron/apticron.conf` :

	EMAIL="you@domain.tld"
	CUSTOM_FROM="server@domain.tld"


That's it.

## Apache 2 et PHP 5

Il suffit d'installer les paquages suivants :


	apt-get install apache2 php5 php5-sqlite memcached php5-memchached

On va commencer par sécuriser un peu ça, puis activer les modules couramment utilisés.

### Installation et configuration

#### Sécurisation

Dans le fichier `/etc/apache2/conf.d/security`, on vérifie les directives suivantes :

	ServerTokens Prod
	ServerSignature Off
	TraceEnable Off

On décommente (début du fichier normalement) et ajoute le paramétrage pour désactiver le listage des dossiers sans fichier d'index :

	<Directory />
		Options -Indexes
		AllowOverride None
	#	Order Deny,Allow
	#	Deny from all
	</Directory>

#### Format des logs

Comme on va utiliser plusieurs vhosts pour nos sites web, et pour un meilleur affichage dans certains reports (logwatch par exemple), il faut modifier le format de log `vhost_combined` dans le fichier `/etc/apache2/apache2.conf`.

	LogFormat "%V:%p %h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" vhost_combined

#### Support des fichiers RSS

Pour que les fichiers RSS soient automatiquement reconnus comme des fichiers XML, on crée le fichier `/etc/apache2/conf.d/xml-rss-support` dans lequel on inscrit :

	AddType application/xml .xml .rss

#### Modules

On va activer les modules suivants :

	a2enmod rewrite
	a2enmod headers
	a2enmod deflate
	a2enmod mem_cache


On peut aussi en profiter pour modifier le paramètre `DirectoryIndex` (ordre de préférence des fichiers) dans le fichier `/etc/apache2/mods-available/dir.conf` :

	 DirectoryIndex index.php index.html index.htm

### Virtual hosts

Pour servir plusieurs sites web avec des noms de domaines différents, on va utiliser les vhosts d'Apache.

Il suffit de créer un fichier de configuration dans `/etc/apache2/sites-available` et de l'activer ou désactiver selon ses besoins.

	a2ensite nom_vhost_file
	a2dissite nom_vhost_file

Il faut aussi s'assurer que dans `/etc/apache2/ports.conf` on a bien la directive :

	NameVirtualHost *:80

#### Default vhost

On va modifier le vhost par défaut. Comme on écoute sur le port 80 quelque soit l'IP ou DNS demandée, il faut filtrer un peu et refuser de répondre aux requêtes non attendues.

On désactive le vhost par défaut avec `a2endissite default` puis on modifier le fichier `/etc/apache2/sites-available/default`.  
_Personnellement, pour mieux me répérer, je préfixe tous mes fichiers vhost de 2 chiffres, je modifierai donc `00_default` ou `xx_vhost`._

	<VirtualHost _default_:80>
		ServerName default.local
		Redirect / https://google.com

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

	<VirtualHost _default_:443>
		ServerName default.local
		DocumentRoot /var/www/

		SSLEngine on
		SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
		SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

La particularité de ce vhost est qu'il redirige toute requête entrante vers un autre serveur (Google en l'occurence). Pour quelqu'un qui essayerait d'obtenir une réponse HTTP en tapant sur une IP ou un NDD que l'on a pas configuré, il est redirigé plutôt que d'obtenir des infos sur le serveur alors qu'il ne devrait pas se trouver là.

_On voit qu'on a aussi ajouté le comportement par défaut pour un vhost SSL qui n'existe pas ; voir le parapgraphe sur l'activitation du SSL._

#### Local vhost

On va ensuite créer le vhost « localhost » pour qu'Apache accepte les requêtes internes :

	<VirtualHost *:80>
		ServerName localhost
		ServerAlias 127.0.0.1

		ServerAdmin you@domain.tld
		DocumentRoot /var/www

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

#### Generic vhosts

Une fois que les deux précédents vhosts sont configurés, on peut maintenant créer les vhosts qui nous intéressent sur le modèle suivant, à adapter !

	<VirtualHost *:80>
		ServerName domain.tld
		ServerAlias www.domain.tld

		ServerAdmin you@domain.tld
		DocumentRoot /data/www/domain.tld
		
		<Directory /data/www/domain.tld>
			ErrorDocument 404 /404.html
		</Directory>
		<Directory /data/www/domain.tld/beta>
			Options +Indexes
			Include /data/apache/auth/auth_server.conf
			Require group developers
		</Directory>

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

#### Authentification

Dans la configuration du vhost précédent, on a inclus le fichier de configuration `auth_server.conf` (permet de l'inclure depuis plusieurs vhosts) :

		AuthType Basic
		AuthName "Beta area"
		AuthUserFile /data/apache/auth/users
		AuthGroupFile /data/apache/auth/groups

Pour créer un utilisateur et l'ajouter à un fichier non encore existant :

	htpasswd -c users username

Si le fichier existe déjà et que l'on veut ajouter un autre utilisateur :

	htpasswd users otherusername

Pour le fichier des groupes utilisateur, il faut juste respecter la syntaxe suivante :

	nom_groupe: user1, user2, user3
	autre_groupe: user1, user3


### Activation du SSL (HTTPS)

Pour servir tout ou partie des sites web en HTTPS, il faut quelques étapes supplémentaires.  
On suppose que l'on a déjà les certificats qui seront utilisés, hormis le certificat auto-signé.

#### Certificat auto-signé

Ce certificat est utilisé pour le vhost SSL par « défaut ».  
Il suffit de taper la commande suivante et d'y entrer les informations nécessaires (dans mon cas, comme je souhaite donner le moins d'informations, je ne renseigne que les champs obligatoires) :

	openssl req -x509  -newkey rsa:2048 -nodes -days 365 -keyout /etc/ssl/private/ssl-cert-snakeoil.key -out /etc/ssl/certs/ssl-cert-snakeoil.pem

_On peut aussi changer la durée de la validité du certificat (`-days 365`) voire diminuer ou augmenter la taille du chiffrement de la clef (`rsa:2048`)._

Pour éviter que n'importe qui puisse aller voir nos clefs privées, on fait un `chmod 400 ssl-cert-*.key` sur chaque de nos clefs privées !  
On peut aussi faire un `chown :ssl-cert ssl-cert-*.key` pour leur attribuer comme groupe « ssl-cert ».

Pour les certificats signés, je les stocke dans `/etc/ssl/public` et `/etc/ssl/private`.


#### Écoute sur le port 443

On commence par activer le module SSL :

	a2enmode ssl

Ensuite on s'assure qu'on écoute sur le port 443 pour les vhosts à vérifier dans le fichier `/etc/apache2/ports.conf` :

	NameVirtualHost *:443
    Listen 443

#### Configuration des vhost SSL

Pour faciliter la navigation, on ajoute dans le vhost SSL générique un vhost sur le port 80 pour une redirection en HTTPS, à adapter selon ;-)

	<VirtualHost *:80>
		ServerName sub.domain.tld

		RewriteEngine on
		RewriteRule ^/(.*) https://sub.domain.tld/$1 [NC,R,L]
	</VirtualHost>

	<VirtualHost *:443>
		ServerName sub.domain.tld
		
		ServerAdmin you@domain.tld
		DocumentRoot /data/www/sub.domain.tld

		<Directory /data/www/sub.domain.tld>
			ErrorDocument 403 /403.html
			ErrorDocument 404 /404.html
		</Directory>

		SSLEngine on
		SSLProtocol all -SSLv2
		SSLCipherSuite ALL:!ADH:!EXPORT:!SSLv2:RC4+RSA:+HIGH:+MEDIUM

		SSLCertificateFile    /etc/ssl/public/ssl-vhost.crt
		SSLCertificateKeyFile /etc/ssl/private/ssl-vhost.key
		SSLCertificateChainFile /etc/ssl/public/sub.class1.server.ca.pem
		SSLCACertificateFile /etc/ssl/public/ca.pem

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

Vous pouvez regarder du côté de StartSSL, une autorité de certification gratuite.


### Application des paramètres

On redémarre ensuite le serveur pour que les modifications soient prises en compte :

	service apache2 restart


On peut aussi relancer le server avec `service apache2 reload` pour certains changements qui ne nécessitent pas le rédémarrage du service.

## Git

On installe juste de quoi cloner et mettre à jour un dépôt :

	apt-get install git-core

Sauf exception, les dépôts git seront clonés dans `/data/git`, et on fera des liens symboliques vers les dépôts si besoin (permet, sauf exception, de rationnaliser).

	ln -s /data/git/moodpicker /data/www/vhost/moods

