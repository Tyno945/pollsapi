# Building APIs with Django and Django Rest Framework

## Setup, Models and Admin

### Creating a project

```bash
$ cd ~/Desktop
$ mkdir pollsapi
$ cd pollsapi
$ pipenv install django==2.1
$ pipenv install djangorestframework
$ pipenv shell
$ django-admin startproject pollsapi .
$ python manage.py startapp polls
```

```python
# pollsapi/settings.py
INSTALLED_APPS = [
'django.contrib.admin',
'django.contrib.auth',
'django.contrib.contenttypes',
'django.contrib.sessions',
'django.contrib.messages',
'django.contrib.staticfiles',

# Local
'rest_framework',

# 3rd Party
'polls.apps.PollsConfig', # new
]
```

```bash
$ python manage.py migrate
```

### Creating models

```python
# polls/models.py
from django.db import models
from django.contrib.auth.models import User

class Poll(models.Model):
    question = models.CharField(max_length=100)
    created_by = models.ForeignKey(User, on_delete=models.CASCADE)
    pub_date = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.question

class Choice(models.Model):
    poll = models.ForeignKey(Poll, related_name='choices', on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=100)

    def __str__(self):
        return self.choice_text

class Vote(models.Model):
    choice = models.ForeignKey(Choice, related_name='votes', on_delete=models.CASCADE)
    poll = models.ForeignKey(Poll, on_delete=models.CASCADE)
    voted_by = models.ForeignKey(User, on_delete=models.CASCADE)
    
    class Meta:
        unique_together = ("poll", "voted_by")
```

```bash
$ python manage.py makemigrations polls
$ python manage.py migrate polls
```

### Django Admin

```bash
$ python manage.py createsuperuser
```

将`Poll`和`Choice`注册到admin上，从而显示出来。

```python
# polls/admin.py
from django.contrib import admin

from .models import Poll, Choice

admin.site.register(Poll)
admin.site.register(Choice)
```

## A simple API with pure Django

### Writing the urls

```python
# pollsapi/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('polls.urls')),
]
# polls/urls.py
from django.urls import path
from .views import polls_list, polls_detail

urlpatterns = [
    path("polls/", polls_list, name="polls_list"),
    path("polls/<int:pk>/", polls_detail, name="polls_detail")
]
```

### Writing the views

```python
# polls/views.py
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse

from .models import Poll

def polls_list(request):
    MAX_OBJECTS = 20
    polls = Poll.objects.all()[:MAX_OBJECTS]
    data = {"results": list(polls.values("question", "created_by__username", "pub_date"))}
    return JsonResponse(data)


def polls_detail(request, pk):
    poll = get_object_or_404(Poll, pk=pk)
    data = {"results": {
        "question": poll.question,
        "created_by": poll.created_by.username,
        "pub_date": poll.pub_date
    }}
    return JsonResponse(data)
```

## API with DRF

Create a file named `polls/serializers.py`

```python
# polls/serializers.py
from rest_framework import serializers

from .models import Poll, Choice, Vote


class VoteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Vote
        fields = '__all__'


class ChoiceSerializer(serializers.ModelSerializer):
    votes = VoteSerializer(many=True, required=False)

    class Meta:
        model = Choice
        fields = '__all__'


class PollSerializer(serializers.ModelSerializer):
    choices = ChoiceSerializer(many=True, read_only=True, required=False)

    class Meta:
        model = Poll
        fields = '__all__'
```

### Creating Views with APIView

use the APIView to build the polls list and poll detail API

```python
# polls/apiviews.py
from rest_framework.views import APIView
from rest_framework.response import Response
from django.shortcuts import get_object_or_404

from .models import Poll, Choice
from  .serializers import PollSerializer

class PollList(APIView):
    def get(self, request):
        polls = Poll.objects.all()[:20]
        data = PollSerializer(polls, many=True).data
        return Response(data)


class PollDetail(APIView):
    def get(self, request, pk):
        poll = get_object_or_404(Poll, pk=pk)
        data = PollSerializer(poll).data
        return Response(data)
```

```python
# polls/urls.py
from django.urls import path

from .apiviews import PollList, PollDetail

urlpatterns = [
    path("polls/", PollList.as_view(), name="polls_list"),
    path("polls/<int:pk>/", PollDetail.as_view(), name="polls_detail")
]
```

### Using DRF generic views

```python
# polls/apiviews.py
from rest_framework import generics

from .models import Poll, Choice
from .serializers import PollSerializer, ChoiceSerializer, VoteSerializer


class PollList(generics.ListCreateAPIView):
    queryset = Poll.objects.all()
    serializer_class = PollSerializer


class PollDetail(generics.RetrieveDestroyAPIView):
    queryset = Poll.objects.all()
    serializer_class = PollSerializer


class ChoiceList(generics.ListCreateAPIView):
    queryset = Choice.objects.all()
    serializer_class = ChoiceSerializer


class CreateVote(generics.CreateAPIView):
    serializer_class = VoteSerializer
```

```python
# polls/urls.py
from django.urls import path

from .apiviews import ChoiceList, CreateVote, PollList, PollDetail

urlpatterns = [
    path("polls/", PollList.as_view(), name="polls_list"),
    path("polls/<int:pk>/", PollDetail.as_view(), name="polls_detail"),
    path("choices/", ChoiceList.as_view(), name="choice_list"),
    path("vote/", CreateVote.as_view(), name="create_vote")
]
```

### More views and viewsets

