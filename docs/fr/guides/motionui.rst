=====================================================
[Linux] - Vidéosurveillance avec Motion-UI
=====================================================

EN version : https://en.linuxdocs.net/en/latest/guides/motionui.html

**Motion-UI** est une interface web (UI pour "User Interface") développée pour gérer plus aisément le fonctionnement et la configuration de **motion**, un célèbre logiciel **open-source** de détection de mouvement généralement utilisé pour faire de la vidéosurveillance.

Il s'agit d'un projet open-source disponible sur github : https://github.com/lbr38/motion-UI

Présentation
------------

L'interface se présente comme étant très simpliste et **responsive**, ce qui permet une utilisation depuis un **mobile** (application android disponible ici : https://github.com/lbr38/motion-UI/releases/tag/android-1.0).

Elle permet en outre de mettre en place des **alertes mail** en cas de détection et **d'activer automatiquement** ou non la vidéosurveillance en fonction d'une plage horaire ou de la présence de périphériques "de confiance" sur le réseau local (smartphone...).

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/bdae2550-819d-40c4-895b-541ee64bdc03">
        <img src="https://github.com/user-attachments/assets/bdae2550-819d-40c4-895b-541ee64bdc03" width=25% align="top"> 
        </a>

        <a href="https://github.com/user-attachments/assets/afe3e48a-3a26-4e75-a6a7-a97b2ac2bf9e">
        <img src="https://github.com/user-attachments/assets/afe3e48a-3a26-4e75-a6a7-a97b2ac2bf9e" width=25% align="top">
        </a>

        <a href="https://github.com/user-attachments/assets/a2472f8b-24fc-4967-bb6a-f8ad8af95270">
        <img src="https://github.com/user-attachments/assets/a2472f8b-24fc-4967-bb6a-f8ad8af95270" width=25% align="top">
        </a>
    </div>
    <br>
    <div align="center">
        <a href="https://github.com/user-attachments/assets/cb9137c7-484a-4c2c-ad0f-c33ef7a602bd">
        <img src="https://github.com/user-attachments/assets/cb9137c7-484a-4c2c-ad0f-c33ef7a602bd" width=25% align="top">
        </a>

        <a href="https://github.com/user-attachments/assets/81c05e3f-599d-4cc1-9d9a-9748fce54763">
        <img src="https://github.com/user-attachments/assets/81c05e3f-599d-4cc1-9d9a-9748fce54763" width=25% align="top">
        </a>

        <a href="https://github.com/user-attachments/assets/04b18116-2af0-4bd3-8438-e9f1fed8c7ed">
        <img src="https://github.com/user-attachments/assets/04b18116-2af0-4bd3-8438-e9f1fed8c7ed" width=25% align="top">
        </a>
    </div>

    <br>


L'interface se décompose en plusieurs onglets :

- Un onglet dédié aux caméras et au **stream** en direct. Les caméras sont alors disposées en grilles à l'écran (du moins sur un écran PC) un peu à la manière des écrans de vidéosurveillance d'un établissement par exemple.
- Un onglet permettant de démarrer et arrêter le service **motion** et les services associés (**démarrage automatique**, **alertes** en cas de détection).
- Un onglet listant les **évènements** (events) aillant eu lieu et détectés par motion, avec également la possibilité de visualiser les images ou vidéos capturées directement depuis la page web.
- Un onglet avec quelques graphiques permettent de résumer l'activité récente du service motion et des évènements aillant eu lieu.


Pré-requis
----------

Il est préconisé de dédier un serveur uniquement à l'exécution de **motion-UI**, et qu'il soit le point d'entrée unique pour la vidéosurveillance sur le réseau local : les caméras diffusent leur stream au serveur et c'est le serveur qui analyse les images et détecte d'éventuels mouvements et avertit l'utilisateur. La visualisation des caméras se fait également par le biais du serveur depuis l'interface **motion-UI**. C'est ce cas de figure qui sera détaillé ici.

L'installation doit se faire en **root** ou avec **sudo**.

Installer docker :

..  code-block:: shell

    # Installation du repository docker (cas pour Debian ici, voir la doc officielle pour d'autres distributions : https://docs.docker.com/engine/install/)
    apt install ca-certificates curl gnupg -y

    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    echo \ 
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    # Installation de docker
    apt update -y
    apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin -y

Il faudra également :

