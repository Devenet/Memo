# Astuces Git

* [Ajouter ses clefs de connexion Git SSH](#ajout-ses-clefs-de-connexion-git-ssh)
* [Faire un remisage](#faire-un-remisage)
* [Récupérer son remisage](#récupérer-son-remisage)
* [Créer une branche vide](#créer-une-branche-vide)
* [Cloner un repository avec seulement les derniers commits](#cloner-un-repository-avec-seulement-les-derniers-commits)
* [Supprimer tous les anciens commits d'un historique Git](#supprimer-tous-les-anciens-commit-dun-historique-git)
* [Antidater son dernier commit](#antidater-son-dernier-commit)

***

## Ajouter ses clefs de connexion Git SSH

Pour ajouter ses clefs, il faut normalement faire la démarche suivante :

	ssh-agent /bin/bash
	ssh-add ~/.ssh/id_rsa

Au redémarrage de votre machine, les clefs seront bien ajoutées, mais ce n'est pas le cas si vous vous loguez de nouveau.  

Pour éviter cela, il faut les ajouter dans le fichier `#/.ssh/config` file :

	IdentityFile ~/.ssh/id_rsa

La configuration ne s'applique qu'à l'utilisateur concerné.  
Pour que tous les utilisateurs soient concernés, il suffit de mettre la configuration dans le fichier `/etc/ssh/ssh_config`.

Petit bonus : actuellement la clef ajoutée sera testée pour toute connexion, quel que soit l'hôte. Pour restreindre la clef à un hôte spécifique :

	Host domain.tld
    	HostName domain.tld
    	User git
    	IdentityFile ~/.ssh/id_rsa

_Astuces tirées de [Add private key permanently with ssh-add on Ubuntu](https://stackoverflow.com/questions/3466626/add-private-key-permanently-with-ssh-add-on-ubuntu/4246809#4246809)._

Pour ajouter une clef de déploiement à un utilisateur virtuel, voir [deployment_key.sh](https://gist.github.com/nicolabricot/2d488601712b2723544e)

***

## Faire un remisage

Pour récupérer un espace de travail propre sans perdre ses modifications et sans commiter, on peut remiser son travail actuel et le récupérer plus tard, grâce à la commande  

	git stash

## Récupérer son remisage

On parcourt la liste des remisages effectués avec

	git stash list

Pour appliquer le dernier remisage effectué (le plus récent) :

	git stash apply

Si vous souhaitez récupérer le remisage numéro `x` en particulier, il suffit de le préciser :

	git stash apply stash@{x}

On peut maintenant reprendre la suite.

***

## Créer une branche vide

Si pour une certaine raison vous souhaitez créer une branche "vide", c'est-à-dire sans historique des précédentes commits, il suffit de préciser que vous souhaiter une branche "orpheline" :

	git checkout --orphan nouvelle-branche

Il peut être utile de supprimer les fichiers présents dans le répetoire avec :

	git rm -rf .

pour ne pas être polluer des fichiers de la branche principale.

***

## Cloner un repository avec seulement les derniers commits

Lorsque l'on clone un dépôt Git, tout l'historique est récupéré. Il n'est pas toujours nécessaire de les récupérer tous, il est possible en précisant la profondeur de ne récupérer qu'un certain nombres de commit depuis la dernière version :

	git clone --depth=5 https://github.com/nicolabricot/Memo.git memo

Cela permet de ne récupérer que les 5 derniers commits par exemple.

## Supprimer tous les anciens commits d'un historique Git

Il faut commencer par récupérer le hash correspondant au dernier commit, par exemple `a089db6`.  

Ensuite, en passant par une branche orpheline on peut réécrire la branche master :

	git checkout --orphan temp a089db6
	git commit -m "Truncated history"
	git rebase --onto temp a089db6 master
	git branch -D temp 

Pour que les modifications soient prises en compte sur le serveur distant de référence, il faut forcer la mise à jour avec 

	git push --force

Ce genre d'opération est à réserver pour un dépôt privé !

_[Source](http://web.archive.org/web/20130116195128/http://bogdan.org.ua/2011/03/28/how-to-truncate-git-history-sample-script-included.html) et [inspiration](https://stackoverflow.com/questions/17673771/git-remove-earlier-commit-but-keep-recent-changes)._

***

## Antidater son dernier commit

Si pour d'obscures raisons vous avez besoin d'antidater votre dernier commit, voici la méthode.

On commence par définir la date souhaitée en respectant le format 

	Tue, 16 Dec 2014 23:27:42 +0100

On ouvre ensuite une console en se placant dans le répertoire correspondant au dépôt Git :

	GIT_AUTHOR_DATE='Tue, 16 Dec 2014 23:27:42 +0100'
	GIT_COMMITTER_DATE='Tue, 16 Dec 2014 23:27:42 +0100'
	git commit —amend --date "Tue, 16 Dec 2014 23:27:42 +0100"

Il est possible qu'il soit nécessaire, en fonction de votre environnement, de remplacer les deux premières instructions par 

	export GIT_AUTHOR_DATE='Tue, 16 Dec 2014 23:27:42 +0100'
	export GIT_COMMITTER_DATE='Tue, 16 Dec 2014 23:27:42 +0100'

Si vous utilisez Github, si la synchronization a déjà été effectuée, il faudra forcer la synchronisation (le fait de faire un `Sync` depuis l'application Github ne réécrira pas le commit) :

	git push --force

Voici le commit devrait maintenant avoir été effectué à la date souhaitée.
