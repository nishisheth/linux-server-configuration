# Linux Server Configuration 

In this project, the Catalog Web Application will be hosted on a Ubuntu Linux server using an Amazon Lightsail instance. This document demonstrate steps to configure linux server to host any website. You can visit http://ec2-13-237-224-118.ap-southeast-2.compute.amazonaws.com/ for the website deployed.

Public IP address: 13.237.224.118
SSH port: 2200
URL: http://ec2-13-237-224-118.ap-southeast-2.compute.amazonaws.com

## Create a new Ubuntu Linux Server instance using Amazon Lightsail
1. Create an AWS account
2. Click **Create instance** button on the home page
3. Select **Linux/Unix** platform
4. Select **OS Only** and **Ubuntu** as blueprint
5. Select an instance plan
6. Name your instance
7. Click **Create** button

Please wait until instance comes in running status. 

## SSH into your server 
1. Download private key from the **SSH keys** section in the **Account** section on Amazon Lightsail. The file name should be like `LightsailDefaultKey-ap-southeast-2.pem`
2. Create a new file named **lightsail_key.rsa** under ~/.ssh folder on your local machine
3. Copy and paste content from downloaded private key file to **lightsail_key.rsa**
4. Set file permission as owner only : `$ chmod 600 ~/.ssh/lightsail_key.rsa`
5. SSH into the instance: `$ ssh -i ~/.ssh/lightsail_key.rsa ubuntu@13.237.224.118`

## Update installed packages
1. Run `sudo apt-get update` to update packages
2. Run `sudo apt-get upgrade` to install newest versions of packages
3. Set for future updates: `sudo apt-get dist-upgrade`


## Apply security settings 

####  Change the SSH port from 22 to 2200 and restrict root login
1. Run `$ sudo nano /etc/ssh/sshd_config` to open up the configuration file
2. Change the port number from **22** to **2200** in this file
3. Change `PermitRootLogin` setting to `no`. This will not allow root user to change any configuration of the server.
3. Save and exit the file
4. Restart SSH: `$ sudo service ssh restart`

#### Configure firewall

Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), NTP (port 123) and HTTP (port 80).

* Check firewall status: `$ sudo ufw status`

```
sudo ufw status                  # The UFW should be inactive.
sudo ufw default deny incoming   # Deny any incoming traffic.
sudo ufw default allow outgoing  # Enable outgoing traffic.
sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
sudo ufw allow www               # Allow HTTP traffic in.
sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
sudo ufw deny 22                 # Deny tcp and udp packets on port 53.
```
* Enable firewall: `$ sudo ufw enable`

* Check out current firewall status: `$ sudo ufw status`. Output should be like this: 

```
Status: active

To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
123/udp                    ALLOW       Anywhere                  
22                         DENY        Anywhere                  
2200/tcp (v6)              ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
123/udp (v6)               ALLOW       Anywhere (v6)             
22 (v6)                    DENY        Anywhere (v6)
```

* Log in to Amazon Ligtsail and update the firewall configuration under **Networking**. Delete default SSH port 22 and add **port 80, 123, 2200**

* Open up a new terminal and you can now ssh in via the new port 2200: `$ ssh -i ~/.ssh/lightsail_key.rsa ubuntu@13.237.224.118 -p 2200`

## Add new user named `grader` and configure sudo access for it 

 - While logged in as `ubuntu`, create a new user account **grader**:`$ sudo adduser grader`. Please enter a password (twice) and fill out information for this new user.

- Edits the sudoers file: `sudo visudo`.
- Search for the line that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `grader` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

- Save and exit using CTRL+X and confirm with Y.
- Verify that `grader` has sudo permissions. Run `su - grader`, enter the password, 
run `sudo -l` and enter the password again. The output should be like this:

  ```
  Matching Defaults entries for grader on ip-172-26-13-170.us-east-2.compute.internal:
      env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
  
  User grader may run the following commands on ip-172-26-13-170.us-east-2.compute.internal:
      (ALL : ALL) ALL
  ```

### Create an SSH key pair for `grader` using the `ssh-keygen` tool

#### On the local machine:
  - Run `ssh-keygen`
  - Enter file in which to save the key (e.g. `grader_key`) in the local directory `~/.ssh`
  - Enter in a passphrase twice. Two files will be generated (`~/.ssh/grader_key` and `~/.ssh/grader_key.pub`)
  - Run `cat ~/.ssh/grader_key.pub` and copy the contents of the file

