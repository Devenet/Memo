# Installation et configuration de Webalizer pour Apache

Ce guide permet d'installer et configurer Webalizer pour visualiser graphiquement les logs et statistiques d'Apache.

* [Installation](#installation)
* [Configuration](#configuration)
	* [Prise en compte des modifications](#prise-en-compte-des-modifications)
* [Génération des rapports](#génération-des-rapports)


***

## Installation

Le package `webalizer` est disponible directement depuis Debian ; il suffit de l'installer.

  apt-get install webalizer

## Configuration

La configuration se fait via le fichier `webalizer.conf` que l'on va modifier

  nano /etc/webalizer/webalizer.conf

De nombreux paramètres sont configurables. Dans notre cas, on va seulement s'assurer que les suivants le sont bien :

  OutputDir /var/www/webalizer
  ReportTitle Webalizer stats
  HostName ServerName

  PageType        htm*
  #PageType       cgi
  #PageType       phtml
  #PageType       php3
  #PageType       pl
  PageType        php

  # Si votre site web est en HTTPS
  UseHTTPS       yes

### Prise en compte des modifications

Comme il ne s'agit pas d'un service, les modifications seront prises en compte dès que le programme sera lancé.  
Pour vérifier que nos modifications constituent toujours un fichier de paramtréage correct, on peut lancer manuellement la génération, en se rapportant au paragraphe suivant.


## Génération des rapports

Une tâche CRON a automatiquement été créée dans `/etc/cron.daily` et la génération « statique » des pages HTML à partir des logs d'Apache se fera automatiquement.

Pour lancer manuellement l'actualisation des données — notamment après un changement de configuration — il faut lancer la commande en _root_ :

  webalizer

C'est tout !
