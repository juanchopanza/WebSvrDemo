# WebSvrDemo
Web server for Udacity full stack nanodegree project 5

## Set up

Unless specified otherwise, these steps have been implemented using information either from Udacity's *Configuring Linux Web Servers* course or from experienced accrued over many years of playing around with Linux boxes.

#### 1. Create a new user named `grader`

    addusr grader
    
#### 2. Give the grader the permission to sudo

Create file `/etc/sudoers.d/grader` with contents

    grader ALL=(ALL) NOPASSWD:ALL
    
#### 3. Set up ssh login for `grader`

    su - grader
    mkdir .ssh
    chmod 700 .ssh
    
Then copy `root`'s `.ssh/authorized_keys` and change permissions

    sudo cp /root/.ssh/authorized_keys .ssh/authorized_keys
    sudo chown grader:grader .ssh/authorized_keys 
    chmod 644 .ssh/authorized_keys

#### 4. Upgrade packages

##### 4.a Initial update of all currently installed packages

Change user to `grader` and use `sudo apt-get`

    su - grader
    sudo apt-get update
    sudo apt-get upgrade

##### 4.b Set up automatic security updates
    
**Note**: This is an extra requirement of the project. However, in a real life, critical application I would not have enabled automatic upgrading of packages. In the interest of stability, upgrages would be applied manually after careful evaluation. We would then phase un upgrades in a dev machine(s) before pushing into production.

We will use the `unattended-upgrades` package.

```shell
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

This opens a console application that prompts the user. Select "yes".



Source [Ubuntu doecumentation on AutomaticSecurityUpdates](https://help.ubuntu.com/community/AutomaticSecurityUpdates).

#### 5. Fix warning `sudo: unable to resolve host ip-xx-yy-zz-xyz`

Hostname is in `/etc/hostname`. Add that hostname to `/etc/hosts`:

    127.0.1.1 ip-xx-yy-zz-xyz
    
And reboot the machine

    sudo reboot
    
* Source: [AskUbuntu](http://askubuntu.com/questions/59458/error-message-when-i-run-sudo-unable-to-resolve-host-none)

#### 6. Extra: disable root ssh access

`sudo vim /etc/ssh/sshd_config` and set

    PermitRootLogin no
    
For safety, remove `root`'s `.ssh/authorized_keys`. These are not needed anymore.

    sudo rm /root/.ssh/authorized_keys
    
Then restart ssh:

    sudo service ssh restart

#### 7. change SSH port to 2200

Modify relevant line in `/etc/ssh/sshd_config` to

    Port 2200
    
and restart `ssh`

    sudo service ssh restart
    
We can now log in with

    ssh -i ~/.ssh/udacity_key.rsa grader@52.88.73.214 -p 2200
    
#### 8. Configure firewall and protect against repeated failed login attempts

##### 8.a Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

Check UFW status

    $ sudo ufw status
    Status: inactive

Allow SSH, HTTP, NTP

    # Disallow incoming
    sudo ufw default deny incoming
    # Allow outgoing
    sudo ufw default allow outgoing
    # Allow ssh (port 2200)
    sudo ufw allow 2200/tcp
    # Allow HTTP (port 80)
    sudo ufw allow www
    # Allow NTP (port 123)
    sudo ufw allow ntp
    
Enable the UFW

    sudo ufw enable
    
Check status

    $ ufw status
    Status: active
     
    To                         Action      From
    --                         ------      ----
    2200/tcp                   ALLOW       Anywhere
    80/tcp                     ALLOW       Anywhere
    123                        ALLOW       Anywhere
    2200/tcp (v6)              ALLOW       Anywhere (v6) 
    80/tcp (v6)                ALLOW       Anywhere (v6) 
    123 (v6)                   ALLOW       Anywhere (v6)
    
##### 8.b Protect against repeated unsuccesful login attempts

We will use [fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page), and configure
it to e-mail to admin.


```shell
sudo apt-get install fail2ban
sudo apt-get install sendmail
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Modify the configuration file `/etc/fail2ban/jail.local`:

Set the ssh port to 2200. Under `[ssh]`,

```shell
port = 2200
```

Change the default action to ban, send e-mail with whois repots and loglines.

```shell
action = %(action_mwl)s
```

Re-start the service

```shell
sudo service fail2ban restart
```

Source: ["How To Protect SSH with Fail2Ban on Ubuntu 14.04 (digital ocean)"](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04).

#### 9. Configure the local timezone to UTC

