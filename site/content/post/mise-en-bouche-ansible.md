+++
date = "2015-05-04T15:28:00+02:00"
draft = false
title = "Mise en bouche d'Ansible"
slug = "mise-en-bouche-ansible"
aliases = [
	"mise-en-bouche-ansible"
]
+++

## Introduction
>Ansible is a radically simple IT automation platform that makes your applications and systems easier to deploy. Avoid writing scripts or custom code to deploy and update your applications— automate in a language that approaches plain English, using SSH, with no agents to install on remote systems.  
<http://www.ansible.com>

Outil pour configurer et gérer des serveurs, avec une configuration simple grâce à des "playbooks" (fichier de configuration), le tout exécuté en SSH.

Quelques avantages d'Ansible :

* Des fichiers en YAML, très facilement utilisable, lisible.
* Une structure bien pensée avec des rôles comprenant des templates, des fichiers, des variables....
* Une simple installation de l'outil sur la machine hôte, tout ce fait en ssh.

<br/>
Le but de cet article va être de déployer une simple application Django (un simple blog par exemple) et de détailler les playbooks d"Ansible.

L'adresse du git <https://github.com/pabardina/ansible-django>
<br/>
### Structure du repo git

A la racine, on trouve trois fichiers :

* hosts : qui contient les ip des serveurs
* vars.yml : qui contient les variables "globales" du projet, c'est à dire nécessaire dans tous les rôles
* start.yml : qui correspond au fichier de base, c'est ce fichier qui sera lancé, et qui appelera les différents rôles

 On retrouve également un dossier "rôles" :

```
roles/
  base/
    tasks/            #
      main.yml      #  <-- fichier lu en premier dans ce rôle, qui va contenir différentes actions
    web/               # <-- web role (django, celery, memcached)
      files/ #
        main.yml #
      handlers/
      tasks/            #
         main.yml      #  <-- fichier lu en premier dans ce rôle, qui va contenir différentes actions
      templates/        #  <-- dossier templates
        zshrc.in   #  <------- template pour le fichier zshrc
      vars/
        main.yml
start.yml
vars.yml
```
<br/>
Extrait du fichier roles/base/tasks/main.yml :

```
- name: Install base packages
  apt: name={{ item }} update_cache={{ update_apt_cache }} force=yes state=installed
  with_items:
    - build-essential
    - ntp
    - htop
    - git
    - tig
    - tmux
    - ncdu
    - python-dev
    - python-pip
    - python-pycurl
    - python3-dev
    - supervisor
  tags: packages
```
Dans cet extrait, on va installer les paquets de la liste, et s'assurant que le gestionnaire de paquets soit à jour.
<br/>

D'autres possibilités existe :

```
- name: Create the application user
  user: name={{ gunicorn_user }} state=present
```
<br/>
Création d'un utilisateur, dont le nom est une variable (gunicorn_user) contenu dans le fichier vars.yml à la racine du repo.

La variable : `gunicorn_user: test`
<br/>

Fichier template (roles/web/templates/local_settings.py) :

```
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql_psycopg2",
        "NAME": "{{ db_name }}",
        "USER": "{{ db_user }}",
        "PASSWORD": "{{ db_password }}",
        "HOST": "localhost",
        "PORT": "",
    }
}
STATIC_ROOT = "/webapps/{{ application_name }}/static"
```
<br/>
### Ansible et Django
Un module Django existe pour Ansible, ce qui permet l'utilisation de manage.py, c'est simple, facile et c'est cool.
<http://docs.ansible.com/django_manage_module.html>

Exemple de `manage.py migrate` dans le fichier /roles/web/tasks/setup_django_app.yml

```
- name: Run Django migrations
  django_manage:
    command: migrate
    app_path: "{{ application_path }}"
    virtualenv: "{{ virtualenv_path }}"
    settings: "{{ django_settings_file }}"
```
<br/>

## Installation et configuration
- Sur la machine hôte :

	```sudo pip install ansible```

- Ensuite, comme dit précédemment, Ansible utilise ssh, il faut donc ajouter notre certificat ssh sur la machine distante :

	```ssh-copy-id root@xx.xx.xx.xx ```

- Ajouter l'ip de votre serveur dans le fichiers ```hosts```

- Un dernier test :

	```ansible all -m ping -u root```

<br/>
## Utilisation

```ansible-playbook -i hosts start.yml -vvv```

Explication de la commande:

- le `-i` permet de spécifier l'endroit du fichier qui contient les ip
- le `-vvv` permet de verbose mode plus détaillé que le simple -v


Repo git basé sur <https://github.com/jcalazan/ansible-django-stack> très bien fait, avec des rôles en plus :

- celery
- memcached
- rabbitmq

N'hésitez pas à y faire un saut, ça vaut le coup.




