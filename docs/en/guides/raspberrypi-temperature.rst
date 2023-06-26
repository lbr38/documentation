=====================================================
[Raspberry Pi] - Display Raspberry Pi CPU Temperature
=====================================================

For those concerned about monitoring the temperature of the **Raspberry Pi**, there are two commands that can be used to retrieve temperature readings from the temperature sensors.

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/raspberrypi/thermal.png

Commands
========

The first command returns the result in the following format: **temp=44.9'C**:

..  code-block:: shell

    /opt/vc/bin/vcgencmd measure_temp

    temp=44.9’C

The second command returns a five-digit number, for example **44912**. To get the temperature reading in two digits, simply divide the result by 1000:

..  code-block:: shell

    cat /sys/class/thermal/thermal_zone0/temp | awk '{ print $1 / 1000 }'

    44

Daily History
=============

To go further, I propose the following **bash** script that retrieves the temperature **every 5 minutes** and writes it to a file, keeping a **daily temperature history**.

Under certain conditions, such as **overheating**, the script is capable of performing actions like **sending an email alert** or even ordering the Raspberry Pi to **shut down**.

The generated file is in HTML format, allowing for the addition of colors and easy viewing in any web browser.

Extract from the generated file:

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/raspberrypi/temp_repport.png

Conditions
----------

.. role:: bluetext
.. role:: greentext
.. role:: orangetext
.. role:: redtext

- If the temperature is below **40°C**, it is considered **normal**. The value is displayed in :bluetext:`blue` in the file.
- If the temperature is between **40°C** and **50°C**, it is considered **normal**. The value is displayed in :greentext:`green` in the file.
- If the temperature is between **50°C** and **70°C**, there is a **slight heat increase**. The value is displayed in :orangetext:`orange` in the file.
- If the temperature is between **70°C** and **75°C**, there is a **heat increase**. The value is displayed in :redtext:`red`` in the file, and an alert email is sent to report overheating.
- If the temperature exceeds **75°C**, the value is displayed in **black** in the file, and **an email alert is sent** to report abnormal temperature. Finally, the command **shutdown -h now** is executed to **shut down the Raspberry Pi** and prevent any damage to the components.

While it may seem excessive to talk about overheating when the temperature exceeds **70°C**, it appears that the Raspberry Pi can withstand temperatures above **80°C** according to some internet sources.

However, since my Raspberry Pi has rarely reached +70°C, I consider that if that were to happen, it would be abnormal and worth being alerted by email.

Code
----

Create a new script as **pi** user:

.. code-block:: shell

    vim /home/pi/scripts/temperature.sh

Insert the following code:

.. code-block:: shell

    #!/bin/bash

    SEND_ALERT="no"
    ALERT_MSG=""
    ALERT_MAIL_RECIPIENT="email@mail.com" # email address to receive alerts
    ALERT_MUTT_CONF="/home/pi/.muttrc" # path to the .muttrc configuration file
    SHUTDOWN="no"
    COLOR=""

    # Get the temperature; here we obtain a 5-digit value without decimals (e.g., 44123):
    TEMP=$(cat /sys/class/thermal/thermal_zone0/temp)

    # Divide the obtained value by 1000 to get a result with two digits only (e.g., 44):
    TEMP=$(($TEMP/1000))

    # Get the current date and time in the format "Wednesday 31 December 2014, 00:15:01":
    DATE=$(date +"%A %d %B %Y, %H:%M:%S")

    # Get the current date and time in another format (e.g., AAAA-MM-DD):
    DATE_ALT=$(date +"%Y-%m-%d")

    # Target directory (where the values will be stored). I store my values on my NAS for easy access to the generated files:
    DIR="/mnt/NAS/raspberry/temperatures"

    # The file to create in this directory is "temperature.html"
    TEMP_FILE="${DIR}/${DATE_ALT}_temperature.html"

    # If the target directory doesn't exist, create it
    mkdir -p "$DIR"

    # If the temperature.html file doesn't exist, create it and inject the minimum HTML code
    if [ ! -f "$TEMP_FILE" ];then
        echo "<!DOCTYPE html><html><head><meta charset='utf-8' /></head><body><center>" > "$TEMP_FILE"
    fi

    # Test the measured temperature

    # If the measured temperature is below 40°C:
    if [ "$TEMP" -lt "40" ]; then
        COLOR="blue"

    # If the measured temperature is between 40°C and 50°C:
    elif [ "$TEMP" -ge "40" ] && [ "$TEMP" -lt "50" ];then
        COLOR="green"

    # If the measured temperature is between 50°C and 70°C:
    elif [ "$TEMP" -ge "50" ] && [ "$TEMP" -lt "70" ];then
        COLOR="orange"

    # If the measured temperature is between 70°C and 75°C, send an "overheating" alert via email:
    elif [ "$TEMP" -ge "70" ] && [ "$TEMP" -lt "75" ];then
        COLOR="red"
        SEND_ALERT="yes"
        ALERT_MSG="Overheating alert, temperature = ${TEMP}°C"

    # If the measured temperature exceeds 75°C, send an email alert and order the RPi to shut down:
    elif [ "$TEMP" -ge "75" ];then
        COLOR="black"
        SHUTDOWN="yes"
        SEND_ALERT="yes"
        ALERT_MSG="Abnormal temperature alert, immediate shutdown of the Pi, temperature = ${TEMP}°C"
    fi

    # Write the measured temperature to the file
    echo "<font face='Courier'>${DATE}<br><strong><font color='${COLOR}'>${TEMP}°C</font></font></strong><br><br>" >> "$TEMP_FILE"

    # If an alert is to be sent
    if [ "$SEND_ALERT" == "yes" ];then
        echo "" | mutt -s "$ALERT_MSG" -F "$ALERT_MUTT_CONF" -- "$ALERT_MAIL_RECIPIENT"
    fi

    # If the RPi needs to be shut down
    if [ "$SHUTDOWN" == "yes" ];then
        sudo shutdown -h now
    fi

    exit


Automatic Execution
-------------------

To have the script executed every 5 minutes, add a line to the crontab:

.. code-block:: shell

    crontab -e

Insert the following line:

.. code-block:: shell

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