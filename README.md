# udacity-linux-server-configuration

### Project Description

Take a baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

- IP address: 52.33.103.134

- Accessible SSH port: 2200

- Application URL: http://ec2-52-33-103-134.us-west-2.compute.amazonaws.com/catalog

Configuration changes

Add user

Add user grader with command: useradd -m -s /bin/bash grader

Add user grader to sudo group

Assuming your Linux distro has a sudo group (like Ubuntu 16.04), simply add the user to this admin group:

usermod -aG sudo grader
Update all currently installed packages

apt-get update - to update the package indexes

apt-get upgrade - to actually upgrade the installed packages

If at login the message *** System restart required *** is display, run the following command to reboot the machine:

reboot

Set-up SSH keys for user grader

As root user do:

mkdir /home/grader/.ssh
chown grader:grader /home/grader/.ssh
chmod 700 /home/grader/.ssh
cp /root/.ssh/authorized_keys /home/grader/.ssh/
chown grader:grader /home/grader/.ssh/authorized_keys
chmod 644 /home/grader/.ssh/authorized_keys
Can now login as the grader user using the command: ssh -i ~/.ssh/udacity_key.rsa grader@52.33.103.134

Fix sudo resolve host error

When the grader user issues a sudo command, got the following warning: sudo: unable to resolve host ip-10-20-47-177

To fix this, the hostname was added to the loopback address in the /etc/hosts file so that th first line now reads: 127.0.0.1 localhost ip-10-20-47-177

This solution was found on the Ubuntu Forms here.

Disable root login

Change the following line in the file /etc/ssh/sshd_config:

From PermitRootLogin without-password to PermitRootLogin no.

Also, uncomment the following line so it reads:

PasswordAuthentication no
Do service ssh restart for the changes to take effect.

Will now do all commands using the grader user, using sudo when required.

Configure the local timezone to UTC

Run sudo dpkg-reconfigure tzdata and then choose PST


Change SSH port from 22 to 2200

Edit the file /etc/ssh/sshd_config and change the line Port 22 to:

Port 2200

Then restart the SSH service:

sudo service ssh restart

Will now need to use the following command to login to the server:

ssh -i ~/.ssh/udacity_key.rsa grader@52.33.103.134 -p 2200

Install Apache

sudo apt-get install apache2
Install mod_wsgi

Run sudo apt-get install libapache2-mod-wsgi python-dev
Enable mod_wsgi with sudo a2enmod wsgi
Start the web server with sudo service apache2 start
Clone the Catalog app from Github

Install git using: sudo apt-get install git
cd /var/www
sudo mkdir catalog
Change owner of the newly created catalog folder sudo chown -R grader:grader catalog
cd /catalog
Clone your project from github git clone https://github.com/rrjoson/udacity-item-catalog.git catalog

Create a catalog.wsgi file, then add this inside:
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'supersecretkey'
Rename application.py to init.py mv application.py __init__.py

Install virtual environment

Install the virtual environment sudo pip install virtualenv
Create a new virtual environment with sudo virtualenv venv
Activate the virutal environment source venv/bin/activate
Change permissions sudo chmod -R 777 venv
Install Flask and other dependencies

Install pip with sudo apt-get install python-pip
Install Flask pip install Flask
Install other project dependencies sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils
Update path of client_secrets.json file

nano __init__.py
Change client_secrets.json path to /var/www/catalog/catalog/client_secrets.json
Configure and enable a new virtual host

Run this: sudo nano /etc/apache2/sites-available/catalog.conf
Paste this code:
<VirtualHost *:80>
    ServerName    52.33.103.134
    ServerAlias ec2-52-33-103-134.us-west-2.compute.amazonaws.com
    ServerAdmin admin@52.33.103.134
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
Enable the virtual host sudo a2ensite catalog
Install and configure PostgreSQL

sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit

Change create engine line in your __init__.py and database_setup.py to: engine = create_engine('postgresql://catalog:password@localhost/catalog')
python /var/www/catalog/catalog/database_setup.py

Make sure no remote connections to the database are allowed. 

Check if the contents of this file sudo nano /etc/postgresql/9.3/main/pg_hba.conf looks like this:
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
Restart Apache

sudo service apache2 restart

Visit site at http://52.33.103.134

Special Thanks to rrjoson and SteveWooding for a very helpful README