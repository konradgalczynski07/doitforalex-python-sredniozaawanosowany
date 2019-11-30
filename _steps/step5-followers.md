---
layout: step
number: 5
title: Followersi
permalink: step5/
---

Dobra teraz followersów dodamy. Znowu zaczynamy od modelu: 

```python
from django.conf import settings
...

class User(AbstractUser):
    ...
    followers = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        related_name="user_followers",
        blank=True,
        symmetrical=False,
    )
    following = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        related_name="user_following",
        blank=True,
        symmetrical=False,
    )

    def number_of_followers(self):
        return self.followers.count()

    def number_of_following(self):
        return self.following.count()

    ...
```

Lecimy do serializera:

```python
class UserProfileSerializer(serializers.ModelSerializer):
    followed_by_req_user = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = (
            'id',
            'username',
            'fullname',
            'bio',
            'profile_pic',
            'number_of_followers',
            'number_of_following',
            'followed_by_req_user',
        )

    def get_followed_by_req_user(self, obj):
        user = self.context['request'].user
        return user in obj.followers.all()


class FollowSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('username', 'profile_pic')
```

Potem do views'a:

```python
from django.contrib.auth import get_user_model
from rest_framework import generics, permissions
from users.serializers import UserSerializer, RegisterUserSerializer
from rest_framework.views import APIView
from rest_framework.response import Response

from users.serializers import (
    UserSerializer,
    RegisterUserSerializer,
    UserProfileSerializer,
    FollowSerializer,
)


User = get_user_model()

...

class UserProfileView(generics.RetrieveAPIView):
    lookup_field = 'username'
    queryset = User.objects.all()
    serializer_class = UserProfileSerializer
    permission_classes = (permissions.AllowAny,)


class FollowUserView(APIView):
    def get(self, request, format=None, username=None):
        to_user = User.objects.get(username=username)
        from_user = self.request.user
        follow = None
        if from_user != to_user:
            if from_user in to_user.followers.all():
                follow = False
                from_user.following.remove(to_user)
                to_user.followers.remove(from_user)
            else:
                follow = True
                from_user.following.add(to_user)
                to_user.followers.add(from_user)
        data = {'follow': follow}
        return Response(data)


class GetFollowersView(generics.ListAPIView):
    serializer_class = FollowSerializer
    permission_classes = (permissions.AllowAny,)

    def get_queryset(self):
        username = self.kwargs['username']
        queryset = User.objects.get(username=username).followers.all()
        return queryset


class GetFollowingView(generics.ListAPIView):
    serializer_class = FollowSerializer
    permission_classes = (permissions.AllowAny,)

    def get_queryset(self):
        username = self.kwargs['username']
        queryset = User.objects.get(username=username).following.all()
        return queryset
```

Urls: 

```python
from django.urls import path

from rest_framework_jwt.views import obtain_jwt_token

from users.views import RegisterUserView, UserView
from users.views import (
    RegisterUserView,
    UserView,
    UserProfileView,
    FollowUserView,
    GetFollowersView,
    GetFollowingView,
)

app_name = 'users'

urlpatterns = [
    path('register/', RegisterUserView.as_view(), name='register'),
    path('login/', obtain_jwt_token, name='login'),
    path('me/', UserView.as_view(), name='me'),
    path('<slug:username>/', UserProfileView.as_view(), name='user-profile'),
    path('<slug:username>/follow/', FollowUserView.as_view(), name='follow-user'),
    path(
        '<slug:username>/get-followers/',
        GetFollowersView.as_view(),
        name='get-followers',
    ),
    path(
        '<slug:username>/get-following/',
        GetFollowingView.as_view(),
        name='get-following',
    ),
]
```

Gites lecimy dalej !