#### On the grader's virtual machine:

  - Create a new directory called `~/.ssh` (`mkdir .ssh`)
  - Run `sudo nano ~/.ssh/authorized_keys` and paste the copied content into this file, save and exit
  - Give the permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
  - Check in `/etc/ssh/sshd_config` file if `PasswordAuthentication` is set to `no`
  - Restart SSH: `sudo service ssh restart`
  - On the local machine, run: `ssh -i ~/.ssh/grader_key -p 2200 grader@13.237.224.118`. You should be logged in successfully as `grader` user using your SSH key pair. 

### How to deploy application code to server? 

##### Step 1: Configure the local timezone to UTC

- While logged in as `grader`, configure the time zone: `sudo dpkg-reconfigure tzdata`. Choose **None of the above** to set timezone to UTC. 

##### Step 2: Install and configure apache server

1. Install **Apache**: `$ sudo apt-get install apache2`
2. Go to http://13.237.224.118/. If Apache is working correctly, a **Apache2 Ubuntu Default Page** will show up

##### Step 3: Install and configure Python mod_wsgi

1. Install the **mod_wsgi** package: `$ sudo apt-get install libapache2-mod-wsgi python-dev`
2. Enable **mod_wsgi**: `$ sudo a2enmod wsgi`
3. Restart **Apache**: `$ sudo service apache2 restart`
4. Check if Python is installed: `$ python`

##### Step 4: Install PostgreSQL

1. Run `$ sudo apt-get install postgresql`
2. Please Open file: `$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf` and make sure PostgreSQL does not allow remote connections.
3. Check to make sure it looks like this:
   ```
   # Database administrative login by Unix domain socket
   local   all             postgres                                peer

   # TYPE  DATABASE        USER            ADDRESS                 METHOD

   # "local" is for Unix domain socket connections only
   local   all             all                                     peer
   # IPv4 local connections:
   host    all             all             127.0.0.1/32            md5
   # IPv6 local connections:
   host    all             all             ::1/128                 md5
   ```

##### Step 5: Create new PostgreSQL user called **catalog**
1. Switch to PostgreSQL defualt user **postgres**: `$ sudo su - postgres`
2. Connect to PostgreSQL: `$ psql`
3. Create user **catalog** with LOGIN role: `# CREATE ROLE catalog WITH PASSWORD 'password';`
4. Allow user to create database tables: `# ALTER USER catalog CREATEDB;`
5. Create database: `# CREATE DATABASE catalog WITH OWNER catalog;`
6. Connect to database **catalog**: `# \c catalog`
7. Revoke all the rights: `# REVOKE ALL ON SCHEMA public FROM public;`
8. Grant access to **catalog**: `# GRANT ALL ON SCHEMA public TO catalog;`
9. Exit psql: `\q` and Exit user **postgres**: `exit`

##### Step 6: Create new Linux user called **catalog** and new database called `catalog`
1. Create a new Linux user: `$ sudo adduser catalog`
2. Give **catalog** user sudo access:
   * `$ sudo visudo`
   * Add `$ catalog ALL=(ALL:ALL) ALL` under line `$ grader ALL=(ALL:ALL) ALL`
   * Save and exit using CTRL+X and confirm with Y.
3. Log in as **catalog**: `$ sudo su - catalog`
4. Create database **catalog**: `createdb catalog`
5. Exit user **catalog**: `exit`


##### Step 7: Install git and clone item catalog application from github
1. Run `$ sudo apt-get install git`
2. Create dictionary: `$ mkdir /var/www/catalog`
3. CD to this directory: `$ cd /var/www/catalog`
4. Clone the catalog app: `$ sudo git clone https://github.com/nishisheth/item-catalog.git catalog`
5. Change the ownership: `$ sudo chown -R ubuntu:ubuntu catalog/`
6. CD to `/var/www/catalog/catalog`
7. Change file **application.py** to **__init__.py**: `$ mv application.py __init__.py`
8. Change line `app.run(host='0.0.0.0', port=8000)` to `app.run()` in **__init__.py** file

##### Step 8: Authenticate login using Google OAuth

