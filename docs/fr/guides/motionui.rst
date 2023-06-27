=====================================================
[Linux] - Video surveillance avec motion et motion-UI
=====================================================

**Motion-UI** est une interface web (User Interface) développée pour gérer plus aisémment le fonctionnement et la configuration de **motion**, un célèbre logiciel **open-source** de détection de mouvement généralement utilisé pour faire de la vidéo surveillance.

Il s'agit d'un projet open-source disponible sur github : https://github.com/lbr38/motion-UI

Présentation
------------

L'interface se présente comme étant très simpliste et **responsive**, ce qui permet une utilisation depuis un **mobile** sans avoir à installer une application. Les gros boutons principaux permettent d'exécuter des actions rapides avec précision sur mobile même lorsque la vision n'est pas optimale (soleil, mouvements...).

Elle permet en outre de mettre en place des **alertes mail** en cas de détection et **d'activer automatiquement** ou non la vidéo-surveillance en fonction d'une plage horaire ou de la présence de périphériques "de confiance" sur le réseau local (smartphone...).

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-1.png">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-1.png" width=25% align="top"> 
        </a>

        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-events.png">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-events.png" width=25% align="top">
        </a>

        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-metrics.png">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-metrics.png" width=25% align="top">
        </a>
    </div>
    <br>
    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-autostart.png">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-autostart.png" width=25% align="top">
        </a>

        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-autostart.png">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-autostart.png" width=25% align="top">
        </a>

        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-4.png">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-4.png" width=25% align="top">
        </a>
    </div>

    <br>


L'interface web se décompose en deux parties :

- La page principale dédiée principalement dédiée à **motion**, permettant de démarrer/stopper le service ou de configurer des alertes en cas de détection. Quelques graphiques permettent de résumer l'activité récente du service et des évènements (events) aillant eu lieu, avec également la possibilité de visualiser les images ou vidéos capturées directement depuis la page web.
- Une page **live** dédiée à la **visualisation en direct** des flux des caméras. Les caméras sont alors disposées en grilles à l'écran (du moins sur un écran PC) un peu à la manière des écrans de vidéo-surveillance d'un établissement par exemple.


Pré-requis
----------

**Motion-UI** doit être installé sur le même hôte/serveur exécutant le service **motion**.

L'installation préconisée est de dédier un serveur uniquement à l'exécution de **motion** et de **motion-UI**, et qu'il soit le point d'entrée unique pour la vidéo-surveillance sur le réseau local : les caméras diffusent leur stream au serveur et c'est le serveur qui analyse les images et détecte d'éventuels mouvements et avertit l'utilisateur. La visualisation des caméras se fait également par le biais du serveur depuis l'interface **motion-UI**. C'est ce cas de figure qui sera détaillé ici.

- Le paquet **motion** doit être installé (version minimale >= **4.4**). Si ce n'est pas le cas, motionui tentera de l'installer dans la bonne version.
- Un serveur web **nginx** avec une version récente de **php-fpm** (PHP 8.1).
- Un certificat SSL pour naviger en https.

Si vous souhaitez pouvoir vous rendre sur **motion-UI** depuis l'extérieur, il faudra également :

- Un nom de domaine avec un **enregistrement DNS** pointant vers l'adresse IP publique de votre box.
- Il faudra mettre en place les redirections de ports qui vont bien depuis l'interface de votre box/routeur, ainsi que **les règles de pare-feu n'autorisant que vous même** à vous connecter à l'interface web **motion-UI**.


Installation de nginx et PHP
----------------------------

Installer **nginx** et **PHP-FPM 8.1**.

L'installation doit se faire en **root** ou avec **sudo**

..  code-block:: shell

    # Debian (il faut au prélable installer un repo de paquets pour PHP8.1, trouvable sur internet)
    apt install nginx php8.1-fpm php8.1-cli php8.1-sqlite3 php8.1-curl

    # RHEL/CentOS (il faut au prélable installer un repo de paquets pour PHP8.1, fourni par Remi Collet)
    yum install nginx php-fpm php-cli php-pdo php-curl

Je ne peux pas détailler la configuration générale de **nginx** et **PHP** mais voici l'exemple de vhost nginx préconisé permettant de servir motion-UI.

