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

#### 4. Update all currently installed packages

Change user to `grader` and use `sudo apt-get`

    su - grader
    sudo apt-get update
    sudo apt-get upgrade
    
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
    
#### 8. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

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

#### 12. Install and configure PodtgreSQL

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

#### 12. Install other item catalog app dependencies

A few packages are needed to ease the installation and running of the item catalog
application.

Install `git`,`pip`, `virtualenv`, `supervisord`:

```shell
sudo apt-get install git
sudo apt-get install python-pip
sudo apt-get install python-virtualenv
sudo apt-get install supervisor
```

The remaining python dependencies will be installed with `pip` into a `virtualenv`.

#### 13. WSGI configuration

We will configure Apache to run an WSGI app from a `virtualenv`. The app should be
demonized such that it starts whenever the server re-starts.

Source: [Flask mod_wsgi(Apache) Configuration](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)

##### 13.1 Clone item catalog repo, initialize a virtualenv and initialize database

```shell
# clone the repo
sudo git clone https://github.com/juanchopanza/ItemCatalog.git /var/www/catalog
# initialize a virtualenv
sudo virtualenv --system-site-packages /var/www/catalog/venv
# transfer ownership to user catalog
sudo chown -R catalog:catalog ./catalog
# install requirements into virtualenv as user catalog
sudo su catalog -c "/var/www/catalog/venv/bin/pip install -r /var/www/catalog/requirements.txt"
# initialize the database
sudo su catalog -c "/var/www/catalog/venv/bin/python /var/www/catalog/init_db.py -c /var/www/catalog/config_prod.py"
```

##### 13.2 Create virtual host configuration

A template for the virtual host apache configuration itemcatalog.conf can be found in the itemcatalog repository. Modify this file and move if to `/etc/apache2/sites-available/catalog.conf`.

##### 13.3 Create WSGI application file
