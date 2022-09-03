<h1>Video surveillance avec motion et motion-UI</h1>

<b>Motion-UI</b> est une interface web (User Interface) développée pour gérer plus aisémment le fonctionnement et la configuration de <b>motion</b>, un célèbre logiciel <b>open-source</b> de détection de mouvement généralement utilisé pour faire de la vidéo surveillance.

Il s'agit d'un projet open-source disponible sur github : <a href="https://github.com/lbr38/motion-UI">github de motion-UI</a>

<h2>Présentation</h2>

L'interface se présente comme étant très simpliste et <b>responsive</b>, ce qui permet une utilisation depuis un <b>mobile</b> sans avoir à installer une application. Les gros boutons principaux permettent d'exécuter des actions avec précision sur mobile même lorsque la vision n'est pas optimale (soleil, mouvements...).

<div align="center">
<a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-1.png">
<img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-1.png" width=24% align="top"> 
</a>

<a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-3.png">
<img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-3.png" width=24% align="top">
</a>

<a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-2.png">
<img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-2.png" width=24% align="top">
</a>

<a href="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-4.png">
<img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/motion-UI-4.png" width=24% align="top">
</a>
</div>

<br>

L'interface web se décompose en deux parties : 

- Une partie dédiée à <b>motion</b>, permettant de démarrer/stopper le service ou de configurer des alertes en cas de détection. Quelques graphiques permettent de résumer l'activité récente du service et des évènements (events) aillant eu lieu, avec également la possibilité de visualiser les images ou vidéos capturées directement depuis la page web.
- Une partie dédiée à la <b>visualisation en direct</b> (image par image raffraichie toutes les X secondes) de caméras http sur le réseau local. Les caméras sont alors disposées en grilles à l'écran (du moins sur un écran PC) un peu à la manière des écrans de vidéo-surveillance d'un établissement par exemple.

<h2>Pré-requis</h2>

<b>Motion-UI</b> doit être installé sur le même hôte/serveur exécutant le service <b>motion</b>.

L'installation préconisée est de dédier un serveur uniquement à l'exécution de <b>motion</b> et de <b>motion-UI</b>, et qu'il soit le point d'entrée unique pour la vidéo surveillance sur le réseau local : les caméras diffusent leur stream au serveur et c'est le serveur qui analyse les images et détecte d'éventuels mouvements et avertit l'utilisateur. La visualisation des caméras se fait également par le biais du serveur depuis l'interface <b>motion-UI</b>. C'est ce cas de figure qui sera détaillé ici.

- Le paquet <b>motion</b> doit être installé
- Un serveur web <b>nginx</b> doit être à minima configuré
- Une version récente de <b>php-fpm</b> (PHP 8.1 par ex.).
- Quelques dépendances pour motion-UI, pour sa base de données, pour l'envoi de mail de notification et cas de détection et afin qu'il puisse récupérer ses dernières mises à jour depuis github

Installer les dépendances :

```
apt/yum install motion sqlite3 mutt wget curl git
```

Si vous souhaitez pouvoir vous rendre sur <b>motion-UI</b> depuis l'extérieur, il faudra également :

Un nom de domaine avec un <b>enregistrement DNS</b> pointant vers l'adresse IP publique de votre box. Il faudra mettre en place les redirections de ports qui vont bien depuis l'interface de votre box/routeur, ainsi que <b>les règles de pare-feu n'autorisant que vous même</b> à vous connecter à l'interface web <b>motion-UI</b>.

<h2>Installation</h2>

Installer le paquet <b>git</b> si ce n'est pas déjà fait :

```
apt/yum install git
```

Cloner le projet <b>motion-UI</b> :

```
git clone https://github.com/lbr38/motion-UI.git
```

Exécuter le script d'installation et se laisser guider. Le script nécessite des droits sudo car il devra être en mesure de créer le répertoire où seront stockées les sources web (par défaut <b>/var/www/motionui</b>), de créer le répertoire où seront stockées les données (<b>/var/lib/motionui</b>) ainsi que de créer un service systemd 'motionui' :

```
cd motion-UI
sudo ./motionui --install
```

Une fois l'installation terminée, il ne reste plus qu'à mettre en place un vhost qui diffusera l'interface web de motion-UI.

<h2>Vhost nginx</h2>

Créer un nouveau fichier de vhost dans le répertoire dédié. Insérer le contenu suivant en adaptant certaines valeurs :

- Le chemin vers le socket unix dédié à PHP
- La valeur de la variable $WWW_DIR = indiquer le répertoire racine où vous avez choisi de stocker les sources web de motion-UI (notamment demandé lors de l'installation avec le script d'installation)
- Le paramètre SERVER-IP = l'adresse IP du serveur nginx
- Les paramètres SERVERNAME.MYDOMAIN.COM = le nom de domaine dédié à motion-UI
- Les chemins vers le certificat SSL et clé privée associée

```
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

    # Motion-UI does not have any login page for the moment. You can use a .htpasswd file to set up basic authentication.
    # Uncomment the lines below and generate a .htpasswd file:
    # auth_basic "You must login";
    # auth_basic_user_file /var/www/.htpasswd;

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

```

