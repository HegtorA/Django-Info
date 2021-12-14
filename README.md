# Process for Making a Django App

## 1. Setting up venv
 - ```python3 -m venv ~/venvs/{project-name}```
 - ```source ~/venvs/{project-name}/bin/activate```

## 2. Install packages
 - ```pip3 install django psycopg2```
 -  Depending on DB: ```pip3 install psycopg2``` or ```pip3 install mysqlclient```

## 3. Setup Django project (makes a new project directory in current)
 - ```django-admin startproject {project-name}```

## 4. Create apps (run in directory with manage.py)
 - ```python manage.py startapp {app1-name}```
 - ```python manage.py startapp {app2-name}```
 
## 5. Install Desired Database if Needed
### PostgreSQL
 - Install: ```sudo apt-get install postgresql postgresql-contrib```
 - Additional Linux Dependencies: ```sudo apt-get install libpq-dev python3-dev```
### MySQL
 - Install: ```sudo apt install mysql-server```
 - Additional Linux Dependencies: ```sudo apt install python3-dev libmysqlclient-dev default-libmysqlclient-dev```

## 6. Setup App Databases
### PostgreSQL
 - Connect: ```sudo -i -u postgres```
 - Prompt: ```psql```
 - ```CREATE DATABASE {db1-name};```
 - ```CREATE DATABASE {db2-name};```
 - ```CREATE USER {user-name} WITH ENCRYPTED PASSWORD '{password}';```
 - ```ALTER ROLE myuser SET client_encoding TO 'utf8';```
 - ```ALTER ROLE myuser SET default_transaction_isolation TO 'read committed';```
 - ```ALTER ROLE myuser SET timezone TO 'UTC';```
 - ```GRANT ALL PRIVILEGES ON DATABASE {db1-name} TO {user-name};```
 - ```GRANT ALL PRIVILEGES ON DATABASE {db2-name} TO {user-name};```
#### Helpful commands
 - List Databases: ```\list | \l```
 - Connect to Database: ```\connect | \c```
 - Display Tables: ```\dt```
 - Quit: ```\quit | \q```
### MySQL
 - Connect: ```sudo mysql -u root```
 - ```CREATE DATABASE {db1-name};```
 - ```CREATE DATABASE {db2-name};```
 - ```CREATE USER '{user-name}'@'%' IDENTIFIED WITH mysql_native_password BY '{password}';```
 - ```GRANT ALL ON {db1-name}.* TO '{user-name}'@'%';```
 - ```GRANT ALL ON {db2-name}.* TO '{user-name}'@'%';```
 - ```FLUSH PRIVILEGES```
#### Helpful Commands
- List Databases: ```SHOW DATABASES;```
- Quit: ```EXIT```

## 7. Django Project Database Config
 - In `settings.py` of project:
 ```python
 DATABASES = {
    'default': {},
    '{db1-name}': {
        'NAME': '{db1-name}',
        'ENGINE': 'django.db.backends.{postgresql|mysql}',
        'USER': '{user-name}',
        'PASSWORD': '{password}',
        'HOST': 'localhost',
        'PORT': ''
    },
    '{db2-name}': {
        'NAME': '{db2-name}',
        'ENGINE': 'django.db.backends.{postgresql|mysql}',
        'USER': '{user-name}',
        'PASSWORD': '{password}',
        'HOST': 'localhost',
        'PORT': ''
    }
}
 ```

## 8. Write some simple Views
 - In `app-name/views.py` of each app:
```python
from django.http import HttpResponse


def index(request):
    return HttpResponse("App Index")
 ```
 
 ## 9. Create URLs for each App
 - Create then add following code to `app-name/urls.py`:
```python
from django.urls import path

from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
 ```

## 10. Add App URLs to Project
 - Add following code to `project-name/urls.py`:
```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('app1-name/', include('app1-name.urls')),
    path('app2-name/', include('app2-name.urls')),
    path('admin/', admin.site.urls),
]
```
 
## 11. Add Models to Apps
 - Add model code to `app-name/models.py`:
