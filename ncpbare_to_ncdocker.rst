######################################################################
Migrating from NextCloudPi bare metal installation to Nextcloud Docker
######################################################################

NextCloudPi is a great way to quickly get started with a private instance of Nextcloud that can be self-hosted on small computers like the Raspberry Pi.
Setup is really easy because it includes sane default configurations.
Also making backups of the instance (config and/or data) is easy with the inbuilt tools.
In addition, if one uses the btrfs filesystem for the data folder, the bare metal installation offers convient ways to create snapshots.

***********
Why change?
***********

While it was great in the beginning, I actually don't use a lot of the features that NextCloudPi offers.
Or I slightly changed scripts from it, so that I fear that a restore or upgrade might overwrite those modifications.
For example, by default, NCP allows you to take btrfs snapshots of `only` the data directory, and they `have` to be stored in the parent folder.
I now have both the data and database directory (what by default would be /var/www/html/nextcloud/data and /var/lib/mysql) in a single btrfs subvolume.
This allows me to take atomic snapshots of both, which in the case of a restore would avoid mismatches.
In addition my modification updates a `latest` symlink in the snapshots folder, so I can conveniently backup the latest snapshot via restic.
Apart from the backup situation, I fear that the smaller number of developers for NCP might make it more likely that it is abandoned.
At least the core Nextcloud team seems a lot bigger and better funded.

**********************
The basic architecture
**********************

The goal is to use the official docker image provided by the Nextcloud team.
To stay similar to the NCP setup, I use the -fpm image which includes FastCGI.
This won't work out of the box, because it needs a few other services.
In the end, there will be five docker containers:

1. nc-app: The main Nextcloud application
2. nc-db: A MariaDB database that nc-app will access.
3. nc-redis: The Redis server.
4. nc-cron: Executes Nextcloud cron jobs.
5. nc-web: A web server that serves the website.

The best way to set up all these requirements is a docker-compose project (docker-compose.yml at the end).

Before just running the docker-compose command, some migrations are needed.

**********
Migrations
**********

.. tip::

    It's best to do a test-run on a copy of the nextcloud files in case something goes wrong.
    I used a btrfs snapshot of my datadir and databasedir and a copy of /var/www/html/nextcloud.

First, get the versions of MariaDB and the current Nextcloud instance.
Having different versions for the bare metal installation and the docker setup just asks for trouble.

.. code-block:: shell
    
    db -V
    mariadb  Ver 15.1 Distrib 10.3.29-MariaDB, for debian-linux-gnu (aarch64) using readline 5.2

This needs docker image mariadb:10.3.29.

.. code-block:: shell

    $ sudo cat /var/www/html/nextcloud/version.php |grep VersionString
    $OC_VersionString = '20.0.11';

And that's Nextcloud 20.0.11.
I'm using the fpm flavour and the lightweight alpine linux as base, so the image is nextcloud:20.0.11-fpm-alpine.

I will work with three folders

1. ncdata: Contains all Nextcloud user data. By default at /var/www/html/nextcloud/data.
2. ncdb: Contains MariaDB files. By default at /var/lib/mysql.
3. php: Contains php configuration. By default at /var/www/html/nextcloud.


Database (MariaDB)
------------------

The database user `ncadmin` created by NextcloudPi can only be accessed from `localhost`, i.e. the machine that the database is running on.
That makes sense for a bare-bones installation where the nextcloud app runs on the same computer, but not for a multi-container setup.
In that case other containers look like other hosts from one container's perspective.
Therefore, access rights must be modified to allow access from the future `nc-app` container.
That container will be part of the `foonet` docker network which has subnet 172.19.0.0/16.
Since I don't know what IP the `nc-app` container will have, I grant access for the whole subnet.
Check a docker network's subnet via `docker network inspect foonet |grep Subnet`.

Changing the access host can be done via a single `mariadb` container.
So I will fire up a `mariadb:10.3.29` container and mount the database folder into the container:

.. code-block:: shell

    $ sudo docker run -v ncdb:/var/lib/myqsl -d --name mariadb --rm mariadb:10.3.29 mysqld_safe --skip-grant-tables

