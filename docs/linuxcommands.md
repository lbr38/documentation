<h1>Commandes Linux</h1>

<h2>Rechercher dans l’arborescence</h2>

<h3>find</h3>

Parcourir les fichiers à la recherche d’une chaîne de caractères « toto » et afficher ces fichiers :

```
find /etc -type f -exec grep -Hn "toto" {} \;
```

Trouver les gros fichiers (1Go ou plus) :

```
find / -xdev -type f -size +1G -ls
```

<h2>Opérations sur des variables, dans des fichiers, ou sur des chaines de caractères</h2>

<h3>cat</h3>

Afficher les caractères spéciaux invisibles d’un fichier :

```
cat -v fichier
```

Ajouter l’option -T pour également afficher les tabulations :

```
cat -v -T fichier
```

<h3>sed</h3>

Options :

<b>-i</b> : appliquer les modifications dans le fichier indiqué

Caractères qui ont besoin d’être échappés :

```
$.*[\]^
```

Caractères qu’il n’est pas utile d’échapper :

```
Lettres, chiffres et (){}+?|
```

Rechercher/remplacer dans un fichier :

```
sed "s/occurence/remplacement/g" monfichier
```

Alternative : on peut utiliser des <b>|</b> ou un autre caractère, afin de ne pas avoir à échapper les slashs :

```
sed "s|occurence|remplacement|g" monfichier
```

Remplacer une occurrence dans un fichier par une ligne vide :

```
sed "s/occurence//g" monfichier
```

Supprimer une ligne d’un fichier sans laisser de vide (retour chariot) :

```
sed '/occurence/d' monfichier
```

Remplacer deux ou plusieurs lignes vides par une seule :

```
sed '/^$/N;/^\n$/D' monfichier
```

Supprimer toutes les lignes vides d’un fichier :

```
sed '/^$/d' monfichier
```

Supprimer tous les commentaires d’un fichier :

```
sed '/^#/d' monfichier
```

Afficher une partie d’un fichier (en définissant un pattern de début et un pattern de fin) :

```
sed -n -e '/patterndebut/,/patternfin/p'
```

Supprimer un pattern et toutes les lignes qui suivent jusqu’à rencontrer une ligne vide :

```
sed '/^pattern/,/^$/{d;}' monfichier
```

Remplacer un bloc entier par un autre en définissant un pattern de début, un pattern de fin et le bloc à insérer :

```
sed '/pattern-debut/,/pattern-fin/c\
ligne1\
ligne2\
ligne3\
ligne4\
ligne5\' monfichier
```

Insérer une ligne avant un pattern :

```
sed '/^pattern/i maligne' monfichier
```

<h3>awk</h3>

Scinder une chaîne en 2 ou plusieurs parties, en fonction d’un caractère de séparation et afficher le terme souhaité

```
var="terme1:terme2"
```

Le caractère de séparation est ":", on le définit avec l'option -F

```
echo "$var" | awk -F: '{print $1}'
terme1
```
```
echo "$var" | awk -F: '{print $2}'
terme2
```

<h3>grep</h3>

Compter le nombre d’occurrences trouvées par grep :

```
grep -c "occurence" monfichier
```

<h2>Formatage</h2>

<h3>Supprimer des caractères au début d’une variable</h3>

Exemple : www.toto.com

Supprimer les www. :

```
NDD="www.toto.com"
NDD=$(echo "${NDD#www.}")
```

<h3>Supprimer des caractères à la fin d’une variable</h3>

Exemple : www.toto.com

Supprimer .com :

```
NDD="www.toto.com"
NDD=$(echo "${NDD%.com}")
```

<h2>Encodage</h2>

<h3>Voir l’encodage d’un fichier</h3>

```
file -bi FICHIER
text/x-shellscript; charset=iso-8859-1
```

<h3>Modifier l’encodage d’un fichier</h3>

-f : format source

-t : format cible

```
iconv -f iso-8859-1 -t utf-8 -c FICHIER
```