- Un nom de domaine dédié à **motion-UI** (**motionui.mondomaine.com** par exemple) ainsi qu'un enregistrement **SPF** pour ce nom de domaine (utile pour pouvoir recevoir correctement les alertes mails).
- Un certificat SSL pour ce nom de domaine afin de sécuriser l'accès à **motion-UI** (HTTPS).

Si vous souhaitez pouvoir vous rendre sur **motion-UI** depuis l'extérieur, il faudra également :

- Soit un **VPN** vous permettant de vous connecter à votre réseau local depuis l'extérieur.
- Soit un **enregistrement DNS** faisant pointer **motionui.mondomaine.com** vers votre box, avec des redirections de ports de votre **box/routeur vers le serveur motion-UI** (attention le site sera alors accessible publiquement, veiller à mettre en place des règles de pare-feu pour limiter l'accès si cela est possible).


Installation
------------

L'installation doit se faire avec un utilisateur lambda (non root).

Installer la dernière image disponible en adaptant la valeur de ``FQDN`` par votre nom de domaine dédié à motion-UI :

..  code-block:: shell

    docker run -d --restart always --name motionui \
       -e FQDN=motionui.example.com \
       -p 8080:8080 \
       -p 8555:8555 \
       -v /etc/localtime:/etc/localtime:ro \
       -v /var/lib/docker/volumes/motionui-data:/var/lib/motionui \
       -v /var/lib/docker/volumes/motionui-captures:/var/lib/motion \
       lbr38/motionui:latest

Deux volumes persistants sont alors créés sur le système hôte :

- **motionui_data** ``/var/lib/docker/volumes/motionui-data/`` : contient la base de données de motion-UI
- **motionui-captures** ``/var/lib/docker/volumes/motionui-captures/`` : contient les captures d'images et vidéos réalisées par motion (à conserver donc!)

Une fois l'installation terminée, poursuivre par la mise en place d'un reverse-proxy pour accéder à motion-UI par son nom de domaine.


Reverse-proxy
-------------

La mise en place d'un reverse-proxy va permettre d'accéder à **motion-UI** avec le nom de domaine qui lui a été dédié et de manière sécurisée (HTTPS).

L'installation doit se faire en **root** ou avec **sudo**.

Installer **nginx** si ce n'est pas déjà fait :

..  code-block:: shell

    apt install nginx -y

Supprimer le vhost par défaut :

..  code-block:: shell

    rm /etc/nginx/sites-enabled/default

Puis créer un nouveau vhost dédié à **motion-UI** :

..  code-block:: shell

    vim /etc/nginx/sites-available/motionui.conf

Insérer le contenu suivant en remplacant les valeurs :

- **<SERVER-IP>** : l'adresse IP du serveur
- **<FQDN>** : le nom de domaine dédié à motion-UI
- **<PATH_TO_CERTIFICATE>** : le chemin vers le certificat SSL
- **<PATH_TO_PRIVATE_KEY>** : le chemin vers la clé privée du certificat SSL

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
            proxy_read_timeout 86400;
            proxy_pass http://motionui_docker;
        }
    }

Activer le vhost :

..  code-block:: shell

    ln -s /etc/nginx/sites-available/motionui.conf /etc/nginx/sites-enabled/motionui.conf

Recharger nginx :

..  code-block:: shell

    nginx -t && systemctl reload nginx

Se connecter à **motion-UI** depuis un navigateur web via https://motionui.mondomaine.com

Utiliser les identifiants par défaut pour s'authentifier :

- Login : **admin**
- Mot de passe : **motionui**

Une fois connecté, il est possible de modifier son mot de passe depuis l'espace utilisateur (en haut à droite).



Ajout d'une caméra
------------------

Utiliser le bouton **+** pour ajouter une caméra.

- Préciser un nom et l'URL de la caméra ou le nom du périphérique local (/dev/video0 par exemple).
- Préciser un utilisateur / mot de passe si le flux est protégé.
- Choisir d'activer ou non la détection de mouvement (motion) sur cette caméra.

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/0413cb57-a87f-4779-87ca-7bcbe8e50fa5">
        <img src="https://github.com/user-attachments/assets/0413cb57-a87f-4779-87ca-7bcbe8e50fa5" align="top"> 
        </a>
    </div> 

    <br>

Configuration d'une caméra
--------------------------

Si le besoin de modifier la configuration d'une caméra se fait sentir, il suffit de cliquer sur le bouton **Configure**.

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/42c09a68-b4d1-4950-aa8c-b5dbebf18f52">
        <img src="https://github.com/user-attachments/assets/42c09a68-b4d1-4950-aa8c-b5dbebf18f52" align="top"> 
        </a>
    </div> 

    <br>

