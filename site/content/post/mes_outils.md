+++
date = "2017-03-30T10:42:00+02:00"
draft = false
tags = [ "i3wm", "linux", "fzf", "polybar", "rofi" ]
title = "Mes outils"
slug = "mes-outils"
+++


Cet article a pour but de faire un panel des outils que j'utilise quotidiennement. **Je ne dis en aucun cas que ce sont les meilleurs.**

J'utilise Debian testing comme système d'exploitation, tous les logiciels ci-dessous fonctionnent sur Ubuntu/Debian.


# Window Manager

#### [i3wm](https://i3wm.org/)
Le window manager à la mode ! Après avoir vu pas mal de personnes l'utiliser en conférence, j'ai voulu essayer, et je suis toujours dessus. Cela me permet de gérer mes fenêtres rapidement, il y a aussi beaucoup de fonctionnalités, une configuration simple et lisible. D'autres window manager ont des fichiers de configurations vraiment repoussant...

Tout n'est pas rose non plus, il faut passer plusieurs minutes (heures...) à le configurer. Pour ma part, la gestion multi écrans n'est pas encore parfaite...

[Ma configuration](https://github.com/pabardina/dotfiles/blob/master/i3/config)


### Outils window manager

#### [Polybar](https://github.com/jaagr/polybar)
Personnellement, je n'ai jamais trop accroché avec la bar d'I3. Il existe pas mal de projets pour la remplacer (py3status..). J'utilise Polybar, simple d'utilisation, et plutôt joli de base. Sur Debian, il faut installer une multitude de dépendances...

![Polybar](https://u.teknik.io/x32YI.png)


#### [Rofi](https://davedavenport.github.io/rofi/)
Cet outil me rappelle Spotlight sur Mac. Je l'utilise quotidiennement pour lancer mes applications, switcher entre mes fenêtres. Il est possible de faire plus de choses en le combinant avec les commandes Unix (lancer directement une recherche Google : [exemple](https://github.com/pabardina/dotfiles/blob/master/i3/config#L156))

![Rofi](https://davedavenport.github.io/rofi/images/rofi/run-dialog.png)


# Développement

#### [Visual Studio Code](https://code.visualstudio.com/)
Après avoir utilisé Sublime Text pendant plusieurs mois, et testé Atom plusieurs semaines, je trouve que VSCode est vraiment le mieux des trois. Super interface, plein de plugins et même un debugger intégré.

Mes plugins :

* [Git Blame](https://marketplace.visualstudio.com/items?itemName=waderyan.gitblame)
* [Git History](https://marketplace.visualstudio.com/items?itemName=donjayamanne.githistory)
* [Go](https://marketplace.visualstudio.com/items?itemName=lukehoban.Go)
* [Log File Highlighter](https://marketplace.visualstudio.com/items?itemName=emilast.LogFileHighlighter)
* [Open in GitHub](https://marketplace.visualstudio.com/items?itemName=ziyasal.vscode-open-in-github)
* [Path Intellisense](https://marketplace.visualstudio.com/items?itemName=christian-kohler.path-intellisense)
* [Python](https://marketplace.visualstudio.com/items?itemName=donjayamanne.python)
* [Rainbow Brackets](https://marketplace.visualstudio.com/items?itemName=2gua.rainbow-brackets)
* [Settings Sync](https://marketplace.visualstudio.com/items?itemName=Shan.code-settings-sync)
* [Vim](https://marketplace.visualstudio.com/items?itemName=vscodevim.vim)
* [Vscode-icons](https://marketplace.visualstudio.com/items?itemName=robertohuertasm.vscode-icons)


![Vscode](https://cloud.githubusercontent.com/assets/11839736/16642200/6624dde0-43bd-11e6-8595-c81885ba0dc2.png)

#### [NeoVim](https://neovim.io/)
Avec [Vim-Plug](https://github.com/junegunn/vim-plug)  pour gérer les plugins.

[Ma configuration](https://github.com/pabardina/dotfiles/blob/master/nvim/init.vim)

#### [Pycharm](https://www.jetbrains.com/pycharm/)
Super IDE python, les contributeurs OpenStack peuvent même avoir une version professionnelle gratuite !

#### [GitKraken](https://www.gitkraken.com/)
C'est mon interface préféré pour Git, quand j'en ai marre de jouer avec les commandes git, je vais dessus.

#### [Virtualenv](https://virtualenv.pypa.io/en/stable/)
Indispensable pour avoir plusieurs environnements Python isolés. Couplé à [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/) pour les gérer plus facilement.


# Les indispensables

#### Zsh
Avec le framework [oh-my-Zsh](https://github.com/robbyrussell/oh-my-zsh)

#### [Termite](https://github.com/thestinger/termite)
Je l'utilise depuis quelques mois comme terminal, il ne fait rien d'extraordinaire, mais il me convient.

#### [Tmux](https://tmux.github.io/)
Indispensable si vous passez vos journées dans un terminal.

![Tmux](https://tmux.github.io/ss-tmux1.png)

#### [Ranger](http://ranger.nongnu.org/)
Un explorateur de fichier dans son terminal.

![ranger](https://raw.githubusercontent.com/ranger/ranger-assets/master/screenshots/screenshot.png  "Ranger")

#### [Autojump](https://github.com/wting/autojump)
Découvert récemment, je me force à l'utiliser. Il permet de se déplacer plus rapidement dans des dossiers qu'on visite fréquemment.

#### [Fkill](https://github.com/sindresorhus/fkill-cli)
J'aime beaucoup l'idée, je trouve l'outil super. En revanche, installer NodeJS que pour ça....

![Fkill](https://github.com/sindresorhus/fkill-cli/raw/master/screenshot.gif)

#### Chromium
Avec les extensions [Vimium](https://vimium.github.io/), Lastpass et uBlock.

#### [Docker, docker-compose et machine](https://www.docker.com)
Of course !

#### [Ansible](https://www.ansible.com/)
Je l'utilise dès que je dois refaire plusieurs fois les mêmes choses. Dernier petit projet en date, un playbook qui va provisionner des droplets DigitalOcean, télécharger des torrents dessus, et les récupérer en local : [https://github.com/pabardina/get-torrent-ansible](https://github.com/pabardina/get-torrent-ansible)

#### [Fzf](https://github.com/junegunn/fzf)
Vraiment indispensable si vous faites des recherches dans votre historique bash.

![Fzf](https://camo.githubusercontent.com/0b07def9e05309281212369b118fcf9b9fc7948e/68747470733a2f2f7261772e6769746875622e636f6d2f6a756e6567756e6e2f692f6d61737465722f667a662e676966  "Fzf")

#### [Rambox](http://rambox.pro/)
Centraliser toutes ses messageries instantanées (slack, rocket, irc, messenger...)

![Rambox](https://raw.githubusercontent.com/saenzramiro/rambox/master/resources/screenshots/mac.png)


#### Pour vos yeux
[RedShift](http://jonls.dk/redshift/) et [Flux](https://justgetflux.com/)

Ces deux outils vont ajuster les couleurs de votre écran suivant l'heure qu'il est. Lorsque le soleil se couche, votre écran va "jaunir". Moins de mal aux yeux et à la tête en perspective. Je n'ai pas de préférence entre les deux, ils font la même chose. En ce moment, j'utilise Redshift.
