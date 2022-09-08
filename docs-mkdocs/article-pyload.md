<h1>Pyload un gestionnaire de téléchargement en python</h1>

![pyload](https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/pyload/pyload.png) 

**Pyload** est un gestionnaire de téléchargement direct (à l’instar de **Jdownloader**) développé en **python**.

Il est administrable depuis une **interface web** et ne nécessite pas forcément une interface graphique pour fonctionner.

Cet article décrit une installation sur serveur headless ARM (type Raspberry Pi) basé sur **Ubuntu-minimal 22.04**.

![pyload-ui](https://raw.githubusercontent.com/lbr38/documentation/main/docs/images/pyload/pyload-ui.png)

##Pré-requis

- Un **terminal** ouvert sur le serveur sur lequel sera installé pyload, logué en **root**.
- Avoir désactivé **IPv6** sur le serveur car l’installation décrite ne prend pas en charge IPv6. Vous trouverez facilement sur Internet les commandes permettant de désactiver IPv6.
- Être à l’aise sous **Linux**. Chacun possède son propre système et sa configuration, par exemple ici je suis sur **Ubuntu** sur **ARM** et j’utilise **nginx**, peut être que vous êtes sur CentOS et que vous utilisez apache. Je ne peux pas détailler tous les cas de figures et prendre en compte toutes les erreurs rencontrées par chacun pendant l’installation, il faut savoir s’adapter. Je reste néanmoins disponible dans l’espace commentaire si vous êtes bloqué.

##Préparation

Exécuter les commandes suivantes en tant que **root**.

- Créer un utilisateur **pyload** dédié à l’exécution de **pyload** :

```
useradd -s /usr/sbin/nologin -d /home/pyload -m pyload
```

Installer quelques dépendances pour pyload et notamment de quoi créer un virtual env python dans lequel pyload et ses dépendances seront installés :

```
apt install python3-venv libjpeg8-dev
```

## Installation

En tant que **root**, se loguer en tant que **pyload** :

```
su pyload -s /bin/bash
```

Installer le compilateur **Rust** sans quoi l’une des dépendances python de pyload ne s’installera pas. Choisir l’installation par défaut lorsque c’est demandé. L’installateur installera tout le nécessaire dans le home directory de pyload **/home/pyload**.

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh


Welcome to Rust!

This will download and install the official compiler for the Rust
programming language, and its package manager, Cargo.

Rustup metadata and toolchains will be installed into the Rustup
home directory, located at:

/home/pyload/.rustup

This can be modified with the RUSTUP_HOME environment variable.

The Cargo home directory is located at:

/home/pyload/.cargo

This can be modified with the CARGO_HOME environment variable.

The cargo, rustc, rustup and other commands will be added to
Cargo's bin directory, located at:

/home/pyload/.cargo/bin

This path will then be added to your PATH environment variable by
modifying the profile files located at:

/home/pyload/.profile
/home/pyload/.bashrc

You can uninstall at any time with rustup self uninstall and
these changes will be reverted.

Current installation options:


default host triple: armv7-unknown-linux-gnueabihf
default toolchain: stable (default)
profile: default
modify PATH variable: yes

1) Proceed with installation (default)
2) Customize installation
3) Cancel installation
>1
```

Puis prendre en compte l’installation de **Rust** en chargeant quelques variables d’environnement :

```
source "$HOME/.cargo/env"
```

Créer un environnement virtuel dans lequel on isolera l’installation de pyload et ses dépendances :

```
cd /home/pyload
python3 -m venv pyload
```

Puis se placer à l’intérieur de cet environnement virtuel afin de débuter l’installation de pyload :

```
. ./pyload/bin/activate
```

Le terminal devient alors préfixé par le nom de l’environnement virtuel (pyload). 

Installer **wheel** si ce n’est pas déjà fait, puis installer le package **pyload-ng** :

```
pip3 install wheel
pip3 install --pre pyload-ng[all]
```

Si des erreurs surviennent lors de l’installation il est fort probable que cela provienne de dépendances manquantes. Généralement une recherche sur Internent sur l’erreur rencontrée permet de s’en sortir.

Sortir de l’environnement virtuel lorsque terminé :

```
deactivate
```

## Démarrage de pyload

Toujours en étant logué en tant que **pyload**, exécuter le binaire pyload en mode **daemon** :

```
/home/pyload/pyload/bin/pyload --daemon
```

Un **serveur web** embarqué est alors lancé et écoute sur **http://127.0.0.1:8000**

Vous pouvez vous arrêter là si vous utilisez pyload en local, il suffit d’ouvrir **http://127.0.0.1:8000** sur un navigateur.

Sinon c’est le moment de mettre en place un **reverse proxy nginx** afin d’accéder à l’interface web depuis l’extérieur.

## Configuration de nginx

C’est le reverse proxy **nginx** qui fera office d’intermédiaire entre le serveur web embarqué de pyload et le navigateur web.

Le serveur web de pyload écoute en local sur le port **8000**, le reverse proxy se chargera de rediriger les requêtes vers ce port.

Je ne vais pas entrer dans les détails concernant la configuration générale de nginx. Je détaille ici uniquement la mise en place du **vhost** faisant office de reverse proxy.

Aussi il me semble essentiel de posséder un **nom de domaine** pour accéder à pyload depuis le web. Si vous n’avez pas de nom de domaine, vous pouvez en acheter un chez OVH (les .ovh ne sont vraiment pas cher, environ 3€/an). C’est toujours possible de faire sans mais il faudra bidouiller son fichier /etc/hosts. 

Ici pour l’exemple j’utiliserai **dl.mondomain.com**

### Vhost 80

Si ce n’est pas déjà fait, installer nginx :

```
apt install nginx
```

Créer un nouveau vhost dans sites-available (attention si vous n’êtes pas sur une distribution basée sur Debian il est possible que ce répertoire n’existe pas et que les vhosts doivent être placés ailleurs) :

```
sudo vim /etc/nginx/sites-available/reverse-proxy-pyload.conf
```

```
server {
	listen 80;
	server_name dl.mondomaine.com;

	# Forcer https
	# return 301 https://$server_name$request_uri; # Commenter cette ligne qu'on gardera pour plus tard
	root /var/www/dl.mondomaine.com;

	access_log /var/log/nginx/dl.mondomaine.com_access.log;
	error_log /var/log/nginx/dl.mondomaine.com_error.log;
}
```

Activer ce nouveau vhost :

```
cd /etc/nginx/sites-enabled/
ln -s ../sites-available/reverse-proxy-pyload.conf
```

Tester la configuration, nginx ne doit pas retourner d’erreur :

```
sudo nginx -t
```

Redémarrer le service :

```
service nginx restart
```

A ce stade et sous réserve que le paramétrage DNS et les redirections de ports de votre box sont en place, le vhost devrait fonctionner et votre navigateur devrait afficher la page d’accueil nginx ou au moins une page blanche.. mais pas d’erreur 404 ou autre.

Nous reviendrons plus tard pour la configuration du SSL (https) du reverse proxy car il faut d’abord commander un certificat, ce que nous allons faire tout de suite.

### Certificat Let’s Encrypt 

J’ai déjà créé un article sur **getssl**, un script bash qui permet de commander un certificat SSL. Pour éviter les doublons, je vous invite à suivre cet article jusqu’à la fin et de commander un certificat pour le nom de domaine **dl.mondomaine.com**.

Lien vers l’article : [Getssl](article-getssl.md)

A ce stade, vous devriez exécuter la commande suivante pour commander votre certificat (exemple) : 

```
./getssl dl.mondomaine.com
```

Maintenant que nous avons un certificat SSL, la mise en place du vhost '**https**' devient alors possible.

D’abord, il faut limiter le vhost 80 à **renvoyer** vers le vhost 443, c’est tout ce qu’il devra faire. Editer le vhost précédemment créé :

```
vim /etc/nginx/sites-available/reverse-proxy-pyload.conf
```

Et décommenter la ligne précédemment commentée, afin de **rediriger toutes les requêtes** sur le port 80 (http) vers le port 443 (https) :

```
server {
	listen 80;
	server_name dl.mondomaine.com;

	# Forcer https
	return 301 https://$server_name$request_uri;
	root /var/www/dl.mondomaine.com;

	access_log /var/log/nginx/dl.mondomaine.com_access.log;
	error_log /var/log/nginx/dl.mondomaine.com_error.log;
}
```

### Vhost 443

Ceci étant fait, créer le nouveau vhost écoutant sur le port 443 :

```
vim /etc/nginx/sites-available/reverse-proxy-pyload_ssl.conf
```

C’est ce vhost qui fera office de reverse proxy et qui renverra les requêtes vers le serveur web embarqué de pyload :

```
upstream pyload { # Défini le groupe de serveurs qui va répondre aux requêtes derrière le reverse proxy. Ici en l’occurrence c'est ce même serveur car pyload écoute en local sur le port 8000
    server 127.0.0.1:8000;
}

