=====================================================
[Linux] - Video Surveillance with Motion-UI
=====================================================

**Motion-UI** is a web-based User Interface developed to facilitate the operation and configuration of **motion**, a popular open-source motion detection software commonly used for video surveillance.

It is an open-source project available on GitHub: https://github.com/lbr38/motion-UI

Overview
--------

The interface presents itself as very simplistic and **responsive**, allowing for mobile usage (Android application available here: https://github.com/lbr38/motion-UI/releases/tag/android-1.0).

It also allows the setup of **email alerts** in case of motion detection, and it can automatically enable or disable video surveillance based on a specified time range or the presence of "trusted" devices on the local network (e.g., smartphones).

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


The interface is divided into several tabs:

- An tab dedicated to cameras and **live stream**. The cameras are then arranged in grids on the screen (at least on a PC screen), somewhat like the surveillance screens of a facility, for example.
- An tab for starting and stopping the service **motion** and associated services (**automatic startup**, **alerts** in case of detection).
- An tab listing the **events** that have occurred and been detected by motion, with the ability to view the images or videos captured directly from the web page.
- An tab with a few graphs summarizing the recent activity of the motion service and the events that have occurred.


Prerequisites
-------------

It is recommended to dedicate a server solely for running **motion-UI**, and it should be the unique entry point for video surveillance on the local network: cameras stream their footage to the server, which analyzes the images, detects any movements, and notifies the user. Viewing the cameras is also done through the server using the **motion-UI** interface. This scenario will be detailed here.

The installation must be done as **root** or with **sudo**.

Install docker:

.. code-block:: shell

    # Installing the Docker repository (for Debian in this case, refer to the official documentation for other distributions: https://docs.docker.com/engine/install/)
    apt install ca-certificates curl gnupg -y

    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    echo \
    "deb [arch=\"$(dpkg --print-architecture)\" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
    $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    # Installing Docker
    apt update -y
    apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin -y

Additionally, you will need:

- A dedicated domain name for **motion-UI** (**motionui.mydomain.com**, for example), along with an **SPF record** for this domain (useful for correctly receiving email alerts).
- An SSL certificate for this domain to secure access to **motion-UI** (HTTPS).

If you want to access **motion-UI** from outside your local network, you will also need:

- Either a **VPN** that allows you to connect to your local network from outside.
- Or a **DNS record** that points **motionui.mydomain.com** to your router, with port forwarding from your **router/ gateway to the motion-UI server** (please note that the site will then be publicly accessible, so make sure to implement firewall rules to limit access if possible).


Installation
------------

The installation should be done with a regular user (non-root).

Pull the latest available image and adapt the ``FQDN`` value to your domain name:

.. code-block:: shell

    docker run -d --restart unless-stopped --name motionui --network=host \
       -e FQDN=motionui.example.com \
       -v /etc/localtime:/etc/localtime:ro \
       -v /var/lib/docker/volumes/motionui-data:/var/lib/motionui \
       -v /var/lib/docker/volumes/motionui-captures:/var/lib/motion \
       lbr38/motionui:latest

Two persistent volumes are created on the host system:

- **motionui_data** ``/var/lib/docker/volumes/motionui-data/``: contains the motion-UI database.
- **motionui-captures** ``/var/lib/docker/volumes/motionui-captures/``: contains the captures of images and videos taken by motion (keep them!).

Once the installation is complete, proceed with setting up a reverse-proxy to access motion-UI via its domain name.


Reverse-proxy
-------------

Setting up a reverse-proxy will allow accessing **motion-UI** using its dedicated domain name securely (HTTPS).

Installation should be done as **root** or using **sudo**.

Install **nginx** if it is not already installed:

..  code-block:: shell

    apt install nginx -y

Remove the default vhost:

..  code-block:: shell

    rm /etc/nginx/sites-enabled/default

Then, create a new vhost dedicated to **motion-UI**:

..  code-block:: shell

    vim /etc/nginx/sites-available/motionui.conf

Insert the following content, replacing the values:

- **<SERVER-IP>**: Server's IP address
- **<FQDN>**: The domain name dedicated to motion-UI
- **<PATH_TO_CERTIFICATE>**: Path to the SSL certificate
- **<PATH_TO_PRIVATE_KEY>**: Path to the SSL certificate's private key

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

Activate the vhost:

.. code-block:: shell

    ln -s /etc/nginx/sites-available/motionui.conf /etc/nginx/sites-enabled/motionui.conf

Reload nginx:

.. code-block:: shell

    nginx -t && systemctl reload nginx

Connect to **motion-UI** from a web browser via https://motionui.mydomain.com

Use the default credentials to authenticate:

- Login: **admin**
- Password: **motionui**

Once logged in, you can change your password from the user area (top right).



Adding a camera
---------------

Use the **+** button to add a camera.

- Provide a name and the URL to the camera or the local device (/dev/video0, for example).
- Specify a username/password if the stream is protected.
- Choose to enable motion detection on this camera.

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/0413cb57-a87f-4779-87ca-7bcbe8e50fa5">
        <img src="https://github.com/user-attachments/assets/0413cb57-a87f-4779-87ca-7bcbe8e50fa5" align="top"> 
        </a>
    </div> 

    <br>

Camera configuration
--------------------

If there is a need to modify the configuration of a camera, simply click on the **Configure** button.

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/42c09a68-b4d1-4950-aa8c-b5dbebf18f52">
        <img src="https://github.com/user-attachments/assets/42c09a68-b4d1-4950-aa8c-b5dbebf18f52" align="top"> 
        </a>
    </div> 

    <br>

From here, it is possible to modify the general parameters of the camera (**name**, **URL**, etc.), and change the **rotation** of the image.

It is also possible to modify the **motion configuration** of the camera (motion detection).

However, it is recommended to **avoid modifying motion parameters in advanced mode**, or at least not without knowing exactly what you are doing.

For example, **it is better to avoid** modifying the following parameters:

- The name and URL parameters (**device_name**, **netcam_url**, **netcam_userpass**, and **rotate**) have values that come from the general parameters of the camera. Therefore, it is best to modify them directly from the **Global settings** fields.
- The parameters related to codecs (**picture_type** and **movie_container**) should not be modified, or you may no longer be able to view the captures directly from motion-UI. 
- The event parameters (**on_event_start**, **on_event_end**, **on_movie_end**, and **on_picture_save**) should not be modified, or you may no longer be able to record motion detection events and receive alerts.


Testing event recording
~~~~~~~~~~~~~~~~~~~~~~~

To do this from the **motion-UI** interface: manually start motion (button **Start capture**).

.. raw:: html

    <div align="center">
        <img src="https://github.com/lbr38/motion-UI/assets/54670129/34fd7ac9-0ea0-4b5f-95a0-bbdb9f7b5c01" align="top"> 
    </div>  

    <br>

Then **make a movement** in front of a camera to trigger an event.

If everything goes well, a new ongoing event should appear after a few seconds in the **motion-UI** interface.


Automatic start and stop of Motion
----------------------------------

Use the **Enable and configure autostart** button to activate and configure automatic startup.

.. raw:: html

    <div align="center">
        <img src="https://github.com/lbr38/motion-UI/assets/54670129/e3007d7e-f4de-41c2-8c0d-506c393ad59f" align="top"> 
    </div> 

    <br>

There are two types of automatic startups and shutdowns of motion that can be configured:

- Based on the specified time slots for each day. The **motion** service will be active **between** the specified time slot.
- Based on the presence of one or more IP devices connected to the local network. If none of the configured devices are present on the local network, then the motion service will start, assuming that no one is present at home. Motion-UI regularly sends a **ping** to determine if the device is present on the network, so it is necessary to configure static IP leases from the router for each device in the home (smartphones).

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/373219d1-588f-4097-80d4-e0b533115098">
        <img src="https://github.com/user-attachments/assets/373219d1-588f-4097-80d4-e0b533115098" width=49% align="top"> 
        </a>
    </div>

    <br>


Configure alerts
----------------

Use the **Enable and configure alerts** button to activate and configure alerts.

.. raw:: html

    <div align="center">
        <img src="https://github.com/lbr38/motion-UI/assets/54670129/7a630e6c-d271-455f-9921-b8adc84d1e49" align="top"> 
    </div> 

    <br>

Configuring alerts requires two configuration points:

- An **SPF record** for the domain name dedicated to motion-UI.
- Event recording must be functional (see '**Testing Event Recording**').


Configuration of alert time slots
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Fill in the **time slots** during which you wish to **receive alerts** if there is any motion detection. To enable alerts for the **entire day**, you should enter 00:00 for both the start and end time slots.
- Provide the recipient email address that will receive the email alerts. Multiple email addresses can be specified by separating them with a comma.

.. raw:: html

    <div align="center">
        <a href="https://github.com/user-attachments/assets/0dd3bc5b-71f4-46ac-8937-c928716987cb">
            <img src="https://github.com/user-attachments/assets/0dd3bc5b-71f4-46ac-8937-c928716987cb" width=49% align="top"> 
        </a>
    </div>

    <br>


Testing alerts
~~~~~~~~~~~~~~

Once the previously mentioned points have been properly configured, and the **motionui** service is running, you can test the sending of alerts.

To do this from the **motion-UI** interface:

- Temporarily disable motion autostart if it's enabled, to prevent it from stopping motion in case of alerts.
- Manually start motion (**Start capture**).

Then **create motion** in front of a camera to trigger an alert.

If you encounter any issues, feel free to ask a **question** on the developer's repository or open a new **issue**:

- https://github.com/lbr38/motion-UI/discussions
- https://github.com/lbr38/motion-UI/issues


External access

To access motion-UI from outside, two options are possible:

**Option 1** (VPN) (recommended):

- Set up a VPN to access your local network from outside.

**Option 2** (Port forwarding from the router):

- Point the domain name dedicated to motion-UI to your router.
- Forward ports 80, 443, and 8555 (the latter is necessary to view the live stream) from your router to the motion-UI server.
- Set up firewall rules to restrict access to motion-UI from outside only to trusted IP addresses.


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


