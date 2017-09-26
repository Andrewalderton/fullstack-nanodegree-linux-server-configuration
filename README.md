Udacity Linux Server Configuration
==================================

## Description

Final project for the Udacity Full Stack Nanodegree:

> You will take a baseline installation of a Linux server and prepare it to host your web applications. You will secure your server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it.

Amazon Lightsail was used to host the server and is accessible at:

* IP Address: ~~35.177.37.255~~

* SSH Port: ~~2200~~

* Application URL: ~~[http://ec2-35-177-37-255.eu-west-2.compute.amazonaws.com](http://ec2-35-177-37-255.eu-west-2.compute.amazonaws.com)~~

The application deployed on the server is the [Item Catalog](https://github.com/Andrewalderton/fullstack-nanodegree-item-catalog) project developed earlier in the course.

**Please Note: Amazon account has been closed since completing the course, therefore the server is no longer accessible.**

## Server Configuration

Start a new Ubuntu Linux server instance on Amazon Lightsail and follow the instructions provided to SSH into the server.

Update all currently installed packages:

* `sudo apt-get update`
* `sudo apt-get upgrade`

Change the SSH port from 22 to 2200:

* `sudo nano /etc/ssh/sshd_config`
* Edit the 'ports listened for' to: `Port 2200`
* Save the file and run: `sudo service ssh restart`

Configure the Lightsail firewall to allow the change by navigating to the 'Networking' tab on the Lightsail dashboard in the browser and adjusting the firewall settings.

Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for **SSH (port 2200), HTTP (port 80), and NTP (port 123)**:

* `sudo ufw default deny incoming`
* `sudo ufw default allow outgoing`
* `sudo ufw allow ssh`
* `sudo ufw allow 2200/tcp`
* `sudo ufw allow www`
* `sudo ufw allow ntp`
* `sudo ufw enable`

Check firewall status:

* `sudo ufw status`

Delete any unwanted connection rules:

* `sudo ufw delete <nameOfRule>`

Adjust, as needed, the firewall settings in Lightsail to reflect these changes.


### Create a new user named 'grader':

* `sudo adduser grader`
* `nano /etc/sudoers`
* `touch /etc/sudoers.d/grader`
* `nano /etc/sudoers.d/grader`
* Add the text: `grader ALL=(ALL:ALL) ALL`
* Save and quit

Set up SSH login for grader:

* Generate keys on local machine using `ssh-keygen`
* Save the private key in `~/.ssh` on the local machine

Add public key to the server:

* `su - grader`
* `mkdir .ssh`
* `touch .ssh/authorized_keys`
* `nano .ssh/authorized_keys`
* Copy the public key generated on the local machine to this file and save

Set permissions:

* `chmod 700 .ssh`
* `chmod 644 .ssh/authorized_keys`
* Reload SSH: `service ssh restart`

**Log in as 'grader' using SSH**: `ssh grader@35.177.37.255 -p 2200 -i ~/.ssh/<privateKeyFilename>`

### Configure local time zone to UTC:

* `sudo dpkg-reconfigure tzdata`
* If time zone is not already UTC, select the option '**none of the above**', then choose '**UTC**' from the list which follows

### Install and configure Apache to serve a Python mod_wsgi application:

Install Apache:

* `sudo apt-get install apache2`

Install mod_wsgi:

* `sudo apt-get install python-setuptools libapache2-mod-wsgi`

Enable mod_wsgi:

* `sudo a2enmod wsgi`

Start Apache:

* `sudo service apache2 start`

### Install and configure PostgreSQL:

Install PostgreSQL:

* `sudo apt-get install postgresql`

Check that no remote connections are allowed to the database:

* `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

The relevant part of the file should look like this:

    local   all             postgres                                peer
    local   all             all                                     peer
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5

Login as user 'postgres':

* `sudo su - postgres`

Access postgreSQL shell:

* `psql`

Create a new database named 'catalog' and a new user named 'catalog' in postgreSQL shell:

* `CREATE DATABASE catalog;`
* `CREATE USER catalog;`

Set a password for user 'catalog':

* `ALTER ROLE catalog WITH PASSWORD 'password';`

Give user 'catalog' permission to access the 'catalog' database:

* `GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`

Quit psql: `\q`

Exit postgreSQL: `exit`

**Sources**:

* [Digital Ocean - How to Install and use PostgresSQL on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
* [Digital Ocean - How to Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

### Set up the Item Catalog project to be deployed:

Install git:

* `sudo apt-get install git`

Create the application directory:

* `cd /var/www`
* `sudo mkdir catalog`

Move inside this directory:

* `cd catalog`

Clone the repository:

* `git clone https://github.com/Andrewalderton/fullstack-nanodegree-item-catalog`

Rename the repository:

* `sudo mv ./fullstack-nanodegree-item-catalog ./catalog`

Move to the inner catalog directory:

* `cd catalog`

Rename '**application.py**' to '**\__init__.py**':

* `sudo mv application.py __init__.py`

Edit '**database_config.py**', '**books_database.py**' and '**\__init__.py**' using the `sudo nano <filename>` command and change all references to the databse, `engine = create_engine('sqlite:///itemcatalog.db')`, to `engine = create_engine('postgresql://catalog:password@localhost/catalog')`

Update path of '**client_secrets.json**' file:

* `nano __init__.py`
* Change 'client_secrets.json' path to `/var/www/catalog/catalog/client_secrets.json` (this appears twice in the file)

Make sure the .git directory is not publicly accessible via browser:

* `cd /var/www/catalog/`
* `sudo nano .htaccess`
* Add the following to the file and save: `RedirectMatch 404 /\.git`

### Set up the Virtual Environment:

Install pip and virtualenv:

* `sudo apt-get install python-pip`
* `sudo pip install virtualenv`

Move to the catalog folder and create a new virtual environment:

* `cd /var/www/catalog/catalog`
* `sudo virtualenv venv`

Activate the virtual environment:

* `source venv/bin/activate`

Install Flask, psycopg2 and SQAlchemy Utils:

* `sudo pip install Flask`
* `sudo apt-get -qqy install postgresql python-psycopg2`
* `sudo pip install sqlalchemy_utils`

Use pip to install project dependencies:

* `sudo pip install -r requirements.txt`

Configure postgreSQL database schema and add dummy data to Item Catalog:

* `sudo python database_config.py`
* `sudo python books_database.py`


### Configure and Enable the New Virtual Host:

Create a '**catalog.conf**' file:

* `sudo nano /etc/apache2/sites-available/catalog.conf`

Add the following code to configure the virtual host:

    <VirtualHost *:80>
    	ServerName 35.177.37.255
    	ServerAdmin admin@35.177.37.255
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

Enable the virtual host:
* `sudo a2ensite catalog`

Disable default Apache configuration file:
* `sudo a2dissite 000-default`

Create the .wsgi file:
* `cd /var/www/catalog`
* `sudo nano catalog.wsgi`

Add the following code to the file:

    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/catalog/")

    from catalog import app as application
    application.secret_key = 'secret_muffin'

Restart Apache:
* `sudo service apache2 restart`

**Source**: [How to Deploy a Flask App on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
