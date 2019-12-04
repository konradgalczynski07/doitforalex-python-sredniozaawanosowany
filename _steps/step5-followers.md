---
layout: step
number: 5
title: Followersi
permalink: step5/
---

Dobra teraz dodamy followersów. Znowu zaczynamy od modelu: 

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

Zgodnie z schematem bazy dodaliśmy dwa nowe pola ManyToMany a w nich odwołanie do klasy User (czyli samej siebie). My użyliśmy do tego atrybutu pochodzącego z ustawień który określa gdzie znajduję się model użytkownika ale można to też zrobić za pomocą 'self' w pierwszym parametrze. Kolejne parametry to wymagany przez Django related_name, blank określający czy pole może być puste/opcjonalne oraz symmetrical. Ten ostatni ustawiony na False będzie odpowiadał za to żeby nie dodawać użytkowniką siebie nawzajem np. gdybyśmy mieli pole friends wtedy gdy ja jestem twoim znajomym to ty moim wiec m2m musiałoby być symetryczne.

Dodaliśmy również dwie przydane metody zliczające ilość osób obserwujących oraz obserwowanych.

Robimy migracje i lecimy do serializera:

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
```

Dodajemy nowy UserProfileSerializer tak żeby posiadał wszystkie pola które potrzebne są w profilu użytkownika. Zwróć uwagę ze ma on tez metody klasy User tj. `number_of_followers` oraz `number_of_following`. Co więcej dodaliśmy customowe pole za pomocą `serializers.SerializerMethodField()` w której okreslamy czy użytkownik wysyłający zapytanie obserwuję użytkownika którego dotyczy zapytanie.

Potem we views'ach importujemy te serializery oraz dodajemy pierwszy widok:

```python
from django.contrib.auth import get_user_model
from rest_framework import generics, permissions
from rest_framework.views import APIView
from rest_framework.response import Response

from users.serializers import (
    UserSerializer,
    RegisterUserSerializer,
    UserProfileSerializer,
)


User = get_user_model()

...

class UserProfileView(generics.RetrieveAPIView):
    lookup_field = 'username'
    queryset = User.objects.all()
    serializer_class = UserProfileSerializer
    permission_classes = (permissions.AllowAny,)
```

Ten krótki widok pozwoli na wyświetlanie profilu użytkownika. Poprzez `lookup_field` określamy po jakim polu chcemy wyszukiwać użytkowników (dodamy to potem w scieżce za pomocą `<slug:username>`)


Dodajmy kolejny widok:

```python
...

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
```

Oraz zapiszmy je w urls.py: 

```python
from django.urls import path

from rest_framework_jwt.views import obtain_jwt_token

from users.views import (
    RegisterUserView,
    UserView,
    UserProfileView,
    FollowUserView,
)

app_name = 'users'

urlpatterns = [
    path('register/', RegisterUserView.as_view(), name='register'),
    path('login/', obtain_jwt_token, name='login'),
    path('me/', UserView.as_view(), name='me'),
    path('<slug:username>/', UserProfileView.as_view(), name='user-profile'),
    path('<slug:username>/follow/', FollowUserView.as_view(), name='follow-user'),
]
```

Wejdzmy teraz na localhost:8000 i pobawmy się z tymi endpointami. Założmy nowego użytkownika i wyślimy im nawzajem followy.
