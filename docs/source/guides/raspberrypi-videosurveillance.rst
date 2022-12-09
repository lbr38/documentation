===================================================
[Raspberry Pi] - Vidéo-surveillance avancée avec ustreamer et motion
===================================================

Introduction
============

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/raspberrypi/motion/cctv.jpg

La vidéo-surveillance est l'un des projets phare sur **Raspberry Pi** et cela depuis les débuts de la carte.
Des outils comme **motion** et des OS dédiés comme **motionPie** sont vite devenus une référence pour ce type d’utilisation.

Pourtant, les limitations se font vite ressentir malgré les évolutions de la carte. L'encodage vidéo étant un processus gourmand en CPU et en RAM, le Raspberry Pi se retrouve rapidement surchargé lorsqu'on commence à mettre en place un système de vidéo-surveillance avancé avec plusieurs caméras.

Solution
========

La solution proposée ici tente de répondre à la problématique de surcharge en dispatchant les tâches :

- Les **cartes ARM** (Raspberry Pi ou autre) s'occupent de capturer et renvoyer le flux vidéo généré par la caméra qui y est rattachée.
- Un **serveur central** recéptionne les flux et prend en charge les tâches lourdes de **détection de mouvement** et **d'encodage** (motion)

Les "**cartes-caméras**" sont placées aux différents endroits à surveiller de l'habitation, en intérieur ou extérieur, en prenant les précautions nécéssaires pour les protéger de l'humidité le cas échéant.

Le **serveur** est stocké au chaud à l'intérieur et sa sécurité sera renforcée puisqu'il s'agira du point central par lequel l'utilisateur consultera tous les évènements ayant eu lieu et visualisera le stream en direct.

Le tout étant relié au réseau local de manière **filaire**. On excluera les caméras WI-FI ici puisque celles-ci peuvent être facilement **neutralisées** avec un simple brouilleur Wi-FI.

Tout ceci illustré :

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/raspberrypi/motion/motion.png

Pré-requis
==========

- **1 ou plusieurs Raspberry Pi** (ou carte concurrente) pour la partie "caméra".

Leur puissance peut être faible à modérée puisque leur rôle est seulement de diffuser du flux sans traitement. 

Personnellement j'utilise des `Orange-Pi zero LTS <https://orangepi.com/index.php?route=product/product&product_id=846>`_ (4CPU, 512Mo de RAM, 1 port Ethernet et 1 port USB).
Leur **petite taille** permet de les faufiler partout et leur port **POE** peut permettre de n'utiliser qu'un câble Ethernet pour les alimenter.
Pour la caméra, j'utilise un dôme USB étanche acheté sur `amazon <https://www.amazon.fr/dp/B01JG43TD0/ref=dp_prsubs_1>`_ relié à l'unique port USB de l'Orange-Pi.

- **1 serveur central**

De préférence un serveur "maison", **le plus puissant possible**. Il est préférable d'éviter d'utiliser une carte ARM qui risque de se retrouver rapidement surchargée lors des traitements vidéos. Il faut prendre en compte que plus il y aura de caméras et plus le traitement sera lourd.
Le serveur devra faire tourner un OS tel que Debian, CentOS... là où il est possible d'installer **motion** dans une version suffisamment récente.

Préparer chaque élément :

- Installer les OS nécéssaires (raspbian ou armbian par exemple) sur chaque cartes ARM et sur le serveur central (Debian par exemple)
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
    
    mkdir -p /home/pi/scripts

Script de démarrage :

..  code-block:: shell

    vim /home/pi/scripts/start-camera.sh

Insérer le contenu suivant :

..  code-block:: shell

    #!/bin/bash

    RESOLUTION="1920x1080" # Resolution du stream, à adapter en fonction de la résolution maximale dont est capable la camera
    FRAMERATE="25" # Nombre d'images par seconde qui seront diffusées par le stream, si la camera en est capable
    LOG="/home/pi/scripts/ustreamer-live.log" # Emplacement du fichier de log 

    echo -n> "$LOG" # Vidage du fichier de log

    echo "Démarrage du stream" 
    /home/pi/ustreamer/ustreamer --device=/dev/video0 --slowdown -e 30 -K 0 -r $RESOLUTION -m MJPEG --host 0.0.0.0 --port 8888 --device-timeout 2 --device-error-delay 1 2>&1 | tee "$LOG" &

    exit

Script d'arrêt :

..  code-block:: shell

    vim /home/pi/scripts/stop-camera.sh

Insérer le contenu suivant :

..  code-block:: shell

    #!/bin/bash

    # Pour arrêter le stream, il faut tuer le processus, du coup on cherche le PID correspondant :

    PID="$(/bin/ps -aux | /bin/grep 'ustreamer' | egrep -v 'grep|ustreamer-live.log' | /usr/bin/awk '{print $2}')"

    if [ -z "$PID" ];then
        echo "Aucun processus actif de ustreamer"
        exit
    fi

    echo "Arrêt de ustreamer :"
    kill "$PID"

    sleep 1

    # Vraiment au cas où le processus n'a pas été tué, on retente une deuxième fois :

    if /bin/ps -aux | /bin/grep '/home/pi/ustreamer/ustreamer' | /bin/grep -v 'grep';then
        echo "Le processus n'a pas été tué, nouvelle tentative..."
        kill -9 "$PID"
    else
        echo "OK"
    fi

    exit

