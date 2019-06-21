
# Linux Server Configuration Project: 
### By Ahmed Alhawsawi
## ReadMe.md 

### Introduction
This project took a baseline installation of a Linux server and prepare it to host the web applications. The need of securing the server from a number of attack vectors, install and configure a database server, and finally deploying Item Catalog web application to this server.

A full understanding of how the web applications work, how they are hosted, and the interactions between multiple systems. All in all, full skills of turning a brand-new, bare bones, Linux server into the secure and efficient web application host applications need will be granted.

### 1st step: Getting the server:
1.	Setting and starting up a new Ubuntu Linux server instance on [Amazon Lightsail](https://aws.amazon.com/lightsail). 
### 2nd step: Securing the server:
1.	Updating all currently installed packages. 
`
sudo apt-get update && apt-get upgrade
`
### Update
Automatically install updates
: This step was added after first review to meet the specifications.

The `unattended-upgrades` package can be used to automatically install important system updates.
- Enable automatic (security) updates: `sudo apt-get install unattended-upgrades`.
- Edit `/etc/apt/apt.conf.d/50unattended-upgrades`, uncomment the line `${distro_id}:${distro_codename}-updates` and save it.
- Modify `/etc/apt/apt.conf.d/20auto-upgrades` file so that the upgrades are downloaded and installed every day:
  ```
  APT::Periodic::Update-Package-Lists "1";
  APT::Periodic::Download-Upgradeable-Packages "1";
  APT::Periodic::AutocleanInterval "7";
  APT::Periodic::Unattended-Upgrade "1";
  ```
- Enable it: `sudo dpkg-reconfigure --priority=low unattended-upgrades`.
- Restart Apache: `sudo service apache2 restart`.

2.	Changing the SSH port from 22 to 2200. Make sure to configure the 
`
sudo nano /etc/ssh/sshd_config
`

```
`Change the ‘Port 22’ to ‘Port 2200’.`
`Change the ‘PasswordAuthentication yes’ to ‘PasswordAuthentication no’.`
`Change the ‘PermitRootLogin yes’ to ‘PermitRootLogin no’.`
`Ctrl+X to save the file and then restart ssh `
```

`
 sudo service ssh restart
`

3. Configure the Uncomplicated Firewall (UFW) to:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
```

#### 3rd step: Giving grader access to the server:
```bash
sudo adduser grader
sudo nano /etc/sudoers.d/grader 
```
_Then add the following text_
>‘grader ALL=(ALL) ALL’
1.	Setup SSH keys for **grader**

On local machine `ssh-keygen` Then choose the path for storing **public** and **private keys** on remote machine home as user **grader**
```bash
sudo su - grader
mkdir .ssh
touch .ssh/authorized_keys 
sudo chmod 700 .ssh
sudo chmod 600 .ssh/authorized_keys 
nano .ssh/authorized_keys 
```
_Then paste the contents of the **public key** created on the local machine_
### 4th step: Preparing to deploy the project:
1.	Configure the local timezone to UTC.

_Log in as_ **grader**, _configure the time zone:_
```bash
sudo dpkg-reconfigure tzdata.
sudo apt-get install ntp
```

2.	Install and configure Apache to serve a Python mod_wsgi application.
`
sudo apt-get install apache2
`

install the Python 2 mod_wsgi package on your server: 
`
sudo apt-get install libapache2-mod-wsgi.
`

install the Python 3 mod_wsgi package on your server: 
`
sudo apt-get install libapache2-mod-wsgi-py3.
`

3.	Install and configure PostgreSQL:
```bash
 sudo apt-get install libpq-dev python3-dev
 sudo apt-get install postgresql postgrwsql-contrib
 sudo su - postgres
 psql
```
Then
```bash
CREATE USER catalog WITH PASSWORD 'catalog';
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit
```
4.	Installing **git**.
`
sudo apt-get install git
`

### 5th step: Deploying the Item Catalog project:
1.	Clone and setup **Item Catalog** project from Github repo.
```bash
cd /var/www/
git clone https://github.com/ahd959/Item_Catalog.git catalog
sudo chown grader:grader catalog
cd catalog
nano catalog.wsgi
```
Then place these lines in it 
```python
activate_this = '/var/www/catalog/catalog/venv3/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

#!/usr/bin/python3
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application
application.secret_key = "super_secret_key"
```
_Restart Apache:_
`
sudo service apache2 restart.
`

2.	Install the virtual environment and dependencies
```bash
sudo apt-get install python3-pip
sudo apt-get install python-virtualenv
cd /var/www/catalog/catalog
```
Create the virtual environment: 
`
sudo virtualenv -p python3 venv3
`

Change the ownership to grader with: 
`
sudo chown -R grader:grader catalog/
`
`
sudo chown -R grader:grader venv3/
`

Activate the new environment:  `
source venv3/bin/activate
`

Install the following dependencies:
```bash
pip install httplib2
pip install requests
pip install --upgrade oauth2client
pip install sqlalchemy
pip install flask
sudo apt-get install libpq-dev
pip install psycopg2
```
Run
`
python3 __init__.py 
`

To deactivate the virtual environment: 
`
deactivate.
`

3.	Set up and enable a virtual host

Add the following line in
`
 /etc/apache2/mods-enabled/wsgi.conf 
`
```
#WSGIPythonPath directory|directory-1:directory-2:...

WSGIPythonPath /var/www/catalog/catalog/venv3/lib/python3.6/site-packages
```
`
sudo nano /etc/apache2/sites-available/catalog.conf 
`

   Add the following lines to configure the virtual host:
   
```
<VirtualHost *:80>
    ServerName 3.19.88.5
  ServerAlias ec2-3-19-88-5.us-west-2.compute.amazonaws.com
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv3/lib/python3.6/site-packages
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
```
##### Server login Info:
```
Application URL: [ec2-3-19-88-5.us-east-2.compute.amazonaws.com](http://ec2-3-19-88-5.us-east-2.compute.amazonaws.com/)
DNS: [carcompany.ddns.net](http://carcompany.ddns.net/)
IP address: [3.19.88.5](http://3.19.88.5/)
SSH port: 2200.
```
##### Useful commands: 

Virtual host:

Enable
`
sudo a2ensite catalog.
`

Disable
`
sudo a2dissite catalog.
`

Apache:
```bash
sudo service apache2 reload.
sudo service apache2 restart. 
sudo service apache2 status.
```

#### References:
```
* https://aws.amazon.com/lightsail/
* http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/#working-with-virtual-environments
* https://httpd.apache.org/
* https://aws.amazon.com/premiumsupport/knowledge-center/new-user-accounts-linux-instance/
* https://www.youtube.com/watch?v=_P29ZHu_feU
* https://www.youtube.com/watch?v=goToXTC96Co
* https://docs.sqlalchemy.org/en/13/orm/cascades.html
* https://raw.githubusercontent.com/stueken/FSND-P5_Linux-Server-Configuration/master/README.md
* https://libraries.io/github/golgtwins/Udacity-P7-Linux-Server-Configuration#udacity-full-stack-web-developer-nanodegree-project-7-linux_server_configuration
* https://devhub.io/repos/iraquitan-udacity-fsnd-p5-linux-server-config
* https://github.com/boisalai/udacity-linux-server-configuration
* https://help.ubuntu.com/community/UFW 
* http://askubuntu.com/questions/256013/could-not-reliably-determine-the-
* https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps 
* https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps 
* https://help.ubuntu.com/lts/serverguide/automatic-updates.html

```
