# WebSvrDemo
Web server for Udacity full stack nanodegree project 5

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
    
    
