# Django Deployment to Ubuntu

## Server Security & Access
### Updating Packages:
```
# apt update
# apt upgrade
```

### Creating a new User and assigning a `sudo` group to it:
```
# useradd -m nabil
# usermod -aG sudo nabil
# passwd nabil
```

### SSH Configuration
Creating a SSH key
```
$ ssh-keygen -t rsa -b 4096 -f ~/.ssh/public_key
```

Move the public key to the server:
First method:
```
$ scp ~/.ssh/public_key.pub user@server-IP:~/.ssh/public_key.pub
```
then from server side, the public key needs to be appended to `authorized_keys` file:
```
# cd ~/.ssh
# cat public_key.pub >> authorized_keys
# rm public_key.pub
```
changing permissions for `.ssh` folder (required step):
```
# chmod 700 ~/.ssh
# chmod 600 ~/.ssh/*
```

Second method which is easier:
```
$ ssh-copy-id -i ~/.ssh/public_key.pub user@server-IP
```

Change ssh configuration:
```
# nano /etc/ssh/sshd_config
```

then change these parameters:
```
Port 2222                  # whatever port you want (shouldn't be reserved)

AllowUsers nabil           # change `nabil` with whatever user created
AllowGroups nabil          # change `nabil` with whatever user created

PermitRootLogin no         # Permit the root account from using ssh
PasswordAuthentication no  # Don't use Passwords to authenticate (just public keys)
```
Restart SSH server:
```
# systemctl restart sshd
```
### Firewall Configuration:
installing `ufw` if doesn't exist
```
# apt install ufw
```
Setting up permissions:
```
# ufw default allow outgoing
# ufw default deny incoming
```
See which apps are registered with the firewall:
```
# ufw app list
```
to allow ssh ports run this:
```
# ufw allow ssh
```
but for a custom ssh port (e.g: 2222) run this:
```
# ufw allow 2222
```
allowing 8000 port for testing:
```
# ufw allow 8000
```

to enable & check ufw:
```
# ufw enable
# ufw status
```
## Deploy Our Project to the Server
Login with your new user account (e.g: nabil) to the server via a SSH connection, then continue using it from now on :D

### Moving Our Project to the server
We could use `git clone` or `scp` to move our project to the server:
```
$ scp -r project_folder_path user@server-IP:~/my_app
```

### Installing Python 3, Postgres & Nginx
```
$ apt install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx curl
```
### Postgres & Database User Setup
```
$ sudo -u postgres psql
```
You should now be logged into the pg shell

#### Create a database
```
CREATE DATABASE db_name;
```

#### Create user
```
CREATE USER db_user WITH PASSWORD 'abc123!';
```
#### Give User access to database
```
GRANT ALL PRIVILEGES ON DATABASE db_name TO db_user;
```
Set default encoding, tansaction isolation scheme (Recommended from Django)
```
ALTER ROLE db_user SET client_encoding TO 'utf8';
ALTER ROLE db_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE db_user SET timezone TO 'UTC';
```
Quit out of Postgres
```
\q
```
### Vitrual Environment
changing the path to the project folder:
```
$ cd ~/my_app
```
You need to install the python3-venv package
```
$ sudo apt install python3-venv
```
Creating a virtual envirement:
```
$ python3 -m venv ./venv
```
#### Activate the environment
```
$ source venv/bin/activate
```
#### Install pip modules from requirements
You could manually install each one as well
```
pip install -r requirements.txt
```
#### Setup Local Settings or Envirement Variables
Add code to your settings.py file and push to server
```
try:
    from .local_settings import *
except ImportError:
    pass
```
Create a file called local_settings.py on your server along side of settings.py and add the following

- SECRET_KEY
- ALLOWED_HOSTS
- DATABASES
- DEBUG
- EMAIL_*

#### Run Migrations
```
python manage.py makemigrations
python manage.py migrate
```
#### Create super user
```
python manage.py createsuperuser
```
#### Create static files
```
python manage.py collectstatic
```
#### Run Server
```
python manage.py runserver 0.0.0.0:8000
```
Test the site at YOUR_SERVER_IP:8000
Add some data in the admin area and check if script works properly

### Gunicorn Setup
Install gunicorn
```
$ pip install gunicorn
```
Test Gunicorn serve
```
$ gunicorn --bind 0.0.0.0:8000 app.wsgi
```
Your images, etc will be gone

Stop server & deactivate virtual env
```
ctrl-c
$ deactivate
```
### NGINX Setup
Removing the default site
```
$ sudo rm /etc/nginx/sites-enabled/default
```
Create project folder
```
$ sudo nano /etc/nginx/sites-available/django-web
```
Copy this code and paste into the file
```
server {
    listen 80;
    server_name YOUR_IP_OR_DOMAIN yourdomain.com www.yourdomain.com;

    location /static {
        alias /home/YOUR_USER/YOUR_PROJECT/static;
    }
    
    location /media {
        alias /home/YOUR_USER/YOUR_PROJECT/media;    
    }
    
    location / {
        proxy_pass http://localhost:8000;
        include /etc/nginx/proxy_params;
        proxy_redirect off;
    }
}
```

Enable the file by linking to the sites-enabled dir
```
$ sudo ln -s /etc/nginx/sites-available/django-web /etc/nginx/sites-enabled
```
#### Test NGINX config
```
$ sudo nginx -t
```
Remove port 8000 from firewall and open up our firewall to allow normal traffic on port 80
```
$ sudo ufw allow http
$ sudo ufw delete allow 8000
```
#### Max Upload file size
To change the max upload file size we should add this parameter `client_max_body_size 10M;` to this file `nginx.conf`:
```
$ sudo nano /etc/nginx/nginx.conf
```
#### Restart NGINX
```
$ sudo systemctl restart nginx
```
### Supervisor Setup
To make sure that gunicorn up and running, need to use `supervisor` (or `systemd`)
#### Install Supervisor
```
$ sudo apt install supervisor
```
#### Configure Supervisor
Adding config file
```
$ sudo nano /etc/supervisor/conf.d/django-web.conf
```
Pasting this content and save the file:
```
[program:django-web]
directory=/home/ubuntu/app
command=/home/USER/my_app/venv/bin/gunicorn -w 3 app.wsgi
user=ubuntu
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
stderr_logfile=/var/log/django-web/django-web.err.log
stdout_logfile=/var/log/django-web/django-web.out.log
```
then create some files:
```
$ sudo mkdir -p /var/log/django_web
$ sudo touch /var/log/django-web/django-web.err.log
$ sudo touch /var/log/django-web/django-web.out.log
```
#### Reload Supervisor & Nginx
```
$ sudo supervisor reload
$ sudo supervisor start django-web
$ sudo systemctl restart nginx
```
## SSL/TLS Certificate

Edit /etc/nginx/sites-available/django-web
```
server {
    listen: 80;
    server_name example.com www.example.com;
    ...
}
```

#### Add SSL with LetsEncrypt
```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Only valid for 90 days, test the renewal process with
certbot renew --dry-run
```
#### Allow 443 port:
```
$ sudo ufw allow https
```
#### SSL Certificate Auto renewal
```
$ sudo crontab -e
```
append this line in it and save
```
0 0 1 */2 * sudo certbot renew --quiet
```
#### Restart Nginx
```
$ sudo systemctl restart nginx
```