Using the `date` command we can see the timezone is already set to UTC:

    $ date +%Z
    UTC

Otherwise, reconfigure `tzdata` and set to UTC:

    $ sudo dpkg-reconfigure tzdata

However, we can set up the Network Time Protocol (NTP) to regularly update the time.

Install ntp and the ntp documentation:

    sudo apt-get install ntp ntp-doc

Check if the NTP daemon is running:

    sudo /etc/init.d/ntp status

If it is running, the command above should output

     * NTP server is running

Otherwise, restart it:

    sudo /etc/init.d/ntp restart

Note that we could select different time servers depending on location if necessary.

Source: [UbuntuTime](https://help.ubuntu.com/community/UbuntuTime).

#### 10. Install Apache2 and documentation

Install apache and apache docs:

    sudo apt-get install apache2 apache2-doc

This produces warning message

    AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message

We fix this by creating file `/etc/apache2/conf-available/fqdn.conf` with the line `ServerName $HOSTNAME`

    echo "ServerName $HOSTNAME" | sudo tee /etc/apache2/conf-available/fqdn.conf
    sudo a2enconf fqdn
    sudo service apache2 reload
    
Test that the warning has disappeared

    sudo service apache2 restart
    
Expected output:

     * Restarting web server apache2
       ...done.

Source: [AskUbuntu](http://askubuntu.com/questions/256013/could-not-reliably-determine-the-servers-fully-qualified-domain-name).

Check that server is running by accessing `http://52.88.73.214/`. That should
produce the Ubuntu Apache2 welcome page.

Install apache2 docs because they are quite useful

    sudo apt-get install apache2-doc

#### 11. Install apache2-mod-wsgi and configure apache to use WSGI module

Install apache WSGI module

    sudo apt-get install libapache2-mod-wsgi

Set up a trivial WSGI application to test that the module is working.

First, add the following line to `/etc/apache2/sites-enabled/000-default.conf`, before
the closing `<VirtualHost>`:

    WSGIScriptAlias / /var/www/html/hello.wsgi

Next, put the following hello world application in `/var/www/html/hello.wsgi`:

    def application(environ, start_response):
        status = '200 OK'
        output = 'Hello, WSGI World!'
    
        response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))]
        start_response(status, response_headers)
    
        return [output]

And re-start apache

    sudo service apache2 restart

Now, visiting `http://52.88.73.214/` should produce a page with

    Hello, WSGI World!

