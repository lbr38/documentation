==============================================================================
[Raspberry Pi] - Vidéo-surveillance avancée avec ustreamer et motion/motion-UI
==============================================================================

EN version : https://en.linuxdocs.net/en/latest/guides/raspberrypi-videosurveillance.html

Introduction
============

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/raspberrypi/motion/cctv.jpg

La vidéo-surveillance est l'un des projets phare sur **Raspberry Pi** et cela depuis les débuts de la carte.
Des outils comme **motion** et des OS dédiés comme **motionPie** sont vite devenus une référence pour ce type d’utilisation.

Pourtant, les limitations se font vite ressentir malgré les évolutions de la carte. L'encodage vidéo étant un processus gourmand en CPU et en RAM, le Raspberry Pi se retrouve rapidement surchargé lorsqu'on commence à mettre en place un système de vidéo-surveillance avancé avec plusieurs caméras.

Solution
========

La solution proposée ici tente de répondre à la problématique de surcharge en séparant les tâches :

- Les **cartes ARM** (Raspberry Pi ou autre) s'occupent de capturer et renvoyer le flux vidéo généré par la caméra qui y est rattachée.
- Un **serveur central** recéptionne les flux et prend en charge les tâches lourdes de **détection de mouvement** et **d'encodage** (motion)

Les "**cartes-caméras**" sont placées aux différents endroits à surveiller de l'habitation, en intérieur ou extérieur, en prenant les précautions nécéssaires pour les protéger de l'humidité le cas échéant.

Le **serveur** est stocké au chaud à l'intérieur et sa sécurité sera renforcée puisqu'il s'agira du point central par lequel l'utilisateur consultera tous les évènements ayant eu lieu et visualisera le stream en direct.

Le tout étant relié au réseau local de manière **filaire**. On excluera les caméras WI-FI ici puisque celles-ci peuvent être facilement **neutralisées** avec un simple brouilleur Wi-FI.

Tout ceci illustré :

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/raspberrypi/motion/motion.png

Pré-requis
==========

- **1 ou plusieurs Raspberry Pi** (ou carte concurrente) pour la partie "caméra".

Leur puissance peut être faible à modérée puisque leur rôle est seulement de diffuser du flux sans traitement. 

Personnellement j'utilise des `Orange-Pi zero LTS <https://orangepi.com/index.php?route=product/product&product_id=846>`_ (4CPU, 512Mo de RAM, 1 port Ethernet et 1 port USB).
Leur **petite taille** permet de les faufiler partout et leur port **POE** peut permettre de n'utiliser qu'un câble Ethernet pour les alimenter.
Pour la caméra, j'utilise un dôme USB étanche acheté sur `amazon <https://www.amazon.fr/dp/B01JG43TD0/ref=dp_prsubs_1>`_ relié à l'unique port USB de l'Orange-Pi.

- **1 serveur central**

De préférence un serveur "maison", **le plus puissant possible**. Il est préférable d'éviter d'utiliser une carte ARM qui risque de se retrouver rapidement surchargée lors des traitements vidéos. Il faut prendre en compte que plus il y aura de caméras et plus le traitement sera lourd.
Le serveur devra faire tourner un OS tel que Debian, CentOS...

Préparer chaque élément :

- Installer les OS nécéssaires (raspbian ou armbian par exemple) sur chaque cartes ARM et sur le serveur central (Debian 11 par exemple)
- Mettre à jour les paquets systèmes et firmware si besoin
- Configurer des adresses IP fixes sur les cartes et le serveur


Configuration
=============

Pour la suite de l'article :

- J'estimerai que l'installation côté surveillance s'effectue sur carte **Raspberry Pi** (car c'est la plus répandue) reliée à une **caméra ou dôme USB**, et tournera sur un OS de type Debian (Armbian/Raspbian).
- J'estimerai que le serveur tournera sur un OS de type Debian.

