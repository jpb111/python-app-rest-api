# **Python Django app docker setup and Github actions for running linting and unit test**

## **Authenticate Github actions with DockerHub using AccessTokens**

Authenticate Github actions with DockerHub using AccessTokens to use the free the docker images pull request of 200 per 6 hours.

## **GitHub**

1. Create an account in github and create a new repository with readme and gitignore file for python Django.
2. Clone the repo to in the local computer.


## **In the cloned folder in the local machine create the following files**

.dockerignore

```txt

# Git
.git
.gitignore

# Docker
.docker

# Python
app/__pycache__/
app/*/__pycache__/
app/*/*/__pycache__/
app/*/*/*/__pycache__/
.env/
.venv/
venv/

```

requirements.txt

For latest [Django version](https://www.djangoproject.com/download/) and [Django rest framework](https://www.django-rest-framework.org/community/release-notes/#314x-series)

```txt

Django>=4.1,<4.2
djangorestframework>=3.14,<3.15

```

For latest [Flake versions](https://pypi.org/project/flake8/)

requirements.dev.txt

```txt

flake8>=6.0,<6.1

```

Dockerfile

```Dockerfile

FROM python:3.11.1-alpine3.17
LABEL maintainer="johnpaulbabu@gmail.com"
ENV PYTHONUNBUFFERED 1
COPY ./requirements.txt /tmp/requirements.txt
COPY ./requirements.dev.txt /tmp/requirements.dev.txt
COPY ./app /app
WORKDIR /app
EXPOSE 8000
ARG DEV=false
RUN python -m venv /py && \
    /py/bin/pip install --upgrade pip && \
    /py/bin/pip install -r /tmp/requirements.txt && \
    if [ $DEV = "true" ]; \
        then /py/bin/pip install -r /tmp/requirements.dev.txt ; \
    fi && \
    rm -rf /tmp && \
    adduser \
        --disabled-password \
        --no-create-home \
        django-user
ENV PATH="/py/bin:$PATH"
USER django-user

```
## Docker file explanation 

Setting PYTHONUNBUFFERED to a non-empty value different from 0 ensures that the python output i.e. the stdout and stderr streams are sent straight to terminal (e.g. your container log) without being first buffered and that you can see the output of your application (e.g. django logs) in real time.

```bash

ENV PYTHONUNBUFFERED 1

```


The below bash script in the above docker files put a condition to include development dependencies in the build or not.

```bash

if [ $DEV = "true" ]; \
        then /py/bin/pip install -r /tmp/requirements.dev.txt ; \
    fi && \

```

The ARG DEV variable is set to false by default, but we can set it to true from the docker compose file to
install the development dependencies from the requirement.dev.txt file.

```bash
ARG DEV=false 
```

In the dockerfile for security reasons we create another user so as not to run the container as a default root user.

```bash

adduser \
        --disabled-password \
        --no-create-home \
        django-user

```

docker-compose.yml

```yml
version: "3.9"

services:
  app:
    build:
      context: .
      args:
        - DEV=true
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app
    command: >
      sh -c "python manage.py runserver 0.0.0.0:8000"
```

Create a folder app and create a file .flake8 in it and add the following to it to exclude the files from testing. 

```text

[flake8]
exclude =
  migrations,
  __pycache__,
  manage.py,
  settings.py


```

Build the docker containers

```ShellCommand

docker-compose build

```

Run the linting tool

```ShellCommand

docker-compose run --rm app sh -c "flake8"

```

To create Django project use the Django CLI command to create a new project called app in the current directory.

```ShellCommand

docker-compose run --rm app sh -c "django-admin startproject app ."

```

To run the development server

```ShellCommand

docker-compose up

```
## **Generate access token in DockerHub**

1. Create an account in DockerHub
2. DockerHub > Settings > Security > Access Tokens > New Access Token
3. Give a description eg. give the repository name created to easily identify
4. Click Create and leave the page like that we need to use this token in github.

Add the DockerHub username and access Token in Github repository

1. In github repository > settings > secrets and variables > actions > New repository secrets
2. Name > enter a name, eg: DockerHub_User
3. In value > enter username used to login to DockerHub
4. Add Secret
5. Click New repository secrets
6. Add a new secret name, eg: DockerHub_Token
7. In value > copy and paste the dockerHub token.
8. Add Secret

## Github actions checks.yml file 

1. Create a folder .github/workflows
2. Create a file checks.yml in it. 

```yml 
---
name: Checks

on: [push]

jobs:
  test-lint:
    name: Test and Lint
    runs-on: ubuntu-20.04
    steps:
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USER }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Checkout
        uses: actions/checkout@v2
      - name: Test
        run: docker-compose run --rm app sh -c "python manage.py wait_for_db && python manage.py test"
      - name: Lint
        run: docker-compose run --rm app sh -c "flake8"


```


Run a simple python test. Create a simple function for calculator 

```python 
"""
Calculator functions
"""


def add(x, y):
    """Add x and y and return result."""
    return x + y


```

Create another file in the app/app directory called test.py
```python


"""
Sample tests
"""
from django.test import SimpleTestCase

from app import calc



class CalcTests(SimpleTestCase):
    """Test the calc module."""

    def test_add_numbers(self):
        """Test adding numbers together."""
        res = calc.add(5, 6)

        self.assertEqual(res, 11)


```

import django.test and import the function that needed testing Create a class and write the test function. 

## Configure data base in docker compose 

```yaml 
  db: 
    image: postgres:13-alpine
    volumes: 
      - dev-db-data:/var/lib/postgresql/data

volumes: 
  dev-db-data:
  dev-static-data:


```

The path /var/lib/postgresql/data is given in the postgres documentation.  

Adding environment variable to docker-compose file,  services app and db 

```yaml
version: "3.9"

services:
  app:
    build:
      context: .
      args:
        - DEV=true
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app
    command: >
      sh -c "python manage.py runserver 0.0.0.0:8000"
    environment:
      - DB_HOST=db
      - DB_NAME=devdb
      - DB_USER=devuser
      - DB_PASS=changeme

    depends_on:
      - db
  db: 
    image: postgres:13-alpine
    volumes: 
      - dev-db-data:/var/lib/postgresql/data
    environment: 
       - POSTGRES_DB=devdb 
       - POSTGRES_USER=devuser 
       - POSTGRES_PASSWORD=changeme
volumes: 
  dev-db-data:
  dev-static-data:


```

we add the same environment to the app and to the db so they can communicate with each other. We also specify a DB_HOST=db in the app environment to db is the service name of the database. 