.. note::

    The command `mysqld_safe --skip-grant-tables` is necessary because otherwise you can't log into the database.

.. caution::

    Best practice would be to restore a mysql dump.
    However, MariaDB's crash recovery just worked on my file snapshot.
    This probably works because my Nextcloud is used by only a few people and there aren't many changes.


Log into the database and change the access host for ncadmin.

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
    MariaDB [nextcloud]> update mysql.user set host='172.19.0.0/255.255.0.0' where user='ncadmin';
    MariaDB [nextcloud]> update mysql.db set host='172.19.0.0/255.255.0.0' where user='ncadmin';
    MariaDB [nextcloud]> flush privileges;
    MariaDB [nextcloud]> quit;
    # exit
    $ sudo docker stop mariadb


Change config.php
-----------------

In order to have the nextcloud app find everything, some references need to be adapted in `php/config/config.php`.

.. list-table:: Change these
    :header-rows: 1

    * - key
      - change to
    * - :code:`datadirectory`
      - :code:`/var/www/html/data`
    * - :code:`dbhost`
      - :code:`nextcloud-db:3306`
    * - :code:`redis` array: :code:`host`
      - :code:`nextcloud-redis`
    * - :code:`redis` array: :code:`port`
      - :code:`6379`
    * - :code:`tempdirectory`
      - :code:`/var/www/html/data/tmp`
    * - :code:`logfile`
      - :code:`/var/www/html/data/nextcloud.log`

It is also advisable to change the redis password to something that doesn't contain special characters.
Special characters are somehow jumbled up and then the password of redis and the one in docker-compose don't match.
I think the password doesn't even need to be very secure if the container doesn't expose its port to outside of the docker network.
It's a good idea to temporarily add a test domain under `trusted_domains`.

Alpine image: Change file permissions
-------------------------------------

User `www-data` runs both the Nextcloud app and fpm.
On armbian `www-data` is user 33.
Under alpine images user 33, however, is `xfs`, and `www-data` is 82.
Therefore, if one wants to use an alpine-based image, everything in `/var/www/html` needs to be `chown`'d to `82`.
If the datadir for nextcloud was moved, it doesn't reside in `/var/www/html/data` anymore.

.. note::

    This also applies to the data directory which will be mounted at /var/www/html/data

.. code-block:: shell
    
    sudo chown -R 82:root {php,ncdata}

Alpine image: Nginx www-data
----------------------------

The nginx webserver needs to run as user `www-data` to be able to access the php files.
But under nginx:alpine that user doesn't exist.
In current images you can't add the user because you get an error that it exists (but it doesn't).
Removing and adding also doesn't work.

The last image that `does` allow one to add the user is nginx:10.15.12-alpine.
For that, the image has to be slightly modified via a Dockerfile, that I put under the docker-compose.yml's subdirectory web:

.. code-block:: yaml

    FROM nginx:1.15.12-alpine

    RUN set -x \
            && addgroup -g 82 -S www-data \
            && adduser -u 82 -D -S -G www-data www-data
    # 82 is the standard uid/gid for "www-data" in Alpine

Nginx config
------------

Put this config under web/nginx.conf (mainly copied from the nextcloud docker repo):