Additional sources: [Apache configuration documentation](https://httpd.apache.org/docs/2.2/configuring.html).

#### 12. Install and configure PostgreSQL

We will use `postgresql` as a database backend for the item catalog app. We need to install
the package and take care of some configuration and database creation.

Install `postgresql`:

```shell
sudo apt-get install postgresql
```

Check only local connections are allowed. See `/etc/postgresql/9.3/main/pg_hba.conf`.

Create unix user `catalog`:

```shell
sudo adduser catalog # provide a suitable password
```

Create `postgres` user `catalog`:

```shell
sudo su postgres -c 'createuser -dRS catalog'
```

Create database `itemcatalog`, owned by user `catalog`

```shell
sudo su postgres -c 'createdb itemcatalog -O catalog'
```

Disable access for all users except `catalog` and set a password:

Launch the `psql` interpreter as `postgres`
```shell
sudo su postgres -c 'psql'
```

Once in the interpreter, issue the following commands:

```shell
REVOKE connect ON DATABASE itemcatalog FROM PUBLIC;
GRANT connect ON DATABASE itemcatalog TO catalog;
ALTER USER catalog WITH PASSWORD '<CATALOG_PWD>';
```

where `<CATALOG_PWD>` is to be set to something sensible.

#### 13. Install other item catalog app dependencies

A few packages are needed to ease the installation and running of the item catalog
application.

Install `git`,`pip`, `virtualenv`, `supervisord`:

```shell
sudo apt-get install git
sudo apt-get install python-dev
sudo apt-get install python-psycopg2
sudo apt-get install python-pip
sudo apt-get install python-virtualenv
sudo apt-get install supervisor
```

The remaining python dependencies will be installed with `pip` into a `virtualenv`.

#### 14. WSGI configuration

We will configure Apache to run an WSGI app from a `virtualenv`. The app should be
demonized such that it starts whenever the server re-starts.

Source: [Flask mod_wsgi(Apache) Configuration](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)

##### 14.1 Install item catalog app and initialize database

We install the item catalog application from project 3 into a virtualenv. By doing this instead of cloning the repo,
we ensure we only install the code necessary to run the application.

```shell
# initialize a virtualenv
sudo virtualenv --system-site-packages /var/www/catalog
# transfer ownership to user catalog
sudo chown -R catalog:catalog /var/www/catalog
# install ItemCatalog app into virtualenv as user catalog
sudo su catalog -c "/var/www/catalog/bin/pip install git+https://github.com/juanchopanza/ItemCatalog.git@v0.0.0"
# initialize the database tables
sudo su catalog -c "/var/www/catalog/bin/init_db.py -t psql"
```

Note: here we install tagged version `v0.0.0`. We do not want to blindly pick up the HEAD
revision.

##### 14.2 Create a directory for secrets and populate it

This application has authentication for for google and facebook sign-in and requires secrets files for both. The
instructions for obtaining these are given in [Udacity's Authentication and Authorization course](https://www.udacity.com/course/viewer#!/c-ud330/l-3951228603/m-3949778775). See also the Full stack nanodegree project 3 for more details. Here, we assume the secrets files have been correctly obtained elsewhere.

```shell
sudo su catalog -c "mkdir -p /var/www/catalog/.secrets"
sudo cp <some_secret_location>/client_secrets.json /var/www/catalog/.secrets/.
sudo cp <some_secret_location>/fb_client_secrets.json /var/www/catalog/.secrets/.
sudo chown -R catalog:catalog /var/www/catalog/.secrets
```

##### 14.3 Obtain hostname and enable google, facebook credentials

We will need the hostname to ease authentication with google and facebook. The host name can be obtained from the IP address using `nslookup`:

```shell
nslookup 52.88.73.214
Server:         192.168.1.1

Non-authoritative answer:
214.73.88.52.in-addr.arpa       name = ec2-52-88-73-214.us-west-2.compute.amazonaws.com.
```

Here, the host name is `ec2-52-88-73-214.us-west-2.compute.amazonaws.com`. This is the address that should be used to access the application.

Next, enable the URL in google and facebook's developer settings. We don't go into detail here because it is assumed this part has been covered in project 3.

* Go to [Google Developer Console](https://console.developers.google.com/project) and set javascript origins and redirect URLS using the hostname.
* Go to [facebook developer apps](https://developers.facebook.com/apps), select this app and set the URL in both basic and advanced settings.

##### 14.3 Create virtual host configuration

An example virtual host apache configuration file `catalog.conf` can be found in this repository. This configuration runs the application from the `virtualenv` as user `catalog`.
Check the file, modify is necessary, and move it to `/etc/apache2/sites-available/catalog.conf`. The final result shoudl look something like

```xml
<VirtualHost *:80>

    ServerName 52.88.73.214
    ServerAdmin admin@52.88.73.214
    ServerAlias ec2-52-88-73-214.us-west-2.compute.amazonaws.com
    DocumentRoot /var/www/catalog

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    WSGIScriptAlias / /var/www/catalog/bin/itemcatalog.wsgi
    WSGIDaemonProcess itemcatalog user=catalog group=catalog threads=5
    <Directory /var/www/catalog/bin/>
      WSGIScriptReloading On
      Order allow,deny
      Allow from all
    </Directory>

</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Enable the new site, disable the default site, and reload `apache2`.

```shell
sudo a2ensite catalog
sudo a2dissite 000-default
sudo service apache2 reload
```

##### 14.4 Modify the WSGI application file

A WSGI script `/var/www/catalog/bin/itemcatalog.wsgi` has been installed into the application virtualenv. Check that the script and secret paths are correct, and set the postgres user `catalog`'s password (see step 12.):

```python
...
VENV = '/var/www/catalog' # virtualenv location
...
os.environ['SECRETS_PATH'] = '%s/.secrets' % VENV # secrets location
...
# Replace <CATALOG_PWD> for the real password of the POSTGRES user "catalog"
application.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://catalog:<CATALOG_PWD>@localhost/itemcatalog'
```

Re-start `apache2`.

```shell
sudo service apache2 restart
```

#### 15 Install monitoring application

We use [glances](http://glances.readthedocs.org/en/latest/glances-doc.html#introduction)
because it is easy to install and is usable in a plain terminal.

````shell
sudo apt-get install build-essential lm-sensors
sudo pip install Glances
sudo pip install PySensors
```

To start type `glances` in the terminal commandline.


Source: [AskUbuntu](http://askubuntu.com/questions/293426/system-monitoring-tools-for-ubuntu).
