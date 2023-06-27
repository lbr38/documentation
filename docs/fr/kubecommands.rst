==========
Kubernetes
==========

Pré-requis
==========

Swap
----

Désactiver le swap sans quoi kubelet ne démarrera pas :

..  code-block:: shell

    swapoff -a
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

SELinux
-------

Sur un système RHEL/CentOS, désactiver SELinux :

..  code-block:: shell

    setenforce 0
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

Kubectl
=======

Lister
------

Lister les nodes

..  code-block:: shell

    kubectl get nodes -A

Lister les pods

..  code-block:: shell
    
    kubectl get pods -A

Lister les persistent volumes (pv)

..  code-block:: shell

    kubectl get pv -A

Lister les containers à l'intérieur d'un pod

..  code-block:: shell

    kubectl get pods pod-xxx -o jsonpath='{.spec.containers[*].name}'

Décrire une ressource
---------------------

Décrire un node

..  code-block:: shell

    kubectl describe node XXXX

Décrire un pod

..  code-block:: shell

    kubectl describe pod XXXX

Lire les logs
-------------

Lire les logs d'un pod

..  code-block:: shell

    kubectl logs pod-xxxxxx

Lire les évènements

..  code-block:: shell

    kubectl get events



(WIP)