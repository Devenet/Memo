# Installation et configuration de MariaDB

## Installation

On vérifie que ton système est à jour (`apt update && apt upgrade`), puis on installe les paquets MariaDB :

    apt install mariadb-server


À la fin de l'installation, on peut vérifier la version ou que la base de données est bien active :

    mariadb --version
    systemctl status mariadb

## Sécurisation

On sécurise ensuite immédiatement notre serveur SQL avec la commande :

    mariadb-secure-installation

où l'on va successivement :

1. `Enter current password for root (enter for none)` → mot de passe du compte `root` de MariaDB.
2. `Switch to unix_socket authentication` → répondre `n` (comme on a fait l’étape 1).
3. `Change the root password?` → répondre `n` (comme on a fait l’étape 1).
4. `Remove anonymous users?` → `Y` pour désactiver les utilisateurs anonymes.
5. `Disallow root login remotely?` → `Y` pour que la connexion de l’utilisateur `root` ne soit acceptée que depuis la machine elle-même.
6. `Remove test database and access to it?` → `Y` pour supprimer la base de test présente.
7. `Reload privilege tables now?` → `Y` pour appliquer les paramètres que l’on vient de configurer.


## Ajout de bases de données

On se connecte maintenant à la base de données

    mysql
    # ou mariadb

et on liste les bases présentes (aucune à part les bases systèmes) :

    show databases;

Par défaut, connecté·e en tant que _root_ sur la machine, on est automatiquement identifié `root` via la commande `mysql`. Si on veut s’y connecter avec un utilisateur non-root de la machine, il faut alors utiliser `mysql -p`.

### Ajout d’un·e utilisateur·trice

Pour créer un·e utilisateur·trice administrateur·trices autre que `root` :

    create user 'nicolas'@'localhost' identified by '<mot de passe complexe>';
    grant all privileges on *.* to 'nicolas'@'localhost';
    flush privileges;

### Ajout d’une base de données

Pour créer une base de données :

    create database nextcloud;

Et ensuite créer un·e utilisateur·trice ayant tous les droits uniquement sur cette base de données :

    create user 'nextcloud'@'localhost' identified by '<mot de passe>';
    grant all privileges on nextcloud.* to 'nextcloud'@'localhost';
    flush privileges;

---

## Configuration de Munin

Si on a installé Munin, on peut activer les plugins associés pour surveiller l’usage de sa base de données.

- https://gallery.munin-monitoring.org/plugins/munin/mysql_/  
- https://memo-linux.com/munin-plugin-mysql-missing-dependency-cachecache/

Il faut ensuite mettre à jour le fichier de configuration de Munin dans `/etc/munin/plugin-conf.d/munin-node` avec :

    [mysql*]
    user root
    env.mysqlopts --defaults-file=/etc/mysql/debian.cnf
    env.mysqluser root
    env.mysqlpassword <mot de passe>
    env.mysqlconnection DBI:mysql:mysql;mysql_read_default_file=/etc/mysql/debian.cnf

(Lignes 2 et 5 à modifier avec `root` + ajout ligne 4.)

On s'assure que les paquets suivant sont bien installés :

    apt install libcache-cache-perl
    apt install libdbd-mysql-perl

On active les informations SQL souhaitées :

    cd plugins
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_commands'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_connections'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_files_tables'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_bpool'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_bpool_act'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_insert_buf'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_io'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_io_pend'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_log'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_rows'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_semaphores'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_tnx'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_myisam_indexes'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_network_traffic'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_qcache'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_qcache_mem'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_select_types'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_slow'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_sorts'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_table_locks'
    ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_tmp_tables'


On recharche notre nœud Munin avec `systemctl restart munin-node`.
