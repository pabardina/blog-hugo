+++
date = "2015-05-01T14:42:00+02:00"
draft = false
title = "Ajout d'actions dans l'admin Django"
slug = "ajout-dactions-dans-ladmin-django"
aliases = [
	"ajout-dactions-dans-ladmin-django"
]
+++
# Ajouter des actions

Pour ajouter des actions personnalisées dans l'administration de Django, il est nécessaire d'éditer le fichier `admin.py`.

Prenons par exemple, dans une application [blog](https://github.com/pabardina/django-blog), une action qui va publier tous les articles sélectionnés.
<!--more-->
![](/images/2015/05/admin_1.png)

```python
class ArticleAdmin(admin.ModelAdmin):
    actions = ('publish_event',)

    def publish_event(self, request, queryset):
        for article in queryset:
            if article.published:
                article.published = False
            else:
                article.published = True
                article.save()
        return
```

Pour ce faire, il a fallu rajouter une méthode `publish_event` dans notre exemple.

La documentation de Django étant très bien faite, pour aller plus loin, n'hésitez pas à y faire un [tour](https://docs.djangoproject.com/en/1.7/ref/contrib/admin/actions/). 


#Ajouter un lien dans un list_display

Dernièrement, j'avais besoin de rajouter dans la liste d'objets d'un model, un lien pour voir l'objet en front.

![](/images/2015/05/admin_2.png)

Il a fallu comme pour ajouter une action, rajouter une méthode dans ma classe admin :

```python
class ExampleAdmin(admin.ModelAdmin):
    list_display = ('name', 'link_front')

    def link_front(self, form):
        return "<a href='/view/%s'>click me</a>" % form.id
    link_front.allow_tags = True
```


