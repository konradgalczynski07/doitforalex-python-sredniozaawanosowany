---
layout: step
number: 7
title: Uprawnienia
permalink: step7/
---

Być może zwróciłeś uwagę ze każdy post może edytować każdy użytkownik? By to zmienić stwórzmy w folderze posts plik permissions.py w którym stworzymy własne klasy uprawnień. By to zrobić musimy dziedziczyć po klasie `permissions.BasePermission`.

```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """Custom permission class which allow
    object owner to do all http methods"""

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True

        return obj.author.id == request.user.id


class IsOwnerOrPostOwnerOrReadOnly(permissions.BasePermission):
    """Custom permission class which allow comment owner to do all http methods
    and Post Owner to DELETE comment"""

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True

        if request.method == 'DELETE' and obj.post.author.id == request.user.id:
            return True

        return obj.author.id == request.user.id
```

a w viesach dodajemy import i permission_class:

```python
...
from posts.permissions import IsOwnerOrReadOnly, IsOwnerOrPostOwnerOrReadOnly


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    queryset = Post.objects.all()
    permission_classes = (IsOwnerOrReadOnly, permissions.IsAuthenticatedOrReadOnly)
```

Teraz posty będą edytowalne tylko dla właścicieli.

Czas na komentarze.