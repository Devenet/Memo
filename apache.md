# Installation et configuration d’Apache 2 et PHP 7

Ce guide permet d’installer et configurer un serveur Apache avec PHP pour servir plusieurs sites web.

* [Installation](#installation)
* [Configuration](#configuration)
	* [Sécurisation](#sécurisation)
	* [Format des logs](#format-des-logs)
	* [Support des fichiers RSS](#support-des-fichiers-rss)
	* [Images favicon](#images-favicon)
	* [Modules](#modules)
	* [Envoi d'e-mails](#envoi-de-mails)
* [Virual hosts](#virtual-hosts)
	* [Default vhost](#default-vhost)
	* [Local vhost](#local-vhost)
	* [Generic vhosts](#generic-vhosts)
	* [Proxy vhost](#proxy-vhost)
* [Authentification](#authentification)
* [Activation du SSL (HHTPS)](#activation-du-ssl-https)
	* [Certificat auto-signé](#certificat-auto-signé)
	* [Écoute sur le port 443](#écoute-sur-le-port-443)
	* [Configuration des vhost SSL](#configuration-des-vhost-ssl)
* [Application des paramètres](#application-des-paramètres)
* [Statistiques d’utilisation](#statistiques-dutilisation)
	* [Webalizer](#webalizer)
	* [Munin](#munin)	

***


## Installation

Il suffit d’installer les paquages suivants :


	apt install apache2 php php-common php-cli php-fpm php-json php-pdo php-sqlite3 php-zip php-gd php-mbstring php-curl php-xml memcached php-memcached php-imagick imagemagick

On peut aussi installer d’autres extensions selon les besoins, par exemple `php-pear`, `php-bcmath`.

Pour utiliser PHP avec Apache (ce qui est un peu le but), on installe :

	apt install libapache2-mod-php


## Configuration

### Sécurisation

Dans le fichier `/etc/apache2/conf-available/security.conf`, on vérifie que les directives suivantes sont bien configurées :

	ServerTokens Prod
	ServerSignature Off
	TraceEnable Off

On décommente (début du fichier normalement) et ajoute le paramétrage pour désactiver le listage des dossiers sans fichier d’index :

	<Directory />
		Options -Indexes
		AllowOverride None
		Require all denied
	</Directory>

#### Git et SVN

S’il vous arrive de cloner un dépôt et de l’utiliser, il faut veuiller à ce que les répertoires `.git` ne soient pas accessibles.  
Pour cela, on modifie le fichier précédent `security.conf`en décommantant la partie `svn` puis en ajoutant une directive pour la partie `git` :

	<DirectoryMatch "/\.svn">
	       Require all denied
	</DirectoryMatch>
	<DirectoryMatch "/\.git">
	       Require all denied
	</DirectoryMatch>

Si vous stockez des infos SSH de déploiement dans `/var/www/.ssh` par exemple, il faut aussi ajouter :

	<DirectoryMatch "/\.ssh">
	       Require all denied
	</DirectoryMatch>

### Format des logs

Comme on va utiliser plusieurs _vhosts_ pour nos sites web, et pour un meilleur affichage dans certains reports (logwatch par exemple), il faut modifier le format de log `vhost_combined` dans le fichier `/etc/apache2/apache2.conf`.

	LogFormat "%V:%p %h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" vhost_combined

### Support des fichiers RSS

Pour que les fichiers RSS soient automatiquement reconnus comme des fichiers XML, on crée le fichier `/etc/apache2/conf-available/xml-rss-support.conf` dans lequel on inscrit :

	AddType application/xml .xml .rss

On active la configuration avec :

	a2enconf xml-rss-support.conf

### Images favicon

Comme les navigateurs essaient toujours d'accéder à `/favicon.ico` et si l'on veut éviter des logs d'erreur pour ça, on peut toujours autoriser l'accès à ce fichier. On crée le fichier `/etc/apache2/conf-available/favicon.conf` qui contient :

	<Location /favicon.ico>
	   Require all granted
	</Location>

On active la configuration avec :

	a2enconf favicon.conf

### Modules

On va activer les modules suivants :

	a2enmod rewrite
	a2enmod headers
	a2enmod deflate
	a2enmod http2
	a2enmod expires

On peut aussi en profiter pour modifier le paramètre `DirectoryIndex` (ordre de préférence des fichiers) dans le fichier `/etc/apache2/mods-available/dir.conf` :

	 DirectoryIndex index.html index.htm index.php


### Envoi d’e-mails

Penser à modifier le fichier `/etc/passwd` pour mettre à jour l’utilisateur `www-data` avec quelque chose de plus friendly si Apache est amené à envoyer des e-mails :

	www-data:x:33:33:Servername:/var/www:/bin/sh

## Virtual hosts

Pour servir plusieurs sites web avec des noms de domaines différents, on va utiliser les vhosts d’Apache.

Il suffit de créer un fichier de configuration dans `/etc/apache2/sites-available` et de l’activer ou désactiver selon ses besoins.

	a2ensite nom_vhost_file.conf
	a2dissite nom_vhost_file.conf

Il faut aussi s’assurer que dans `/etc/apache2/ports.conf` on a bien la directive :

	NameVirtualHost *:80

### Default vhost

On va modifier le vhost par défaut. Comme on écoute sur le port 80 quelque soit l’IP ou DNS demandée, il faut filtrer un peu et refuser de répondre aux requêtes non attendues.

On désactive le vhost par défaut avec `a2endissite 000-default` puis on modifie le fichier `/etc/apache2/sites-available/000_default.conf`.  

	<VirtualHost _default_:80>
		ServerName default.local
		DocumentRoot /dev/null

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

	<IfModule mod_ssl.c>
	<VirtualHost _default_:443>
		ServerName default.local
		DocumentRoot /dev/null

		SSLEngine on
		SSLVerifyClient none
		SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
		SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>
	</IfModule>

_On voit qu'on a aussi ajouté le comportement par défaut pour un vhost SSL qui n'existe pas ; voir le parapgraphe sur l'activitation du SSL._

### Local vhost

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

### Generic vhosts

Une fois que les deux précédents _vhosts_ sont configurés, on peut maintenant créer les _vhosts_ qui nous intéressent sur le modèle suivant, à adapter !

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
			Include /data/apache/conf/auth_server.conf
			Require group developers
		</Directory>

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

### Proxy vhost

Apache permet de configurer un _virtual host_ pour l’utiliser comme un proxy et accéder à un élément de réseau interne qui n’est (directement) pas accessible depuis l’extérieur.  
Par exemple si l’on souhaite pouvoir accéder au site web hébergé sur `192.168.1.1` (l’administration de votre box par exemple) qui n’est pas accessible depuis Internet, on peut configuer un _vhost_ pour qu’il fasse le relais.

__Notez bien que cela peut faciliter le piratage de votre réseau, à faire en connaissance de cause !__

On commence par installer et activer les modules nécessaires :

	a2enmod proxy
	a2enmod proxy_http

Puis on configure le _vhost_ :

	<VirtualHost *:80>
		ServerName something.domain.tld

		ProxyRequests           Off
		ProxyPreserveHost       Off
		
		ProxyPass               /       http://192.168.1.1/
		ProxyPassReverse        /       http://192.168.1.1/
		<Proxy *>
				Order Deny,Allow
				Allow from all

				Include /data/apache/conf/auth_server.conf
				Require group administrators
		</Proxy>

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

Il suffit d’activer le _vhost_ et de relancer Apache.  

Voir la section [Authentification](#authentification) pour que tout le monde ne puisse pas y accéder.

## Authentification

Il faut activer le module suivant pour supporter l’authentification par groupes d'utilisateurs :

	a2enmod authz_groupfile

Dans la configuration du _vhost_ précédent, on a inclus le fichier de configuration `auth_server.conf` (permet de l’inclure depuis plusieurs _vhosts_) :

		AuthType Basic
		AuthName "Beta area"
		AuthUserFile /data/apache/credentials/users
		AuthGroupFile /data/apache/credentials/groups

Pour créer un utilisateur et l’ajouter à un fichier non encore existant :

	htpasswd -c users username

Si le fichier existe déjà et que l’on veut ajouter un autre utilisateur :

	htpasswd users otherusername

Pour le fichier des groupes utilisateur, il faut juste respecter la syntaxe suivante :

	nom_groupe: user1, user2, user3
	autre_groupe: user1, user3


## Activation du SSL (HTTPS)

Pour servir tout ou partie des sites web en HTTPS, il faut quelques étapes supplémentaires.  
On suppose que l’on a déjà les certificats qui seront utilisés, hormis le certificat auto-signé.

### Certificat auto-signé

Ce certificat est utilisé pour le _vhost_ SSL par « défaut ».  
Il suffit de taper la commande suivante et d’y entrer les informations nécessaires (dans mon cas, comme je souhaite donner le moins d’informations, je ne renseigne que les champs obligatoires : le pays (EU), la province (Europe) et l’organisation (Domain)) :

	openssl req -x509  -newkey rsa:2048 -nodes -days 365 -keyout /etc/ssl/private/ssl-cert-snakeoil.key -out /etc/ssl/certs/ssl-cert-snakeoil.pem

_On peut aussi changer la durée de la validité du certificat (`-days 365`) voire diminuer ou augmenter la taille du chiffrement de la clef (`rsa:4096`)._

Pour éviter que n’importe qui puisse aller voir nos clefs privées, on fait un `chmod 400 ssl-cert-*.key` sur chaque de nos clefs privées !  
On peut aussi faire un `chown :ssl-cert ssl-cert-*.key` pour leur attribuer comme groupe « ssl-cert ».


### Écoute sur le port 443

On commence par activer le module SSL :

	a2enmod ssl

Ensuite on s’assure qu’on écoute bien sur le port 443 pour les _vhosts_ à vérifier dans le fichier `/etc/apache2/ports.conf`.

### Configuration des vhost SSL

Pour faciliter la navigation, on ajoute dans le _vhost_ SSL générique un _vhost_ sur le port 443.

	# Pour forcer la redirection de http vers https
	<VirtualHost *:80>
		ServerName domain.tld
		ServerAlias www.domain.tld
	
		RewriteEngine on
		RewriteRule ^/(.*) https://www.domain.tld/$1 [NC,R,L]
	</VirtualHost>

	<IfModule mod_ssl.c>
	<VirtualHost *:443>
		ServerName domain.tld
		ServerAlias www.domain.tld
		
		DocumentRoot /data/www/domain.tld
		<Directory /data/www/domain.tld>
			ErrorDocument 404 /404.html
		</Directory>
		<Directory /data/www/domain.tld/beta>
			Options +Indexes
			Include /data/apache/conf/auth_server.conf
			Require group developers
		</Directory>

		SSLEngine on
		SSLProtocol all -SSLv2
		SSLCipherSuite ALL:!ADH:!EXPORT:!SSLv2:RC4+RSA:+HIGH:+MEDIUM

		SSLCertificateFile    /etc/letsencrypt/live/domain.tld/fullchain.pem
		SSLCertificateKeyFile /etc/letsencrypt/live/domain.tld/privkey.pem

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>
	</IfModule>

Pour générer vos certificats SSL gratuitement, vous pouvez regarder du côté de Let’s Encrypt, une autorité de certification.


## Application des paramètres

On redémarre ensuite le serveur pour que les modifications soient prises en compte :

	service apache2 restart

On peut aussi relancer le server avec `service apache2 reload` pour certains changements qui ne nécessitent pas le rédémarrage du service.

## Statistiques d’utilisation

### Webalizer

Si vous souhaitez connaître quelques statistiques sur l’utilisation d’Apache, en se basant sur les logs, vous pouvez suivre [Installation et configuration de Webalizer pour Apache](webalizer.md).

### Munin

Si vous avez activé les plugins Apache pour Munin, mais qu’il n’y a aucune donnée… deux pistes :

1. Assurez-vous que le module `status` d’Apache est bien activé : `a2enmod status`.
2. Ça peut venir d’une dépendance manquante utilisée par Munin.  
   Lançon en mode test la commande utilisée par Munin : `munin-run apache_accesses --debug`. Si l’erreur affichée est « LWP::UserAgent not found », il faut alors installer la bibliothèque Perl manquante avec `apt install libparse-http-useragent-perl`.
