=====================================================
[Linux] - Video surveillance avec motion et motion-UI
=====================================================

**Motion-UI** est une interface web (User Interface) développée pour gérer plus aisémment le fonctionnement et la configuration de **motion**, un célèbre logiciel **open-source** de détection de mouvement généralement utilisé pour faire de la vidéo surveillance.

Il s'agit d'un projet open-source disponible sur github : https://github.com/lbr38/motion-UI

Présentation
------------

L'interface se présente comme étant très simpliste et **responsive**, ce qui permet une utilisation depuis un **mobile** sans avoir à installer une application. Les gros boutons principaux permettent d'exécuter des actions avec précision sur mobile même lorsque la vision n'est pas optimale (soleil, mouvements...).

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

- Une partie dédiée à **motion**, permettant de démarrer/stopper le service ou de configurer des alertes en cas de détection. Quelques graphiques permettent de résumer l'activité récente du service et des évènements (events) aillant eu lieu, avec également la possibilité de visualiser les images ou vidéos capturées directement depuis la page web.
- Une partie dédiée à la **visualisation en direct** (image par image raffraichie toutes les X secondes) de caméras http sur le réseau local. Les caméras sont alors disposées en grilles à l'écran (du moins sur un écran PC) un peu à la manière des écrans de vidéo-surveillance d'un établissement par exemple.

Pré-requis
----------

**Motion-UI** doit être installé sur le même hôte/serveur exécutant le service **motion**.

L'installation préconisée est de dédier un serveur uniquement à l'exécution de **motion** et de **motion-UI**, et qu'il soit le point d'entrée unique pour la vidéo surveillance sur le réseau local : les caméras diffusent leur stream au serveur et c'est le serveur qui analyse les images et détecte d'éventuels mouvements et avertit l'utilisateur. La visualisation des caméras se fait également par le biais du serveur depuis l'interface **motion-UI**. C'est ce cas de figure qui sera détaillé ici.

- Le paquet **motion** doit être installé (version minimale >= 4.2)
- Un serveur web **nginx** doit être à minima configuré
- Une version récente de **php-fpm** (PHP 8.1 par ex.).
- Quelques dépendances pour motion-UI, pour sa base de données, pour l'envoi de mail de notification et cas de détection et afin qu'il puisse récupérer ses dernières mises à jour depuis github

Installer les dépendances :

..  code-block:: shell

    apt/yum install motion sqlite3 mutt curl

Installer nginx et PHP si ce n'est pas déjà fait :

..  code-block:: shell

    # Debian (il faut au prélable installer un repo de paquets pour PHP8.1, trouvable sur internet)
    apt install nginx php8.1-fpm php8.1-cli php8.1-sqlite3 php8.1-curl

    # RHEL/CentOS (il faut au prélable installer un repo de paquets pour PHP8.1, fourni par Remi Collet)
    yum install nginx php-fpm php-cli php-pdo php-curl

Si vous souhaitez pouvoir vous rendre sur **motion-UI** depuis l'extérieur, il faudra également :

- Un nom de domaine avec un **enregistrement DNS** pointant vers l'adresse IP publique de votre box.
- Il faudra mettre en place les redirections de ports qui vont bien depuis l'interface de votre box/routeur, ainsi que **les règles de pare-feu n'autorisant que vous même** à vous connecter à l'interface web **motion-UI**.

Installation
------------

Installer le paquet **git** si ce n'est pas déjà fait :

..  code-block:: shell

    apt/yum install git

Cloner le projet **motion-UI** :

..  code-block:: shell

    git clone https://github.com/lbr38/motion-UI.git

Exécuter le script d'installation et se laisser guider. Le script nécessite des droits sudo car il devra être en mesure de créer le répertoire où seront stockées les sources web (par défaut **/var/www/motionui**), de créer le répertoire où seront stockées les données (**/var/lib/motionui**) ainsi que de créer un service systemd 'motionui' :

..  code-block:: shell

    cd motion-UI
    sudo ./motionui --install

Une fois l'installation terminée, il ne reste plus qu'à mettre en place un vhost qui diffusera l'interface web de motion-UI.

Vhost nginx
-----------

Je ne peux pas détailler la configuration générale de **nginx** et **PHP** mais voici l'exemple de vhost nginx préconisé permettant de servir motion-UI.

Créer un nouveau fichier de vhost dans le répertoire dédié.

Insérer le contenu suivant en adaptant certaines valeurs :

