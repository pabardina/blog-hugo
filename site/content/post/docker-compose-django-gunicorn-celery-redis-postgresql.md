+++
date = "2015-05-20T19:37:00+02:00"
draft = false
title = "Docker : Django, Gunicorn, Celery, Redis, PostgreSQL"
slug = "docker-compose-django-gunicorn-celery-redis-postgresql"
aliases = [
	"docker-compose-django-gunicorn-celery-redis-postgresql"
]
+++
Développement Django avec Docker Compose.
La stack utilisée pour l'exemple : 

* Django
* PostgreSQL
* Gunicorn
* Celery
* Nginx
* Redis 
* Supervisor 

[Git du projet ](https://github.com/pabardina/docker-compose-django)

Docker ?
----
> Docker est un outil qui peut empaqueter une application et ses dépendances dans un conteneur virtuel, qui pourra être exécuté sur n'importe quel serveur Linux.  
[Docker](https://www.docker.com) (documentation officielle)  
[Débuter avec Docker](http://www.it-wars.com/debuter-avec-docker/)
 
![](/images/2015/05/big-docker-logo-300x267.png)

<br/>
Docker Compose
---
> Compose est un outil pour définir et exécuter des applications complexes avec Docker. Avec Compose, vous définissez une application multi-conteneur dans un seul fichier, puis tourner votre application en une seule commande qui s'occupe de tout pour le faire marcher.  
[Docker Compose](https://docs.docker.com/compose/)

<br/>
### Dockerfile
Le Dockerfile du projet va créer un conteneur avec l'application Django,  installer ses dépendances ainsi que gérer Gunicorn et Celery avec Supervisor.

```
FROM ubuntu:14.10

ENV PYTHONUNBUFFERED 1

WORKDIR /code

RUN apt-get update && apt-get upgrade -qq
RUN apt-get install -y build-essential git
RUN apt-get install -y python python-dev python-setuptools python-software-properties libpq-dev libxml2 gcc libxslt1-dev vim
RUN apt-get install -y supervisor
RUN apt-get -y autoremove
RUN easy_install pip

RUN mkdir -p /var/log/supervisor /etc/supervisor/conf.d
ADD docker_config/supervisor.conf /etc/supervisor/conf.d/myconf.conf

ADD requirements.txt /code/requirements.txt

RUN pip install -r /code/requirements.txt
```
<br/>
### Utilisation d'un conteneur de données

Un conteneur de données va permettre de stocker des données utilisables par plusieurs conteneurs.   
Deux bonnes ressources : 

* [data container](https://docs.docker.com/userguide/dockervolumes/#creating-and-mounting-a-data-volume-container)  (documentation officielle)
* [un bon discours](https://medium.com/@ramangupta/why-docker-data-containers-are-good-589b3c6c749e)

Il n'est pas possible de créer un container de données avec Compose (bientôt ?).
<br/>Ainsi, pour le créer manuellement :

`docker create -v /data --name data ubuntu:14.10 /bin/true`

J'utilise ubuntu:14.10, c'est une image que j'utilise déjà avec un autre conteneur, cela m'évite d'utiliser de l'espace disque pour rien. 
Le conteneur de données est utilisé pour stocker le "static" du projet Django, ainsi que les fichiers de redis (persistance de données pour Celery). 
<br/>
### COMPOSE

`docker-compose.yml`

```
db:
  image: jamesbrink/postgresql
  environment:
    - SCHEMA=example
    - USER=example
    - PASSWORD=password
    - POSTGIS=true

rediscache:
  image: redis:latest

rediscelery:
  image: redis:latest
  volumes_from:
    - data

app:
  build: .
  command: /usr/bin/supervisord -n
  volumes:
    - ./docker_config/supervisor.conf:/etc/supervisor/conf.d/myconf.conf
    - ./docker_config/start.sh:/code/start.sh
    - ./docker_config/start_cel.sh:/code/start_cel.sh
    - ./project:/code
  volumes_from:
    - data
  links:
    - db
    - rediscache
    - rediscelery

nginx:
  image: nginx:latest
  volumes:
    - ./docker_config/myconf.nginx:/etc/nginx/nginx.conf
  volumes_from:
    - data
  ports:
    - "80:80"
  links:
    - app
```

Nous avons donc 5 conteneurs dans le fichier (6 avec le conteneur de données créé plus haut)  :

* app avec :  
     *  Django
     *  Gunicorn
     *  Celery
* db avec PostgreSQL
* Nginx 
* rediscache pour le cache de Django (sessions...)
* rediscelery broker celery avec persistance de données

### Lancement

Pour créer les conteneurs :  
`docker-compose up -d` (-d pour lancer en mode détaché) 

Pour démarrer les conteneurs déjà créés, il faut utiliser `docker-compose start`, si vous utilisez `up`, ils vont se recréer.

Il est possible de vouloir recréer un conteneur spécifique :  
`docker-compose up --no-deps container_name` (--no-deps signifie que l'on ne veut pas recréer les conteneurs liés)
<br/>
#### Comment utiliser les conteneurs entre eux ?  
Pour pouvoir utiliser par exemple PostgreSQL ou Redis depuis mon conteneur `app`, il n'y a pas besoin de connaître l'ip. On peut utiliser juste le nom du conteneur.
`'HOST': 'db'` ou `BROKER_URL = 'redis://rediscelery:6379/0'`

 En effet, Compose remplit le fichier hosts des conteneurs.

```
root@a230dcd1741c:/code# cat /etc/hosts
127.0.0.1	localhost
172.17.0.68	rediscelery_1
172.17.0.69	dockercomposeexample_db_1
172.17.0.70	rediscache
172.17.0.70	rediscache_1
172.17.0.68	rediscelery
172.17.0.69	db
172.17.0.69	db_1
172.17.0.70	dockercomposeexample_rediscache_1
172.17.0.68	dockercomposeexample_rediscelery_1
```

Il est bien entendu possible d'utiliser moins de conteneurs. J'ai choisi d'avoir deux conteneurs redis parce qu'ils ont une configuration différente, choix discutable. Il est également possible d'avoir Nginx dans le conteneur "app". Néanmoins, si vous voulez utiliser Nginx comme load balancer entre plusieurs conteneurs de votre application, il faut le laisser dans un conteneur à part.
<br/>
### Optimisation

Compose gère la plupart des paramètres de Docker. Ainsi, la gestion des ressources se fait via `mem_limit` pour la mémoire, `cpu_shares`pour le cpu.

Exemple : 

```
db:
  image: jamesbrink/postgresql
  mem_limit:100m
  cpu_shares:50
```
<br/>
### Pour conclure

Pour "entrer" dans un conteneur et avoir un shell ([If you run SSHD in your Docker containers, you're doing it wrong!)](http://jpetazzo.github.io/2014/06/23/docker-ssh-considered-evil/), la façon la plus simple est de lancer cette commande :  
`docker exec -it id_container bash`

Néanmoins, vous trouverez sur ce [site](http://programster.blogspot.fr/2014/01/docker-enter-running-container.html), un petit script pour raccourcir la commande à :  
`docker-enter idcontainer`

C'est quand même plus cool !

[Git du projet ](https://github.com/pabardina/docker-compose-django)

Cet article est susceptible d'être modifié au fur et à mesure des mises à jour de Docker, de Compose et des mes expériences.