server {
    listen 443 ssl;
    server_name dl.mondomaine.com;

    ssl_certificate /etc/nginx/ssl/dl.mondomaine.com/dl.mondomaine.com.crt;
    ssl_certificate_key /etc/nginx/ssl/dl.mondomaine.com/dl.mondomaine.com.key;

    # Add headers to serve security related headers
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains";
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header X-Download-Options noopen;
    add_header X-Permitted-Cross-Domain-Policies none;

    # Racine du site 
    root /var/www/dl.mondomaine.com;

    # Fichiers de logs
    access_log /var/log/nginx/dl.mondomaine.com_ssl_access.log;
    error_log /var/log/nginx/dl.mondomaine.com_ssl_error.log;

    # Ne pas autoriser les robots à indexer le site
    location = /robots.txt {
        deny all;
        log_not_found off;
        access_log off;
    }

    location / {
        include /etc/nginx/proxy_params; # Inclut quelques directives et en-têtes pour les proxys
        proxy_pass http://pyload/;       # On redirige les requêtes vers le groupe de serveurs 'pyload' défini plus haut
    }
}
```

Tester la conf : 

```
nginx -t
```

Si rien n’a été oublié, nginx ne devrait pas retourner d’erreur, redémarrer le service :

```
service nginx restart
```

Tester l’accès dans le navigateur, l’interface de pyload devrait être accessible : https://dl.mondomaine.com

- Utilisateur : **pyload**
- Mot de passe : **pyload** (penser à le modifier)