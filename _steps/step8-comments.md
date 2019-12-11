---
layout: step
number: 8
title: Komentarze
permalink: step8/
---

Ok przechodzimy do komentarzy, dodajemy model:

```python 
class Comment(models.Model):
    post = models.ForeignKey(
        Post, on_delete=models.CASCADE, related_name='post_comments'
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='user_comments'
    )
    text = models.CharField(max_length=100)
    posted_on = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-posted_on']

    def __str__(self):
        return f'{self.author}\'s comment'
```

Importujemy model w pliku serializers.py i robimy serializer:

```python
class CommentSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)

    class Meta:
        model = Comment
        fields = ('id', 'author', 'text', 'posted_on')
        read_only_fields = ('author', 'id', 'posted_on')
```

Dodamy tutaj również komentarze do PostSerializers

```python
post_comments = CommentSerializer(read_only=True, many=True)
```

Dodaj to pole w fieldsach w Class Meta i mozemy przejsc do viewsow. Importujemy tam 3 nowe rzeczy:
- z rest_framework importujemy status 
- CommentSerializer
- model Comment

Dodajemy teraz dwa viesy
```python
class AddCommentView(generics.CreateAPIView):
    serializer_class = CommentSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly,)

    def post(self, request, post_id=None):
        post = Post.objects.get(pk=post_id)
        serializer = CommentSerializer(data=request.data)

        if serializer.is_valid():
            serializer.save(post=post, author=self.request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        else:
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

oraz dwa nowe urle:

```python
    path('comment/<uuid:post_id>/', AddCommentView.as_view(), name='add-comment'),
```