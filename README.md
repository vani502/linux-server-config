# Linux Server Configuration Project

In this project you take a baseline installation of a Linux server and prepare it to host a web applications. You will secure your server, install and configure a database server, and deploy an existing web applications onto it.

Project website
  * [http://18.223.48.31.xip.io] - added xip.io as Google OAuth longer works with a raw IP address
  * [http://ec2-18-223-48-31.us-east-2.compute.amazonaws.com]

## Get server

### Start a new Ubuntu Linux server instance
* Log in to [Amazon Lightsail](https://lightsail.aws.amazon.com/)
* Once logged in create an instance
* Choose Linux/Unix platform, OS Only and Ubuntu
* Choose an instance plan pick the lowest one to get free-tier access for a month
* Give your instance a host name and click create
* Wait for the instance to start up

### SSH into your server

* In Lighsaild select Account on the top navigation bar, and then select Account from the drop-down menu
* Select SSH Keys tab and download default key
* Move downloaded private key into your local folder ~/.ssh
* Connect via terminal ssh -i ~/.ssh/LightsailDefaultPrivateKey-us-east-2.pem -p 2200 ubuntu@18.223.48.31

## Secure server

* Update all currently installed packages
```
sudo apt-get update
sudo apt-get upgrade
```
* Change the SSH port from 22 to 2200
  1. Edit /etc/ssh/sshd_configsudo
    * nano /etc/ssh/sshd_config
  2. Change the Port number from 22 to 2200
  3. Restart SSH service
    * sudo service ssh restart


* Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
sudo ufw status
```

## Give grader access

* Create a new user account named `grader`
```
sudo adduser grader
```
* Give `grader` `sudo` permissions
```
sudo nano /etc/sudoers.d/grader
```
* Create an SSH key pair for `grader`

* Generate key on local machine and save to ~/.ssh folder
```
ssh-keygen
```
* Deploy key
```
su - grader
mkdir .ssh
touch .ssh/authorized_keys
nano .ssh/authorized_keys
```
* Set permissions
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```
* Restart SSH: `sudo service ssh restart`

* Log in as `grader`
  ```
  ssh -i ~/.ssh/linuxGrader -p 2200 grader@18.223.48.31
  ```

## Prepare to deploy project

* Configure the local timezone to UTC
```
sudo dpkg-reconfigure tzdata
```
* Install and configure Apache to serve a Python mod_wsgi application

  * Install Apache `sudo apt-get install apache2`

  * Install mod_wsgi ` sudo apt-get install libapache2-mod-wsgi`

  * Restart Apache `sudo service apache2 restart`

* Install and configure PostgreSQL

  * Install PostgreSQL `sudo apt-get install postgresql postgresql-contrib`

  * Switch to the `postgres` user: `sudo su - postgres`

  * Open PostgreSQL with `psql`

  * Create new catalog database and new user named catalog
  ```
  postgres=# CREATE DATABASE catalog;
  postgres=# CREATE USER catalog;
  ```

  * Set password for user catalog
  ```
  postgres=# ALTER ROLE catalog WITH PASSWORD 'password';
  ```

  * Provide user permission to catalog database
  ```
  postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
  ```

  * Quit PostgreSQL `postgres=# \q`

  * Exit from "postgres" `exit`

* Install git

  `sudo apt-get install git`

## Deploy the Item Catalog project

* Clone and setup Item Catalog project

  * Move to the /var/www directory
  ```
  cd /var/www
  ```
  * Create application directory
  ```
  sudo mkdir FlaskApp
  ```
  * Move inside this directory
  ```
  cd FlaskApp
  ```
  * Clone catalog application
  ```
  git clone https://github.com/vani502/catalog-application.git
  ```
  * Rename the project's name
  ```
  sudo mv ./catalog-application ./FlaskApp
  ```
  * Move inside this directory
  ```
  cd FlaskApp
  ```
  * Rename `project.py` to `__init__.py`
  ```
  sudo mv project.py __init__.py
  ```
  * Update `database_setup.py`, `__init__.py` and `lotsofitems.py`
  ```
  engine = create_engine('sqlite:///catalog.db')
  engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')
  ```

* Update Google OAuth client secrets file
  * Update `credentials` in [Google Cloud Platform](https://console.cloud.google.com/home/dashboard?project=catalog-application-213323)
    * Add urls to `Authorized JavaScript origins`
    ```
      http://18.223.48.31.xip.io

      http://ec2-18-223-48-31.us-east-2.compute.amazonaws.com
    ```
    * Add urls to `Authorized redirect URIs`
    ```
      http://18.223.48.31.xip.io/login

      http://18.223.48.31.xip.io/gconnect

      http://ec2-18-223-48-31.us-east-2.compute.amazonaws.com/login

      http://ec2-18-223-48-31.us-east-2.compute.amazonaws.com/gconnect
    ```
  * Edit `redirect_uris` and `javascript_origins` with urls in the  `client_secrets.json` file
  ```
  sudo nano client_secrets.json
  ```


* Install Flask and other dependencies
```
sudo apt-get install python-psycopg2 python-flask
sudo apt-get install python-sqlalchemy python-pip
sudo pip install oauth2client
sudo pip install requests
sudo pip install httplib2
```
* Configure and Enable Virtual Host

  * Create FlaskApp.conf file
  ```
  sudo nano /etc/apache2/sites-available/FlaskApp.conf
  ```
  * Add the following lines of code, save and close
  ```
  <VirtualHost *:80>
    ServerName http://18.223.48.31
    ServerAdmin admin@18.223.48.31.com
    ServerAlias ec2-18-223-48-31.us-east-2.compute.amazonaws.com/
    WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
		<Directory /var/www/FlaskApp/FlaskApp/>
			Order allow,deny
			Allow from all
		</Directory>
		Alias /static /var/www/FlaskApp/FlaskApp/static
		<Directory /var/www/FlaskApp/FlaskApp/static/>
			Order allow,deny
			Allow from all
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```
  * Enable the virtual host
  ```
  sudo a2ensite FlaskApp.conf
  ```

* Create the .wsgi File

  * Create the .wsgi File under /var/www/FlaskApp
  ```
  cd /var/www/FlaskApp
  sudo nano flaskapp.wsgi
  ```
  * Add the following lines of code to the flaskapp.wsgi file

    ```
    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/FlaskApp/")

    from FlaskApp import app as application

    application.secret_key = 'Add your secret key'
    ```
* Restart Apache
```
sudo service apache2 restart
```

## Resources
  * [https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1604]
  * [https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps]
  * [https://discussions.udacity.com/t/google-oauth-api-and-authorized-redirect-uris-with-ip-adr-not-allowed/515313]