Créer un nouveau fichier de vhost dans le répertoire dédié aux vhosts.

Insérer le contenu suivant en adaptant certaines valeurs :

- Le chemin vers le socket unix dédié à PHP, si différent
- Le paramètre SERVER-IP = l'adresse IP du serveur nginx
- Les paramètres SERVERNAME.MYDOMAIN.COM = le nom de domaine dédié à motion-UI
- Les chemins vers le certificat SSL et clé privée associée (PATH-TO-CERTIFICATE.crt et PATH-TO-PRIVATE-KEY.key)

..  code-block:: shell

    # Path to PHP unix socket
    upstream php-handler {
        server unix:/run/php/php8.1-fpm.sock;
    }

    server {
        listen SERVER-IP:80;
        server_name SERVERNAME.MYDOMAIN.COM;

        # Force https
        return 301 https://$server_name$request_uri;

        # Path to log files
        access_log /var/log/nginx/motionui_access.log;
        error_log /var/log/nginx/motionui_error.log;
    }

    server {
        # Set motion-UI web directory location
        set $WWW_DIR '/var/www/motionui'; # default is /var/www/motionui

        listen SERVER-IP:443 ssl;
        server_name SERVERNAME.MYDOMAIN.COM;

        # Path to log files
        access_log /var/log/nginx/motionui_ssl_access.log combined;
        error_log /var/log/nginx/motionui_ssl_error.log;

        # Path to SSL certificate/key files
        ssl_certificate PATH-TO-CERTIFICATE.crt;
        ssl_certificate_key PATH-TO-PRIVATE-KEY.key;

        # Add headers to serve security related headers
        add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
        add_header Referrer-Policy "no-referrer" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Download-Options "noopen" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Permitted-Cross-Domain-Policies "none" always;
        add_header X-Robots-Tag "none" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Remove X-Powered-By, which is an information leak
        fastcgi_hide_header X-Powered-By;

        # Path to motionui root dir
        root $WWW_DIR/public;

        # Enable gzip
        gzip on;
        gzip_vary on;
        gzip_comp_level 4;
        gzip_min_length 256;
        gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
        gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

        location = /robots.txt {
            deny all;
            log_not_found off;
            access_log off;
        }

        location / {
            rewrite ^ /index.php;
        }

        location ~ \.php$ {
            root $WWW_DIR/public;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            fastcgi_param HTTPS on;
            # Avoid sending the security headers twice
            fastcgi_param modHeadersAvailable true;
            fastcgi_pass php-handler;
            fastcgi_intercept_errors on;
            fastcgi_request_buffering off;
        }

        location ~ \.(?:css|js|svg|gif|map|png|html|ttf|ico|jpg|jpeg)$ {
            try_files $uri $uri/ =404;
            access_log off;
        }
    }

Redémarrer **nginx** pour appliquer la configuration.


Installation de motion-UI
-------------------------

L'installation doit se faire en **root** ou avec **sudo**

**Installation sur un système Debian**:

1. Importer la clé publique du repo de **motion-UI** :

..  code-block:: shell

    curl -sS https://packages.bespin.ovh/repo/gpgkeys/packages.bespin.ovh_deb.pub | gpg --dearmor > /etc/apt/trusted.gpg.d/packages.bespin.ovh_deb.gpg

2. Installer le repo de paquets de **motion-UI** :

..  code-block:: shell

    # Pour Debian 11
    echo "deb https://packages.bespin.ovh/repo/motionui/bullseye/main_prod bullseye main" > /etc/apt/sources.list.d/motionui.list

    # Pour Debian 10
    echo "deb https://packages.bespin.ovh/repo/motionui/buster/main_prod buster main" > /etc/apt/sources.list.d/motionui.list

3. Mettre à jour la liste des paquets et installer le paquet **motionui** :

..  code-block:: shell

    apt update
    apt install motionui


**Installation sur un système RHEL**:

1. Installer le repo de paquets de **motion-UI** :

..  code-block:: shell

    echo -e "[motionui]
    name=motionui repo on packages.bespin.ovh
    comment=motionui repo on packages.bespin.ovh
    baseurl=https://packages.bespin.ovh/repo/motionui_prod
    enabled=1
    gpgkey=https://packages.bespin.ovh/repo/gpgkeys/packages.bespin.ovh_rpm.pub
    gpgcheck=1" > /etc/yum.repos.d/motionui.repo