<h2>Locales</h2>

Exécuter une commande avec une locale différente (exemple avec date) :

```
LC_ALL="fr_FR.UTF-8" date +%A
```

<h2>Vim</h2>

Toutes les commandes ci-dessous s'effectuent en dehors du mode insertion (ECHAP)

Rechercher un terme dans un fichier avec vim :

```
/toto
utiliser n pour aller au terme suivant, et N pour aller au terme précédent
```

Se rendre à la ligne numéro '123' :

```
:123
```

Afficher ou masquer les numéros de lignes :

```
:set nu
:set nonu
```

Afficher ou masquer les caractères invisibles (tabulations, saut de ligne)

```
:set list
:set nolist
```

Dans vim, remplacer une occurrence par une autre dans tout le fichier :

```
:%s/chaine1/chaine2/g
```

Aller en début de fichier :

```
gg
```

Aller a la fin du fichier :

```
G
```

Supprimer une ligne :

```
dd
```

Copier-coller une ligne :

```
yy (copier)
p (coller)
```

<h2>Espace disque</h2>

Afficher l’espace disponible/utilisé sur les disques :

```
df -h
```

Afficher le nombre d’inodes utilisés :

```
df -i
```

Calculer l’espace utilisé par un fichier ou répertoire :

```
du -hs fichier
```

<h2>Apache</h2>

Tester la conf Apache :

```
apachectl configtest
```

Rechargement d’Apache sans couper les requêtes en cours :

```
service httpd graceful
```

Test de la conf et rechargement d’Apache sans couper les requêtes en cours :

```
apachectl configtest && service httpd graceful
```

Déclarer un Vhost écoutant sur plusieurs IP :

```
NameVirtualHost 192.168.1.1:80 
NameVirtualHost 172.20.30.40:80


<VirtualHost 192.168.1.1:80 172.20.30.40:80>
...
</VirtualHost>
```

<h2>Mysql et base de données</h2>

Changer le mot de passe root :

```
/usr/bin/mysqladmin -u root -p"MOT_DE_PASSE_ACTUEL" password

New password :
```

Modifier la politique de mot de passe de MySQL :

```
mysql>SET GLOBAL validate_password_policy=LOW;
```

<h2>Paquets</h2>

<h3>Debian/Ubuntu</h3>

Lister les paquets installés :

```
dpkg -l
```

Rechercher dans tous les paquets si le paquet php est installé :

```
dpkg -l *php*
```

<h3>Red Hat/CentOS</h3>

Rechercher dans tous les paquets si le paquet php est installé :

```
rpm -qa | grep php
```

Rechercher un paquet par son nom :

```
yum list php
yum list *php*
```

Étendre la recherche à la description :

```
yum search php
```

Obtenir des informations détaillées sur un paquet :

```
yum info php
```

<h2>Curl</h2>

Afficher/tester les entêtes HTTP d’un site web :

```
curl -I https://toto.com
```

<h2>Iptables</h2>

Bloquer / bannir une adresse IP :

```
iptables -I INPUT -s X.X.X.X -j DROP
```

<h2>GPG</h2>

Générer une paire de clés :

