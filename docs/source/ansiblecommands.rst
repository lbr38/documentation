=======
Ansible
=======

Variables ansible
=================

Hostname de la machine :

..  code-block:: shell

      {{ ansible_fqdn }}

IP de la machine :

..  code-block:: shell

      ansible_default_ipv4.address

Templates Jinja
===============

Concerne les fichiers de templates .j2

Appliquer du code si le nom de mon serveur commence par ^frontal-

..  code-block:: shell

      {% if hostvars[inventory_hostname]['inventory_hostname_short'] | regex_search('^frontal-') %}
      code
      {% endif %}

La même chose si le nom de mon serveur commence par ^frontal- OU ^backoffice-

..  code-block:: shell

      {% if hostvars[inventory_hostname]['inventory_hostname_short'] | regex_search('^frontal-') or regex_search('^backoffice-') %}
      code
      {% endif %}

Appliquer du code si une variable spécifique a été définie sur mon serveur :

..  code-block:: shell

      {% if serveur_type == "frontal" %}
      mon_code
      {% endif %}

D’autres exemples de conditions dans un fichier jinja :

https://stackoverflow.com/questions/25552766/change-variable-in-ansible-template-based-on-group

https://stackoverflow.com/questions/3842690/in-jinja2-how-do-you-test-if-a-variable-is-undefined

https://automatisation.pressbooks.com/chapter/templates/