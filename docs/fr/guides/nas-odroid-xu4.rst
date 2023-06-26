========================================================
[Linux - Odroid] - NAS et nextcloud perso sur Odroid XU4
========================================================

La carte ARM **Odroid XU4** est une des puissantes moutures du fabricant coréen **Hardkernel** dont la renommée n'est plus à faire. Elle embarque un SoC Samsung Exynos 5422 (8 coeurs) et 2Go de RAM, accompagnés par 1 port Ethernet Gigabit, 2 ports USB 3.0 et 1 port USB2.0.

Mon choix en **2016** d'utiliser cette carte pour faire un **NAS** était qu'elle présentait une puissance globalement 3 fois plus puissante que son concurrent anglais **Raspberry Pi 2** et surtout qu'elle possédait des interfaces **Ethernet** et **USB** suffisamment rapides pour le transfert de fichiers.

Aujourd'hui, **6 ans** plus tard, mon **Odroid XU4** rempli toujours autant ses fonctions. Il y a certes eu plusieurs réinstallations du système depuis, pour rester à jour, mais matériellement la carte n'a présenté aucun défaut de fonctionnement.

.. raw:: html

    <div align="center">
        <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/nas-nextcloud/odroid-xu4.png" width=49% align="top"> 
    </div>


Pré-requis
==========

- **Une carte ARM**. Mon choix s'était porté sur l'Odroid XU4 il y a quelques années mais une carte ARM comme le Raspberry Pi 4 semblerait faire l'affaire aujourd'hui.
- **Un ou plusieurs disques durs externes**. Ici aussi mon choix s'était porté sur l'Odroid XU4 car le fabricant était le seul à l'époque à proposer un boitier capable de brancher deux disques durs (boitier **Cloudshell**).


Préparation du système
======================

**Note** : toutes les opérations sont effectuée en **root**.

