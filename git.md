# Astuces Git

* [Antidater son dernier commit](#antidater-son-dernier-commit)

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