Ajuster les permissions sur ce qui vient d'être créé :

..  code-block:: shell

    chmod 700 /home/pi/scripts/start-camera.sh 
    chmod 700 /home/pi/scripts/stop-camera.sh
    chown -R pi:pi /home/pi/scripts

Se loguer temporairement en tant que **pi** et démarrer le stream pour tester :

..  code-block:: shell

    su pi
    /home/pi/scripts/start-camera.sh &

Ça devrait afficher quelques logs à l’écran.

Ouvrir http://ADRESSE_IP_CAMERA:8888 dans un navigateur, la page d'accueil de ustreamer doit être accessible et le **stream** est visualisable en cliquant sur **/stream**.

Toujours en tant que **pi** créer une tâche cron qui démarrera le stream automatiquement après un reboot du Raspberry Pi :

..  code-block:: shell

    crontab -e

    @reboot /home/pi/scripts/start-camera.sh


Configuration du serveur
------------------------

Le but ici est de mettre en place **motion** pour analyser le flux des caméras disposées dans l'habitation et détecter des mouvements.

**motion-UI** pourra également être installé afin de pouvoir administrer plus facilement motion, pouvoir **configurer des alertes** et pouvoir **visualiser le stream en direct** des caméras sans jamais avoir besoin de se connecter aux caméras elles-mêmes.

Notes :

- Le système utilisé ici est Debian
- La version de motion installée est au minimum la **4.3.X**. Les versions plus anciennes peuvent ne pas comporter certains paramètres disponibles uniquement sur les versions récentes.
- L’ensemble des configurations s’effectuent en **root**.

Motion
++++++

Installer motion :

..  code-block:: shell

    apt install motion

Configuration générale
~~~~~~~~~~~~~~~~~~~~~~

Motion est livré avec un fichier de configuration principal **motion.conf** ainsi que plusieurs sous-fichiers de caméras optionnels :

..  code-block:: shell

    -rw-r--r-- 1 root root  726 nov.  15  2020 camera1-dist.conf
    -rw-r--r-- 1 root root  817 nov.  15  2020 camera2-dist.conf
    -rw-r--r-- 1 root root  881 nov.  15  2020 camera3-dist.conf
    -rw-r--r-- 1 root root  798 nov.  15  2020 camera4-dist.conf
    -rw-r--r-- 1 root root 5190 nov.  15  2020 motion.conf

Par défaut lorsqu'il n'y a qu'une seule caméra à traiter, on peut utiliser uniquement le fichier principal et s'affranchir des sous-fichiers.
Dans notre cas, nous avons plusieurs caméras à gérer et nous devrons utiliser ces sous-fichiers (1 pour chaque caméra).

Commencer par désactiver/adapter certains paramètres dans le fichier de configuration principal :

..  code-block:: shell

    vim /etc/motion/motion.conf

Désactiver le mode daemon car c'est **systemd** qui exécutera motion :

..  code-block:: shell

    daemon off

Spécifier l'emplacement du fichier de log.
Veillez à ce que le répertoire où il est stocké existe et que l'utilisateur **motion** a le droit d'écriture sur le fichier.

..  code-block:: shell

    log_file /var/log/motion/motion.log

Commenter les paramètres suivants :

..  code-block:: shell

    ;target_dir
    ;videodevice

Désactiver le système de stream proposé par motion en le forçant à streamer uniquement sur localhost :

..  code-block:: shell

    stream_localhost on

Enfin, tout en bas du fichier il est possible d'inclure des fichiers de configuration supplémentaires.
Inclure autant de fichiers que nécessaire (1 par caméra), en les nommant explicitement si besoin. Par exemple pour inclure 2 caméras :

..  code-block:: shell

    camera /etc/motion/camera-exterieur.conf
    camera /etc/motion/camera-interieur.conf

Enregistrer et sortir du fichier de configuration principal.

Puis utiliser les fichiers de configuration supplémentaires déjà présents et les renommer :

..  code-block:: shell

    cd /etc/motion/
    mv camera1-dist.conf camera-exterieur.conf
    mv camera2-dist.conf camera-interieur.conf

Configuration par caméra
~~~~~~~~~~~~~~~~~~~~~~~~

Editer chacun des fichiers de caméras précédemment inclus dans la configuration principale et ajouter/adapter les paramètres suivants.


Modifier le nom de la caméra, ce sera notamment utile dans motion-UI pour identifier la caméra.
Le nom doit être unique pour chaque caméra.

..  code-block:: shell

    camera_name Exterieur

