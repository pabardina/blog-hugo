+++
date = "2015-05-09T15:28:00+02:00"
draft = false
title = "Django REST Framework : Authentification"
slug = "django-rest-framework-auth"
aliases = [
	"django-rest-framework-auth"
]
+++
# Introduction

>Django REST framework is a powerful and flexible toolkit that makes it easy to build Web APIs.  
<http://www.django-rest-framework.org/>

Le but de cet article est de présenter Django REST framework avec un exemple.


Pour ce premier article, il y aura l'application d'authentification du projet avec : 

- extension du modèle user de django  
- authentification avec username ou email avec gestion token pour les requêtes  
- requêtes GET/POST/PATCH/DELETE sur modèle user

Les sources du tutorial sont sur mon **[github](https://github.com/pabardina/django-course/)**.

Il est nécessaire pour bien comprendre le tutorial d'avoir un minimum de bases avec Django.
<br/>
# Mise en place

On commence par créer un nouveau projet :  
`django-admin startproject examplename`  

On installe [Django REST framework](http://www.django-rest-framework.org/#installation) :  
`pip install djangorestframework`

Et on l'ajoute dans les INSTALLED_APPS :

```python
INSTALLED_APPS = (
    ...
    'rest_framework',
)
```

Il y a également des paramètres à mettre dans votre fichiers settings.py, tout dépend de comment vous souhaitez l'utiliser.

Exemple de mon fichier [settings.py](https://github.com/pabardina/django-course/blob/master/course/config/settings.py) :

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.TokenAuthentication', #  on utilise une authentification avec token, il est possible d'utiliser Oauth2.
    ),

    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.XMLRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
        'rest_framework.renderers.YAMLRenderer',
    ), 

    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated', #  par défaut, toutes les requêtes auront besoin du token
    ),

    'TEST_REQUEST_DEFAULT_FORMAT': 'json',

    'TEST_REQUEST_RENDERER_CLASSES': (
        'rest_framework.renderers.MultiPartRenderer',
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.YAMLRenderer'
    )
}

CORS_ORIGIN_ALLOW_ALL = True
```
<br/>
### Utilisation d'un modèle utilisateur personnalisé

Une fois votre projet Django configuré aux petits [oignons](http://www.miximum.fr/checklist-bonnes-pratiques-django.html.html), on crée l'application authentification :  
`django-admin startapp authentication`

#### Modèle Account
Comme dit plus haut, on va étendre le model utilisateur de Django. 
Dans `authentication/models.py` :

```python
from django.contrib.auth.models import AbstractBaseUser


class Account(AbstractBaseUser):
    email = models.EmailField(unique=True)
    username = models.CharField(max_length=40, unique=True)
    first_name = models.CharField(max_length=70, blank=True)
    last_name = models.CharField(max_length=70, blank=True)
    is_admin = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    objects = AccountManager()

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    def __unicode__(self):
        return self.email + " - " + self.username

    @property
    def is_superuser(self):
        return self.is_admin

    @property
    def is_staff(self):
        return self.is_admin

    def has_perm(self, perm, obj=None):
        return self.is_admin

    def has_module_perms(self, app_label):
        return self.is_admin
```
Simple modèle qui hérite de AbstractBaseUser. 
<br/>
#### AccountManager

Il nous reste à coder le manager dans le même fichier :
 
```python
from django.contrib.auth.models import  BaseUserManager


class AccountManager(BaseUserManager):
    def create_user(self, email, password=None, **kwargs):
        if not email:
            raise ValueError('Users must have a valid email address.')

        if not kwargs.get('username'):
            raise ValueError('Users must have a valid username.')

        account = self.model(
            email=self.normalize_email(email), username=kwargs.get('username')
        )

        account.set_password(password)
        account.save()

        return account

    def create_superuser(self, email, password, **kwargs):
        account = self.create_user(email, password, **kwargs)

        account.is_admin = True
        account.save()

        return account

