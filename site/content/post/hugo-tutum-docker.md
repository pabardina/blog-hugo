+++
date = "2015-11-29T14:42:00+02:00"
draft = false
title = "Déploiement continu d'Hugo avec Docker et Tutum"
slug = "deploiement-continu-hugo-docker-tutum"
+++
J'ai commencé mon blog avec Pelican, le générateur de site statique en Python. Plus tard, je suis passé sur Ghost qui me semblait plus intéressant, répondant d'avantage à mes besoins. Depuis que j'ai découvert Docker, j'ai toujours eu l'idée de stocker l'ensemble de mon blog dans un conteneur et ce d'une façon simple. Ainsi, Ghost ne répondait plus à ma problématique, je suis reparti avec les générateurs de sites statiques. Étant donnée les louanges que j'entends sur le langage Go et ses performances, j'ai privilégié mes recherches sur un générateur en Go. C'est là qu'intervient Hugo, un générateur de site statique pure Go, avec une grosse communauté derrière. Très simple en prendre en main, beaucoup de fonctionnalités, léger et parfait pour être dockerizé.

Migration Ghost -> Hugo
======

Rien de plus simple, il suffit d'exporter ses articles depuis l'interface Ghost, et d'utiliser ce script https://github.com/jbarone/ghostToHugo.

Docker
=====

Une fois votre blog configuré (thème, articles...), il est temps de passer à sa dockérization. Pour ma part, j'ai décidé d'utiliser le serveur intégré dans Hugo plutôt que de rajouter Nginx ou Apache. Il me suffit pour le moment.

Mon Dockerfile :

```sh
FROM debian:wheezy

ENV HUGO_VERSION 0.15
ENV HUGO_BINARY hugo_${HUGO_VERSION}_linux_amd64

ADD https://github.com/spf13/hugo/releases/download/v${HUGO_VERSION}/${HUGO_BINARY}.tar.gz /usr/local/
RUN tar xzf /usr/local/${HUGO_BINARY}.tar.gz -C /usr/local/ \
        && ln -s /usr/local/${HUGO_BINARY}/${HUGO_BINARY} /usr/local/bin/hugo \
        && rm /usr/local/${HUGO_BINARY}.tar.gz

ADD site/ /usr/share/blog
WORKDIR /usr/share/blog

EXPOSE 1313

CMD hugo server --watch --baseUrl=${HUGO_BASE_URL} --port=1313 --appendPort=False --bind=0.0.0.0
```

<u>Architecture du repository :</u>

![Architecture blog](/img/docker-tutum-hugo/architecture.png)


On crée l'image :  

`docker build -t blog .`

Et on lance le conteneur :

`docker run -d -p 80:1313 -e HUGO_BASE_URL=http://192.168.99.100 blog` 

<br>

Déploiement Continue
====
<br>
Prérequis :

* un repository github
* un compte Docker Hub
* un compte Tutum (connexion avec le compte Docker Hub)

La première étape pour le déploiement continue va être de créer un "automated build" sur le Hub de Docker et de le lier avec votre repository Github.

![create automated build](/img/docker-tutum-hugo/automated.png)

Choisissez votre répertoire Github, la visibilité de votre image et écrivez une description.

![create automated build](/img/docker-tutum-hugo/create.png)

Pour tester le lien, soit vous pousser une modification sur Github soit vous allez dans **Build Settings** et vous cliquez sur **Trigger**

Un build va être lancé, visible dans **Build Details** :

![test automated build](/img/docker-tutum-hugo/test-build.png)
<br>
De plus, si vous allez dans les settings de votre repository Github ( **Settings -> Webhooks & services**), le lien est bien visible :

![Github link](/img/docker-tutum-hugo/github.png)
<br>
À chaque modification sur Github, l'image sera recrée avec les dernières modifications. Un bon début, mais comment faire pour que mon blog en production se mette à jour tout seul ? 

Tutum
======

Tutum, récemment racheté par Docker Inc est un service cloud pour déployer et gérer des conteneurs. Il permet de gérer plusieurs types de nodes (AWS, DigitalOcean, serveur personnel...). Très facile pour déployer un simple conteneur ou bien une stack plus complète avec du load balancing. Pour ma part, je trouve que c'est un super outil qui pour le moment est grauit puisqu'il est en béta. Reste à voir la politique tarifaire une fois la béta finie.

Néanmoins, une fois connecté sur l'interface avec votre compte Docker Hub, il faut tout d'abord rajouter un node. 

![Node Tutm](/img/docker-tutum-hugo/node.png)

Lorsque vous êtes sur la page "**Nodes**", deux choix possibles :  

* Soit vous utilisez votre propre serveur, dans ce cas là choisissez : **Bring your own node**
* Soit (ce qui est mon cas) vous passez par un IaaS comme DigitalOcean : **Launch new node cluster**

Je ne vais pas détailler l'utilisation de son propre serveur, tant la mise en place est simple (il suffit de copier une commande sur sa machine).

### Création du noeud DigitalOcean

J'ai choisi de créer une seule droplet DigitalOcean, si vous voulez que votre blog/site soit en load balacing, vous pouvez créer plusieurs noeuds.

![Node DigitalOcean](/img/docker-tutum-hugo/nodedo.png)

Une fois le noeud crée, et déployé, passons au service.

![Node DigitalOcean Deployed](/img/docker-tutum-hugo/node-deployed.png)

### Création du service

Prochaine étape, création du service, c'est-à-dire déployer le blog sur mon noeud. Pour ce faire, une fois sur la page des services, cherchez votre image Docker :

![Image Hub on Tutum](/img/docker-tutum-hugo/docker-tutum.png)

Quelques configurations :
![Configuration Service Tutum](/img/docker-tutum-hugo/config.png)

Il faut également renseigner la variable d'environnement `HUGO_BASE_URL` pour ne pas avoir de problème pour le statique du site :

![Environnement variable](/img/docker-tutum-hugo/env.png)


Pour finir, cliquez sur "**Create and Deploy**"

![Service deployed](/img/docker-tutum-hugo/deployed.png)


Votre conteneur est maintenant déployé sur votre serveur, et votre blog est accessible.

La dernière étape pour terminer ce déploiement continu va être de rajouter un lien entre Tutum et le Hub de Docker.

Tutum + Docker Hub
=======

Lorsque vous êtes sur votre service dans Tutum, allez sur le menu **Triggers**.

### Création du trigger

![Create Trigger](/img/docker-tutum-hugo/create-trigger.png)

Copiez l'url, et retournez sur l'interface du Hub de Docker.      
<br>
![Create Trigger](/img/docker-tutum-hugo/show-trigger.png)
<br>

### Création du webhook sur Docker Hub 
<br>
Sur l'interface du Docker Hub, allez dans la partie "**Webhooks**" et créez un webhook avec l'url copié.

![Create Webhook on Hub](/img/docker-tutum-hugo/finish.png)


Fini ! Maintenant, lorsque vous effectuerez une modification sur votre blog, une nouvelle image sera crée, votre conteneur en production sera détruit, et un nouveau sera déployé. 

* [Github](https://github.com/pabardina/blog-hugo)
* [Docker hub](https://hub.docker.com/r/pabardina/blog)