2. Installer le paquet **motionui** :

..  code-block:: shell

    yum install motionui

Si l'installateur a indiqué qu'il fallait redémarrer PHP-FPM, le faire :

..  code-block:: shell

    # Debian
    systemctl restart php8.1-fpm

    # RHEL
    systemctl restart php-fpm


Une fois l'installation terminée, accéder à motion-UI depuis un navigateur à partir du domaine renseigné dans son vhost nginx :

http(s)://SERVERNAME.MYDOMAIN.COM


Puis utiliser les identifiants par défaut pour s'authentifier :

- Login : **admin**
- Mot de passe : **motionui**

Une fois connecté, il est possible de modifier son mot de passe depuis l'espace utilisateur (en haut à droite).


Ajout d'une caméra
------------------

Utiliser le bouton **+** en haut de page pour ajouter une caméra.

- Préciser si la caméra diffuse un **flux video** ou seulement une **image statique** qui nécessite un rechargement (si oui préciser l'intervalle de rafraîchissement en secondes).
- Préciser alors un nom et l'URL vers le **flux video/image** de la caméra
- Choisir ou non de rediffuser le flux video/image sur motion-UI (dans les paramètres généraux on peut ensuite choisir de diffuser ce flux sur la page principale, sur la page **live** ou les deux).
- Choisir d'activer la détection de mouvement (motion) sur cette caméra. Attention si le flux sélectionné est une image statique alors il faudra préciser une seconde URL pointant vers un flux video car motion est incapable de faire de la détection de mouvement sur un flux d'images statiques (il n'est pas capable de recharger automatiquement l'image).
- Préciser un utilisateur / mot de passe si le flux est protégé (beta).

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/camera/add.gif">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/camera/add.gif" align="top"> 
        </a>
    </div> 

    <br>

Une fois la camera ajoutée : 

- motion-UI se charge de créer automatiquement la **configuration motion** pour cette caméra. A noter que la configuration motion créée est relativement minimaliste mais suffisante pour fonctionner dans tous les cas. Il est possible de modifier cette configuration en mode avancé et d'ajouter ses propres paramètres si besoin (voir partie **Configuration d'une caméra**).
- Le flux de la caméra devient visible depuis la page principale, la page **live** (ou les deux) selon la configuration globale choisie.


Configuration d'une caméra
--------------------------

Si le besoin de modifier la configuration d'une caméra se fait sentir, il suffit de cliquer sur le bouton **Configure**.

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/camera/configure.gif">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/camera/configure.gif" align="top"> 
        </a>
    </div> 

    <br>

D'ici il est possible de modifier les paramètres généraux de la caméra (**nom**, **URL**, etc.), de changer la **rotation** de l'image.

Il est également possible de modifier la **configuration motion** de la caméra (détection de mouvement).

Attention, il est préconisé d'**éviter de modifier les paramètres motion en mode avancé**, ou du moins pas sans savoir précisément ce que l'on fait.

Par exemple **il vaut mieux éviter** de modifier les paramètres suivants :

- les paramètres de nom et d'URL (**camera_name**, **netcam_url**, **netcam_userpass** et **rotate**) ont des valeurs issues des paramètres généraux de la caméra. Il convient donc de les modifier directement depuis les champs **Global settings**.
- les paramètres liés aux codecs (**picture_type** et **movie_codec**) ne doivent pas être modifiés sous peine de ne plus pouvoir visualier les captures directement depuis motion-UI. 
- les paramètres d'évènements (**on_event_start**, **on_event_end**, **on_movie_end** et **on_picture_save**) ne doivent pas être modifiés sous peine de ne plus pouvoir enregistrer les évènements de détection de mouvement, et de ne plus recevoir d'alertes.


Tester l'enregistrement des évènements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pour cela depuis l'interface **motion-UI** : démarrer manuellement motion (bouton **Start capture**).

.. raw:: html

    <div align="center">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/start-stop-button.png" align="top"> 
    </div> 

    <br>

