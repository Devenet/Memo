# Installation et configuration d'Apache 2 et PHP 5

Ce guide permet d'installer et configurer un serveur Apache avec PHP pour servir plusieurs sites web.

* [Installation](#installation)
* [Configuration](#configuration)
	* [Sécurisation](#sécurisation)
	* [Format des logs](#format-des-logs)
	* [Support des fichiers XML](#support-des-fichiers-xml)
	* [Modules](#modules)
	* [Envoi d'e-mails](#envoi-de-mails)
* [Virual hosts](#virutal-hosts)
	* [Default vhost](#default-vhost)
	* [Local vhost](#local-vhost)
	* [Generic vhost](#generic-vhost)
	* [Proxy vhost](#proxy-vhost)
* [Authentification](#authentification)
* [Activation du SSL (HHTPS)](#activation-du-ssl-https)
	* [Certificat auto-signé](#certificat-auto-signé)
	* [Écoute sur le port 443](#ecoute-sur-le-port-443)
	* [Configuration des vhost SSL](#configuration-des-vhost-ssl)
* [Application des paramètres](#application-des-paramètres)

***


## Installation

Il suffit d'installer les paquages suivants :


	apt-get install apache2 php5 php5-sqlite memcached php5-memchached


## Configuration

### Sécurisation

Dans le fichier `/etc/apache2/conf.d/security`, on vérifie que les directives suivantes sont bien configurées :

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

#### Git et SVN

S'il vous arrice de cloner un dépôt et de l'utiliser, il faut veuiller à ce que les répertoires `.git` ne soient pas accessibles.  
Pour cela, on modifie le fichier `/etc/apache2/conf.d/security`en décommantant la partie `svn` et en ajoutant une directive pour la partie `git` :

	<DirectoryMatch "/\.svn">
	       Deny from all
	       Satisfy all
	</DirectoryMatch>	
	<DirectoryMatch "/\.git">
	       Deny from all
	       Satisfy all
	</DirectoryMatch>

### Format des logs

Comme on va utiliser plusieurs vhosts pour nos sites web, et pour un meilleur affichage dans certains reports (logwatch par exemple), il faut modifier le format de log `vhost_combined` dans le fichier `/etc/apache2/apache2.conf`.

	LogFormat "%V:%p %h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" vhost_combined

### Support des fichiers RSS

Pour que les fichiers RSS soient automatiquement reconnus comme des fichiers XML, on crée le fichier `/etc/apache2/conf.d/xml-rss-support` dans lequel on inscrit :

	AddType application/xml .xml .rss

### Modules

On va activer les modules suivants :

	a2enmod rewrite
	a2enmod headers
	a2enmod deflate
	a2enmod mem_cache


On peut aussi en profiter pour modifier le paramètre `DirectoryIndex` (ordre de préférence des fichiers) dans le fichier `/etc/apache2/mods-available/dir.conf` :

	 DirectoryIndex index.php index.html index.htm


### Envoi d'e-mails

Penser à modifier le fichier `/etc/passwd` pour mettre à jour l'utilisateur `www-data` avec quelque chose de plus friendly si Apache est amené à envoyer des e-mails :

	www-data:x:33:33:Server Name:/var/www:/bin/sh

## Virtual hosts

Pour servir plusieurs sites web avec des noms de domaines différents, on va utiliser les vhosts d'Apache.

Il suffit de créer un fichier de configuration dans `/etc/apache2/sites-available` et de l'activer ou désactiver selon ses besoins.

	a2ensite nom_vhost_file
	a2dissite nom_vhost_file

Il faut aussi s'assurer que dans `/etc/apache2/ports.conf` on a bien la directive :

	NameVirtualHost *:80

### Default vhost

On va modifier le vhost par défaut. Comme on écoute sur le port 80 quelque soit l'IP ou DNS demandée, il faut filtrer un peu et refuser de répondre aux requêtes non attendues.

On désactive le vhost par défaut avec `a2endissite default` puis on modifier le fichier `/etc/apache2/sites-available/default`.  
_Personnellement, pour mieux me répérer, je préfixe tous mes fichiers vhost de 2 chiffres, je modifierai donc `00_default` ou `xx_vhost`._

	<VirtualHost _default_:80>
		ServerName default.local
		Redirect / http://example.net

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

	<VirtualHost _default_:443>
		ServerName default.local
		Redirect / http://example.net

		SSLEngine on
		SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
		SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

La particularité de ce vhost est qu'il redirige toute requête entrante vers un autre serveur (Google en l'occurence). Pour quelqu'un qui essayerait d'obtenir une réponse HTTP en tapant sur une IP ou un NDD que l'on a pas configuré, il est redirigé plutôt que d'obtenir des infos sur le serveur alors qu'il ne devrait pas se trouver là.

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
			Include /data/apache/conf/auth_server.conf
			Require group developers
		</Directory>

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log vhost_combined
	</VirtualHost>

### Proxy vhosts

Apache permet de configurer un virtual host pour l'utiliser comme un peoxy et accéder à un élément de réseau interne qui n'est (directement) pas accessible depuis l'extérieur.  
Par exemple si l'on souhaite pouvoir accéder au site web hébergé sur `192.168.1.1` (l'administration de votre box par exemple) qui n'est pas accessible depuis Internet, on peut configuer un vhost pour qu'il fasse le relais.

__Notez bien que cela peut faciliter le piratage de votre réseau, à faire en connaissance de cause !__

On commence par installer et activer les modules nécessaires :

	a2enmod proxy
	a2enmod proxy_http

Puis on configure le vhost :

	<VirtualHost *:80>
		ServerName something.domain.tld

		ProxyPass               /       http://192.168.1.1/
		ProxyPassReverse        /       http://192.168.1.1/
		ProxyRequests           Off
		ProxyPreserveHost       Off
		SSLProxyEngine          On
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

Il suffit d'activer le vhost et de relancer Apache.  

Voir la section [Authentification](#authentification) pour que tout le monde ne puisse pas y accéder.

## Authentification

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


## Activation du SSL (HTTPS)

Pour servir tout ou partie des sites web en HTTPS, il faut quelques étapes supplémentaires.  
On suppose que l'on a déjà les certificats qui seront utilisés, hormis le certificat auto-signé.

### Certificat auto-signé

Ce certificat est utilisé pour le vhost SSL par « défaut ».  
Il suffit de taper la commande suivante et d'y entrer les informations nécessaires (dans mon cas, comme je souhaite donner le moins d'informations, je ne renseigne que les champs obligatoires) :

	openssl req -x509  -newkey rsa:2048 -nodes -days 365 -keyout /etc/ssl/private/ssl-cert-snakeoil.key -out /etc/ssl/certs/ssl-cert-snakeoil.pem

_On peut aussi changer la durée de la validité du certificat (`-days 365`) voire diminuer ou augmenter la taille du chiffrement de la clef (`rsa:2048`)._

Pour éviter que n'importe qui puisse aller voir nos clefs privées, on fait un `chmod 400 ssl-cert-*.key` sur chaque de nos clefs privées !  
On peut aussi faire un `chown :ssl-cert ssl-cert-*.key` pour leur attribuer comme groupe « ssl-cert ».

Pour les certificats signés, je les stocke dans `/etc/ssl/public` et `/etc/ssl/private`.


### Écoute sur le port 443

On commence par activer le module SSL :

	a2enmode ssl

Ensuite on s'assure qu'on écoute sur le port 443 pour les vhosts à vérifier dans le fichier `/etc/apache2/ports.conf` :

	NameVirtualHost *:443
    Listen 443

### Configuration des vhost SSL

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


## Application des paramètres

On redémarre ensuite le serveur pour que les modifications soient prises en compte :

	service apache2 restart

On peut aussi relancer le server avec `service apache2 reload` pour certains changements qui ne nécessitent pas le rédémarrage du service.