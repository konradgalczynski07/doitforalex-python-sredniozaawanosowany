---
layout: step
number: 3
title: Użytkownicy, logowanie i rejestracja
permalink: step3/
---

Przejdzmy zatem do implementacji naszego pierwszego modelu. 

Najpierw skopiuj do modelu:

```python
import uuid
import os
from django.db import models
from django.contrib.auth.models import AbstractUser


def image_file_path(instance, filename):
    """Generate file path for new recipe image"""
    ext = filename.split('.')[-1]
    filename = f'{uuid.uuid4()}.{ext}'

    return os.path.join('uploads/', filename)


class User(AbstractUser):
    email = models.EmailField(max_length=255, unique=True)
    fullname = models.CharField(max_length=60, blank=True)
    bio = models.TextField(blank=True)
    profile_pic = models.ImageField(upload_to=image_file_path, default='avatar.png')

    def __str__(self):
        return self.username
```

Puszczamy migracje i zeby mozna rejestrowac nowych uzytkowników piszemy serializer:

```python
from django.contrib.auth import get_user_model
from rest_framework import serializers

User = get_user_model()


class RegisterUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'email', 'fullname', 'username', 'password')
        extra_kwargs = {
            'password': {'write_only': True, 'min_length': 5},
            'username': {'min_length': 3},
        }

    def create(self, validated_data):
        return User.objects.create_user(**validated_data)
```

Przechodzimy do viewsów:

```python
from rest_framework import generics, permissions
from users.serializers import RegisterUserSerializer


class RegisterUserView(generics.CreateAPIView):
    serializer_class = RegisterUserSerializer
    permission_classes = (permissions.AllowAny,)
```

Teraz pozostały juz tylko urls'y:

```python
from django.urls import path

from rest_framework_jwt.views import obtain_jwt_token

from users.views import RegisterUserView

app_name = 'users'

urlpatterns = [
    path('register/', RegisterUserView.as_view(), name='register'),
    path('login/', obtain_jwt_token, name='login'),
]
```

no i jeszcze załaczmy powyższe urls'y w głównym pliku ulrs.py:

```python
from django.contrib import admin
from django.urls import path, include
from django.conf.urls.static import static
from django.conf import settings

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/users/', include('users.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

Super mamy juz rejestracie i logowanie ! Dopiszmy jeszcze funkcjonalność zarzadania profilem użytkownika. Model juz mamy, zaczynamy od serializera, napiszmy wieć:

```python
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = (
            'id',
            'email',
            'username',
            'password',
            'fullname',
            'bio',
            'profile_pic',
        )
        extra_kwargs = {
            'password': {'write_only': True, 'min_length': 5},
            'username': {'min_length': 3},
        }

    def update(self, instance, validated_data):
        password = validated_data.pop('password', None)
        user = super().update(instance, validated_data)

        if password:
            user.set_password(password)
            user.save()

        return user
```

Lecimy do viesów: 

```python
class UserView(generics.RetrieveUpdateDestroyAPIView):
    serializer_class = UserSerializer

    def get_object(self):
        return self.request.user
```

i dodajemy url: 

```python
from users.serializers import UserSerializer, RegisterUserSerializer

urlpatterns = [
    path('register/', RegisterUserView.as_view(), name='register'),
    path('login/', obtain_jwt_token, name='login'),
    path('me/', UserView.as_view(), name='me'),
]
```

Jest gites ! Sprawdzmy zatem nasza apke reactowa. Jak widzicie mozemy z jej poziomu sie zarejestrować oraz zalogowac, czad right ?
A zauyważyliscie może ze nie ładuje nam sie logo usera ? Jak wejdziemy w dev toolsy w Network tab możemy zwrocić uwage że nasz frontend wysyła tylko request na /feed, skąd zatem mamy username i inne dane a avatar nie. Tutaj wkracza ciekawa właściwość JWT - możemy zakodować w nim payload czyli jakies dane ktore nastepnie odkodujemy na frontendzie. Defaultowo zakodowany jest tylko username ale możemy do zmienić. Stworzmy w projekcie nowy folder utils a w nim plik `jwt_payload.py`

Plik jwt_payload.py:

```python
import uuid
from datetime import datetime

from rest_framework_jwt.compat import get_username
from rest_framework_jwt.compat import get_username_field
from rest_framework_jwt.settings import api_settings


def jwt_payload_handler(user):
    """Slightly customized jwt payload that include user profile picture"""

    username_field = get_username_field()
    username = get_username(user)

    payload = {
        'user_id': user.pk,
        'username': username,
        'profile_pic': user.profile_pic.url,
        'exp': datetime.utcnow() + api_settings.JWT_EXPIRATION_DELTA,
    }
    if hasattr(user, 'fullname'):
        payload['fullname'] = user.fullname

    payload[username_field] = username

    return payload
```

Potem zarejestrujmy to w settingsach, na góre dodajmy import a na samym dole JWT_AUTH:

```python 
from datetime import timedelta

...

# JWT
JWT_AUTH = {
    'JWT_EXPIRATION_DELTA': timedelta(hours=36),
    'JWT_PAYLOAD_HANDLER': 'utils.jwt_payload.jwt_payload_handler',
}
```

Jest gites w kuj !