- Le chemin vers le socket unix dédié à PHP
- La valeur de la variable $WWW_DIR = indiquer le répertoire racine où vous avez choisi de stocker les sources web de motion-UI (notamment demandé lors de l'installation avec le script d'installation)
- Le paramètre SERVER-IP = l'adresse IP du serveur nginx
- Les paramètres SERVERNAME.MYDOMAIN.COM = le nom de domaine dédié à motion-UI
- Les chemins vers le certificat SSL et clé privée associée

..  code-block:: shell

    # Path to unix socket
    upstream php-handler {
        server unix:/var/run/php-fpm/php-fpm.sock;
    }

    server {
        listen SERVER-IP:80;
        server_name SERVERNAME.MYDOMAIN.COM;

        # Force https
        return 301 https://$server_name$request_uri;

        # Path to log files
        access_log /var/log/nginx/SERVERNAME.MYDOMAIN.COM_access.log;
        error_log /var/log/nginx/SERVERNAME.MYDOMAIN.COM_error.log;
    }

    server {
        # Set motion-UI web directory location
        set $WWW_DIR '/var/www/motionui'; # default is /var/www/motionui

        listen SERVER-IP:443 ssl;
        server_name SERVERNAME.MYDOMAIN.COM;

        # Path to log files
        access_log /var/log/nginx/SERVERNAME.MYDOMAIN.COM_ssl_access.log combined;
        error_log /var/log/nginx/SERVERNAME.MYDOMAIN.COM_ssl_error.log;

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
            index index.php;
        }

        location ~ \.php$ {
            root $WWW_DIR/public;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            #include fastcgi.conf;
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

Redémarrer **nginx** pour appliquer la configuration et se rendre sur motion-UI **depuis un navigateur web** en se connectant avec les identifiants par défaut :

- Login : **admin**
- Mot de passe : **motionui**

Il est possible de modifier son mot de passe depuis l'espace utilisateur (en haut à droite).

Si un message indique que le service motionui n'est pas démarré, le démarrer depuis le terminal :

..  code-block:: shell

    sudo systemctl start motionui


Configuration de motion
-----------------------

La version minimale du paquet motion doit être **>= 4.2**. Sans quoi certaines fonctionnalités de **motion-UI** seront indisponibles.

La configuration générale de **motion** est propre à chacun et à chaque utilisation. Par défaut motion met à disposition plusieurs fichiers de configuration :

- **motion.conf** qui est le fichier de configuration principal
- **des fichiers de configuration supplémentaires**, 1 pour chaque caméra.

La bonne pratique étant d'utiliser **motion.conf** pour la configuration générale et d'utiliser les fichiers de configuration supplémentaires pour configurer individuellement chaque caméra. Puis de faire prendre en compte ces fichiers de configuration par le fichier principal (voir tout en bas de motion.conf pour inclure des fichiers supplémentaires).

Pour chaque caméra :

- veiller à préciser à minima un Id de caméra (camera_id).
- veiller si possible et si la version de motion le prend en charge, de préciser un nom de caméra (camera_name).
- veiller à préciser un répertoire de destination pour l'enregistrement des images et/ou vidéos et que celui-ci soit accessible en lecture et écriture au groupe **motionui**.

Voir la documentation de motion pour plus d'informations sur chaque paramètre : https://motion-project.github.io/motion_config.html#Configuration_OptionsAlpha


Paramétrer l'enregistrement des évènements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pré-requis :

- La version minimale du paquet motion doit être **>= 4.2**.
- Le paramètre **camera_id** doit être configuré pour chaque caméra.

Motion propose plusieurs déclencheurs permettant d'exécuter une commande lorsqu'ils sont invoqués :

- on_event_start = lorsqu'un nouvel évènement démarre 
- on_event_end = lorsqu'un évènement prend fin
- on_motion_detected = lorsqu'un mouvement est détecté
- on_movie_start = lorsqu'un nouveau fichier vidéo vient d'être généré suite à une détection
- on_movie_end = lorsqu'un fichier vidéo a terminé sa génération suite à une détection
- on_picture_save = lorsqu'une image a été générée suite à une détection

**motion-UI** propose de paramétrer automatiquement l'enregistrement des évènements (on_event_start) en base de données lorsqu'une nouvelle détection a lieu. Ces évènements deviennent alors visibles depuis l'interface **motion-UI** avec les images et vidéos associées.

L'enregistrement des évènement est également nécessaire pour la réception d'alertes mail (on reçoit une alerte lorsqu'un nouvel évènement a lieu).

Pour chaque fichier de caméra, utiliser le bouton **Set up event registering** pour paramétrer automatiquement l'enregistrement d'évènements :

.. raw:: html
    
    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/motion-UI-setup-event.png">
        <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/motion-UI-setup-event.png" width=49% align="top"> 
        </a>
    </div>

    <br>

Ceci aura pour effet de configurer les 3 paramètres de motion suivants :

- on_event_start
- on_event_end
- on_movie_end


Tester l'enregistrement des évènements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pour cela depuis l'interface **motion-UI** :

- Démarrer manuellement motion (gros bouton power 'Start capture')

Depuis un terminal sur le serveur exécutant motion-UI, vérifier en continu l'état du service motionui pour s'assurer qu'il ne remonte pas de message d'erreur : 

..  code-block:: shell

    watch systemctl status motionui

Puis **faire un mouvement** devant une caméra pour déclencher un évènement.

Si tout se passe bien, un nouvel évènement en cours devrait apparaitre dans l'interface **motion-UI**.


Démarrage et arrêt automatique de motion
----------------------------------------

Il est possible de configurer deux types de démarrages et arrêts automatiques de motion :

- En fonction des plages horaires renseignées pour chaque journée. Le service **motion** sera alors actif **entre** la plage d'horaire renseignée.
- En fonction de la présence d'un ou plusieurs appareils IP connecté(s) sur le réseau local. Si aucun des appareils configurés n'est présent sur le réseau local alors le service motion démarrera, considérant que personne n'est présent au domicile. Motion-UI envoi régulièrement un **ping** pour déterminer si l'appareil est présent sur le réseau, il faut donc veiller à configurer des baux d'IP statiques depuis la box pour chaque appareil du domicile (smartphones).

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/autostart-1.png">
        <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/autostart-1.png" width=49% align="top"> 
        </a>

        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/autostart-2.png">
        <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/autostart-2.png" width=49% align="top"> 
        </a>
    </div> 

    <br>


Configurer les alertes
----------------------

La configuration des alertes nécessite trois points de configuration :

- Configurer le client mail **mutt** pour qu'il puisse envoyer des alertes depuis l'un de vos comptes mail (gmail, etc...)
- L'enregistrement des évènements doit être paramétré (voir Paramétrer l'enregistrement des évènements)
- Le service **motionui** doit être en cours d'exécution


Configuration de mutt
~~~~~~~~~~~~~~~~~~~~~

- Utiliser le bouton **Generate muttrc config template** pour générer un nouveau fichier de configuration mutt. Ce fichier est créé dans **/var/lib/motionui/.muttrc**.

- Entrer les informations concernant l'adresse mail qui sera émettrice des messages d'alertes ainsi que le mot de passe associé. Utiliser une adresse dédiée ou bien la même adresse qui recevra les mails (et qui s'enverra des alertes à elle même du coup).
- Entrer les informations concernant le serveur SMTP à utiliser. Par défaut le template propose d'utiliser le smtp de **gmail**, ceci est valide uniquement si votre adresse mail émettrice est une adresse gmail. Sinon vous devrez chercher sur Internet les informations concernant le serveur SMTP à utiliser pour votre compte mail :

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/configure-mutt.png">
            <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/configure-mutt.png" width=49% align="top"> 
        </a>
    </div>

    <br>


Configuration des créneaux d'alertes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Renseigner les **créneaux horaires** entre lesquels vous souhaitez **recevoir des alertes** si détection il y a. Pour activer les alertes **toute une journée**, il convient de renseigner 00:00 pour le créneau de début ET de fin.
- Renseigner l'adresse mail destinataire qui recevra les alertes mails. Plusieurs adresses mails peuvent être spécifiées en les séparant par une virgule.

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/alert1.png">
            <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/motionui/alert1.png" width=49% align="top"> 
        </a>
    </div>

    <br>


Tester les alertes
~~~~~~~~~~~~~~~~~~

Une fois que les points précédemment évoqués ont été correctement configurés et que le service motionui est bien en cours d'exécution, il est possible de tester l'envoi d'alertes.

Pour cela depuis l'interface **motion-UI** :

- S'assurer d'avoir activé les alertes (le gros bouton avec la cloche doit être rouge)
- Désactiver provisoirement l'autostart de motion si activé
- Démarrer manuellement motion (gros bouton power 'Start capture')

Depuis un terminal sur le serveur exécutant motion-UI, vérifier en continu l'état du service motionui pour s'assurer qu'il ne remonte pas de message d'erreur : 

..  code-block:: shell

    watch -n1 systemctl status motionui

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