- Go to [Google Cloud Plateform](https://console.cloud.google.com/).
- Click `APIs & services` on left menu.
- Click `Credentials`.
- Create an OAuth Client ID (under the Credentials tab), and add http://13.237.224.118 and 
http://ec2-13-237-224-118.ap-southeast-2.compute.amazonaws.com as authorized JavaScript origins.
- Add http://ec2-13-237-224-118.ap-southeast-2.compute.amazonaws.com/oauth2callback as authorized redirect URI.
- Download the corresponding JSON file, open it et copy the contents.
- Open `/var/www/catalog/catalog/client_secret.json` and paste the copied contents into the this file.
- Replace the client ID to line 15 of the `templates/login.html` file in the project directory. It should look like: 

```
<meta name="google-signin-client_id" content="<CLIENT_ID>.apps.googleusercontent.com">
```

##### Step 9: Install dependencies

1. Install pip: `$ sudo apt-get install python-pip`
2. Install packages:
```
   $ sudo pip install httplib2
   $ sudo pip install requests
   $ sudo pip install --upgrade oauth2client
   $ sudo pip install sqlalchemy
   $ sudo pip install flask
   $ sudo apt-get install libpq-dev
   $ sudo pip install psycopg2
   ```

##### Step 10: Configure and enable virtual host 

1. Create file: `$ sudo touch /etc/apache2/sites-available/catalog.conf`
2. Add the following to the file:
```
   <VirtualHost *:80>
    ServerName 13.237.224.118
    ServerAlias ec2-13-237-224-118.ap-southeast-2.compute.amazonaws.com
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
3. Run `$ sudo a2ensite catalog` to enable the virtual host
4. Restart **Apache**: `$ sudo service apache2 reload`

##### Step 11: Configure .wsgi file

1. Create file: `$ sudo touch /var/www/catalog/catalog.wsgi`
2. Add content below to this file and save:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application
application.secret_key = "..."
```
3. Restart **Apache**: `$ sudo service apache2 reload`

##### Step 12: Configure postgresql the database path 

1. CD to `/var/www/catalog/catalog` folder. 
2. Open `connect_database.py` file. Update database path as below.
```
engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
```
3. Open `database_setup.py` file. Update database path as below on line 103. 
```
engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
```
##### Step 13: Disable default apache config 

1. `$ sudo a2dissite 000-default.conf`
2. Restart **Apache**: `$ sudo service apache2 reload`

##### Step 14: Setup database schema 

1. Run `$ sudo python database_setup.py`
2. Run `$ sudo python database_populate.py`
3. Restart **Apache**: `$ sudo service apache2 reload`

##### Step 15: Test application on browser

Follow the link to http://ec2-13-237-224-118.ap-southeast-2.compute.amazonaws.com the application should be running online

If internal errors occur please chech error log: `sudo tail /var/log/apache2/error.log`

### After applying all above settings `var/www` folder structure on web server looks like this:

```
.
└── catalog
    ├── catalog         # item catalog application folder from github
    │   ├── client_secrets.json
    │   ├── connect_database.py
    │   ├── connect_database.pyc
    │   ├── database_populate.py
    │   ├── database_setup.py
    │   ├── database_setup.pyc
    │   ├── __init__.py
    │   ├── static
    │   │   ├── css
    │   │   │   └── styles.css
    │   │   ├── js
    │   │   │   └── js.cookie-2.0.4.min.js
    │   │   └── mdl
    │   │       ├── bower.json
    │   │       ├── LICENSE
    │   │       ├── material.css
    │   │       ├── material.js
    │   │       ├── material.min.css
    │   │       ├── material.min.css.map
    │   │       ├── material.min.js
    │   │       ├── material.min.js.map
    │   │       └── package.json
    │   └── templates
    │       ├── add_item.html
    │       ├── delete_item.html
    │       ├── edit_item.html
    │       ├── index.html
    │       ├── item.html
    │       ├── items.html
    │       ├── layout.html
    │       ├── login.html
    │       ├── new_item.html
    │       └── user_items.html
    ├── catalog.wsgi
    ├── README.md
    └── Vagrantfile

```
## Sources
1. [Amazon Lightsail Website](https://aws.amazon.com/lightsail/?p=tile)
2. [Google API Concole](https://console.cloud.google.com/)
3. [Apache](https://httpd.apache.org/docs/2.2/configuring.html)