Redémarrer <b>nginx</b> pour appliquer la configuration et se rendre sur motion-UI <b>depuis un navigateur web</b>.

Si un message indique que le service motionui n'est pas démarré, le démarrer depuis le terminal :

```
sudo systemctl start motionui
```

<h2>Démarrage et arrêt automatique</h2>

Il est possible de configurer deux types de démarrages et arrêts automatiques :

- En fonction des plages horaires renseignées pour chaque journée. Le service <b>motion</b> sera alors <b>actif</b> entre la plage d'horaire renseignée.
- En fonction de la présence d'un ou plusieurs appareils IP connecté(s) sur le réseau local. Si aucun des appareils configurés n'est présent sur le réseau local alors le service motion démarrera, considérant que personne n'est présent au domicile. Motion-UI envoi régulièrement un <b>ping</b> pour déterminer si l'appareil est présent sur le réseau, il faut donc veiller à configurer des baux d'IP statiques depuis la box pour chaque appareil du domicile (smartphones).

<div align="center">
<a href="https://raw.githubusercontent.com/lbr38/documentation/main/images/motionui/autostart-1.png">
<img src="https://raw.githubusercontent.com/lbr38/documentation/main/images/motionui/autostart-1.png" width=49% align="top"> 
</a>

<a href="https://raw.githubusercontent.com/lbr38/documentation/main/images/motionui/autostart-2.png">
<img src="https://raw.githubusercontent.com/lbr38/documentation/main/images/motionui/autostart-2.png" width=49% align="top"> 
</a>
</div> 

<br>

<h2>Configurer les alertes</h2>

La configuration des alertes nécessite trois points de configuration :

- Configurer le client mail <b>mutt</b> pour qu'il puisse envoyer des alertes depuis l'un de vos comptes mail (gmail, etc...)
- Configurer motion pour qu'il envoie une ou plusieurs alertes selon les <b>déclencheurs</b> désirés
- Le service <b>motionui</b> doit être démarré

<h3>Configuration de mutt</h3>

Depuis un terminal sur le serveur exécutant motion-UI, créer un nouveau fichier <b>.muttrc</b>. Ce fichier devra être accessible en lecture par l'utilisateur <b>motion</b> :

```
vim /var/lib/motionui/.muttrc
```

Insérer la configuration suivante, ici un exemple pour un compte mail @riseup.net :

```
# Nom de l'expéditeur du message
set realname = "motion-UI"

# Activer TLS si disponible sur le serveur
set ssl_starttls=yes
# Toujours utiliser SSL lors de la connexion à un serveur
set ssl_force_tls=yes

# Configuration SMTP
set smtp_url = "smtps://ACCOUNT@riseup.net@mail.riseup.net:465/"
set smtp_pass = "ACCOUNT_PASSWORD"
set from = "ACCOUNT@riseup.net"
set use_envelope_from=yes

# Paramètres locaux, date 
set date_format="%A %d %b %Y à %H:%M:%S (%Z)"

# Ne pas conserver une copie des mails envoyés
set copy=no
```

```
chown motion:motionui /var/lib/motionui/.muttrc
```

Depuis l'interface motion-UI :

- Renseigner les <b>créneaux horaires</b> entre lesquels vous souhaitez <b>recevoir des alertes</b> si détection il y a. Pour activer les alertes <b>toute une journée</b>, renseigner 00:00 pour le créneau de début ET de fin (comme sur la capture).
- Renseigner le chemin vers le <b>fichier de configuration mutt</b>, ainsi que l'adresse mail destinataire qui recevra les alertes mails. Plusieurs adresses mails peuvent être spécifiées en les séparant par une virgule.

<a href="https://raw.githubusercontent.com/lbr38/documentation/main/images/motionui/alert1.png">
<img src="https://raw.githubusercontent.com/lbr38/documentation/main/images/motionui/alert1.png" width=49% align="top"> 
</a>
</div>

<h3>Configuration de motion</h3>

<p>Motion propose plusieurs déclencheurs permettant d'exécuter une commande lorsqu'ils sont invoqués. Les paramètres proposé par motion sont les suivants :</p>

- on_event_start = lorsqu'un nouvel évènement démarre 
- on_event_end = lorsqu'un évènement prend fin
- on_motion_detected = lorsqu'un mouvement est détecté
- on_movie_start = lorsqu'un nouveau fichier vidéo vient d'être généré suite à une détection
- on_movie_end = lorsqu'un fichier vidéo a terminé sa génération suite à une détection
- on_picture_save = lorsqu'une image a été générée suite à une détection

Depuis l'interface <b>motion-UI</b>, il est possible d'éditer la configuration de motion et donc de modifier ces déclencheurs. Il est conseiller d'utiliser et de configurer les déclencheurs suivants :

<b>Lorsqu'un nouvel évènement démarre</b>

```
on_event_start /var/lib/motionui/tools/event --cam-id %t --cam-name %$ --register-event %v
```

La commande fait appel au script <b>event</b> qui va se charger d'enregistrer le nouvel évènement, ce qui permettra de le faire remonter dans l'interface web de motion-UI. 




