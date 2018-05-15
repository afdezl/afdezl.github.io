---
title: "Getting started with React Native & Django authentication - Part 1"
date: 2018-05-04T18:46:44+01:00
draft: false
tags: [
    "react-native",
    "django",
    "rest-framework",
    "auth",
    "development"
]
---

If you're starting with React Native, chances are you're delegating authentication to services like [Firebase](https://firebase.google.com/) or [Cognito](https://aws.amazon.com/cognito/) and passing back the state to your application backend in order to provide the right content for the user.

These services scale wonderfully and are fully managed by the provider, but sometimes the provided user model does not offer the required flexibility or you simply don't like passing state between the service and your backend.

If this is the case, you're in the right place.

_**NOTE: This post assumes you have basic knowledge of both Django and React Native.**_ 

Let's get to the point.

## 1. Setting up Django with Rest-Framework
### 1.1 Installing the required packages

In your terminal, create and activate a python3 virtual environment:

```bash
pip install virtualenv virtualenvwrapper
mkdir django-rn-auth && cd django-rn-auth
mkvirtualenv -p $(which python3) django-rn-auth
```

And install the required python packages:

```bash
pip install django djangorestframework
```

Once you've installed the required packages, simply create a project, let's call it **backend**:

```bash
django-admin startproject backend
cd backend
```

and the API app:

```bash
django-admin startapp api
```

At this point your directory structure should look like this:

```bash
../backend
    ├── api
    │   ├── admin.py
    │   ├── apps.py
    │   ├── __init__.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── backend
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── db.sqlite3
    └── manage.py
```

### 1.2 Configuring the application settings

Now let's modify the `settings.py` file to start using the Django Rest Framework, we'll stick to most of the defaults for simplicity, but the following need to be added/modified:

_**NOTE: These settings are appropriate for development only, do not use this in production!**_

```python
# backend/settings.py

ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.TokenAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    )
}

```

Now that the application is ready, we must run the initial migration and create an admin user:

```bash
python manage.py migrate
python manage.py createsuperuser
```


You can now navigate to [http://127.0.0.1:8000/admin](http://127.0.0.1:8000/admin) and login.

### 1.3 Creating our authentication API

In order to allow users to access our API, we will need 3 views, **registration**, **login** and **logout**

The workflow is fairly simple, upon registration the React Native app will genereate a `POST` request to the registration endpoint with the new user data (username/email and password) and if properly validated, the endpoint will create a new user and an API token, that can subsequently be used to authenticate requests.

On login, the user credentials will be sent via a `POST` request and the token will be generated and returned if the user had previously logged out, else the existing token will be returned.

Lastly, the logout endpoint will consist of a `GET` request, when the request is authenticated, our backend will delete the API token corresponding to the user and return a 200 status code. If this succeeds we will need to route back our user to the login screen on the app.



#### 1.3.1 Setting up the serializers

Serializers are a way of transforming your django models into a standard format, in this case JSON, that can be transformed back to a model object on request.

We will begin by serializing the default `User` model that Django provides.

Create a file name **serializers.py** in your **backend/api/** directory.

```python
# backend/api/serializers.py

from django.contrib.auth import get_user_model
from rest_framework import serializers


class CreateUserSerializer(serializers.ModelSerializer):
    username = serializers.CharField()
    password = serializers.CharField(write_only=True,
                                     style={'input_type': 'password'})

    class Meta:
        model = get_user_model()
        fields = ('username', 'password', 'first_name', 'last_name')
        write_only_fields = ('password')
        read_only_fields = ('is_staff', 'is_superuser', 'is_active',)

    def create(self, validated_data):
        user = super(CreateUserSerializer, self).create(validated_data)
        user.set_password(validated_data['password'])
        user.save()
        return user


```


#### 1.3.2 Setting up the views



```python
# backend/api/views.py

from django.contrib.auth import get_user_model
from rest_framework.generics import CreateAPIView
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from rest_framework.authtoken.models import Token
from rest_framework import status
from rest_framework.views import APIView
from api.serializers import CreateUserSerializer


class CreateUserAPIView(CreateAPIView):
    serializer_class = CreateUserSerializer
    permission_classes = [AllowAny]

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        # We create a token than will be used for future auth
        token = Token.objects.create(user=serializer.instance)
        token_data = {"token": token.key}
        return Response(
            {**serializer.data, **token_data},
            status=status.HTTP_201_CREATED,
            headers=headers
        )


class LogoutUserAPIView(APIView):
    queryset = get_user_model().objects.all()

    def get(self, request, format=None):
        # simply delete the token to force a login
        request.user.auth_token.delete()
        return Response(status=status.HTTP_200_OK)
```

#### 1.3.3 Adding the API URLs

Create a `urls.py` file inside the `api` directory and add the following contents:

```python
# backend/api/urls.py

from django.conf.urls import url
from rest_framework.authtoken.views import obtain_auth_token
from .views import CreateUserAPIView, LogoutUserAPIView


urlpatterns = [
    url(r'^auth/login/$',
        obtain_auth_token,
        name='auth_user_login'),
    url(r'^auth/register/$',
        CreateUserAPIView.as_view(),
        name='auth_user_create'),
    url(r'^auth/logout/$',
        LogoutUserAPIView.as_view(),
        name='auth_user_logout')
]
```

Make sure you import these URLs in the project wide `backend/backend/urls.py` by adding the following to the urlpatterns:

```python
# backend/backend/urls.py

...
from django.urls import path, include
from django.conf.urls import url

urlpatterns = [
    ...
    url(r'api/', include('api.urls')),
]

```

If you now run the server:

```bash
python manage.py runserver
``` 

And navigate to the [http://127.0.0.1:8000/api/auth/login/](http://127.0.0.1:8000/api/auth/login/) endpoint on the browser, you should be greeted with the following message:

```
{"detail" : "Method \"GET\" not allowed."}
```

Not to worry, we haven't authenticated our request as the endpoint only accepts a `POST` request with the user credentials.

Open up a new terminal window and authenticate as follows (replacing the fields between <>). The output should be the auth token:

```bash
curl -X POST -d "username=<YOUR_USERNAME>&password=<YOUR_PASSWORD>" http://127.0.0.1:8000/api/auth/login/

# Output
-> {"token":"a64824a0dca3cf8f1c4fc0ccd4eccb635a001346"}
```

You can subsequently use this token to authenticate requests as follows on other endpoints:

```bash
curl -X GET http://127.0.0.1:8000/api/mynewendpoint/ -H 'Authorization: Token <YOUR_TOKEN>'
```

Let's test the registration and logout endpoints:

```bash
# Registration
curl -X POST -d "username=<NEW_USER>&password=<PASSWORD>" http://127.0.0.1:8000/api/auth/register/

-> {"username":"testuser","token":"228c77007621a0d479cb7a8db60925e644e5d7fa"}

# Logout
curl -X GET http://127.0.0.1:8000/api/auth/logout/ -H 'Authorization: Token <YOUR_TOKEN>'
```

In [part 2](https://afdezl.github.io/post/authentication-react-native-django-2) we will focus on building the React Native application and biding it to our custom authentication backend.


All the code for this project can be found in this [repo](https://github.com/afdezl/django-rn-auth)
