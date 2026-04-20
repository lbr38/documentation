=============================================
[Linux] - Repomanager, gestionnaire de dépôts
=============================================

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/repomanager/refs/heads/devel/images/readme/github-readme-black.png">
        <img src="https://raw.githubusercontent.com/lbr38/repomanager/refs/heads/devel/images/readme/github-readme-black.png" align="top"> 
        </a>
    </div> 

    <br>

**Repomanager** est une interface web open-source pour créer, maintenir et exploiter des miroirs de dépôts de paquets **deb** et **rpm**.

Le projet est disponible sur GitHub : https://github.com/lbr38/repomanager

Présentation
------------

Dans un parc de serveurs, gérer les mises à jour de manière homogène peut vite devenir complexe. Repomanager répond à ce besoin en centralisant :

- la création de dépôts miroirs (internes ou synchronisés depuis des sources externes),
- la publication contrôlée des versions de dépôts,
- le suivi de l'état des machines clientes,
- l'exécution de tâches planifiées autour des dépôts

L'objectif est de fiabiliser les déploiements et de mieux maîtriser ce qui est installé sur les environnements (test, preprod, prod, etc.).

.. raw:: html

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/demo.gif">
        <img src="https://assets.repomanager.net/repomanager/index/demo.gif" align="top"> 
        </a>
    </div> 

    <br>

Fonctionnalités principales
---------------------------

- **Gestion de dépôts deb/rpm** : création de dépôts locaux ou miroirs, duplication, suppression, reconstruction des métadonnées.
- **Publication par environnement** : possibilité d'associer un snapshot de dépôt à un environnement (par exemple preprod puis prod).
- **Upload de paquets** : ajout de paquets personnalisés dans un dépôt local via l'interface web.
- **Signature GPG** : signature des paquets et des dépôts pour renforcer l'intégrité et la confiance côté clients.
- **Planification de tâches** : exécution immédiate ou différée de tâches (mise à jour de miroir, rebuild, publication, etc.).
- **Suivi des hôtes** : inventaire des paquets installés, paquets à mettre à jour, historique des opérations.
- **Profils hôtes** : définition de profils de configuration (accès aux dépôts, exclusions de paquets) pour standardiser le parc.
- **Statistiques et monitoring** : visualisation de métriques d'accès aux dépôts et activité globale.

.. raw:: html

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-1.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-1.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-2.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-2.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-3.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-3.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-4.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-4.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-5.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-5.png" align="top"> 
        </a>
    </div> 

    <br>


Cas d'usage type
----------------

Un scénario courant consiste à :

1. Créer un miroir d'une distribution (Debian, Rocky Linux, etc.).
2. Valider les paquets sur un environnement de test.
3. Pointer l'environnement de production vers le snapshot validé.
4. Déclencher ou planifier les mises à jour des hôtes concernés.

Cette approche limite les effets de bord et facilite les retours arrière en cas d'incident.

Installation
------------

Pour les prérequis matériels/logiciels et la procédure d'installation, consulter la documentation officielle :

- Exigences : https://docs.repomanager.net/getting-started/requirements/
- Installation standard : https://docs.repomanager.net/getting-started/installation/

La documentation officielle couvre également la configuration des environnements, les dépôts sources, le reverse proxy, ainsi que l'administration avancée.

Aller plus loin
---------------

- Documentation complète : https://docs.repomanager.net/
- Dépôt GitHub : https://github.com/lbr38/repomanager
- Suivi des issues et demandes d'évolutions : https://github.com/lbr38/repomanager/issues