```
gpg2 --full-gen-key
gpg (GnuPG) 2.1.11; Copyright (C) 2016 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: répertoire « /root/.gnupg » créé
gpg: nouveau fichier de configuration « /root/.gnupg/dirmngr.conf » créé
gpg: nouveau fichier de configuration « /root/.gnupg/gpg.conf » créé
gpg: le trousseau local « /root/.gnupg/pubring.kbx » a été créé
Sélectionnez le type de clef désiré :
   (1) RSA et RSA (par défaut)
   (2) DSA et Elgamal
   (3) DSA (signature seule)
   (4) RSA (signature seule)
Quel est votre choix ? 1
les clefs RSA peuvent faire une taille comprise entre 1024 et 4096 bits.
Quelle taille de clef désirez-vous ? (2048) 4096
La taille demandée est 4096 bits
Veuillez indiquer le temps pendant lequel cette clef devrait être valable.
         0 = la clef n'expire pas
        = la clef expire dans n jours
      w = la clef expire dans n semaines
      m = la clef expire dans n mois
      y = la clef expire dans n ans
Pendant combien de temps la clef est-elle valable ? (0) 0
La clef n'expire pas du tout
Est-ce correct ? (o/N) o

GnuPG doit construire une identité pour identifier la clef.

Nom réel : Toto
Adresse électronique : toto@tutu.com
Commentaire : 
Vous avez sélectionné cette identité :
    « Toto <toto@tutu.com> »

Changer le (N)om, le (C)ommentaire, l'(A)dresse électronique
ou (O)ui/(Q)uitter ? 
Changer le (N)om, le (C)ommentaire, l'(A)dresse électronique
ou (O)ui/(Q)uitter ? o
De nombreux octets aléatoires doivent être générés. Vous devriez faire
autre chose (taper au clavier, déplacer la souris, utiliser les disques)
pendant la génération de nombres premiers ; cela donne au générateur de
nombres aléatoires une meilleure chance d'obtenir suffisamment d'entropie.
                        
De nombreux octets aléatoires doivent être générés. Vous devriez faire
autre chose (taper au clavier, déplacer la souris, utiliser les disques)
pendant la génération de nombres premiers ; cela donne au générateur de
nombres aléatoires une meilleure chance d'obtenir suffisamment d'entropie.
gpg: /root/.gnupg/trustdb.gpg : base de confiance créée
gpg: clef A1FEA2C7 marquée de confiance ultime.
gpg: répertoire « /root/.gnupg/openpgp-revocs.d » créé
gpg: revocation certificate stored as '/root/.gnupg/openpgp-revocs.d/475B73652786355D54035D2DC636A094A1FEA2C7.rev'
les clefs publique et secrète ont été créées et signées.

gpg: vérification de la base de confiance
gpg: marginals needed: 3  completes needed: 1  trust model: PGP
gpg: profondeur : 0  valables :   1  signées :   0
     confiance : 0 i., 0 n.d., 0 j., 0 m., 0 t., 1 u.
pub   rsa4096/A1FEA2C7 2017-03-19 [S]
      Empreinte de la clef = 475B 7365 2786 355D 5403  5D2D C636 A094 A1FE A2C7
uid        [  ultime ] Toto <toto@tutu.com>
sub   rsa4096/8E46476A 2017-03-19 []
```

Lister les clés :

```
gpg2 --list-keys
```

Exporter la clé PRIVÉE dans un fichier (d’abord récupérer l’ID de la clé à l’aide de la commande précédente) :

```
gpg2 --list-keys

/root/.gnupg/pubring.kbx
------------------------
pub   rsa4096/A1FEA2C7 2017-03-19 [SC]
uid        [  ultime ] Toto <toto@tutu.com>
sub   rsa4096/8E46476A 2017-03-19 [E]


gpg2 --export-secret-keys -a A1FEA2C7 > cle_secrete.key
```

Chiffrer un fichier :

```
gpg2 --output monfichier.gpg --encrypt --recipient toto@tutu.com monfichier
```

<h2>Arborescence et répertoires</h2>

Compter le nombre de fichiers dans un répertoire :

```
ls -l | wc -l
```

Afficher uniquement les noms de fichiers avec ls :

```
ls -A1
```

Vérifier si un répertoire est vide :

```
[ "$(ls -A /chemin/répertoire/)" ] && echo "Pas vide" || echo "Vide"
```

<h2>Commandes spéciales</h2>

<h3>globstar</h3>

Activer / désactiver globstar (récursivité dans les répertoires) :

```
shopt -s globstar
shopt -u globstar (pour désactiver)
```

<h2>Affichage dans le terminal</h2>

Afficher une ligne sur tout le terminal (utile dans les script par exemple), ici il s’agira d’une ligne de caractères ‘=’ :

