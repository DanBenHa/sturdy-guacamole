Some title
==========

.. tip::

    Do this in a copy of your nextcloud-related files.
    I used btrfs snapshots

.. code-block:: shell

    bla

Database (MariaDB)
------------------

The database user `ncadmin` created by NextcloudPi can only be accessed from `localhost`, i.e. the machine that the database is running on.
That makes sense for a bare-bones installation where the nextcloud app runs on the same computer, but not for a multi-container setup.
In that case other containers look like other hosts from one container's perspective.
Therefore, access rights must be modified to allow access from the future `nextcloud-app` container.
This can be done via a `mariadb` container.
Make sure that the version matches the current database.

.. code-block:: shell

    $ sudo cat db/mysql_upgrade_info
    10.3.29-MariaDBd 

So I will fire up a `mariadb:10.3.29` container and mount the database folder into the container:

.. code-block:: shell

    $ sudo docker run -v /path/to/db:/var/lib/myqsl -d --name mariadb --rm mariadb:10.3.29 mysqld_safe --skip-grant-tables

.. note::

    The command `mysqld_safe --skip-grant-tables` is necessary because otherwise you can't log into the database.

.. caution::

    Best practice would be to restore a mysql dump.
    However, MariaDB's crash recovery just worked on my file snapshot.
    This probably works because my Nextcloud is used by only a few people and there aren't many changes.


Log into the database

.. code-block:: shell

    $ sudo docker exec -it mariadb /bin/bash
    # mysql
    MariaDB[(none)] use nextcloud;
    MariaDB[nextcloud] select host, user from mysql.user;
    +-----------+---------+
    | host      | user    |
    +-----------+---------+
    | localhost | ncadmin |
    | localhost | root    |
    +-----------+---------+
    MariaDB [nextcloud]> update mysql.user set host='nextcloud-app' where user='ncadmin';
    MariaDB [nextcloud]> update mysql.db set host='nextcloud-app' where user='ncadmin';
    MariaDB [nextcloud]> flush privileges;
    MariaDB [nextcloud]> quit;
    # exit
    $ sudo docker stop mariadb

Change file permissions (optional)
----------------------------------

User `www-data` runs both the Nextcloud app and the fastcgi-php thingy.
On armbian `www-data` is user 33.
Under alpine images user 33, however, is `xfs`, and `www-data` is 82.
Therefore, if one wants to use an alpine-based image, everything in `/var/www/html` needs to be `chown`'d to `82`.
If the datadir for nextcloud was moved, it doesn't reside in `/var/www/html/data` anymore.

.. code-block:: shell
    
    sudo chown -R 82:root {php,data}


Change config.php
-----------------

In order to have the nextcloud app find everything, some references need to be adapted in `php/config/config.php`.

.. list-table:: Change these
    :header-rows: 1

    * - key
      - change to
      - required
    * - trusted_domains
      - (add a test domain)
      - no
    * - data directory
      - :code:`/var/www/html/data`
      - yes
    * - dbhost
      - nextcloud-db:3306 
      - yes
