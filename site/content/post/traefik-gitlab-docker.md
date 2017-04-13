+++
title = "Utilisation de Trafik pour Portainer et Gitlab"
date = "2017-04-07T09:57:50+02:00"
tags = ["traefik", "docker", "gitlab"]
description = ""

+++

Récemment, j'ai voulu déployer sur un serveur plusieurs applications web (Gitlab, des projets perso) dans des conteneurs Docker. Problème, un conteneur se créé aussi vite qu'il se supprime, et il faut faire attention à la publication des ports.

J'aurai pu utiliser un reverse proxy classique comme Nginx. Le problème est qu'à chaque création ou suppression de conteneur, il faut modifier la configuration de Nginx et redémarrer le service. Pas vraiment dynamique. Il existe des projets comme Nginx-proxy, mais je n'ai pas testé...

Je cherchais donc un outil qui repond à ses besoins :

* Reverse proxy pour avoir plusieurs applications sur les ports  `80 / 443`
* Léger avec une configuration simple
* Gérer plusieurs noms de domaines
* Des certificats valides (Let's Encrypt)
* Load balancer si j'ai plusieus serveurs
* Intégration avec Docker

C'est la qu'intervient Traefik !


# Traefik



Traefik est un reverse proxy HTTP et un load balancer écrit en Go spécialisé dans le déploiement de micro-services. Il est récent et a donc pu éviter les lourdeurs des outils comme Nginx ou HAProxy. Au contraire, on retrouve ici un outil avec une configuration simple, qui peut communiquer avec du Docker, du Swarm, du Kubernetes et plein d'autres...

C'est un simple binaire, avec beaucoup de fonctionnalités dont voici un extrait :

* Une API REST pour communiquer avec lui
* Let's Encrypt avec renouvellement automatique de certificat
* Support des Websockets et de HTTP/2
* Scalable

L'avantage de l'outil est qu'il est automatiquement et dynamiquement au courant des nouveaux conteneurs créés, pas besoin de redémarrer le service.

Dans mon cas, je vais utiliser l'intégration avec Docker. Vous verrez dans la suite de l'article que cela est enfantin. Il suffit lors de la création d'un conteneur de rajouter des labels utilisés par Traefik et la magie opère.


https://traefik.io

J'ai choisi le reverse proxy. Je veux également une interface web pour gérer Docker.



# Portainer  


Il existe plusieurs interfaces web pour gérer Docker. Portainer est probablement la plus aboutie. Légère, elle permet de gérer des hôtes Docker et des clusters Swarm. En plus d'avoir une vision sur l'ensemble de ses ressources Docker, on peut lancer une console pour se connecter sur un conteneur depuis son navigateur. 
On peut également bénéficier de templates de conteneurs pour pouvoir déployer des applications en deux clics. 

![Portainer](https://camo.githubusercontent.com/310898fa1ef36666f110b97f1a3023b88fa9a985/687474703a2f2f706f727461696e65722e696f2f696d616765732f73637265656e73686f74732f706f727461696e65722e676966)


http://portainer.io


# Déploiement

Projet Github : https://github.com/pabardina/docker-traefik-gitlab


Pour déployer Traefik, je vais utiliser Compose de Docker. J'ai fais le choix de lancer Traefik dans un conteneur.

Tout d'abord, la configuration de Traefik se définit dans le fichier `traefik.toml` :

```
# traefik.toml
################################################################
# Global configuration
################################################################

defaultEntryPoints = ["http", "https"] // 1

[entryPoints] // 2
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]

[acme] // 3
email = "xxx@xx.com"
storageFile = "/etc/traefik/acme/acme.json"
entryPoint = "https"
OnHostRule = true
onDemand = true

[[acme.domains]] //
  main = "cab.re"
  sans = ["portainer.cab.re", "traefik.cab.re", "gitlab.cab.re"] 

[[acme.domains]]
  main = "test.cab.re"

[web]
address = ":8080"
[web.auth.basic]
  users = ["pab:$apr1$2ZtdZCue$BHq00nxavh6EFmR9OFAfZ1"]

[docker]
endpoint = "unix:///var/run/docker.sock"
domain = "cab.re"
watch = true
exposedbydefault = true

```

Une configuration simple et lisible.

### Points d'entrées

La première partie correspond aux points d'entrées sur le reverse proxy. 

```
defaultEntryPoints = ["http", "https"] // 1

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
    
```

* Je définis l'accès au reverse proxy en HTTP et en HTTPS
* Le point d'entrée HTTP est sur le port 80
* Ensuite, je redirige toutes les requêtes HTTP en HTTPS
* Le point d'entrée HTTPS est sur le port 443


### Let's Encrypt


> Le protocole ACME (de l'anglais Automatic Certificate Management Environment, littéralement « environnement de gestion automatique de certificat ») est un protocole de communication pour l'automatisation des échanges entre les autorités de certification et les propriétaires de serveur web. Il a été conçu par l’Internet Security Research Group pour leur service Let’s Encrypt.
Wikipédia : https://fr.wikipedia.org/wiki/ACME_(protocole)


```
[acme]
email = "xxx@xx.com" # Adresse email utilisé pour la demande de certificat
storageFile = "/etc/traefik/acme/acme.json" # L'emplacement du fichier contenant les informations des certificats
entryPoint = "https" # Obligatoire et le port HTTPS doit être 443
onDemand = true # La demande de certificat se fera lors de la première requête HTTPS sur votre application

# Permet de demander un certificat à Let's Encrypt pour un domaine et des domaines alternatifs.
# Example, mon application est accessible par test1.cab.re et test2.cab.re 
# Dans ce cas là, une demande de certificat va se faire pour le domaine principal "test1.cab.re". 
# Le domaine alternatif "test2.cab.re" utilisera le certificat de "test1.cab.re".
OnHostRule = true 
```

```

[[acme.domains]]
  main = "cab.re" # Domaine pricinpale
  sans = ["portainer.cab.re", "traefik.cab.re", "gitlab.cab.re"] # domaines alternatifs, ils auront le meme certificat que cab.re 
  
[[acme.domains]]
  main = "test.cab.re" # Ce domaine aura son propre certificat
```

Petite précision sur les `Sans` et les limites de Let's Encrypt. Il est possible de faire 20 demandes de certificats par semaine (https://letsencrypt.org/docs/rate-limits/). Si vous avez un seul nom de domaine, vous pouvez utiliser un certificat global, utilisable par 100 domaines alternatifs (avec les `Sans`, comme dans ma configuration).

Par example, dans mon cas, j'utilise le nom de domaine `cab.re`,  mes sous domaines (portainer, traefik, gitlab) utiliseront le certificat principal. En revanche, tous les autres sous domaines comme `example.cab.re` auront leur propre certificat.

### Suite de la configuration

```
[web]
address = ":8080" # Traefik a une interface web
[web.auth.basic] # Simple authentification
  users = ["pab:$apr1$2ZtdZCue$BHq00nxavh6EFmR9OFAfZ1"]

# Backend utilisé :
[docker]
endpoint = "unix:///var/run/docker.sock" # Socket unix ou TCP possible 
domain = "cab.re" # Le domaine par défault utilisé, les conteneurs auront un nom `CONTAINER_NAME.cab.re`
watch = true # Traefik sera au courant des qu'il y a dès changements dans Docker

# Tous les conteneurs seront utilisables par Traefik
# Pour qu'il ne le soit pas, il est nécessaire d'ajouter le label "traefik.enable=false" 
# lors de la création du conteneur
exposedbydefault = true 
```

Maintenant que Traefik a sa configuration, il ne reste plus qu'à le lancer avec Compose.

Le fichier qui va installer Traefik et Portainer `docker-compose.yml`:

```
version: '2'

services:
  proxy:
    image: traefik
    networks:
      - traefik
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    volumes:
      -  "$PWD/traefik.toml:/etc/traefik/traefik.toml"
      -  "$PWD/acme:/etc/traefik/acme"
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    labels:
      - "traefik.frontend.rule=Host:traefik.cab.re"
      - "traefik.port=8080"
      - "traefik.backend=traefik"
      - "traefik.frontend.entryPoints=http,https"

  portainer:
    image: portainer/portainer
    networks:
      - traefik
    labels:
      - "traefik.frontend.rule=Host:portainer.cab.re"
      - "traefik.port=9000"
      - "traefik.backend=portainer"
      - "traefik.frontend.entryPoints=http,https"
    volumes:
        - "/var/run/docker.sock:/var/run/docker.sock"
    restart: unless-stopped

networks:
  traefik:
    external:
      name: traefik
```

Je ne vais pas décrire ce qu'est Compose, il y a de très bonne documentation sur internet.  Dans ce fichier, deux conteneurs, Traefik et Portainer vont être créés. 

Pour le conteneur Traefik, il est nécessaire de publier les ports `80,443` pour pouvoir utiliser sa fonction de reverse proxy, ainsi que le port `8080` afin d'accéder à son interface web.

Pour utiliser Traefik avec des conteneurs, il faut utiliser les labels de Docker. Par example, pour Portainer :

```
# L'adresse sur laquelle l'utilisateur accède à l'application
- "traefik.frontend.rule=Host:portainer.cab.re" 

- "traefik.port=9000" # Le port de l'application dans le conteneur 
- "traefik.backend=portainer" # Utile pour le load balancing
- "traefik.frontend.entryPoints=http,https" # Accessible en HTTP et HTTPS
```

Pour lancer la stack :

`docker-compose up -d`

A partir d'ici, vous pouvez accéder à Traefik et Portainer sur les domaines que vous avez indiqué.

Pour ma part : https://traefik.cab.re et https://portainer.cab.re

### Gitlab

Maintenant, déployons Gitlab, toujours avec Compose de Docker.

J'ai utilisé la stack https://github.com/sameersbn/docker-gitlab 

Note : Changez la variable d'environnement dans le fichier `docker-compose.yml` :  `GITLAB_HOST=gitlab.cab.re`


L'avantage d'utiliser Traefik est que vous pouvez utiliser vos fichiers Compose si vous avez déjà d'existant. Il y a simplement des labels et un réseau Docker à rajouter :

```
# Pour les conteneurs qui n'ont pas besoin d'être accessible par Traefik (PostgreSQL, Redis):

redis:
  labels:
    - "traefik.enable=false"
  
# Pour Gitlab :

gitlab:
  [...]
  networks:
    - gitlab
    - traefik
  labels:
    - "traefik.frontend.rule=Host:gitlab.cab.re"
    - "traefik.port=80"
    - "traefik.backend=gitlab"
    - "traefik.frontend.entryPoints=http,https"
    - "traefik.docker.network=traefik"
  
  
networks:
  gitlab:
    driver: bridge
  traefik:
    external:
      name: traefik
```

Pour que Traefik communique avec Gitlab, il faut qu'il soit sur le même réseau et que le label qui indique le réseau Docker à utiliser soit présent. Dans le conteneur Gitlab, il y aura donc deux réseaux, et un label :

* Son réseau de base pour communiquer avec Redis et PostgreSQL
* Le réseau de Traefik
* `traefik.docker.network=traefik`


Pour lancer la stack :

`docker-compose -f gitlab/docker-compose.yml up -d`



# Load balancer


Traefik gère facilement le load balancing. Mon déploiement ne se fait que sur un seul serveur, mais il est facile de faire une démonstration. Prenons cette application qui affiche l'ID du conteneur dans lequel elle est lancée :

```
services:
  example:
    image: pabardina/gohostname
    networks:
      - testnetwork
      - traefik
    labels:
      - "traefik.frontend.rule=Host:test.cab.re"
      - "traefik.port=8000"
      - "traefik.backend=example"
      - "traefik.frontend.entryPoints=http,https"
      - "traefik.docker.network=traefik"

networks:
  testnetwork:
    driver: bridge
  traefik:
    external:
      name: traefik

```

```
➜  ~ curl https://test.cab.re
Hostname: 21e68dc6383d

# Maintenant, si on scale l'application avec Compose :

➜  ~ docker-compose scale example=4
Creating and starting cloud_example_2 ... done
Creating and starting cloud_example_3 ... done
Creating and starting cloud_example_4 ... done

➜  ~ curl https://test.cab.re
Hostname: 5025449388ae
➜  ~ curl https://test.cab.re
Hostname: 21e68dc6383d
➜  ~ curl https://test.cab.re
Hostname: 95e9ecce9297%

```

# Le mot de la fin

Traefik est un super outil, facile à utiliser. Si vous avez besoin d'un reverse proxy pour vos micro-services je ne peux que vous le conseiller.