Je ne détaillerai pas la partie installation du système puisque celle-ci est généralement expliquée dans les documentations et tutoriels des fabricants (écriture d'une image système sur carte SD, etc...).

Quelques pré-requis cependant :

- Un OS basé sur **Debian**.
- Paquets à jour (apt update et apt upgrade).
- Configurer son adresse IP de manère **fixe**. La configuration peut varier selon l'OS. Sur Ubuntu-server par exemple c'est généralement netplan qui gère la configuration réseau.
- Un ou deux disques durs externes reconnus et formattés dans un format reconnu par Linux (ext4 par exemple). J'ai personnellement fait le choix de ne pas utiliser le système de RAID proposé par le boitier **Cloudshell** de l'Odroid XU4 mais plutôt de répliquer à intervalle régulière le premier disque vers le deuxième avec rsync. Le problème étant avec ce genre de carte RAID est que si celle-ci venait à avoir un défaut matériel dans quelques années, il n'est pas certain de pouvoir la remplacer (arrêt de production par le fabricant par exemple), rendant potentiellment les données impossible à récupérer.

Je suis personnellement sur **Ubuntu-server 22.04 LTS** sur mon Odroid XU4. L'image système est fournie par le fabricant. Il existe d'autres OS compatibles avec cette carte tels que **Armbian** et **Openmediavault** (une image dédiée aux solutions NAS).

Une fois le premier démarrage et les mises à jour effectués, poursuivre la préparation en installant quelques paquets utiles :

..  code-block:: shell

    apt install vim htop rsync unzip

Puis configurer les locales en français. Sélectionner **fr_FR.UTF-8 UTF-8** dans la liste, puis confirmer à nouveau ce choix à l’étape suivante afin qu’il soit configuré par défaut :

..  code-block:: shell

    apt install language-pack-fr
    echo "LANG=fr_FR.UTF-8" > /etc/default/locale
    dpkg-reconfigure locales

Configurer la time zone. Choisir **Europe** puis **Paris**.

..  code-block:: shell
    
    dpkg-reconfigure tzdata

Monter le disque dur externe. Si avez l'intention d'utiliser 2 disques durs comme moi, nous verrons cela plus tard, on s'occupe uniquement du premier disque (disque dur principal) pour le moment.

Créer le point de montage :

..  code-block:: shell

    mkdir /mnt/disque

Avec **blkid**, repérer l'UUID du disque dur :

..  code-block:: shell

    blkid

    /dev/mmcblk1p2: LABEL="rootfs" UUID="e139ce78-9841-40fe-8823-96a304a09859" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="3cedfd53-02"
    /dev/sdb1: LABEL="xu4-disque-sdb" UUID="20756356-5b84-43ee-a396-fad3ba5ea34c" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="f543fd74-dcc6-4318-bad0-21e8d6d508a6" # Second disque dur externe
    /dev/mmcblk1p1: SEC_TYPE="msdos" LABEL_FATBOOT="boot" LABEL="boot" UUID="52AA-6867" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="3cedfd53-01"
    /dev/sda1: LABEL="xu4-disque-sda" UUID="94b4dc47-43ee-423c-a325-f1c2ae5e7495" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="e4192971-1e5f-45ec-af2f-60e5b5c9bcc3" # Premier disque dur externe

Puis éditer **/etc/fstab** puis indiquer les paramètres de montage pour ce premier disque dur en précisant l'UUID précédemment récupéré :

..  code-block:: shell

    vim /etc/fstab

..  code-block:: shell

    UUID=94b4dc47-43ee-423c-a325-f1c2ae5e7495 /mnt/disque ext4 defaults 0 2


Installation des services
=========================

**Note** : toutes les opérations sont effectuée en **root**.

Samba
-----

Préparation
~~~~~~~~~~~

La mise en place de **Samba** permet d’accéder aux fichiers stockés sur le disque principal depuis un PC **Windows** ou **Linux**.

Personnellement, j’ai choisi de mettre en place la configuration suivante :

- 1 répertoire **Partage** accessible à tous les PC du réseau local. Ce répertoire contiendrait par exemple des photos, de la musique, des films...
- 1 répertoire **Perso** pour chaque utilisateur. L’utilisateur aura accès à son répertoire mais n’aura pas accès à celui des autres.

Créer un répertoire pour samba dans **/mnt/disque** :

..  code-block:: shell

    mkdir /mnt/disque/samba

Puis créer le répertoire **Partage** ainsi que les répertoires personnels de chaque utilisateur :

..  code-block:: shell

    mkdir /mnt/disque/samba/Partage
    mkdir /mnt/disque/samba/toto # Ici on crée le répertoire pour l'utilisateur 'toto', faire de même pour tout autre utilisateur
    ...

Installer Samba et ses outils de test :

..  code-block:: shell

    apt install samba smbclient samba-testsuite

Samba crée par défaut un groupe **sambashare**. Attribuer les droits suivants au répertoire précédemment créé :

..  code-block:: shell

    chown root:sambashare /mnt/disque/samba
    chmod 550 /mnt/disque/samba


Configuration
~~~~~~~~~~~~~

Vider le contenu de **/etc/samba/smb.conf** actuel :

..  code-block:: shell

    echo -n > /etc/samba/smb.conf

Éditer **/etc/samba/smb.conf** et ajouter la configuration suivante :

..  code-block:: shell

    vim /etc/samba/smb.conf


..  code-block:: shell

    #======================= Global Settings =======================

    [global]
    workgroup = WORKGROUP
    netbios name = NAS
    server string = %h server (Samba, Ubuntu)
    security = user
    dns proxy = no
    log level = 2
    log file = /var/log/samba/samba.log
    max log size = 50
    

    #======================= Share Definitions =======================
    # Répertoire perso de chaque utilisateur. 
    [Perso]
    comment = Repertoire Perso
    # Si l'utilisateur s'appelle toto, alors son répertoire perso sera automatiquement traduit par /mnt/disque/samba/toto grâce à la variable %u:
    path = /mnt/disque/samba/%u
    browseable = yes
    public = no
    writeable = yes
    create mask = 0700
    directory mask = 0700
    printable = no

    # Répertoire de partage entre utilisateurs 
    [Partage]
    comment = Repertoire Partage
    path = /mnt/disque/samba/Partage
    browseable = yes
    public = no
    writeable = yes
    create mask = 0770
    directory mask = 0770
    force group = sambashare
    printable = no


Enregistrer puis tester la configuration avec l'outil de test fourni par samba. La commande ne doit pas renvoyer d’erreurs :

..  code-block:: shell

    testparm -s

Redémarrer les services samba :

..  code-block:: shell

    systemctl restart smbd
    systemctl restart nmbd

**Note** : pour que le partage utilisateur se monte **automatiquement** sur un PC Windows sans demander de mot de passe, il est conseillé de créer un compte utilisateur **identique** (respecter la casse) à son compte de session Windows ainsi que le même mot de passe. Car par défaut Windows tentera d'utiliser son nom d'utilisateur + mot de passe de session Windows pour se connecter à un partage réseau.

Pour Samba il est nécessaire à chaque fois de créer doublement un utilisateur :

- Un compte utilisateur **Samba** pour l'accès aux partages
- Un compte utilisateur **Linux** qui permettra de gérer et de cloisonner les permissions sur les répertoires

Commencer par créer un utilisateur Linux (exemple ici **toto**) qui fera partie du groupe **sambashare** et sans accès au shell :

..  code-block:: shell

    useradd -s /usr/sbin/nologin -G sambashare toto

Créer un mot de passe pour cet utilisateur.

..  code-block:: shell

    passwd toto

L’utilisateur Linux est prêt, créer ensuite un utilisateur **Samba** du même nom et lui attribuer un mot de passe (c'est à cette étape qu'on indique le même que sa session Windows le cas échéant).

..  code-block:: shell

    smbpasswd -a toto

Enfin, appliquer les permissions suivantes sur les répertoires de partage. Tous les utilisateurs pourront accéder à **Partage** mais seuls les utilisateurs pourront accéder à leur propre répertoire personnel (ici c'est la cas pour **toto**) :

..  code-block:: shell

    chown -R root:sambashare /mnt/disque/samba/Partage
    chmod -R 770 /mnt/disque/samba/Partage

    # Ici on ajuste les permissions du répertoire pour l'utilisateur 'toto', faire de même pour tout autre utilisateur
    chown -R toto:root /mnt/disque/samba/toto
    chmod -R 700 /mnt/disque/samba/toto

Tester les accès :

**Sur PC Windows** : 

- Utiliser l'explorateur de fichiers pour explorer le réseau et accéder au NAS.

**Sur PC Linux** :

- Se loguer à sa session avec son utilisateur, par exemple **toto**.
- Créer les répertoires qui seront dédiés à monter les partages du NAS :

..  code-block:: shell

    mkdir -p /mnt/NAS/Partage
    mkdir -p /mnt/NAS/Perso

    chown toto:toto /mnt/NAS/*
    chmod 700 /mnt/NAS/*

Créer un fichier de credentials qui contiendra l'utilisateur et le mot de passe **Samba** à utiliser pour s'authentifier. En l'occurence pour **toto** :

..  code-block:: shell

    vim /home/toto/.smbcredentials

..  code-block:: shell

    username=toto
    password=mdp_de_toto

Puis ajouter les entrées suivantes dans **/etc/fstab** :

..  code-block:: shell

    vim /etc/fstab

..  code-block:: shell

    //IP_du_NAS/Partage /mnt/NAS/Partage  cifs credentials=/home/toto/.smbcredentials,iocharset=utf8,file_mode=0660,dir_mode=0770,users,uid=toto,_netdev,sec=ntlmv2,noauto 0 2
    //IP_du_NAS/Perso /mnt/NAS/Perso  cifs credentials=/home/toto/.smbcredentials,iocharset=utf8,file_mode=0660,dir_mode=0770,users,uid=toto,_netdev,sec=ntlmv2,noauto 0 2

Avec ces lignes, les partages seront automatiquement montés au démarrage du PC. En attendant les monter manuellement :

..  code-block:: shell

    mount /mnt/NAS/Partage
    mount /mnt/NAS/Perso


Nextcloud
---------

Pour accéder à mes fichiers depuis Internet et les partager, j’ai choisi la solution **nextcloud**, un fork gratuit de **owncloud**.

Il va donc falloir installer un serveur web de type **LEMP** (Linux EngineX (nginx), MySQL, PHP).


Pré-requis
~~~~~~~~~~

- Réserver un nom de domaine pour les accès depuis l'extérieur. Si besoin **OVH** propose des noms de domaines **.ovh** à petit prix.


MySQL
~~~~~

Installation
++++++++++++

Installer MySQL 8.0 :

..  code-block:: shell

    apt install mysql-server

Changer le mot de passe du compte **root** MySQL actuellement vide en passant par le prompt mysql :

..  code-block:: shell

    mysql -u root

    mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'NOUVEAU_MDP';
    mysql> exit

Puis terminer l’installation en lançant le script suivant :

..  code-block:: shell

    /usr/bin/mysql_secure_installation

La première question propose de changer le mot de passe root de MySQL. Répondre **N** à cette question car nous l'avons déjà changé précédemment : 

..  code-block:: shell

    Enter password for user root: 
    The 'validate_password' component is installed on the server.
    The subsequent steps will run with the existing configuration
    of the component.
    Using existing password for root.

    Estimated strength of the password: 100 
    Change the password for root ? ((Press y|Y for Yes, any other key for No) : N

Par défaut, MySQL crée un utilisateur **anonymous** permettant à quiconque de se connecter à MySQL sans compte attitré. Répondre **Y** à la question suivante pour supprimer cet utilisateur :

..  code-block:: shell

    Remove anonymous users? (Press y|Y for Yes, any other key for No) : Y

La question suivante demande si l’on veut autoriser **root** à se connecter à MySQL depuis une machine distante (différente de **localhost**). Dans notre cas ce ne sera pas nécessaire, on choisi de ne pas autoriser ce type de connexion en répondant par **Y** :

..  code-block:: shell

    Disallow root login remotely? (Press y|Y for Yes, any other key for No) : Y

Par défaut, MySQL crée une base de données **test** accessible à tout le monde. Répondre **Y** à la question suivante pour supprimer cette base de données :

..  code-block:: shell

    Remove test database and access to it? (Press y|Y for Yes, any other key for No) : Y

Appliquer les changements en répondant **Y** à la question suivante :

..  code-block:: shell

    Reload privilege tables now? (Press y|Y for Yes, any other key for No) : Y

Vérifier que le **service mysql est bien démarré** :

..  code-block:: shell

    systemctl status mysql


PHP 8.1
~~~~~~~

Installation
++++++++++++

Installer un repo de paquet supplémentaire afin d'avoir accès aux paquets **PHP 8.1** :

..  code-block:: shell

    apt install apt-transport-https -y
    add-apt-repository ppa:ondrej/php -y
    apt update

Installer **PHP (FPM) 8.1** et tous les **modules** nécessaires pour Nextcloud :

..  code-block:: shell

    apt install php8.1-fpm php8.1-mysql php8.1-gd php8.1-curl php8.1-intl php-imagick php8.1-zip php8.1-xml php8.1-mbstring php8.1-imagick php8.1-bcmath php8.1-gmp


Configuration
+++++++++++++

Éditer le fichier **/etc/php/8.1/fpm/php.ini** et modifier la valeur du paramètre **cgi.fix_pathinfo** en passant sa valeur à 0. Dé-commenter la ligne si elle est commentée :

..  code-block:: shell

    vim /etc/php/8.1/fpm/php.ini

..  code-block:: shell

    cgi.fix_pathinfo=0

Puis modifier les lignes suivantes afin d’augmenter la limite d’upload et le timeout :

..  code-block:: shell

    memory_limit = 512M       # Définit la mémoire max allouée pour chaque script PHP. La valeur recommendée par Nextcloud est 512 Mo
    post_max_size = 5G        # Définit la taille maximale des données reçues par la méthode POST. Ici fixée à 5Go.
    upload_max_filesize = 5G  # Définit la taille maximale d'un fichier à charger. Ici fixée à 5Go.
    max_input_time = 300      # Timeout fixé à 5min
    max_execution_time = 300  # Timeout d'exécution des scripts PHP fixé à 5min

Éditer le fichier **/etc/php/8.1/fpm/pool.d/www.conf** et décommenter la ligne suivante :

..  code-block:: shell

    vim /etc/php/8.1/fpm/pool.d/www.conf

..  code-block:: shell

    env[PATH] = /usr/local/bin:/usr/bin:/bin

Récupérer le nom du fichier de **socket Unix** utilisé par PHP, cela sera utile par la suite pour le paramètrage du vhost nginx :

..  code-block:: shell

    grep "^listen =" /etc/php/8.1/fpm/pool.d/www.conf
    listen = /run/php/php8.1-fpm.sock

Redémarrer PHP-FPM pour appliquer les modifications :

..  code-block:: shell

    systemctl restart php8.1-fpm


Nginx
~~~~~

Installation
++++++++++++

Installer nginx :

..  code-block:: shell

    apt install nginx

Préparation
+++++++++++

Comme préconisé, vous devez avoir réservé un **nom de domaine** et éventuellement créé un sous-domaine dédié à nextcloud. Ici pour l'exemple ce sera **nextcloud.mondomaine.ovh**.

Pour un accès depuis l'extérieur il faudra faire en sorte que ce sous-domaine pointe vers l'**adresse IP publique de votre box Internet**, et mettre en place une redirection de port 80 et 443 vers le serveur NAS. Attention toutefois une fois les redirections de ports en place, les robots qui scannent en permanence l'Internet mondial pourraient parvenir à atteindre votre serveur web et pourraient tenter de se connecter. Il faut donc veiller à mettre en place des règles de firewall (iptables par exemple).

Pour un accès depuis le réseau local seulement, un nom de domaine est facultatif. Par contre il ne sera pas possible de commander un certificat SSL.


Vhost nextcloud (:80)
+++++++++++++++++++++

Préparer le répertoire qui contiendra les sources de **Nextcloud** :

..  code-block:: shell

    mkdir -p /var/www/nextcloud
    mkdir -p /var/www/nextcloud/.well-known/acme-challenge/ # Pour la commande de certificat SSL
    chown -R www-data:www-data /var/www/nextcloud
    chmod -R 750 /var/www/nextcloud

La déclaration et la configuration de **vhosts** s’effectue dans deux répertoires :

- **/etc/nginx/sites-available** : qui contient les fichiers de configuration des vhosts. Les fichiers stockées ici ne sont pas automatiquement pris en compte.
- **/etc/nginx/sites-enabled** : contient des liens symboliques vers les fichiers de vhosts présents dans sites-available. Une fois le lien symbolique ajouté, le fichier est pris en compte et le site est activé.

Par défaut, Nginx génère un fichier **default** dans **/etc/nginx/sites-available**. Supprimer ce fichier :

..  code-block:: shell

    rm -f /etc/nginx/sites-available/default
    rm -f /etc/nginx/sites-enabled/default

Créer un nouveau fichier de vhost pour **Nextcloud** :

..  code-block:: shell

    touch /etc/nginx/sites-available/nextcloud.conf
    chown www-data:www-data /etc/nginx/sites-available/nextcloud.conf
    chmod 660 /etc/nginx/sites-available/nextcloud.conf

Editer le fichier puis insérer la configuration suivante :

..  code-block:: shell

    vim /etc/nginx/sites-available/nextcloud.conf

..  code-block:: shell

    server {
        listen 80;
        server_name nextcloud.mondomaine.ovh;
        root /var/www/nextcloud;

        # Forcer https ; on laisse ce paramètre commenté pour le moment
        # return 301 https://$server_name$request_uri;

        access_log /var/log/nginx/nextcloud.mondomaine.ovh_access.log;
        error_log /var/log/nginx/nextcloud.mondomaine.ovh_error.log;
    }

Créer un lien symbolique dans **sites-enabled** afin d’activer ce fichier de configuration puis redémarrer **nginx** :

..  code-block:: shell

    cd /etc/nginx/sites-enabled
    ln -s ../sites-available/nextcloud.conf
    systemctl restart nginx

C’est le strict minimum pour le moment. On a ici un **vhost** qui écoute sur le port **80**. Nous ajouterons un second vhost qui écoutera sur le port 443 (https) lorsque nous aurons un certificat SSL.

J’ai déjà créé un article sur **getssl**, un script bash qui permet de commander un certificat SSL. Pour éviter les doublons, je vous invite à suivre cet article jusqu’à la fin et de commander un certificat pour le nom de domaine **nextcloud.mondomaine.ovh**.

Lien vers l’article : `getssl <getssl.html>`_

A ce stade, vous devriez exécuter la commande suivante pour commander votre certificat (exemple) : 

..  code-block:: shell

    ./getssl nextcloud.mondomaine.ovh


Vhost SSL nextcloud (:443)
++++++++++++++++++++++++++

D’abord, dans le Vhost **80**, dé-commenter la redirection vers **https** :

..  code-block:: shell

    vim /etc/nginx/sites-available/nextcloud.conf

..  code-block:: shell

    return 301 https://$server_name$request_uri;

Créer un nouveau fichier de vhost :

..  code-block:: shell

    touch /etc/nginx/sites-available/nextcloud_ssl.conf
    chown www-data:www-data /etc/nginx/sites-available/nextcloud_ssl.conf
    chmod 660 /etc/nginx/sites-available/nextcloud_ssl.conf

Editer le fichier puis insérer la configuration suivante (sur les premières lignes, indiquer le bon socket Unix récupéré précédemment (partie configuration de PHP), si différent) :

..  code-block:: shell

    vim /etc/nginx/sites-available/nextcloud_ssl.conf

..  code-block:: shell

    upstream php-handler {
        # Socket Unix PHP
        server unix:/run/php/php8.1-fpm.sock;
    }

    # Set the `immutable` cache control options only for assets with a cache busting `v` argument
    map $arg_v $asset_immutable {
        "" "";
        default "immutable";
    }

    server {
        listen 443 ssl http2;
        server_name nextcloud.mondomaine.ovh;

        # Path to the root of your installation
        root /var/www/nextcloud;

        # Use Mozilla's guidelines for SSL/TLS settings
        # https://mozilla.github.io/server-side-tls/ssl-config-generator/
        ssl_certificate CHEMIN-VERS-CERTIFICAT.crt;
        ssl_certificate_key CHEMIN-VERS-CLE-PRIVEE.key;

        # Prevent nginx HTTP Server Detection
        server_tokens off;

        # HSTS settings
        # WARNING: Only add the preload option once you read about
        # the consequences in https://hstspreload.org/. This option
        # will add the domain to a hardcoded list that is shipped
        # in all major browsers and getting removed from this list
        # could take several months.
        #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload" always;

        # set max upload size and increase upload timeout:
        client_max_body_size 512M;
        client_body_timeout 300s;
        fastcgi_buffers 64 4K;

        # Enable gzip but do not remove ETag headers
        gzip on;
        gzip_vary on;
        gzip_comp_level 4;
        gzip_min_length 256;
        gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
        gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

        # Pagespeed is not supported by Nextcloud, so if your server is built
        # with the `ngx_pagespeed` module, uncomment this line to disable it.
        #pagespeed off;

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

        # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
        location = / {
            if ( $http_user_agent ~ ^DavClnt ) {
                return 302 /remote.php/webdav/$is_args$args;
            }
        }

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        # Make a regex exception for `/.well-known` so that clients can still
        # access it despite the existence of the regex rule
        # `location ~ /(\.|autotest|...)` which would otherwise handle requests
        # for `/.well-known`.
        location ^~ /.well-known {
            # The rules in this block are an adaptation of the rules
            # in `.htaccess` that concern `/.well-known`.

            location = /.well-known/carddav { return 301 /remote.php/dav/; }
            location = /.well-known/caldav  { return 301 /remote.php/dav/; }

            location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
            location /.well-known/pki-validation    { try_files $uri $uri/ =404; }

            # Let Nextcloud's API for `/.well-known` URIs handle all other
            # requests by passing them to the front-end controller.
            return 301 /index.php$request_uri;
        }

        # Rules borrowed from `.htaccess` to hide certain paths from clients
        location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
        location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

        # Ensure this block, which passes PHP files to the PHP process, is above the blocks
        # which handle static assets (as seen below). If this block is not declared first,
        # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
        # to the URI, resulting in a HTTP 500 error response.
        location ~ \.php(?:$|/) {
            # Required for legacy support
            rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+|.+\/richdocumentscode\/proxy) /index.php$request_uri;

            fastcgi_split_path_info ^(.+?\.php)(/.*)$;
            set $path_info $fastcgi_path_info;

            try_files $fastcgi_script_name =404;

            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $path_info;
            fastcgi_param HTTPS on;

            fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
            fastcgi_param front_controller_active true;     # Enable pretty urls
            fastcgi_pass php-handler;

            fastcgi_intercept_errors on;
            fastcgi_request_buffering off;

            fastcgi_max_temp_file_size 0;
        }

        location ~ \.(?:css|js|svg|gif|png|jpg|ico|wasm|tflite|map)$ {
            try_files $uri /index.php$request_uri;
            add_header Cache-Control "public, max-age=15778463, $asset_immutable";
            access_log off;     # Optional: Don't log access to assets

            location ~ \.wasm$ {
                default_type application/wasm;
            }
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


Créer un lien symbolique dans **sites-enabled** afin d’activer ce fichier de configuration puis redémarrer **nginx** :

..  code-block:: shell

    cd /etc/nginx/sites-enabled
    ln -s ../sites-available/nextcloud_ssl.conf
    systemctl restart nginx


Nextcloud
~~~~~~~~~

Télécharger la dernière version de Nextcloud et décompresser le contenu de l'archive dans **/var/www/** :

..  code-block:: shell

    cd /var/www/
    wget https://download.nextcloud.com/server/releases/latest.zip


..  code-block:: shell

    unzip latest.zip
    rm -f latest.zip

Appliquer les bonnes permissions sur les fichiers extraits :

..  code-block:: shell

    chown -R www-data:www-data nextcloud
    find nextcloud -type f -exec chmod -v 0640 {} \;
    find nextcloud -type d -exec chmod -v 0750 {} \;

Puis préparer une **base de données** pour Nextcloud. Se connecter à MySQL en tant que **root** :

..  code-block:: shell

    mysql -u root -p

Créer une nouvelle base de données **nextcloud** :

..  code-block:: shell

    mysql> CREATE DATABASE nextcloud;

Créer un utilisateur **nextcloud** dédié à cette base de données en lui spécifiant un nouveau mot de passe :

..  code-block:: shell

    mysql> CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'mdp_de_nextcloud'; # veiller à préciser un mot de passe compliqué sans quoi il ne sera refusé par mysql
    mysql> GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';

Appliquer les changements puis quitter le prompt :

..  code-block:: shell

    mysql> FLUSH PRIVILEGES;
    mysql> exit

Enfin, préparer le répertoire dédié à stocker les données de **Nextcloud** (notamment les fichiers uploadés par les utilisateurs). Comme il peut vite s'agir d'un répertoire volumineux, le mieux est de créer ce répertoire sur le disque externe :

..  code-block:: shell
    
    mkdir -p /mnt/disque/nextcloud/data
    chown -R www-data:www-data /mnt/disque/nextcloud
    chmod 750 /mnt/disque/nextcloud/

C'est terminé. Il est temps de se connecter à **Nextcloud** depuis un navigateur à l’adresse https://nextcloud.mondomaine.ovh

La mire de connexion demande quelques informations pour terminer l'installation.

- Renseigner un nom d’utilisateur afin de créer un **compte administrateur** (par exemple admin) avec un mot de passe fort.
- Renseigner le chemin vers le répertoire de stockage des données (créé précedemment) **/mnt/disque/nextcloud/data**
- Renseigner l'utilisateur de base de données **nextcloud** créé précédemment pour l’occasion et son mot de passe associé, ainsi que le nom de la base de données **nextcloud**
- Puis cliquer sur **Installer**.

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/nas-nextcloud/nextcloud-first-login.png


Résoudre les erreurs de configuration de la page d'administration de Nextcloud
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Après la première connexion et la découverte de l'environnement Nextcloud, il reste quelques points importants à ajuster pour que l'utilisation de Nextcloud soit optimale et respecte les préconisations de sécurité.

Depuis l'icône de profil utilisateur en haut à droite, se rendre dans **Paramètres d'administration**. A partir d'ici la section **Avertissements de sécurité & configuration** affiche quelques points plus ou moins importants à corriger.


Montage des partages Samba dans Nextcloud
+++++++++++++++++++++++++++++++++++++++++

Afin d'accéder au contenu des partages créés avec **Samba**, il existe un plugin **Nextcloud** à activer et paramétrer.

- Avec un compte administrateur, depuis l'icône de profil utilisateur en haut à droite, se rendre dans **Applications** puis dans **Vos applications**, rechercher et activer l'application **External storage support**.
- Depuis l'icône de profil utilisateur en haut à droite, se rendre dans **Paramètres d'administration** puis **Stockage externe** (de la partie **Administration** sur la droite). 
- Indiquer les paramètres d'accès au partage Samba, ici le partage est sur le même hôte (localhost) et le nom du partage est **Partage**. Le répertoire sera accessible à l'utilisateur **toto**. Faire de même pour le partage **Perso** de l'utilisateur.

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/nas-nextcloud/nextcloud-samba.png


Le(s) partage(s) devient alors accessible directement depuis l'explorateur de fichier de **Nextcloud**.


Mise à jour de Nextcloud
++++++++++++++++++++++++

Nextcloud préviendra lorsqu'une nouvelle mise à jour est disponible. Si c'est le cas, alors il est préférable d'effectuer la mise à jour depuis le terminal pour **éviter tout timeout** depuis l'interface web (car l'opération peut prendre un peu de temps).

Se rendre dans le répertoire d'installation de **Nextcloud** :

..  code-block:: shell

    cd /var/www/nextcloud

Puis exécuter le script **occ** dédié à la maintenance de **Nextcloud** en tant que **www-data**, pour vérifier si une mise à jour est disponible :

..  code-block:: shell

    chmod +x occ
    sudo -u www-data ./occ update:check

Si c'est le cas alors exécuter l'installation de la mise à jour :

..  code-block:: shell

    chmod +x /var/www/nextcloud/updater/updater.phar
    sudo -u www-data /var/www/nextcloud/updater/updater.phar

..  code-block:: shell

    Nextcloud Updater - version: v24.0.0beta3-1-g67bf13b dirty

    Current version is 24.0.2.

    Update to Nextcloud 24.0.8 available. (channel: "stable")
    Following file will be downloaded automatically: https://download.nextcloud.com/server/releases/nextcloud-24.0.8.zip
    Open changelog ↗

    Steps that will be executed:
    [ ] Check for expected files
    [ ] Check for write permissions
    [ ] Create backup
    [ ] Downloading
    [ ] Verify integrity
    [ ] Extracting
    [ ] Enable maintenance mode
    [ ] Replace entry points
    [ ] Delete old files
    [ ] Move new files in place
    [ ] Done

    Start update? [y/N] y

    Info: Pressing Ctrl-C will finish the currently running step and then stops the updater.

    [✔] Check for expected files
    [✔] Check for write permissions
    [✔] Create backup
    [✔] Downloading
    [✔] Verify integrity
    [✔] Extracting
    [✔] Enable maintenance mode
    [✔] Replace entry points
    [✔] Delete old files
    [✔] Move new files in place
    [✔] Done

    Update of code successful.

    Should the "occ upgrade" command be executed? [Y/n] y


Réplication sur second disque
=============================

Si comme moi votre installation comporte un second disque dur externe dédié à répliquer les données du premier disque, il faut alors mettre en place une copie régulière des données. 

Créer le point de montage pour ce disque secondaire (présumé vierge pour le moment) :

..  code-block:: shell

    mkdir /mnt/disque_sauvegarde

Avec **blkid**, repérer l'UUID du disque secondaire :

..  code-block:: shell

    blkid

    /dev/mmcblk1p2: LABEL="rootfs" UUID="e139ce78-9841-40fe-8823-96a304a09859" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="3cedfd53-02"
    /dev/sdb1: LABEL="xu4-disque-sdb" UUID="20756356-5b84-43ee-a396-fad3ba5ea34c" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="f543fd74-dcc6-4318-bad0-21e8d6d508a6" # Second disque dur externe
    /dev/mmcblk1p1: SEC_TYPE="msdos" LABEL_FATBOOT="boot" LABEL="boot" UUID="52AA-6867" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="3cedfd53-01"
    /dev/sda1: LABEL="xu4-disque-sda" UUID="94b4dc47-43ee-423c-a325-f1c2ae5e7495" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="e4192971-1e5f-45ec-af2f-60e5b5c9bcc3" # Premier disque dur externe

Puis éditer **/etc/fstab** puis indiquer les paramètres de montage pour ce disque secondaire en précisant l'UUID précédemment récupéré :

..  code-block:: shell

    vim /etc/fstab

..  code-block:: shell

    20756356-5b84-43ee-a396-fad3ba5ea34c /mnt/disque_sauvegarde ext4 defaults 0 2

Créer un nouveau script dédié à répliquer le contenu des partages Samba et des données Nextcloud. Ce script enverra un mail en cas d'erreur, il faut donc veiller à adapter l'adresse mail de destination (MAIL_RECIPIENT) :

..  code-block:: shell
    
    mkdir /root/scripts
    vim /root/scripts/script-rsync.sh

..  code-block:: shell

    #!/usr/bin/env bash

    set -u
    GREEN=$(tput setaf 2)
    RED=$(tput setaf 1)
    RESET=$(tput sgr0)
    DATE="date +%d-%m-%Y"
    TIME="date +%Hh%M"
    SOURCE_DISK="/mnt/disque"
    TARGET_DISK="/mnt/disque_sauvegarde"
    ERROR="0"
    ERROR_MSG=""
    LOG_DIR="/var/log/scripts"
    LOG="${LOG_DIR}/script-rsync.log"
    MAIL_RECIPIENT="monmail@mail.com" # Adresse mail destinatrice des mails d'erreurs
    RSYNC_PARAMS="-a"


    ## Fonctions ##

    # Affichage de l'aide
    help()
    {
        echo -e "\nParamètres :"
        echo "   -v         ➤  Afficher la progression de rsync"
        echo "   --delete   ➤  Supprimer les fichiers sur la cible qui n'existent plus sur la source"
        echo "   --help     ➤  Afficher l'aide"
    }

    # Envoi d'un mail d'erreur et quitte
    send_error_mail()
    {
        echo "$ERROR_MSG" | mutt -s "[ ERREUR ] - Copie RSYNC échouée" -- "$MAIL_RECIPIENT"
        exit 1
    }

    # Envoi d'un mail de succès et quitte
    send_mail()
    {
        echo "" | mutt -s "[ OK ] - Copie RSYNC terminée" -- "$MAIL_RECIPIENT"
        exit 0
    }


    ## Traitement ##

    mkdir -p "$LOG_DIR"

    # Vidage du fichier de log
    echo -n> "$LOG"
    exec &> >(tee -a "$LOG")

    while [ $# -ge 1 ];do
        case "$1" in
            --help)
                help
                exit
            ;;
            --v|-v)
                RSYNC_PARAMS+=" -P"
                ;;
            --delete)
                RSYNC_PARAMS+=" --delete-after"
                ;;
            *)
        esac
        shift
    done


    # Vérification que les 2 disques sont bien montés aux emplacements prévus
    echo -ne "\nVérification des points de montage : "

    if ! grep -qs "$SOURCE_DISK" /proc/mounts && grep -qs "$TARGET_DISK" /proc/mounts; then
        ERROR_MSG="Le disque n'est pas monté sur le point de montage ou il est en erreur. Sauvegarde interrompue."
        echo -e "[$RED ERREUR $RESET] $ERROR_MSG"
        send_error_mail
    fi

    echo -e "[$GREEN OK $RESET]"

    echo -e "\n`$DATE` à `$TIME` - Démarrage de la sauvegarde"


    rsync $RSYNC_PARAMS $SOURCE_DISK/samba $TARGET_DISK/
    if [ $? -ne "0" ];then
        (( ERROR++ ))
    fi

    rsync $RSYNC_PARAMS $SOURCE_DISK/nextcloud $TARGET_DISK/
    if [ $? -ne "0" ];then
        (( ERROR++ ))
    fi

    # Si il y a eu des erreurs
    if [ "$ERROR" -ne "0" ];then
        send_error_mail
    fi

    echo -e "\nOpération terminée"

    send_mail


Installer **mutt** pour l'envoi de mail :

..  code-block:: shell

    apt install mutt

Puis configurer un nouveau fichier de configuration pour mutt :

..  code-block:: shell

    vim /root/.muttrc

Indiquer le contenu suivant afin de configurer le compte mail émetteur, ici il s'agira d'un compte **Gmail**. Adapter les valeurs pour **ADRESSE_MAIL_GMAIL** et **MDP_DU_COMPTE_GMAIL** :

..  code-block:: shell

    set ssl_starttls = "yes"
    set ssl_force_tls = "yes"
    set use_envelope_from = "yes"
    set copy = "no"
    set charset = "utf-8"
    set realname = "Serveur NAS"
    set from = "ADRESSE_MAIL_GMAIL"
    set smtp_url = "smtps://ADRESSE_MAIL_GMAIL@smtp.gmail.com:465/"
    set smtp_pass = "MDP_DU_COMPTE_GMAIL"

Tester l'envoi d'un mail avec **mutt** :

..  code-block:: shell

    echo 'test' | mutt -s 'test' -- email_destinataire@mail.com

Enfin, configurer une **tâche cron** pour exécuter autant de fois que nécessaire la réplication du **disque dur principal vers le disque dur secondaire**. A vous de trouver la bonne combinaison en fonction de votre utilisation du NAS :

- Pour une faible utilisation du NAS, deux synchronisations par jour (1 à minuit et 1 à midi) peuvent suffire.
- Pour une utilisation très régulière du NAS, des synchronisations fréquentes toutes les **2h** ou toutes les **4h** sont envisageables. Attention toutefois selon la capacité des disques durs et leur vitesse, le calcul de la différence de fichiers effectuée par rsync peut prendre du temps, il ne faut pas planifier de synchronisations trop rapprochées.

..  code-block:: shell

    crontab -e

Exécution tous les jours à minuit 00:00 :

..  code-block:: shell

    0   0   *   *   *   /root/scripts/script-rsync.sh -v --delete-after

.. raw:: html

    <script src="https://giscus.app/client.js"
        data-repo="lbr38/documentation"
        data-repo-id="R_kgDOH7ogDw"
        data-category="Announcements"
        data-category-id="DIC_kwDOH7ogD84CS53q"
        data-mapping="pathname"
        data-strict="1"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="light"
        data-lang="fr"
        crossorigin="anonymous"
        async>
    </script>

    <!-- Google tag (gtag.js) -->
    <script async src="https://www.googletagmanager.com/gtag/js?id=G-SS18FTVFFS"></script>
    <script>
        window.dataLayer = window.dataLayer || [];
        function gtag(){dataLayer.push(arguments);}
        gtag('js', new Date());

        gtag('config', 'G-SS18FTVFFS');
    </script>