D'ici il est possible de modifier les paramètres généraux de la caméra (**nom**, **URL**, etc.), de changer la **rotation** de l'image.

Il est également possible de modifier la **configuration motion** de la caméra (détection de mouvement).

Attention, il est préconisé d'**éviter de modifier les paramètres motion en mode avancé**, ou du moins pas sans savoir précisément ce que l'on fait.

Par exemple **il vaut mieux éviter** de modifier les paramètres suivants :

- les paramètres de nom et d'URL (**device_name**, **netcam_url**, **netcam_userpass** et **rotate**) ont des valeurs issues des paramètres généraux de la caméra. Il convient donc de les modifier directement depuis les champs **Global settings**.
- les paramètres liés aux codecs (**picture_type** et **movie_container**) ne doivent pas être modifiés sous peine de ne plus pouvoir visualier les captures directement depuis motion-UI. 
- les paramètres d'évènements (**on_event_start**, **on_event_end**, **on_movie_end** et **on_picture_save**) ne doivent pas être modifiés sous peine de ne plus pouvoir enregistrer les évènements de détection de mouvement, et de ne plus recevoir d'alertes.


Tester l'enregistrement des évènements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pour cela depuis l'interface **motion-UI** : démarrer manuellement motion (bouton **Start capture**).

.. raw:: html

    <div align="center">
        <img src="https://github.com/lbr38/motion-UI/assets/54670129/34fd7ac9-0ea0-4b5f-95a0-bbdb9f7b5c01" align="top"> 
    </div> 

    <br>

Puis **faire un mouvement** devant une caméra pour déclencher un évènement.

Si tout se passe bien, un nouvel évènement en cours devrait apparaitre après quelques secondes dans l'interface **motion-UI**.


Démarrage et arrêt automatique de motion
----------------------------------------

Utiliser le bouton **Enable and configure autostart** pour activer et configurer le démarrage automatique.

.. raw:: html

    <div align="center">
        <img src="https://github.com/lbr38/motion-UI/assets/54670129/e3007d7e-f4de-41c2-8c0d-506c393ad59f" align="top"> 
    </div> 

    <br>

Il est possible de configurer deux types de démarrages et arrêts automatiques de motion :

- En fonction des plages horaires renseignées pour chaque journée. Le service **motion** sera alors actif **entre** la plage d'horaire renseignée.
- En fonction de la présence d'un ou plusieurs appareils IP connecté(s) sur le réseau local. Si aucun des appareils configurés n'est présent sur le réseau local alors le service motion démarrera, considérant que personne n'est présent au domicile. Motion-UI envoi régulièrement un **ping** pour déterminer si l'appareil est présent sur le réseau, il faut donc veiller à configurer des baux d'IP statiques depuis la box pour chaque appareil du domicile (smartphones).

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/373219d1-588f-4097-80d4-e0b533115098">
        <img src="https://github.com/user-attachments/assets/373219d1-588f-4097-80d4-e0b533115098" width=49% align="top"> 
        </a>
    </div>

    <br>


Configurer les alertes
----------------------

Utiliser le bouton **Enable and configure alerts** pour activer et configurer les alertes.

.. raw:: html

    <div align="center">
        <img src="https://github.com/lbr38/motion-UI/assets/54670129/7a630e6c-d271-455f-9921-b8adc84d1e49" align="top"> 
    </div> 

    <br>

La configuration des alertes nécessite deux points de configuration :

- Un enregistrement **SPF** pour le nom de domaine dédié à motion-UI.
- L'enregistrement des évènements doit fonctionner (voir '**Tester l'enregistrement des évènements**')


Configuration des créneaux horaires d'alertes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Renseigner les **créneaux horaires** entre lesquels vous souhaitez **recevoir des alertes** si détection il y a. Pour activer les alertes **toute une journée**, il convient de renseigner 00:00 pour le créneau de début ET de fin.
- Renseigner l'adresse mail destinataire qui recevra les alertes mails. Plusieurs adresses mails peuvent être spécifiées en les séparant par une virgule.

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/0dd3bc5b-71f4-46ac-8937-c928716987cb">
            <img src="https://github.com/user-attachments/assets/0dd3bc5b-71f4-46ac-8937-c928716987cb" width=49% align="top"> 
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
