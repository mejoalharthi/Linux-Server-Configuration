# Linux Server Configuration Project

This project is for install a Linux server and prepare it to host web applications, secure server from a number of attack vectors, install and configure a database server, and deploy one of existing web applications onto it.

## IP & Hostname & SSH port
- Host Name: ec2-18-130-87-242.eu-west-2.compute.amazonaws.com
- IP Address: 18.130.87.242
- SSH port: 2200

## Amazon Lightsail Setup

1. Login to [Amazon Lightsail](https://aws.amazon.com/lightsail/) website. (if you don't have account you should create one)
2. create instance and Select OS Only and Ubuntu 16.04 then click create.
3. Once instance is running click on it and click "Account Page" at the bottom for download your private.(SSH key The file type is .pem)
4. Go to the networking tab click add another and choose **Custom** for application, **TCP** for protocol, and the port number 123 , 2200. Then click save.

## Linux Configuration
1. Now move your downloaded key to .ssh folder you can find it in windows at this path C:\Users\WinDows\.ssh.
2. For make our key secure type `$ chmod 600 ~/.ssh/udacity.pem` into the terminal.
3. Login into server as the user `ubuntu` with our key `$ ssh -i ~/.ssh/udacity.pem ubuntu@18.130.87.242`.
4. Switch to the root user by typing `sudo su -`.
5. Create `grader` user `$ sudo adduser grader`.
6. To give the user **grader** superuser privileges `$ sudo nano /etc/sudoers.d/grader`. the nano file will open type `grader ALL=(ALL:ALL)`.
7. Now we should **update** package list, **upgrade** the current packages ```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade

```
8. Install **Finger** `$ sudo apt-get install finger`
9. Create an SSH Key for new user **grader** open a new terminal type: `$ ssh-keygen -f ~/.ssh/udacity.rsa`
10. Copy the key from the same terminal `$ cat ~/.ssh/udacity.rsa.pub` Now **Copy** the key from terminal.
11. Go to first terminal and type `$ cd /home/grader`.
12. Create a directory `$ mkdir .ssh`
13. Create a file to store public key `$ touch .ssh/authorized_keys`
14. Open **authorized_keys** file `$ nano .ssh/authorized_keys` and paste into it the public key
15. Change the permissions of folder and file
```
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys
```
16. Change the owner of the `.ssh` directory to **grader** `$ sudo chown -R grader:grader /home/grader/.ssh`.
17. Restart SSh service `$ sudo service ssh restart`
18. Now login as grader from terminal `$ ssh -i ~/.ssh/udacity.rsa grader@18.130.87.242`
19. Open ssh configuration file `$ sudo nano /etc/ssh/sshd_config`. Find **Port 22** and change it to **Port 2200**.Change **PasswordAuthentication** to **no**. then change **PermitRootLogin** to **no**.
20.  Restart SSh service `$ sudo service ssh restart`
21. Login again with change port `$ ssh -i ~/.ssh/udacity.rsa grader@18.130.87.242 -p 2200`
22. Configure the firewall :
```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw deny 22
$ sudo ufw enable
```
## Application Deployment
23. Configure the local timezone to UTC `$ sudo timedatectl set-timezone UTC`.
24. Installing the required software
```
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi python-dev
$ sudo apt-get install git
```
25. Enable mod_wsgi `$ sudo a2enmod wsgi`.
26. Restart Apache using `$ sudo service apache2 restart`.
27. Create a directory for catalog application and make the user **grader** the owner.
```
$ cd /var/www
$ sudo mkdir catalog
$ sudo chown -R grader:grader catalog
$ cd catalog
```
28. Cloning Catalog Application repository by `$ git clone [repository url] catalog`
29. Create the .wsgi file `$ sudo nano catalog.wsgi` and type:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```
30. Rename `application.py` to `__init__.py` by `$ mv project.py __init__.py`.
31. Create virtual environment, you should be into `/var/www/catalog`.
```
$ sudo pip install virtualenv
$ sudo virtualenv venv
$ source venv/bin/activate
$ sudo chmod -R 777 venv
```
32.  Install all packages required for our Flask application.
```
$ sudo apt-get install python-pip
$ sudo pip install flask
$ sudo pip install httplib2 oauth2client sqlalchemy psycopg2
$ sudo pip install requests
$ sudo pip install --upgrade oauth2client
$ sudo apt-get install libpq-dev
$ sudo pip install sqlalchemy_utils
```
33. Open `$ sudo nano __init__.py` and find `client_secrets.json` change it's path to `/var/www/catalog/catalog/client_secrets.json`
34. Configure and enable virtual host to run the site `$ sudo nano /etc/apache2/sites-available/catalog.conf`
Paste in the following:
```
<VirtualHost *:80>
    ServerName [Public IP]
    ServerAlias [Hostname]
    ServerAdmin admin@35.167.27.204
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
```
35. For finding your servers hostname go to [here](https://whatismyipaddress.com/ip-hostname) and paste the IP address.
36. Enable to virtual host: `$ sudo a2ensite catalog.conf`
37. **DISABLE** the default host `$ a2dissite 000-default.conf`.


## Database Setup

1. Install the following:
```
$ sudo apt-get install libpq-dev python-dev
$ sudo apt-get install postgresql postgresql-contrib
$ sudo su - postgres -i
$ psql
```
2. Create a database user and password
```
postgres=# CREATE USER catalog WITH PASSWORD [password];
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog with OWNER catalog;
postgres=# \c catalog
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
catalog=# \q
$ exit
```
3. Nano again to edit this files: ` __init__.py`, `database_setup.py`, and `createitems.py`  change the database **engine** from `sqlite://catalog.db` to `postgresql://username:password@localhost/catalog`.
4. Restart apache server `$ sudo service apache2 restart`



## Google OAuth Setup

1. Go to [console cloud google](https://console.developers.google.com/)
2. Add to Authorized domains **`xip.io`** From **OAuth consent screen** tab.
3. Add to Authorized JavaScript: http://18.130.87.242.xip.io
4. Add to redirect URIs: http://18.130.87.242.xip.io/login , http://18.130.87.242.xip.io/gconnect
5. Download the updated JSON file
6. Edit `client_secrets.json` file `$ sudo nano client_secrets.json` **Paste** into it the updated JSON file
7. Restart apache server `$ sudo service apache2 restart`

### References
- https://github.com/mulligan121/Udacity-Linux-Configuration