```python
# polls/apiviews.py
from rest_framework import generics
from rest_framework.views import APIView
from rest_framework import status
from rest_framework.response import Response

from .models import Poll, Choice
from .serializers import PollSerializer, ChoiceSerializer, VoteSerializer

# ...
# PollList and PollDetail views

class ChoiceList(generics.ListCreateAPIView):
    def get_queryset(self):
        queryset = Choice.objects.filter(poll_id=self.kwargs["pk"])
        return queryset
    serializer_class = ChoiceSerializer


class CreateVote(APIView):
    serializer_class = VoteSerializer

    def post(self, request, pk, choice_pk):
        voted_by = request.data.get("voted_by")
        data = {'choice': choice_pk, 'poll': pk, 'voted_by': voted_by}
        serializer = VoteSerializer(data=data)
        if serializer.is_valid():
            vote = serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        else:
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

```python
# polls/urls.py
#...
urlpatterns = [
    path("polls/<int:pk>/choices/", ChoiceList.as_view(), name="choice_list"),
    path("polls/<int:pk>/choices/<int:choice_pk>/vote/", CreateVote.as_view(), name="create_vote"),

]
```

### Introducing Viewsets and Routers

```python
# urls.py
# ...
from rest_framework.routers import DefaultRouter
from .apiviews import PollViewSet


router = DefaultRouter()
router.register('polls', PollViewSet, base_name='polls')


urlpatterns = [
    # ...
]

urlpatterns += router.urls

# apiviews.py
# ...
from rest_framework import viewsets

from .models import Poll, Choice
from .serializers import PollSerializer, ChoiceSerializer, VoteSerializer


class PollViewSet(viewsets.ModelViewSet):
    queryset = Poll.objects.all()
    serializer_class = PollSerializer
```

### Choosing the base class to use

We have seen 4 ways to build API views until now

- Pure Django views
- `APIView` subclasses
- `generics.*` subclasses
- `viewsets.ModelViewSet`

So which one should you use when? My rule of thumb is,

- Use `viewsets.ModelViewSet` when you are going to allow all or most of CRUD operations on a model.
- Use `generics.*` when you only want to allow some operations on a model
- Use `APIView` when you want to completely customize the behaviour.


## Access Control

Right now our APIs are completely permissive. Anyone can create, access and delete anything. We want to add these access controls.

- A user must be authenticated to access a poll or the list of polls.
- Only an authenticated users can create a poll.
- Only an authenticated user can create a choice.
- Authenticated users can create choices only for polls they have created.
- Authenticated users can delete only polls they have created.
- Only an authenticated user can vote. Users can vote for other people’s polls.
- To enable the access control, we need to add two more APIs

API to create a user, we will call this endpoint `/users/`
API to verify a user and get a token to identify them, we will call this endpoint `/login/`

### Creating a user

```python
# serializers.py
# ...
from django.contrib.auth.models import User

# ...
class UserSerializer(serializers.ModelSerializer):

    class Meta:
        model = User
        fields = ('username', 'email', 'password')
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        return user
```

```python
# in apiviews.py
# ...
from .serializers import PollSerializer, ChoiceSerializer, VoteSerializer, UserSerializer

# ...
class UserCreate(generics.CreateAPIView):
    serializer_class = UserSerializer

# in urls.py
# ...
from .apiviews import PollViewSet, ChoiceList, CreateVote, UserCreate


urlpatterns = [
    # ...
    path("users/", UserCreate.as_view(), name="user_create"),
]
```

### Authentication scheme setup

```python
# settings.py
INSTALLED_APPS = (
    ...
    'rest_framework.authtoken'
)

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    )
}
```

Run `python manage.py migrate` to create the new tables.

```python
# in apiviews.py
# ...
class UserCreate(generics.CreateAPIView):
    authentication_classes = ()
    permission_classes = ()
    serializer_class = UserSerializer
```

```python
# serializers.py
# ...
from rest_framework.authtoken.models import Token

class UserSerializer(serializers.ModelSerializer):

    class Meta:
        model = User
        fields = ('username', 'email', 'password')
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        Token.objects.create(user=user)
        return user
```

### The login API

```python
# in apiviews.py
# ...
from django.contrib.auth import authenticate

class LoginView(APIView):
    permission_classes = ()

    def post(self, request,):
        username = request.data.get("username")
        password = request.data.get("password")
        user = authenticate(username=username, password=password)
        if user:
            return Response({"token": user.auth_token.key})
        else:
            return Response({"error": "Wrong Credentials"}, status=status.HTTP_400_BAD_REQUEST)


# in urls.py
# ...

from .apiviews import PollViewSet, ChoiceList, CreateVote, UserCreate, LoginView



urlpatterns = [
    path("login/", LoginView.as_view(), name="login"),
    # ...
]
```

### Fine grained access control

From now onwards we will use a HTTP header like this, `Authorization: Token <your token>` in all further requests.

We have two remaining things we need to enforce.

- Authenticated users can create choices only for polls they have created.
- Authenticated users can delete only polls they have created.
We will do that by overriding `PollViewSet.destroy` and `ChoiceList.post`.

```python
# ...
from rest_framework.exceptions import PermissionDenied


class PollViewSet(viewsets.ModelViewSet):
    # ...

    def destroy(self, request, *args, **kwargs):
        poll = Poll.objects.get(pk=self.kwargs["pk"])
        if not request.user == poll.created_by:
            raise PermissionDenied("You can not delete this poll.")
        return super().destroy(request, *args, **kwargs)


class ChoiceList(generics.ListCreateAPIView):
    # ...

    def post(self, request, *args, **kwargs):
        poll = Poll.objects.get(pk=self.kwargs["pk"])
        if not request.user == poll.created_by:
            raise PermissionDenied("You can not create choice for this poll.")
        return super().post(request, *args, **kwargs)
```

## Testing and Continuous Integeration