Puis **faire un mouvement** devant une caméra pour déclencher un évènement.

Si tout se passe bien, un nouvel évènement en cours devrait apparaitre après quelques secondes dans l'interface **motion-UI**.


Démarrage et arrêt automatique de motion
----------------------------------------

Utiliser le bouton **Enable and configure autostart** pour activer et configurer le démarrage automatique.

.. raw:: html

    <div align="center">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/autostart-button.png" align="top"> 
    </div> 

    <br>

Il est possible de configurer deux types de démarrages et arrêts automatiques de motion :

- En fonction des plages horaires renseignées pour chaque journée. Le service **motion** sera alors actif **entre** la plage d'horaire renseignée.
- En fonction de la présence d'un ou plusieurs appareils IP connecté(s) sur le réseau local. Si aucun des appareils configurés n'est présent sur le réseau local alors le service motion démarrera, considérant que personne n'est présent au domicile. Motion-UI envoi régulièrement un **ping** pour déterminer si l'appareil est présent sur le réseau, il faut donc veiller à configurer des baux d'IP statiques depuis la box pour chaque appareil du domicile (smartphones).

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/autostart-1.png">
        <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/autostart-1.png" width=49% align="top"> 
        </a>

        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/autostart-2.png">
        <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/autostart-2.png" width=49% align="top"> 
        </a>
    </div> 

    <br>


Configurer les alertes
----------------------

Utiliser le bouton **Enable and configure alerts** pour activer et configurer les alertes.

.. raw:: html

    <div align="center">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/alerts-button.png" align="top"> 
    </div> 

    <br>

La configuration des alertes nécessite trois points de configuration :

- Configurer le client mail **mutt** pour qu'il puisse envoyer des alertes depuis l'un de vos comptes mail (gmail, etc...)
- L'enregistrement des évènements doit fonctionner (voir '**Tester l'enregistrement des évènements**')
- Le service **motionui** doit être en cours d'exécution


Configuration de mutt
~~~~~~~~~~~~~~~~~~~~~

- Utiliser le bouton **Generate muttrc config template** pour générer un nouveau fichier de configuration mutt. Ce fichier est créé dans **/var/lib/motionui/.muttrc**.

- Entrer les informations concernant l'adresse mail qui sera émettrice des messages d'alertes ainsi que le mot de passe associé. Utiliser une adresse dédiée ou bien la même adresse qui recevra les mails (et qui s'enverra des alertes à elle même du coup).
- Entrer les informations concernant le serveur SMTP à utiliser. Par défaut le template propose d'utiliser le smtp de **gmail**, ceci est valide uniquement si votre adresse mail émettrice est une adresse gmail. Sinon vous devrez chercher sur Internet les informations concernant le serveur SMTP à utiliser pour votre compte mail :

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/configure-mutt.png">
            <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/configure-mutt.png" width=49% align="top"> 
        </a>
    </div>

    <br>


Configuration des créneaux horaires d'alertes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Renseigner les **créneaux horaires** entre lesquels vous souhaitez **recevoir des alertes** si détection il y a. Pour activer les alertes **toute une journée**, il convient de renseigner 00:00 pour le créneau de début ET de fin.
- Renseigner l'adresse mail destinataire qui recevra les alertes mails. Plusieurs adresses mails peuvent être spécifiées en les séparant par une virgule.

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/alert1.png">
            <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/motionui/alert1.png" width=49% align="top"> 
        </a>
    </div>

    <br>


Tester les alertes
~~~~~~~~~~~~~~~~~~

Une fois que les points précédemment évoqués ont été correctement configurés et que le service **motionui** est bien en cours d'exécution, il est possible de tester l'envoi d'alertes.

Pour cela depuis l'interface **motion-UI** :

- Désactiver temporairement l'autostart de motion si activé, pour éviter qu'il ne stoppe motion au cas où.
- Démarrer manuellement motion (**Start capture**)

Puis **faire un mouvement** devant une caméra pour déclencher une alerte.

Pour tout problème, n'hésitez pas à poser une **question** sur le dépôt du développeur ou à ouvrir une nouvelle **issue** : 

- https://github.com/lbr38/motion-UI/discussions
- https://github.com/lbr38/motion-UI/issues

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
