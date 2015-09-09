# WebSvrDemo
Web server for Udacity full stack nanodegree project 5

## Set up

### Create a new user named `grader`

    addusr grader
    
### Give the grader the permission to sudo

Create file `/etc/sudoers.d/grader` with contents

    grader ALL=(ALL) NOPASSWD:ALL
    
### Set up ssh login for `grader`

    su - grader
    mkdir .ssh
    chmod 700 .ssh
    
Then copy `root`'s `.ssh/authorized_keys` and change permissions

    sudo cp /root/.ssh/authorized_keys .ssh/authorized_keys
    sudo chown grader:grader .ssh/authorized_keys 
    chmod 644 .ssh/authorized_keys

### Update all currently installed packages

Change user to `grader` and use `sudo apt-get`

    su - grader
    sudo apt-get update
    sudo apt-get upgrade
    
### Fix warning `sudo: unable to resolve host ip-xx-yy-zz-xyz`

Hostname is in `/etc/hostname`. Add that hostname to `/etc/hosts`:

    127.0.1.1 ip-xx-yy-zz-xyz
    
And reboot the machine

    sudo reboot

### Extra: disable root ssh access

`sudo vim /etc/ssh/sshd_config` and set

    PermitRootLogin no
    
Then restart ssh:

    sudo service ssh restart

### change SSH port to 2200

Modify relevant line in `/etc/ssh/sshd_config` to

    Port 2200
    
and restart `ssh`

    sudo service ssh restart
    
We can now log in with

    ssh -i ~/.ssh/udacity_key.rsa grader@52.88.73.214 -p 2200
    
### Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

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
    
### Install apache2, apache2-mod-wsgi and apache docs

    sudo apt-get install apache2 apache2-doc libapache2-mod-wsgi