```python
from django.db import models


class ModelA(models.Model):
    model_fields
    
    class Meta:
        db_table = "table_name"    # Change default Django generated table name
    
    
class ModelB(models.Model):
    model_fields
```

## 12. Register Apps in Project
 - Add following lines to INSTALLED_APPS in `project-name/settings.py`:
```python
INSTALLED_APPS = [
    'app1-name.apps.app1-nameConfig',
    'app2-name.apps.app2-nameConfig',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

## 13. Create Database Routers for the Project
 - Navigate to the project directory where `settings.py` is located.
 - Create a new directory for the database routers (e.g. `database_routers`)
 - Create an empty init file at `database_routers/__init__.py`
 - Create a new router for each application within the new dir:
`database_routers/app1.py`
```python
class App1Router:
    """
    A router to control all database operations on models in the
    app1 application.
    """
    route_app_labels = {'app1-name'}

    def db_for_read(self, model, **hints):
        """
        Attempts to read app1 models go to db1.
        """
        if model._meta.app_label in self.route_app_labels:
            return 'db1-name'
        return None

    def db_for_write(self, model, **hints):
        """
        Attempts to app1 models go to db1.
        """
        if model._meta.app_label in self.route_app_labels:
            return 'db1-name'
        return None

    def allow_relation(self, obj1, obj2, **hints):
        """
        Allow relations if a model in app1 is involved.
        """
        if (
            obj1._meta.app_label in self.route_app_labels or
            obj2._meta.app_label in self.route_app_labels
        ):
           return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """
        Make sure the app1 app only appears in the
        'db1-name' database.
        """
        if app_label in self.route_app_labels:
            return db == 'db1-name'
        return None
```
`database_routers/app2.py`
```python
class App2Router:
    """
    A router to control all database operations on models in the
    app2 application.
    """
    route_app_labels = {'app2-name'}

    def db_for_read(self, model, **hints):
        """
        Attempts to read app2 models go to db2.
        """
        if model._meta.app_label in self.route_app_labels:
            return 'db2-name'
        return None

    def db_for_write(self, model, **hints):
        """
        Attempts to write app2 models go to db2.
        """
        if model._meta.app_label in self.route_app_labels:
            return 'db2-name'
        return None

    def allow_relation(self, obj1, obj2, **hints):
        """
        Allow relations if a model in app2 is involved.
        """
        if (
            obj1._meta.app_label in self.route_app_labels or
            obj2._meta.app_label in self.route_app_labels
        ):
           return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """
        Make sure the app2 app only appears in the
        'db2-name' database.
        """
        if app_label in self.route_app_labels:
            return db == 'db2-name'
        return None
```

## 14. Add Database Router info to Project Settings.
 - Add following lines to `project-name/settings.py`:
```python
DATABASE_ROUTERS = [
    'project_name.database_routers.app1.App1Router', 
    'project_name.database_routers.app2.App2Router'
]
```

## 15. Create App Database Tables
 - `python manage.py migrate --database=db1-name`
 - `python manage.py migrate --database=db2-name`

## 16. Generate Database Migrations for the Apps
 - In project directory run:
 - `python manage.py makemigrations app1-name`
 - `python manage.py makemigrations app2-name`

## 17. Check Generated SQL that will be Run
 - `python manage.py sqlmigrate --database=db1-name app1-name 0001`
 - `python manage.py sqlmigrate --database=db2-name app2-name 0001`
 
## 18. Apply Migrations to the Databases
 - `python manage.py migrate --database=db1-name`
 - `python manage.py migrate --database=db2-name`
 
## 19. Three-Step Guide to making Model Changes
 1. Change models in `models.py`
 2. Run `python manage.py makemigrations app-name`
 3. Run `python manage.py migrate --database=db-name`
 
## 20. Create Admin User
 - `python manage.py createsuperuser`
 
## 21. Make App Modifiable in Admin
- Edit `app/admin.py` and add:
```python
from django.contrib import admin

from .models import ModelA

admin.site.register(ModelA)
```
