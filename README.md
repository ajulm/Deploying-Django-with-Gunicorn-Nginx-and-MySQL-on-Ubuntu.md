# Deploying Django with Gunicorn, Nginx and MySQl on Ubuntu 18.04

## Install Prerequisites

1. Update ubuntu software repository
2. Install 
   1. nginx - Serve our website 
   2. mysql-server and libmysqlclient-dev - For database
   3. Python 3
   4. ufw - Firewall for our system
   5. virtualenv - Virtual environment for our python application
```bash
$ sudo apt-get update
$ sudo apt-get install nginx mysql-server python3-pip python3-dev libmysqlclient-dev ufw virtualenv
```

## Installing python libraries

1. Create a directory to hold all django apps (Eg. django)
2. Create a virtual environment
3. Install django
4. Install mysql client for python
5. Install gunicorn to interact with our python code

```bash
$ mkdir django
$ cd django
$ virtualenv venv
$ source venv/bin/activate
$ pip install mysqlclient django gunicorn
```

Also install any other dependencies needed

## Setting up the firewall

We'll disable access to the server on all ports except 8800 for now. Later on we'll remove this give access to all ports that nginx needs

```bash
$ sudo ufw default deny
$ sudo ufw enable
$ sudo ufw allow 8800
```

## Setting up database

1. Install mysql and give root password
2. Create database for the application
3. Create a user for the application and give all access to the database
   
``` bash
$ sudo mysql_secure_installation
```

Give password for root user

``` bash
$ sudo mysql -u root -p
$ CREATE DATABASE <app database name> CHARACTER SET 'utf8';
$ CREATE USER <app username>
$ GRANT ALL ON <app database name>.* TO '<app username>'@'localhost' IDENTIFIED BY '<app user password>';
$ quit
```


## Setting up the django project

In the same folder (Eg. django) where the virtual environment is present, make a django project or move the existing project here.


#### Current Folder structure
```
\home\<user>\django\
|
|--- venv
|
|--- django_project
    |
    |--- manage.py
    |--- django_project
        |
        |--- __init__.py
        |--- urls.py
        |--- settings.py
        |--- wsgi.py
```

1. Modify the `settings.py` file to use the newly created database.

``` python
DEBUG = False

ALLOWED_HOSTS = ['127.0.0.1', 'domain.name', 'ip-address']

...

DATABASES = {
    'default': {
         'ENGINE': 'django.db.backends.mysql',
         'OPTIONS': {
             'sql_mode': 'traditional',
         },
         'NAME': '<app database name>',
         'USER': '<app username>',
         'PASSWORD': '<app user password>',
         'HOST': 'localhost',
         'PORT': '3306', 
     }
}

...

STATIC_URL = '/static/'
STATIC_ROOT=os.path.join(BASE_DIR, 'static/')

MEDIA_URL='/media/'
MEDIA_ROOT=os.path.join(BASE_DIR, 'media/')

```
Add appropriate ip address or domain name to allowed hosts.

2. Make database migrations

``` bash
(venv) $ python manage.py makemigrations
(venv) $ python manage.py migrate
(venv) $ python manage.py collectstatic
```

These commands create all required tables in the new database created and also collects all static files to a static folder under django_project directory.

3. Run the development server to check whether everything is working fine.
```
(venv) $ python manage.py runserver
```

## Setting up Gunicorn

1. Test gunicorn with django by running the following command inside the django project folder

```
(venv) $ gunicorn --bind 0.0.0.0:8800 django_project.wsgi:application
```
Now if you visit 0.0.0.0:8800/admin you'll see that there is no css or js files. Thats because we are not yet serving static files.
Kill gunicorn and exit the virtual environment.

2. Lets daemonise the gunicorn

    1. Open a new gunicorn.service file
        ```bash
        sudo gedit /etc/systemd/system/gunicorn.service
        ```
    2. Copy the following lines with appropriate modifications
        ```
        [Unit]
        Description=gunicorn service
        After=network.target
        
        [Service]
        User=<user>
        Group=www-data
        WorkingDirectory=/home/<user>/django/django_project/
        ExecStart=/home/<user>/django/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/<user>/django/django_project/django_project.sock django_project.wsgi:application
        
        [Install]
        WantedBy=multi-user.target
        ```

    3. Enable the daemon

        ```bash
        $ sudo systemctl enable gunicorn.service
        $ sudo systemctl start gunicorn.service
        $ sudo systemctl status gunicorn.service
        ```

    4. If something went wrong, do

        ```
        $ journalctl -u gunicorn
        ```

        If journalctl isnt found, ```sudo apt install journalctl```

    5. If you do any changes to your django application, reload the gunicorn server by
        ```
        $ sudo systemctl daemon-reload
        $ sudo systemctl restart gunicorn
        ```

Thats all for gunicorn

## Setting up Nginx

1. Create a new configuration file for nginx

```
$ sudo gedit /etc/nginx/sites-available/django_project
```

2. Copy following lines with appropriate modifications

```
server {
       listen 80;    
       server_name 127.0.0.1;
       location = /favicon.ico {access_log off;log_not_found off;} 
    
        location /static/ {
            alias /home/<user>/django/django_project/static/;    
        }

        location /media/ {
            alias /home/<user>/django/django_project/media/;
        }
    
        location / {
            include proxy_params;
            proxy_pass http://unix:/home/<user>/django/django_project/django_project.sock;
        }
     }
```

`listen 80` tell nginx which port to listen to. 
`server_name 127.0.0.1` tells that we are using 127.0.0.1 as the address to access our website. This can be set to the ip address of the machine.

3. Link the configuration so that nginx can recognise
```
$ sudo ln -s /etc/nginx/sites-available/django_project /etc/nginx/sites-enabled
```

4. Check whether everything is fine
```
$ sudo nginx -t
```
If it returns without any errors, then everything is fine. If not go through the configuration once more and debug.

5. Once everything is done, restart nginx
```
$ sudo systemctl restart nginx
```

6. Now that all setup is done, lets delete the testing firewall rule we had put for 8800 and allow all ports necessary for nginx

```
$ sudo ufw delete allow 8800
$ sudo ufw allow 'Nginx Full'
```


And thats it! If everything worked fine, you should be able to access your django application on 127.0.0.1 or any other ip address you gave on nginx configuration


#### Final Folder structure

```
\home\<user>\django\
|
|--- venv
|
|--- django_project
    |
    |--- manage.py
    |--- django_project
    |    |
    |    |--- __init__.py
    |    |--- urls.py
    |    |--- settings.py
    |    |--- wsgi.py
    |
    |--- static
    |
    |--- media
    |
    |--- django_project.sock

```

--------------------