```
printf '%*s' "${COLUMNS:-$(tput cols)}" '' | tr ' ' '='
```

Explications : pour cela, on va découper la commande :

```
'%*s' → afficher un caractère de type string (%s), l'étoile permet de définir à quelle position sera affiché ce caractère dans le terminal. LA position est définie par le paramètre suivant.
"${COLUMNS:-$(tput cols)}" → lorsque l'étoile (*) est utilisée, ce deuxième paramètre est censé être la position où sera affiché le caractère. En général il s'agit d'un chiffre (ex: 10 pour afficher le caractère après 10 espaces). Dans ce cas précis on calcule le nombre total de colonnes dans le terminal. Le but sera d'afficher le caractère à la fin du terminal (tout à droite).
' ' → c'est le caractère à afficher. Ici rien, la commande va alors afficher une ligne vide sur tout le terminal.
| tr ' ' '=' → la commande tr est une commande de remplacement. Ici tr remplace chaque caractère ' ' (espace ou vide) par un caractère '=' (égal).
```

Pour récapituler : Ici printf va afficher un caractère espace (ou vide) au bout à droite du terminal. Tout ce qui se trouve avant est vide également. Ce caractère et tous les autres vides seront ensuite remplacés par un caractère ‘=’


Couper avant ou après un motif :

Prenons un exemple : root@serveur ; on souhaite ne garder que « serveur », pour cela :

```
cut -d'@' -f2
```

Explication : ici on coupe en deux blocs ce qui se trouve avant et après ‘@’. root étant le premier bloc et serveur le second. Ensuite on choisi de ne garder que le bloc 2 (donc serveur)

<h2>Shell</h2>

Se loguer avec un autre utilisateur et lui assigner temporairement un shell bash :

```
su nginx -s /bin/bash
```

<h2>Virtualisation</h2>

<h3>Proxmox - openVZ</h3>

Lister les containers de l’hôte :

```
pvectl list
```

ou

```
vzlist
```

Modifier les specs d’un container (depuis l’hôte) :

Modifier espace disque (ici exemple avec 100Go) :

```
pvectl set CTID -disk 100
```

Modifier mémoire RAM (ici exemple avec 4Go de RAM) :

```
pvectl set CTID -memory 4096
```

Modifier nombre de CPU (ici exemple avec 2 CPU) :

```
pvectl set CTID --cpus 2
```

Ajouter une IP à un container :

```
vzctl set CTID --ipadd X.X.X.X --save
```

Supprimer une IP d’un container :

```
vzctl set CTID --ipdel X.X.X.X --save
```

Démarrer un container :

```
vzctl start CTID
```

Entrer dans un container depuis l’hôte :

```
vzctl enter CTID
```

<h2>Ansible</h2>

<h3>Variable ansible</h3>

Hostname de la machine :

```
{{ ansible_fqdn }}
```

IP de la machine :

```
ansible_default_ipv4.address
```

<h2>Templates Jinja</h2>

Concerne les fichiers de templates .j2

Appliquer du code si le nom de mon serveur commence par ^frontal-

```
{% if hostvars[inventory_hostname]['inventory_hostname_short'] | regex_search('^frontal-') %}
code
{% endif %}
```

La même chose si le nom de mon serveur commence par ^frontal- OU ^backoffice-

```
{% if hostvars[inventory_hostname]['inventory_hostname_short'] | regex_search('^frontal-') or regex_search('^backoffice-') %}
code
{% endif %}
```

Appliquer du code si une variable spécifique a été définie sur mon serveur :

```
{% if serveur_type == "frontal" %}
mon_code
{% endif %}
```

D’autres exemples de conditions dans un fichier jinja :

https://stackoverflow.com/questions/25552766/change-variable-in-ansible-template-based-on-group

https://stackoverflow.com/questions/3842690/in-jinja2-how-do-you-test-if-a-variable-is-undefined

https://automatisation.pressbooks.com/chapter/templates/