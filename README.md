# Piku Django Postgres

Provides an example Django app using Postgres locally on the PIKU server.

## Requirements
1. Piku
2. Postgres installed on PIKU vm 
``` ./piku-bootstrap install postgres.yml```

## Installation

### Clone this repo and push it to your Piku server
```
git clone ...
git remote add piku piku@<your server here>:<app>
git push piku
```
### Set the hostname for NGINX
```
piku config:set NGINX_SERVER_NAME=<your fqdn>
```
### Provision the Postgres Database  
The Database is given the same name as your NGINX_SERVER_NAME via createdb and runs Django's migrate commmand ```python manage.py migrate```
```
piku run -- ./bin/provision-database.sh
```

### Create your superuser
```
piku shell
./manage.py createsuperuser
```

# Piku features utilized
This app utilizes several of Piku's ```Procfile``` features
```
wsgi: pikupostgres.wsgi:application
release: ./bin/stage_release.sh
cron: 0 0 * * * ./bin/uwsgi_cron_midnight.sh
```
## wsgi
Runs this python app as a WSGI process

## release
Each time you push your code, Piku will run a 'release' cycle and run this code.  ```./bin/stage_release.sh``` only runs Django's collectstatic function, but you can add more

## cron
Piku uses UWSGI to schedule Cron jobs.  ```./bin/uwsgi_cron_midnight.sh``` runs a Django clearsessions command every day at 0:00 (midnight)

## NGINX static file mapping
The ```ENV``` file defines a feature that has NGINX serve this application static files directly from the folder
```
NGINX_STATIC_PATHS=/static:static
```
# Django specifics

## Django settings.py

There are a few minor changes to Django to utilize Piku.  These are added to the end of ```settings.py```
1. ```DEBUG``` - Debug is set to False unless the environment variable ```DEBUG``` is defined (```piku config:set DEBUG=true```)
2. ```ALLOWED_HOSTS```  uses the NGINX_SERVER_NAME or DEBUG for security purposes
3. ```DATABASES``` is updated if NGINX_SERVER_NAME is defined
4. ```STATIC_ROOT``` is defined to reference the /static folder (tied to the above NGINX setting)

## Database Changes
Django makes it easy to modify the schema for your Models.  However, you'll need to update the running DB via 'manage.py migrate' before pushing the code changes.  Piku provides an easy way to accomplish this:

```
piku stop
piku shell 
./manage.py migrate
```
After this, you can ```git push piku``` and you should be OK  

# Pipenv considerations
If you use Pipenv for virtual environments, and given that Piku uses requirements.txt for it's python packages, you need to generate this before being pushed to piku.  A freindly devops method is to use GIT's pre-commit hook.

Add the following file to ./.git/hooks/pre-commit (as +x)
```
#!/bin/bash

# Generate requirements.txt from Pipfile
pipenv sync
pipenv requirements > requirements.txt

# Add requirements.txt to the commit
git add requirements.txt
```
