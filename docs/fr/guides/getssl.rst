====================================
[Linux] - Certificat Let's Encrypt avec getssl
====================================

.. image:: https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/getssl/letsencrypt.png

**getssl** est un script bash permettant d’obtenir gratuitement un certificat SSL auprès de l’autorité de certification **Let’s Encrypt**, valide pour une durée de 3 mois.

Il prend en charge le renouvellement des certificats et contrairement à d’autres scripts tels que certbot, il ne nécessite pas de dépendances faramineuses pour s’installer et fonctionner.

Lien vers le projet github : https://github.com/srvrco/getssl

Pré-requis
==========

- Un serveur web avec au moins un vhost écoutant sur le port 80
- Un nom de domaine avec un enregistrement DNS pointant vers le serveur web
- le paquet **dnsutils** installé si ce n'est pas déjà le cas

Admettons qu'un vhost répondant à l'enregistrement **www.mondomaine.com** et servant les fichiers contenus dans le répertoire racine **/var/www/www.mondomaine.com/**

Créer un sous-répertoire dans /var/www/www.mondomaine.com/ qui servira à la validation de la commande de certificat par le biais d'un token :

..  code-block:: shell

    mkdir -p /var/www/www.mondomaine.com/.well-known/acme-challenge/


Mettre en place les bonnes permissions sur le sous-répertoire créé :

..  code-block:: shell

    # selon le système, l'utilisateur n'est pas www-data mais nginx:nginx ou apache:apache :
    chown -R www-data:www-data /var/www/www.mondomaine.com/.well-known
    chmod -R 750 /var/www/www.mondomaine.com/.well-known


Installation et configuration
=============================

Récupérer le script **getssl** dans un répertoire qui va bien : 

..  code-block:: shell
    
    cd /root/scripts/
    curl --silent https://raw.githubusercontent.com/srvrco/getssl/master/getssl > getssl
    chmod 700 getssl

Générer le fichier de configuration pour votre domaine ou sous-domaine. Par exemple ici pour **www.mondomaine.com** :

..  code-block:: shell
    
    ./getssl -c www.mondomaine.com

Le fichier de configuration est généré dans un répertoire caché **.getssl**, l’éditer :

..  code-block:: shell

    vim /root/.getssl/www.mondomaine.com/getssl.cfg

Adapter les paramètres décommentés ci-dessous. J'ai mis un commentaire à côté de chaque paramètre. Selon la version de **getssl**, le contenu du fichier peut avoir légèrement varié par rapport à cette documentation.

..  code-block:: shell

    # vim: filetype=sh
    #
    # This file is read second (and per domain if running with the -a option)
    # and overwrites any settings from the first file
    #
    # Uncomment and modify any variables you need
    # see https://github.com/srvrco/getssl/wiki/Config-variables for details
    # see https://github.com/srvrco/getssl/wiki/Example-config-files for example configs
    #
    # The staging server is best for testing
    #CA="https://acme-staging-v02.api.letsencrypt.org" # Commenter cette ligne, il s'agit d'un serveur délivrant uniquement des certificats dans le cadre de tests
    # This server issues full certificates, however has rate limits
    CA="https://acme-v02.api.letsencrypt.org" # Dé-commenter cette ligne

    # Private key types - can be rsa, prime256v1, secp384r1 or secp521r1
    #PRIVATE_KEY_ALG="rsa"

    # Additional domains - this could be multiple domains / subdomains in a comma separated list
    # Note: this is Additional domains - so should not include the primary domain.
    SANS="" # A utiliser dans le cas où vous souhaitez obtenir un certificat pour un autre sous-domaine. Dans ce cas il faut indiquer le sous-domaine ici.

    # Acme Challenge Location. The first line for the domain, the following ones for each additional domain.
    # If these start with ssh: then the next variable is assumed to be the hostname and the rest the location.
    # An ssh key will be needed to provide you with access to the remote server.
    # Optionally, you can specify a different userid for ssh/scp to use on the remote server before the @ sign.
    # If left blank, the username on the local server will be used to authenticate against the remote server.
    # If these start with ftp:/ftpes:/ftps: then the next variables are ftpuserid:ftppassword:servername:ACL_location
    # These should be of the form "/path/to/your/website/folder/.well-known/acme-challenge"
    # where "/path/to/your/website/folder/" is the path, on your web server, to the web root for your domain.
    # ftp: uses regular ftp; ftpes: ftp over explicit TLS (port 21); ftps: ftp over implicit TLS (port 990).
    # ftps/ftpes support FTPS_OPTIONS, e.g. to add "--insecure" to the curl command for hosts with self-signed certificates.
    # You can also user WebDAV over HTTPS as transport mechanism. To do so, start with davs: followed by username,
    # password, host, port (explicitly needed even if using default port 443) and path on the server.
    # Multiple locations can be defined for a file by separating the locations with a semi-colon.
    ACL=('/var/www/www.mondomaine.com/.well-known/acme-challenge') # Indiquer ici le répertoire où sera copié le token (c'est le répertoire créé précédemment). Ne pas oublier de clôturer la parenthèse.
    #     'ssh:server5:/var/www/www.mondomaine.com/web/.well-known/acme-challenge'
    #     'ssh:sshuserid@server5:/var/www/www.mondomaine.com/web/.well-known/acme-challenge'
    #     'ftp:ftpuserid:ftppassword:www.mondomaine.com:/web/.well-known/acme-challenge'
    #     'davs:davsuserid:davspassword:{DOMAIN}:443:/web/.well-known/acme-challenge'
    #     'ftps:ftpuserid:ftppassword:www.mondomaine.com:/web/.well-known/acme-challenge'
    #     'ftpes:ftpuserid:ftppassword:www.mondomaine.com:/web/.well-known/acme-challenge')

    # Specify SSH options, e.g. non standard port in SSH_OPTS
    # (Can also use SCP_OPTS and SFTP_OPTS)
    # SSH_OPTS=-p 12345

    # Set USE_SINGLE_ACL="true" to use a single ACL for all checks
    #USE_SINGLE_ACL="false"

    # Preferred Chain - use an different certificate root from the default
    # This uses wildcard matching so requesting "X1" returns the correct certificate - may need to escape characters
    # Staging options are: "(STAGING) Doctored Durian Root CA X3" and "(STAGING) Pretend Pear X1"
    # Production options are: "ISRG Root X1" and "ISRG Root X2"
    #PREFERRED_CHAIN="\(STAGING\) Pretend Pear X1"

    # Uncomment this if you need the full chain file to include the root certificate (Java keystores, Nutanix Prism)
    #FULL_CHAIN_INCLUDE_ROOT="true"

    # Location for all your certs, these can either be on the server (full path name)
    # or using ssh /sftp as for the ACL
    DOMAIN_CERT_LOCATION="/etc/nginx/ssl/www.mondomaine.com/www.mondomaine.com.crt" # Indiquer sur ces trois lignes l'emplacement de destination où seront généré le certificat et sa clé privée. Le répertoire doit exister. 
    DOMAIN_KEY_LOCATION="/etc/nginx/ssl/www.mondomaine.com/www.mondomaine.com.key"
    CA_CERT_LOCATION="/etc/nginx/ssl/www.mondomaine.com/chain.crt"
    #DOMAIN_CHAIN_LOCATION="" # this is the domain cert and CA cert
    #DOMAIN_PEM_LOCATION="" # this is the domain key, domain cert and CA cert

    # The command needed to reload apache / nginx or whatever you use.
    # Several (ssh) commands may be given using a bash array:
    # RELOAD_CMD=('ssh:sshuserid@server5:systemctl reload httpd' 'logger getssl for server5 efficient.')
    #RELOAD_CMD=""

    # Uncomment the following line to prevent non-interactive renewals of certificates
    #PREVENT_NON_INTERACTIVE_RENEWAL="true"

    # Define the server type. This can be https, ftp, ftpi, imap, imaps, pop3, pop3s, smtp,
    # smtps_deprecated, smtps, smtp_submission, xmpp, xmpps, ldaps or a port number which
    # will be checked for certificate expiry and also will be checked after
    # an update to confirm correct certificate is running (if CHECK_REMOTE) is set to true
    #SERVER_TYPE="https"
    #CHECK_REMOTE="true"
    #CHECK_REMOTE_WAIT="2" # wait 2 seconds before checking the remote server

Executer getssl suivi du domaine ou sous-domaine pour lequel on souhaite un certificat : 

..  code-block:: shell

    /root/scripts/getssl www.mondomaine.com

..  code-block:: shell

    Registering account
    Verify each domain
    Verifying www.mondomaine.com
    copying challenge token to /var/www/www.mondomaine.com/.well-known/acme-challenge/ZkFYnTHgj6n0Vl1dcekvwyOwoNEUQ3xXrRZFaA0tKRs
    Pending
    Verified www.mondomaine.com
    Verification completed, obtaining certificate.
    Certificate saved in /root/.getssl/www.mondomaine.com/www.mondomaine.com.crt
    The intermediate CA cert is in /root/.getssl/www.mondomaine.com/chain.crt
    copying domain certificate to /etc/nginx/ssl/www.mondomaine.com/www.mondomaine.com.crt
    copying private key to /etc/nginx/ssl/www.mondomaine.com/www.mondomaine.com.key
    copying CA certificate to /etc/nginx/ssl/www.mondomaine.com/chain.crt
    getssl: www.mondomaine.com - certificate obtained but certificate on server is different from the new certificate


A ce stade et si il n’y a pas eu d’erreurs, le certificat, sa clé privée et la chaine de certification ont été générés et placés dans le répertoire spécifié précédemment dans le fichier de configuration.

Mettre en place un renouvellement automatique de ce certificat (ici tous les dimanches à 00:00) :

..  code-block:: shell

    crontab -e

    0 0 * * 0 /root/scripts/getssl -a

Le paramètre **-a** de getssl tentera de renouveler tous les certificats qui ont été générés. Pour éviter l’abus de renouvellement et d’être bloqué par Let’s Encrypt, getssl ne renouvellera un certificat uniquement si celui-ci expire dans moins de 30j. Inutile donc de planifier la crontab tous les jours.

Passage du site en HTTPS
========================

Maintenant que le certificat pour **www.mondomaine.com** est généré, il peut être utilisé par un vhost écoutant sur le port **443**.

La bonne pratique étant que le vhost 80 **redirige** tout le traffic vers le vhost **443**. Si une telle redirection est en place, pour les renouvellements de certificats il faudra que le répertoire **.well-known/acme-challenge/** soit diffusé par le vhost **443** (et non plus par le vhost 80).

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
