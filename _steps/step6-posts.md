---
layout: step
number: 6
title: Posty
permalink: step6/
---

Robimy posty, stworzmy nową apke za pomocą `python manage.py startapp`, a nastepnie dodajmy ja do INSTALLED_APPS i include w głowym pliku urls.py

Bierzemy się za tworzenie modelu:

```python
import uuid
import os
from django.db import models
from django.conf import settings

from users.models import image_file_path


class Post(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='user_posts'
    )
    photo = models.ImageField(upload_to=image_file_path, blank=False, editable=False)
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

Serializer:

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

Views:

```python
from rest_framework import permissions, viewsets, generics
from rest_framework.response import Response
from rest_framework.views import APIView
from posts.serializers import PostSerializer, AuthorSerializer
from posts.models import Post


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    queryset = Post.objects.all()

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


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


class GetLikersView(generics.ListAPIView):
    serializer_class = AuthorSerializer
    permission_classes = (permissions.AllowAny,)

    def get_queryset(self):
        post_id = self.kwargs['post_id']
        queryset = Post.objects.get(pk=post_id).likes.all()
        return queryset


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

URLs:
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from posts.views import PostViewSet, LikeView, GetLikersView, UserFeedView


router = DefaultRouter()
router.register('', PostViewSet)

app_name = 'posts'

urlpatterns = [
    path('feed/', UserFeedView.as_view(), name='feed'),
    path('', include(router.urls)),
    path('like/<uuid:post_id>/', LikeView.as_view(), name='like'),
    path('<uuid:post_id>/get-likers/', GetLikersView.as_view(), name='get-likers'),
]
```
Dodajemy naszą apke do URLs głównych znajdujących sie w folderze instagram, które po tym jak wprodzimy zmiany powinny wyglądać tak:
```python
from django.contrib import admin
from django.urls import path, include
from django.conf.urls.static import static
from django.conf import settings

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/users/', include('users.urls')),
    path('api/posts/', include('posts.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

No i znowu git :D 