En outre le gestionnaire de paquets utilisé sera **apt** et les noms de paquets installés seront relatifs aux systèmes Debian (leur noms peuvent varier si vous décider d'utiliser un autre OS).

Configuration des caméras
-------------------------

Le but ici est de mettre en place `ustreamer <https://github.com/pikvm/ustreamer>`_ pour diffuser le flux des caméras en **http**.

L’avantage de **ustreamer** par rapport au bien connu **mjpg-streamer** est qu’il possède plus d’options, il est plus clair et plus simple à utiliser et surtout il génère nativement plus de logs que mjpg-streamer, ce qui facilite grandement le debuggage.

Répeter cette partie "**Configuration des caméras**" pour chaque Raspberry Pi relié à une caméra USB.

Note : L’ensemble des configurations s’effectuent en **root**.

Ustreamer
+++++++++

Se connecter au **Raspberry Pi** en **ssh** et installer quelques paquets nécéssaires à la compilation :

..  code-block:: shell
    
    apt install git make gcc build-essential

Lancer l'installation de **ustreamer** en le compilant (c'est facile), il faut au préalable installer quelques librairies supplémentaires :

..  code-block:: shell

    # Si Raspbian (Raspberry Pi OS) :
    apt install libevent-dev libjpeg8-dev libbsd-dev

    # Si autre, voir : https://github.com/pikvm/ustreamer#building

    # Puis cloner le projet ustreamer :
    cd /home/pi/
    git clone --depth=1 https://github.com/pikvm/ustreamer

    # Et compiler :
    cd ustreamer
    make

Vérifier avec **lsusb** que la caméra USB branchée est bien reconnue par le système, dans mon cas avec le dôme USB ça affiche ceci : 

..  code-block:: shell

    lsusb
    Bus 001 Device 008: ID 05a3:9230 ARC International Camera      # Caméra USB
    Bus 001 Device 009: ID 0424:7800 Standard Microsystems Corp. 
    Bus 001 Device 007: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
    Bus 001 Device 006: ID 0424:2514 Standard Microsystems Corp. USB 2.0 Hub
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

Créer les scripts de démarrage et d’arrêt du stream, c’est l'utilisateur **pi** qui exécutera ces scripts :

..  code-block:: shell
    
    mkdir -p /home/pi/scripts/stream

Script de démarrage du stream :

..  code-block:: shell

    vim /home/pi/scripts/stream/start-stream.sh

Insérer le contenu suivant :

..  code-block:: shell

    #!/bin/bash
  
    DATE=$(date +%Y-%m-%d)
    TIME=$(date +%Hh%M)
    RESOLUTION="1920x1080"
    FRAMERATE="25"
    USTREAMER="/home/pi/ustreamer/ustreamer"
    LOG="/home/pi/scripts/stream/ustreamer.log"


    function help()
    {
        echo "Usage: $0 [options]"
        echo "Options:"
        echo "  --1080p"
        echo "  --720p"
        echo "  --low"
        echo "  --fps=FRAMERATE"
        echo "  --help"
    }

    while [ $# -ge 1 ];do
        case "$1" in
            --1080p)
                RESOLUTION="1920x1080"
            ;;
            --720p)
                RESOLUTION="1280x720"
            ;;
            --low)
                RESOLUTION="640x480"
            ;;
            --fps)
                FRAMERATE="$2"
                shift
            ;;
            --help)
                help
                exit
            ;;
            *)
        esac
        shift
    done

    # Cleaning log file
    echo -n> "$LOG"
    exec &> >(tee -a "$LOG")

    echo "$DATE - $TIME - Starting stream" 

    "$USTREAMER" --device=/dev/video0 --slowdown --workers 2 -e 30 -K 0 -r "$RESOLUTION" -m MJPEG --host 0.0.0.0 --port 8888 --device-timeout 2 --device-error-delay 1 2>&1 &

    exit

Script d'arrêt du stream :

..  code-block:: shell

    vim /home/pi/scripts/stream/stop-stream.sh

Insérer le contenu suivant :

..  code-block:: shell

    #!/bin/bash

    # Search for the process ID of ustreamer
    PID="$(/bin/ps -aux | /bin/grep 'ustreamer' | egrep -v 'grep|ustreamer.log' | /usr/bin/awk '{print $2}')"

    if [ -z "$PID" ];then
        echo "No active process found"
        exit
    fi

    echo "Stopping ustreamer... "
    kill "$PID" > /dev/null 2>&1
    sleep 1

    # Check if the process is still running
    if /bin/ps -aux | /bin/grep 'ustreamer' | egrep -v 'grep|ustreamer.log';then
        echo "Process is still running, killing it"
        kill -9 "$PID"
        exit
    fi

    echo "OK"

    exit

Ajuster les permissions sur ce qui vient d'être créé :

