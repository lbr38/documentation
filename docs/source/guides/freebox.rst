==============================================
[Linux] - Accéder au disque dur de la Freebox sous Linux
==============================================

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/freebox/freebox.jpg

Pour les possesseurs d’une Freebox v6 (revolution), il est possible de monter le disque dur intégré à celle-ci sur une distribution **Linux** et de pouvoir ainsi accéder aux fichiers en lecture ou écriture.

Pré-requis
==========

Installer le paquet **cifs-utils** :

..  code-block:: shell

    sudo apt/yum install cifs-utils

Depuis l'interface web de la Freebox, activer les **partages Windows** et s'assurer d'activer la version **SMBv2**. Renseigner un nom d'utilisateur et un mot de passe.

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/freebox/windows-share.png">
            <img src="https://raw.githubusercontent.com/lbr38/documentation/main/docs/source/images/freebox/windows-share.png" width=49% align="top"> 
        </a>
    </div>

    <br>

Appliquer la configuration.

Point de montage
================

Créer le répertoire où sera monté le disque dur de la freebox, par exemple :

..  code-block:: shell

    sudo mkdir -p /mnt/freebox

Créer un fichier .freeboxcredentials dans lequel seront stockés le nom d'utilisateur et le mot de passe précédemment renseigné sur l'interface web de la Freebox.

..  code-block:: shell
    
    vim $HOME/.freeboxcredentials

Insérer le nom d'utilisateur et le mot de passe sous la forme suivante :

..  code-block:: shell

    username=freebox
    password=XXXXXXX

Editer **/etc/fstab** et ajouter la ligne suivante :

..  code-block:: shell

    sudo vim /etc/fstab
    
    //IP_FREEBOX/Disque\040dur/ /mnt/freebox cifs _netdev,rw,iocharset=utf8,uid=toto,credentials=/home/toto/.freeboxcredentials,file_mode=0660,dir_mode=0775 0 2

Adaptez les paramètres :

**IP_FREEBOX** = généralement 192.168.0.254

**uid** = votre nom d’utilisateur ou bien son uid numérique

**/home/toto** = votre home directory

Monter le disque dur de la Freebox dans le répertoire précedemment créé :

..  code-block:: shell

    sudo mount /mnt/freebox

C'est terminé, les fichiers stockés sur le disque dur de la Freebox sont désormais accessibles depuis **/mnt/freebox**

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