```

Pour dire à Django d'utiliser notre nouveau modèle utilisateur plutôt que le sien, on rajoute dans les settings :  
**`AUTH_USER_MODEL = 'authentication.Account'`**

Pour en finir avec le fichier `authentication/models.py` , il nous faut coder le signals qui à chaque création d'utilisateur va créer un token associé (modèle Token de Django rest framework) :

```python
@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)
``` 
<br/>
### Authentification avec username ou email
Je souhaitais mettre en place une authentification avec username ou email, ce qui n'est pas possible de base avec Django, mais qui est très facile à faire.

Création d'un fichier **`authentication/auth_backend.py`**

```python
from django.conf import settings
from django.contrib.auth import get_user_model

class EmailOrUsernameModelBackend(object):
    """
    This is a ModelBacked that allows authentication with either a username or an email address.

    """
    def authenticate(self, username=None, password=None):
        if '@' in username:
            kwargs = {'email': username}
        else:
            kwargs = {'username': username}
        try:
            user = get_user_model().objects.get(**kwargs)
            if user.check_password(password):
                return user
        except get_user_model().DoesNotExist:
            return None

    def get_user(self, username):
        try:
            return get_user_model().objects.get(pk=username)
        except get_user_model().DoesNotExist:
            return None
```

Il faut maintenant changer le **`AUTHENTICATION_BACKENDS`** :  

```python
AUTHENTICATION_BACKENDS = (
	'authentication.auth_backend.EmailOrUsernameModelBackend',
)
```
<br/>
### One more time

Et pour que tout marche, un coup de migration :  
`python manage.py makemigrations`  
`python manage.py migrate`

## Serialization de notre modèle Account

Avant d'entrée dans le vif du sujet, je vous laisse voir la [documentation officiel de Django REST framework sur les serializers](http://www.django-rest-framework.org/api-guide/serializers/), ce qui vous permettra de mieux comprendre la suite du tutorial.

Le serializer de l'application ressemble à :  

```python
# django import
from django.contrib.auth import update_session_auth_hash
# Django REST Framework import
from rest_framework import serializers
# authentication app import
from authentication.models import Account

class AccountSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, required=False)
    confirm_password = serializers.CharField(write_only=True, required=False)

    def __init__(self, *args, **kwargs):
        super(AccountSerializer, self).__init__(*args, **kwargs)

        if self.context.get("request"):
            self.request = self.context['request']
            if not self.request.method == "POST":
                del self.fields['auth_token']

    class Meta:
        model = Account
        fields = ('id', 'email', 'username', 'created_at', 'updated_at',
                  'first_name', 'last_name', 'password', 'confirm_password',
                  'auth_token')
        read_only_fields = ('created_at', 'updated_at', 'auth_token')

        def create(self, validated_data):
            return Account.objects.create(**validated_data)

        def update(self, instance, validated_data):
            instance.username = validated_data.get('username',
                                                   instance.username)

            instance.save()

            password = validated_data.get('password', None)
            confirm_password = validated_data.get('confirm_password', None)

            if password and confirm_password and password == confirm_password:
                instance.set_password(password)
                instance.save()

            update_session_auth_hash(self.context.get('request'), instance)

            return instance
```

Comme je souhaite que toutes mes requêtes soient authentifiées, j'ai besoin du champ `auth_token` sauf lors d'une requête POST sur mon modèle (inscription d'un utilisateur). C'est ce que je fais dans la méthode `__init__` :  

```python
def __init__(self, *args, **kwargs):
    super(AccountSerializer, self).__init__(*args, **kwargs)

    if self.context.get("request"):
        self.request = self.context['request']
        if self.request.method == "POST":
            del self.fields['auth_token']
```
<br/>
### Permissions
Avant de passer au fichier des **`views.py`**, pour le besoin de l'application, j'ai codé une permission supplémentaire. N'hésitez pas à aller voir la [documentation de Django REST framework pour les permissions](http://www.django-rest-framework.org/api-guide/permissions/), la plupart des besoins sont couverts, et en rajouter des personnalisés est très simple, la preuve :  

```python
from rest_framework import permissions
 