..  code-block:: shell

    chmod 700 /home/pi/scripts/stream/*.sh 
    chown -R pi:pi /home/pi/scripts

Se loguer temporairement en tant que **pi** et démarrer le stream pour tester. Il est possible de préciser une résolution et un framerate en paramètre du script de démarrage. Par défaut, le stream est lancé en **1920x1080** et à **25 fps** :

..  code-block:: shell

    su pi
    /home/pi/scripts/stream/start-stream.sh &

    # Exemple pour démarrer le stream en 720p et à 30 fps :
    /home/pi/scripts/stream/start-stream.sh --720p --fps 30 &

Ça devrait afficher quelques logs à l’écran.

Ouvrir http://ADRESSE_IP_CAMERA:8888 dans un navigateur, la page d'accueil de ustreamer doit être accessible et le **stream** est visualisable en cliquant sur **/stream**.

Toujours en tant que **pi** créer une tâche cron qui démarrera le stream automatiquement après un reboot du Raspberry Pi :

..  code-block:: shell

    crontab -e

    @reboot /home/pi/scripts/start-camera.sh &


Configuration du serveur
------------------------

Le but ici est de mettre en place **motion-UI** (interface web) pour analyser le flux des caméras disposées dans l'habitation et détecter des mouvements.

Notes :

- Le système utilisé ici est Debian 11
- L’ensemble des configurations s’effectuent en **root**.

motion-UI
+++++++++

Présentation
~~~~~~~~~~~~

**Motion-UI** est une interface web (User Interface) développée pour gérer plus aisémment le fonctionnement et la configuration de **motion**.

Il s'agit d'un projet open-source disponible sur github : https://github.com/lbr38/motion-UI

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


Installation de docker
~~~~~~~~~~~~~~~~~~~~~~

Commencer par installer le repo de paquets pour **docker** :

..  code-block:: shell

    apt install ca-certificates curl gnupg -y
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    echo \ 
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

Puis installer **docker** :

..  code-block:: shell

    apt update -y
    apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin -y


Installation de motion-UI
~~~~~~~~~~~~~~~~~~~~~~~~~

L'installation doit se faire avec un utilisateur lambda (non root).

Installer la dernière image disponible en adaptant la valeur de ``FQDN`` par votre nom de domaine dédié à motion-UI :

..  code-block:: shell

    docker run -d --restart always --name motionui \
       -e FQDN=motionui.example.com \
       -p 8080:8080 \
       -v /etc/localtime:/etc/localtime:ro \
       -v /var/lib/docker/volumes/motionui-data:/var/lib/motionui \
       -v /var/lib/docker/volumes/motionui-captures:/var/lib/motion \
       lbr38/motionui:latest

Deux volumes persistants sont alors créés sur le système hôte :

- **motionui_data** ``/var/lib/docker/volumes/motionui-data/`` : contient la base de données de motion-UI
- **motionui-captures** ``/var/lib/docker/volumes/motionui-captures/`` : contient les captures d'images et vidéos réalisées par motion (à conserver donc!)

Une fois l'installation terminée, motion-UI est accessible directement (de manière non sécurisée car sans certificat pour le moment) depuis http://<IP_SERVEUR>:8080

Utiliser les identifiants par défaut pour s'authentifier :

- Login : **admin**
- Mot de passe : **motionui**

Une fois connecté, il est possible de modifier son mot de passe depuis l'espace utilisateur (en haut à droite).

Poursuivre par la mise en place d'un reverse-proxy pour accéder à motion-UI par un nom de domaine dédié avec certificat SSL.


Reverse-proxy nginx
~~~~~~~~~~~~~~~~~~~

Installer nginx :

..  code-block:: shell

    apt install nginx -y

Supprimer le vhost par défaut :

..  code-block:: shell

    rm /etc/nginx/sites-enabled/default

Puis créer un nouveau vhost dédié à **motion-UI** :

..  code-block:: shell

    vim /etc/nginx/sites-available/motionui.conf

Puis insérer le contenu suivant en adaptant certaines valeurs :

- Le paramètre <SERVER-IP> = l’adresse IP du serveur
- Les paramètres <FQDN> = le nom de domaine dédié à motion-UI
- Les chemins vers le certificat SSL et la clé privée associée (<PATH-TO-CERTIFICATE> et <PATH-TO-PRIVATE-KEY>)