Modifier le numéro de caméra, ce sera notamment utile dans motion-UI pour identifier la caméra.
L'Id doit être unique pour chaque caméra.

..  code-block:: shell

    camera_id 01

L'URL vers le stream **ustreamer** de la caméra en question. Motion restera connecté en permanence au stream et l'analysera pour détecter des mouvements et capturer des images.

..  code-block:: shell

    netcam_url http://ADRESSE_IP_CAMERA_EXTERIEUR:8888/stream
    netcam_keepalive on
    netcam_tolerant_check on

Résolution et framerate du stream, indiquer les mêmes valeurs que celles indiquées dans le script de démarrage de ustreamer **start-camera.sh** :    

..  code-block:: shell

    width 1920
    height 1080
    framerate 25

Optionnel : il est possible d'inclure un texte dans les vidéos qui seront générées par motion lors d'une détection de mouvement :

..  code-block:: shell

    text_left Exterieur

Nombre d'images pré-détection et post-détection à inclure dans les fichiers vidéos générés par motion lorsqu'une détection à lieu :

..  code-block:: shell

    pre_capture 0
    post_capture 2

Nombre de secondes sans mouvement à l'issue desquelles un évènement prendra fin.
Ici on indique que si 30 secondes ont passé sans nouveau mouvement alors motion peut clore l'évènement en cours.

..  code-block:: shell

    event_gap 30

Désactivation de la génération d'images et activation de la génération de vidéos.

Le nombre d'images (fichiers d'images JPEG) générées par motion lorsqu'un mouvement est détecté peut être énorme et générer plusieurs centaines ou miliers d'images en une seule journée selon les cas.

On préfèrera donc uniquement générer des fichiers vidéos (.avi).

On limite également la durée de chaque vidéo à 30 secondes afin de ne pas générer de trop gros fichier vidéo à la fois. Si l'évènement doit dure plus de 30sec alors plusieurs vidéos de 30sec seront généréés à la suite.

..  code-block:: shell

    picture_output off

    movie_output on
    movie_quality 90
    movie_codec mpeg4
    movie_max_time 30

Répertoire sur le serveur dans lequel enregistrer les fichiers vidéos générés par motion.
Veiller à créer le répertoire au préalable.

..  code-block:: shell

    target_dir /home/camera/exterieur

Nom des fichiers vidéos générés. Ici le fichier vidéo sera préfixé de la date et l'heure à laquelle a eu lieu la détection et sera placé dans un répertoire à la date du jour.

..  code-block:: shell

    movie_filename %Y-%m-%d/%v_%Y-%m-%d_%Hh%Mm%Ss_video

C'est à peu près tout pour la configuration des caméras. Répéter l'opération pour chaque caméra.

Si besoin, tous les paramètres de configuration et leur description sont visibles ici : `documentation de motion <https://motion-project.github.io/motion_config.html#Configuration_OptionsAlpha>`_

Démarrage de motion
~~~~~~~~~~~~~~~~~~~

Il est temps de tester la configuration mise en place.

Démarrer le service **motion** puis vérifier son état :

..  code-block:: shell

    systemctl start motion
    systemctl status motion

Si besoin le fichier de log **/var/log/motion/motion.log** peut être utile pour débugguer un problème de démarrage.

Lorsque tout est au vert, **effectuer un mouvement de la main** devant l'une des caméras paramétrées afin de tester la détection de mouvement.
Puis vérifier qu'un fichier vidéo est en cours de génération par motion dans le répertoire **target_dir** spécifié pour cette caméra.

S'aider des logs si aucun fichier n'est généré, cela provient généralement d'un problème de droit d'écriture.

Motion-UI
+++++++++

**motion-UI** est une interface web permettant d'administrer plus aisément **motion**.

Son installation reste optionnelle, on peut tout à fait s'arrêter ici et utiliser motion tel que configuré actuellement.

L'avantage de **motion-UI** est qu'il permet d'aller plus loin dans l'utilisation de **motion**, il permet en outre de mettre en place des **alertes** et de démarrer/stopper motion **de manière autonome** en fonction de la présence ou l'absence d'une personne dans l'habitation.

Il permet également de **visualiser le stream des caméras en direct** et de lire les vidéos générées par motion lors de détections.

J'ai déjà fait un article sur l'installation de motion-UI qu'il suffit de suivre : https://www.linuxdocs.net/guides/motionui.html

Sécurité
========

Maintenant que le système de vidéo-surveillance est fonctionnel il est temps de **sécuriser** l'ensemble sans attendre.

Je ne peux détailler toutes les configurations de sécurité à mettre en place mais voici quelques idées de base :

- Les flux diffusés par les caméras **ne doivent être accessibles que par le serveur**.

En d'autres termes les URLs d'accès à ustreamer http://ADRESSE_IP_CAMERA:8888 ne doivent être accessibles que par le serveur.

Pour cela mettre en place des règles de **pare-feu** (iptables par ex) sur les Raspberry Pi pour n'autoriser que le serveur à y accéder en http.

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