class IsOwner(permissions.BasePermission):
    """
    Custom permission to only allow owner of an object to use it.
    """

    def has_object_permission(self, request, view, obj):
        if hasattr(obj, 'auth_token') and hasattr(request.auth, 'key') and not request.method == "GET":
            return obj.auth_token.key == request.auth.key
        elif request.method == 'POST' or request.method == "GET":
            return permissions.AllowAny(),
```  
<br/>
### Et voilà les .... views !

Les views sont un sujet assez vaste avec Django REST framework. Plusieurs choix sont possibles. Je vous laisse en juger par vous [mêmes](http://www.django-rest-framework.org/api-guide/views/).  Pour ma part, j'utilise les [modelViewSet](http://www.django-rest-framework.org/api-guide/viewsets/#modelviewset)

#### Account view

Commençons par le modelViewSet de notre modèle Account.
Ouvrez donc le fichier **`authentication/views.py`** :  

```python
from rest_framework import permissions, viewsets, authentication
from rest_framework import status, views
from rest_framework.response import Response

from authentication.models import Account
from authentication.serializers import AccountSerializer
from authentication.permissions import IsOwner


class AccountViewSet(viewsets.ModelViewSet):
    """ Account resource. """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
    authentication_classes = authentication.TokenAuthentication,
    permission_classes = IsOwner,

    def create(self, request):
        """
        Create an account

        """
        serializer = self.serializer_class(data=request.data,
                                           context={'request': request})

        if serializer.is_valid():
            account = Account.objects.create_user(**serializer.validated_data)
            serializer_with_token = AccountSerializer(
                account, context={'request': request})
            return Response(serializer_with_token.data,
                            status=status.HTTP_201_CREATED)

        return Response(serializer.errors,
                        status=status.HTTP_400_BAD_REQUEST)

    def list(self, request, *args, **kwargs):
        """
        Return a list of accounts.

        """
        return super(AccountViewSet, self).list(request, *args, **kwargs)

    def update(self, request, *args, **kwargs):
        """
        Update an object with all fields required

        """
        return super(AccountViewSet, self).update(request, *args, **kwargs)

    def partial_update(self, request, *args, **kwargs):
        """
        Update an object with specific field

        """
        return super(AccountViewSet, self).partial_update(request, *args,
                                                          **kwargs)
```

Comme vous pouvez le constater, il y a très peu de code. Les attributs sont **importants** :  

```python
queryset = Account.objects.all() #  on récupère tout, on filtrera en fonction des droits
serializer_class = AccountSerializer #  notre serializer codé plus haut
authentication_classes = authentication.TokenAuthentication, #  le token sera nécessaire pour les requêtes
permission_classes = IsOwner,
```
<br/>
#### Exemple login view

```python

class LoginView(views.APIView):
    """
    Login Ressource

    """
    permission_classes = permissions.AllowAny,

    def post(self, request, format=None):
        """
        Two arguments:
        username & password
        """
        data = request.DATA

        username = data.get('username', None)
        password = data.get('password', None)

        account = authenticate(username=username, password=password)

        if account is not None:
            if account.is_active:
                login(request, account)

                serialized = AccountSerializer(account)

                return Response(serialized.data)
            else:
                return Response({
                    'status': 'Unauthorized',
                    'message': 'This account has been disabled.'
                }, status=status.HTTP_401_UNAUTHORIZED)
        else:
            return Response({
                'status': 'Unauthorized',
                'message': 'Username/password combination invalid.'
            }, status=status.HTTP_401_UNAUTHORIZED)
```
<br/>
#### Exemple logout view 

```python

class LogoutView(views.APIView):
    permission_classes = (permissions.IsAuthenticated,)

    """
    Logout Ressource
    """

    def post(self, request, format=None):
        """
        Simple Call on /logout in post. No arguments

        """
        logout(request)

        return Response({}, status=status.HTTP_204_NO_CONTENT)
```

<br/>

![](/images/2015/05/djangorest1.png)

Sources :

- https://thinkster.io/django-angularjs-tutorial/  
- http://www.django-rest-framework.org/  
- https://github.com/pabardina/django-course/  