..  code-block:: shell

    upstream motionui_docker {
        server 127.0.0.1:8080;
    }

    # Disable some logging
    map $request_uri $loggable {
        /ajax/controller.php 0;
        default 1;
    }

    server {
        listen <SERVER-IP>:80;
        server_name <FQDN>;

        access_log /var/log/nginx/<FQDN>_access.log combined if=$loggable;
        error_log /var/log/nginx/<FQDN>_error.log;

        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen <SERVER-IP>:443 ssl;
        server_name <FQDN>;

        # Path to SSL certificate/key files
        ssl_certificate <PATH_TO_CERTIFICATE>;
        ssl_certificate_key <PATH_TO_PRIVATE_KEY>;

        # Path to log files
        access_log /var/log/nginx/<FQDN>_ssl_access.log combined if=$loggable;
        error_log /var/log/nginx/<FQDN>_ssl_error.log;
    
        # Security headers
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
    
        location / {
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_pass http://motionui_docker;
        }
    }

Créer un lien symbolique pour activer le vhost :

..  code-block:: shell

    ln -s /etc/nginx/sites-available/motionui.conf /etc/nginx/sites-enabled/motionui.conf

Redémarrer nginx pour appliquer :

..  code-block:: shell

    nginx -t && systemctl restart nginx

motion-UI est alors accessible depuis https://<FQDN>


Ajout d'une caméra
~~~~~~~~~~~~~~~~~~

Utiliser le bouton **+** pour ajouter une caméra.

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
~~~~~~~~~~~~~~~~~~~~~~~~~~

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
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
~~~~~~~~~~~~~~~~~~~~~~

Utiliser le bouton **Enable and configure alerts** pour activer et configurer les alertes.

.. raw:: html

    <div align="center">
        <img src="https://raw.githubusercontent.com/lbr38/resources/main/screenshots/motionui/documentation/alerts-button.png" align="top"> 
    </div> 

    <br>

La configuration des alertes nécessite deux points de configuration :

- Un enregistrement **SPF** pour le nom de domaine dédié à motion-UI.
- L'enregistrement des évènements doit fonctionner (voir '**Tester l'enregistrement des évènements**')


Configuration des créneaux horaires d'alertes
*********************************************

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
******************

Une fois que les points précédemment évoqués ont été correctement configurés et que le service **motionui** est bien en cours d'exécution, il est possible de tester l'envoi d'alertes.

Pour cela depuis l'interface **motion-UI** :

- Désactiver temporairement l'autostart de motion si activé, pour éviter qu'il ne stoppe motion au cas où.
- Démarrer manuellement motion (**Start capture**)

Puis **faire un mouvement** devant une caméra pour déclencher une alerte.


Sécurité
========

Maintenant que le système de vidéo-surveillance est fonctionnel il est temps de **sécuriser** l'ensemble.

Je ne peux détailler toutes les configurations de sécurité à mettre en place mais voici quelques idées de base :

- Les flux diffusés par les caméras **ne doivent être accessibles que par le serveur**.

En d'autres termes les URLs d'accès à ustreamer http://ADRESSE_IP_CAMERA:8888 ne doivent être accessibles que par le serveur.

Pour cela mettre en place des règles de **pare-feu** (iptables par exemple) sur les Raspberry Pi pour n'autoriser que le serveur à y accéder en http.

- La configuration SSH des caméras doit être **renforcée** (par clé, utilisateur root non autorisé, ...)

Avec si possible des règles de pare-feu n'autorisant que le serveur et éventuellement une autre IP du réseau local (de secours) à s'y connecter en SSH.

- Le serveur est le point d'entrée central, il doit être **le plus sécurisé possible**.

Commencer par mettre en place **des règles de pare-feu solides** afin de n'autoriser que certaines IP à s'y connecter en SSH depuis le réseau local.

Mettre en place une configuration SSH **renforcée** (par clé, utilisateur root non autorisé, ...)

Si vous souhaitez pouvoir y accéder depuis l'extérieur (pour aller sur **motion-UI** par exemple), la meilleure solution est la mise en place d'un **VPN** permettant d'accéder au réseau du domicile depuis l'extérieur (la Freebox permet de le faire). Une autre solution consisterai à mettre en place des redirections de port sur la box, mais dans ce cas précis les tentatives d'intrusions seront immédiates et les ports redirigés seront sans cesse scannés par les robots d'Internet.

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