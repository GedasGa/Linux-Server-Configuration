# Linux Server Configuration

**by Gedas Gardauskas**

This is a a guide on how to set up a Amazon Lightsail instance with Ubuntu Linux Server and to host a simple web application built with Flask framework and PostgreSQL database.

# My Server IP and other details
- Public IP: 35.157.41.25
- User name: ubuntu, grader
- SSH port: 2200

Hosted webpage URL: http://35.157.41.25/


# Configuration steps
## Get your server.
### Create a new instance in Amazon Lightsai.
1. Sign in to [Amazon Lightsail](https://amazonlightsail.com) using an Amazon Web Services account.

1. Follow the 'Create an instance' link.

1. First, choose "OS Only". Then 'Ubuntu 16.04 LTS'.

1. Choose a payment plan.

1. Come up with a unique hostname and click 'Create'.

1. Wait for your instance to start.

### Connect to the instance on a local machine
1. Download the instance's private key by navigating to the Amazon Lightsail 'Account page'

1. Click on 'Download default key'

1. Place your downloaded key file in the local ~/.ssh/ directory.

1. Run `chmod 400 ~/.ssh/LightsailDefaultPrivateKey.pem`

1. To log in execute the following command: `ssh -i ~/.ssh/LightsailDefaultPrivateKey.pem ubuntu@54.93.82.190 -p 22`

(Note: Later we're going to use a different port. `2200` instead of `22`)


## Secure your server.
### Update all currently installed packages.
1. Fetch the list of available updates `sudo apt-get update`

1. Strictly upgrade the current packages `sudo apt-get upgrade`

### Change the SSH port from 22 to 2200
1. Open `/etc/ssh/sshd_config` file with any text editor.

1. Change the port number on line 5 from `22` to `2200`.

1. Now restart SSH by executing this command `sudo service ssh restart`

(Note: Don't forget to configure the Lightsail firewall to allow port `2200`)


### Configure the Uncomplicated Firewall (UFW)
Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. First check if the Uncomplicated Firewall is installed `ufw --version` or by executing `sudo ufw status`

1. Set the UFW firewall to block all incoming connections on all ports by executing `sudo ufw default deny incoming`

1. Set the UFW firewall to allow all incoming connections on all ports by executing `sudo ufw default allow outgoing`

1. Execute `sudo ufw allow 2200/tcp` to allow all tcp connections on port `2200`

1. Execute `sudo ufw allow 80/tcp` to allow all tcp connections on port `80`

1. Execute `sudo ufw allow 123/udp` to allow all udp connections on port `123`

1. Now enable the UFW firewall by executing `sudo ufw enable`

1. You can run `sudo ufw status` to check your Uncomplicated Firewall configuration.

(Note: Don't forget to configure your Amazon Lightsail firewall accordingly.)


## Give grader access.
### Create a new user account named `grader`
1. Run `sudo adduser grader` to add a new user named `grader`

1. Enter a new UNIX password for `grader` (x2) in prompt.

1. Fill out the information for the new `grader` user. (It is unnecessary though)

### Give `grader` user sudo permissions
1. Create a new file /etc/sudoers.d/grader by executing `sudo nano /etc/sudoers.d/grader`

1. Now add a following text `grader ALL=(ALL) NOPASSWD:ALL`

1. Save and close the file.

### Create an SSH key pair for `grader`
1. Run `ssh-keygen` on the local machine.

1. Choose a file name for the key pair and save it `~/.ssh/` (Recommended)

1. Now log in to the Amazon Lightsail instance and switch user to `grader`

1. In his home directory create a new `.ssh` folder by executing `mkdir .ssh`

1. Create a new empty file in `.ssh` running `touch .ssh/authorized_keys`

1. Now copy the content from `.pub` file that you have created on your local machine, and paste it to the .ssh/authorized_keys file on the virtual machine.

1. Change permissions for `.ssh` folder by running `chmod 700 .ssh`

1. Change permissions for `authorized_keys` file by running `chmod 644 .ssh/authorized_keys`


## Prepare to deploy your project.
### Configure the local timezone to UTC
1. You can change your local time zone by running `sudo dpkg-reconfigure tzdata` command

(Note: UTC is under the 'None of the above' category)

### Install and configure Apache to serve a Python mod_wsgi application.
1. First, install Apache by running `sudo apt-get install apache2`

1. Then install mod_wsi package with python-dev package by executing `sudo apt-get install libapache2-mod-wsgi python-dev` command.

### Install additional packages
1. Install `pip` by executing the folling command `sudo apt-get install python-pip`
2. Then install additional packages required for this project.
```
pip install httplib2
pip install requests
pip install sqlalchemy
pip install flask
pip install psycopg2
sudo apt-get install libpq-dev
```
### Virtual Host and `/etc/apache2/sites-available/catalogApp.conf` config file
1. Create a new `catalogApp.conf` file in `/etc/apache2/sites-available/` directory
```
<VirtualHost *:80>
		ServerName 35.157.41.25
		ServerAdmin admin@smth.com
		ServerAlias lightsail-35-157-41-25.eu-central-1.amazonaws.com
		WSGIScriptAlias / /var/www/catalogApp/catalogApp.wsgi
		<Directory /var/www/catalogApp/catalogApp>
			Order allow,deny
			Allow from all
			Options -Indexes
		</Directory>
		Alias /static /var/www/catalogApp/catalogApp/static
		<Directory /var/www/catalogApp/catalogApp/static/>
			Order allow,deny
			Allow from all
			Options -Indexes
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

1. Now enable Virtual Host by running `sudo a2ensite catalogApp`

(Note: Don't forget to reload apapche2 service)

### `/var/www/catalogApp/catalogApp.wsgi` file
1. Create a new `catalogApp.wsgi` file in `/var/www/catalogApp/` directory.
```
activate_this = '/var/www/catalogApp/catalogApp/venv/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalogApp/")

from catalogApp import app as application
application.secret_key = 'super_secret_key'
```

(Note: Don't forget to restart apapche2 service)

### Install and configure PostgreSQL:
1. Install PostgreSQL by running `sudo apt-get install postgresql`

1. Check PostgreSQL `.conf` by running `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

### Create a new database user named `catalog` that has limited permissions to your catalog application database.
1. Switch to user `postgres` by running `sudo su - postgres` (This user is created during PostgreSQL installation)

1. Now connect PostgreSQL database by executing `psql`

1. To create a new database user named `catalog` just type `CREATE ROLE catalog WITH LOGIN;` and press Enter.

1. Now alter the `catalog` user table so he could add create a database. Just type `ALTER ROLE catalog CREATEDB;` and press Enter.

1. Connect with user `catalog` by running `\c catalog`

1. You can create a new database with `catalog` user. Creating database named catalog by typing `CREATE DATABASE catalog;` and pressing Enter.

### Install `git`.
1. Just run `sudo apt-get install git`
1. Done!

(Note: Usually `git` is already pre-installed in Amazon Lightsail Ubuntu instance)

#References
[How To Deploy a Flask Application on Ubuntu VPS Tutorial](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

[Pipenv & Virtual Enviroments Turorial](http://docs.python-guide.org/en/latest/dev/virtualenvs)

[Flask mod_wsgi(Apache) documentation](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/#working-with-virtual-environments)

[Tutorialspoint.com PostgreSQl Tutorial](https://www.tutorialspoint.com/postgresql/)

[Flask Application with PostgreSQL blog post](http://www.patricksoftwareblog.com/database-using-postgresql-and-sqlalchemy/)

[Changing postres primary key sequence](http://www.jakobbeyer.de/change-postres-primary-key-sequence)

[Udacity Discussion Forum post: List of items in the directory instead of displaying site](List of items in the directory instead of displaying site)
