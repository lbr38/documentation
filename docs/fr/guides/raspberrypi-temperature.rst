===========================================
[Raspberry Pi] - Afficher la température CPU du Raspberry Pi
===========================================

EN version : https://en.linuxdocs.net/en/latest/guides/raspberrypi-temperature.html

Pour les inquiets qui souhaitent garder un œil sur la température du **Raspberry Pi**, il existe deux commandes permettant de récupérer une valeur de la part des capteurs de température.

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/raspberrypi/thermal.png

Commandes
=========

La première commande qui renvoie un résultat sous la forme suivante : **temp=44.9’C** :

..  code-block:: shell

    /opt/vc/bin/vcgencmd measure_temp

    temp=44.9’C

La seconde qui renvoie un nombre à 5 chiffres, ex: **44912**, il suffit alors de diviser le résultat par 1000 pour ne garder que le résultat sur deux chiffres :

..  code-block:: shell

    cat /sys/class/thermal/thermal_zone0/temp | awk '{ print $1 / 1000 }'

    44

Historique journalier
=====================

Pour aller plus loin, je propose le code ci-dessous qui est un script **bash** récupérant la témpérature **toutes les 5 minutes** et qui l'inscrit un fichier, gardant ainsi un **historique sur toute une journée**.

Dans certaines conditions, une **surchauffe** par exemple, le script est en mesure d’effectuer des actions comme **envoyer une alerte** mail ou encore d’ordonner au Raspberry Pi de **s’arrêter**.

Le fichier généré est au format HTML, ce qui permet d'ajouter quelques couleurs et d'être facilement ouvert depuis n'importe quel navigateur.

Extrait du fichier généré :

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/raspberrypi/temp_repport.png

Conditions
----------

.. role:: bluetext
.. role:: greentext
.. role:: orangetext
.. role:: redtext

- Si la température est inférieure à **40°**, alors on considère qu’elle est **OK**. La valeur est écrite en :bluetext:`bleu` dans le fichier.
- Si la température est comprise entre **40°** et **50°**, alors on considère qu’elle est **normale**. La valeur est écrite en :greentext:`vert` dans le fichier.
- Si la température est comprise entre **50°** et **70°**, alors on considère qu’il y a **une légère chauffe**. La valeur est écrite en :orangetext:`orange` dans le fichier.
- Si la température est comprise entre **70°** et **75°**, alors on considère qu'il y a une **chauffe**. La valeur est écrite en :redtext:`rouge` dans le fichier et **une alerte mail est envoyée** pour signaler une surchauffe.
- Si la température est supérieure à **75°**, alors la valeur est écrite en **noir** dans le fichier et **une alerte mail est envoyée** pour signaler une température anormale. Enfin, la commande d'arrêt **shutdown –h now** est exécutée afin **d'arrêter le Raspberry Pi** et éviter tout dommage sur les composants.

Certes il peut être exagéré de parler de surchauffe lorsque la température dépasse **70°C**, sachant que le Raspberry Pi semble pouvoir résister à des températures allant au delà de **80°C** d’après certaines sources Internet.

Cependant, mon Raspberry Pi n’ayant que très rarement atteint +70°C, je considère que si ça devait arriver, alors cela serait anormal et il serait bon d’en être averti par mail.

Code
----

Créer un nouveau script en tant que pi :

..  code-block:: shell

    vim /home/pi/scripts/temperature.sh

Insérer le code suivant :

..  code-block:: shell

    #!/bin/bash

    SEND_ALERT="no"
    ALERT_MSG=""
    ALERT_MAIL_RECIPIENT="email@mail.com" # adresse mail recevant les alertes
    ALERT_MUTT_CONF="/home/pi/.muttrc" # chemin vers le fichier de configuration .muttrc
    SHUTDOWN="no"
    COLOR=""

    # Récupération de la température ; on obtient ici une valeur à 5 chiffres sans virgules (ex: 44123) :
    TEMP=$(cat /sys/class/thermal/thermal_zone0/temp)

    # On divise alors la valeur obtenue par 1000, pour obtenir un résultat avec deux chiffres seulement (ex: 44) :
    TEMP=$(($TEMP/1000))

    # Récupération de la date et l'heure du jour ; on obtient ici une valeur telle que "mercredi 31 décembre 2014, 00:15:01" :
    DATE=$(date +"%A %d %B %Y, %H:%M:%S")

    # Récupération de la date et l'heure du jour sous un autre format ; on obtient ici un résultat sous la forme suivante : AAAA-MM-DD :
    DATE_ALT=$(date +"%Y-%m-%d")

    # Répertoire cible (où seront stockées les valeurs). Ici je stocke mes valeurs sur mon NAS afin d'avoir facilement accès aux fichiers générés :
    DIR="/mnt/NAS/raspberry/temperatures"

    # Le fichier à créer dans ce répertoire est "temperature.html"
    TEMP_FILE="${DIR}/${DATE_ALT}_temperature.html"

    # Si le répertoire cible n'existe pas, on le crée
    mkdir -p "$DIR"

    # Si le fichier temperature.html n'existe pas, on le crée et on y injecte le code html minimum
    if [ ! -f "$TEMP_FILE" ];then
        echo "<!DOCTYPE html><html><head><meta charset='utf-8' /></head><body><center>" > "$TEMP_FILE"
    fi


    # Test de la température relevée

    # Si la température relevée est inférieure à 40°C :
    if [ "$TEMP" -lt "40" ]; then
        COLOR="blue"

    # Si la température relevée est comprise entre +40 et 50°C :
    elif [ "$TEMP" -ge "40" ] && [ "$TEMP" -lt "50" ];then
        COLOR="green"

    # Si la température relevée est comprise entre +50 et 70°C :
    elif [ "$TEMP" -ge "50" ] && [ "$TEMP" -lt "70" ];then
        COLOR="orange"

    # Si la température relevée est comprise entre +70 et 75°C, on envoie une alerte "surchauffe" par mail :
    elif [ "$TEMP" -ge "70" ] && [ "$TEMP" -lt "75" ];then
        COLOR="red"
        SEND_ALERT="yes"
        ALERT_MSG="Alerte surchauffe, température = ${TEMP}°C"

    # Si la température relevée dépasse 75°, on envoie une alerte par mail et on ordonne l'arrêt du RPi :
    elif [ "$TEMP" -ge "75" ];then
        COLOR="black"
        SHUTDOWN="yes"
        SEND_ALERT="yes"
        ALERT_MSG="Alerte température anormale, arrêt immédiat du pi, température = ${TEMP}°C"
    fi

    # Ecriture de la température relevée dans le fichier
    echo "<font face='Courier'>${DATE}<br><strong><font color='${COLOR}'>${TEMP}°C</font></font></strong><br><br>" >> "$TEMP_FILE"

    # Si une alerte doit être envoyée
    if [ "$SEND_ALERT" == "yes" ];then
        echo "" | mutt -s "$ALERT_MSG" -F "$ALERT_MUTT_CONF" -- "$ALERT_MAIL_RECIPIENT"
    fi

    # Si le RPi doit être arrêté
    if [ "$SHUTDOWN" == "yes" ];then
        sudo shutdown -h now
    fi

    exit

Exécution automatique
---------------------

Pour que le script soit exécuté toutes les 5 minutes, il convient alors de rajouter une ligne dans la crontab :

..  code-block:: shell

    crontab -e

Insérer la ligne suivante :

..  code-block:: shell

    */5 * * * * /home/pi/scripts/temperature.sh

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