.. code-block::

    user www-data;
    worker_processes auto;

    error_log  /var/log/nginx/error.log error;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
    }


    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        keepalive_timeout  65;

        #gzip  on;

        upstream php-handler {
            server nextcloud-app:9000;
        }

        server {
            listen 80;

            # set max upload size
            client_max_body_size 512M;
            fastcgi_buffers 64 4K;

            # Enable gzip but do not remove ETag headers
            gzip on;
            gzip_vary on;
            gzip_comp_level 4;
            gzip_min_length 256;
            gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
            gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

            # HTTP response headers borrowed from Nextcloud `.htaccess`
            add_header Referrer-Policy                      "no-referrer"   always;
            add_header X-Content-Type-Options               "nosniff"       always;
            add_header X-Download-Options                   "noopen"        always;
            add_header X-Frame-Options                      "SAMEORIGIN"    always;
            add_header X-Permitted-Cross-Domain-Policies    "none"          always;
            add_header X-Robots-Tag                         "none"          always;
            add_header X-XSS-Protection                     "1; mode=block" always;

            # Remove X-Powered-By, which is an information leak
            fastcgi_hide_header X-Powered-By;

            # Path to the root of your installation
            root /var/www/html;

            # Specify how to handle directories -- specifying `/index.php$request_uri`
            # here as the fallback means that Nginx always exhibits the desired behaviour
            # when a client requests a path that corresponds to a directory that exists
            # on the server. In particular, if that directory contains an index.php file,
            # that file is correctly served; if it doesn't, then the request is passed to
            # the front-end controller. This consistent behaviour means that we don't need
            # to specify custom rules for certain paths (e.g. images and other assets,
            # `/updater`, `/ocm-provider`, `/ocs-provider`), and thus
            # `try_files $uri $uri/ /index.php$request_uri`
            # always provides the desired behaviour.
            index index.php index.html /index.php$request_uri;

            # Ensure this block, which passes PHP files to the PHP process, is above the blocks
            # which handle static assets (as seen below). If this block is not declared first,
            # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
            # to the URI, resulting in a HTTP 500 error response.
            location ~ \.php(?:$|/) {
                fastcgi_split_path_info ^(.+?\.php)(/.*)$;
                set $path_info $fastcgi_path_info;

                try_files $fastcgi_script_name =404;

                include fastcgi_params;
                # fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param SCRIPT_FILENAME /var/www/html/$fastcgi_script_name;
                fastcgi_param PATH_INFO $path_info;
                fastcgi_param HTTPS on;

                fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
                fastcgi_param front_controller_active true;     # Enable pretty urls
                fastcgi_pass php-handler;

                fastcgi_intercept_errors on;
                fastcgi_request_buffering off;
            }

            location ~ \.(?:css|js|svg|gif)$ {
                try_files $uri /index.php$request_uri;
                expires 6M;         # Cache-Control policy borrowed from `.htaccess`
                access_log off;     # Optional: Don't log access to assets
            }

            location ~ \.woff2?$ {
                try_files $uri /index.php$request_uri;
                expires 7d;         # Cache-Control policy borrowed from `.htaccess`
                access_log off;     # Optional: Don't log access to assets
            }

            # Rule borrowed from `.htaccess`
            location /remote {
                return 301 /remote.php$request_uri;
            }

            location / {
                try_files $uri $uri/ /index.php$request_uri;
            }
        }
    }


:code:`docker-compose.yml`
--------------------------

.. code-block:: yaml

    ---
    version: "3.3"
    services:
      nc-db:
        image: mariadb:10.3.29
        container_name: nc-db
        command: --transaction-isolation=READ-COMMITTED --log-bin=ROW
        restart: unless-stopped
        volumes:
          - ncdb:/var/lib/mysql

      nc-redis:
        image: redis:alpine
        container_name: nc-redis
        hostname: nc-redis
        restart: unless-stopped
        command: redis-server --requirepass asd # use some password

      nc-app:
        image: nextcloud:20.0.11-fpm-alpine
        container_name: nc-app
        restart: unless-stopped
        depends_on:
          - nc-db
          - nc-redis
        environment:
          - REDIS_HOST=nc-redis
          - REDIS_HOST_PASSWORD=asd # redis password given above
          - MYSQL_PASSWORD=xyz # password for user ncadmin
          - MYSQL_DATABASE=nextcloud # database name
          - MYSQL_USER=ncadmin #SQL user name
        volumes:
          - ncdata:/var/www/html/data
          - php:/var/www/html

      nc-web:
        build: ./web
        container_name: nc-web
        restart: always
        ports:
          - 8080:80
        links:
          - nc-app
        volumes:
          - ncdata:/var/www/html/data
          - php:/var/www/html
          - web/nginx.conf:/etc/nginx/nginx.conf:ro
     
      nc-cron:
        image: nextcloud:20.0.11-fpm-alpine
        container_name: nc-cron
        restart: always
        volumes:
          - ncdata:/var/www/html/data
          - php:/var/www/html
        entrypoint: /cron.sh

    networks:
      default:
        external:
          name: foonet

I'm using an external network called foonet that includes my reverse proxy which handles SSL and redirects.
