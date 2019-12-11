---
layout: step
number: 6
title: Posty
permalink: step6/
---

By stworzyć posty, zróbmy nową apke za pomocą `python manage.py startapp posts`, a nastepnie dodajmy ją do INSTALLED_APPS.

Bierzemy się za tworzenie modelu:

```python
import uuid
from django.db import models
from django.conf import settings


class Post(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='user_posts'
    )
    photo = models.ImageField(blank=False, editable=False)
    text = models.TextField(max_length=500, blank=True)
    location = models.CharField(max_length=30, blank=True)
    posted_on = models.DateTimeField(auto_now_add=True)
    likes = models.ManyToManyField(
        settings.AUTH_USER_MODEL, related_name="likers", blank=True, symmetrical=False
    )

    class Meta:
        ordering = ['-posted_on']

    def number_of_likes(self):
        return self.likes.count()

    def __str__(self):
        return f'{self.author}\'s post'
```
Żeby model powstał w bazie danych, znowu tworzymy i puszczamy migracje.

W postach bedziemy chcieli mieć dokładne dane autora, żeby to zrobić stworzymy najpierw `AuthorSerializer` a następnie użyjemy go jako zagnieżdzonego serializera w `PostSerializer`

```python 
from django.contrib.auth import get_user_model
from rest_framework import serializers
from posts.models import Post

User = get_user_model()


class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('username', 'profile_pic')


class PostSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    photo = serializers.ImageField(max_length=None, allow_empty_file=False)
    liked_by_req_user = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = (
            'id',
            'author',
            'photo',
            'text',
            'location',
            'posted_on',
            'number_of_likes',
            'liked_by_req_user',
        )

    def get_liked_by_req_user(self, obj):
        user = self.context['request'].user
        return user in obj.likes.all()
```

Teraz możemy dodać widok bazujący na ModelViewSecie - dzieki niemu bedziemy mogli zrobić CRUD'a (create, retreive, update, delete) na danym obiekcie.

```python
from rest_framework import viewsets, generics
from rest_framework.response import Response
from rest_framework.views import APIView
from posts.serializers import PostSerializer
from posts.models import Post


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    queryset = Post.objects.all()

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

Rejestrujemy ten widok w nowo stworzonym pliku urls.py w folderze posts:

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from posts.views import PostViewSet


router = DefaultRouter()
router.register('', PostViewSet)

app_name = 'posts'

urlpatterns = [
    path('', include(router.urls)),
]
```

i dodajmy je do głównego pliku urls.py, poniżej sciezki '/api/users/' dodajmy posty:

```python
    path('api/posts/', include('posts.urls')),
```


Do rejestrowania viewsetów używa się routerów. Stworzy on za nas wszystkie potrzebne scieżki.

Sprawdzmy teraz czy możemy dodać nowego Posta przy użyciu backendowego interfesju graficznego w przeglądarce. 

Jeśli wszystko działa przejdzmy do lajków.

```python
class LikeView(APIView):
    def get(self, request, format=None, post_id=None):
        post = Post.objects.get(pk=post_id)
        user = self.request.user
        if user in post.likes.all():
            like = False
            post.likes.remove(user)
        else:
            like = True
            post.likes.add(user)
        data = {'like': like}
        return Response(data)
```

Tutaj korzystamy z APIView i implementujemy własną logikę w funkcji GET która będzie obsługiwać zapytania GET i przełączać like/unlike.
Rejestrujemy widok i sprawdzamy aplikacje.

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from posts.views import PostViewSet, LikeView


router = DefaultRouter()
router.register('', PostViewSet)

app_name = 'posts'

urlpatterns = [
    path('like/<uuid:post_id>/', LikeView.as_view(), name='like'),
    path('', include(router.urls)),
]
```

Kolej na strone głowna z postami, tworzymy widok.

```python
class UserFeedView(generics.ListAPIView):
    serializer_class = PostSerializer

    def get_queryset(self):
        user = self.request.user
        following_users = user.following.all().values_list('id', flat=True)
        following_users = list(following_users)
        following_users.append(user.id)
        queryset = Post.objects.filter(author__in=following_users)
        return queryset
```

 Widok znowu rejestrujemy:

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from posts.views import PostViewSet, LikeView, UserFeedView


router = DefaultRouter()
router.register('', PostViewSet)

app_name = 'posts'

urlpatterns = [
    path('feed/', UserFeedView.as_view(), name='feed'),
    path('like/<uuid:post_id>/', LikeView.as_view(), name='like'),
    path('', include(router.urls)),
]
```

Jeśli wszystko działa wychodzi na to że mamy już wiekoszość pracy za sobą! Spójrzmy teraz na uprawnienia.