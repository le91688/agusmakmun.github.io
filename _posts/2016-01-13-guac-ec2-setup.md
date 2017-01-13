---
layout: post
title:  "Guacamole on EC2"
image: ''
date:   2017-01-13 09:18:31
tags: 
- linux
- ubuntu
description: 'setting up guacamole for EC2 instance'
categories:
- Linux
comments: true
---


### INSTALL 
NOTE: the following assumes a fresh EC2 instance with ubuntu server 14.04
Install and setup VNC and a desktop environment. 
```bash
apt-get install vnc4server
apt-get install xfce
```
Configure your vnc startup script
```bash
sudo nano ~/.vnc/xtartup 
------------------------------ 
#!/bin/sh
xrdb $HOME/.Xresources
xsetroot -solid grey
startxfce4 &
------------------------------

sudo add-apt-repository ppa:guacamole/stable

apt-get install guacamole-tomcat 
```

For ssh / rdp support, install the following packages:

```bash
apt-get install libguac-client-ssh0 libguac-client-rdp0
```

you should now be able to hit http://localhost:8080/guacamole and get a login page

### SETUP

configure user-mapping.xml as follows. Please note, <authorize username/pass> can be anything you want, and is only used within guacamole app. 

vncpassword needs to match your vncserver password config, or VNC will fail to initialize. set vnc password with vncpasswd command
port needs to be what the current running vnc session is listening on

```bash
sudo nano /etc/guacamole/user-mapping.xml


    <authorize username="user1" password="pass1">
        <protocol>vnc</protocol>
        <param name="hostname">localhost</param>
        <param name="port">5901</param>
        <param name="password">vncpassword</param>
    </authorize>

    <authorize username="user2" password="pass2">
        <protocol>vnc</protocol>
        <param name="hostname">localhost</param>
        <param name="port">5901</param>
        <param name="password">vncpassword</param>
    </authorize>
```


### REVERSE PROXY SETUP
this setting redirects all incoming port 80 traffic to port 8080 #once it hits your webserver. This is useful for accessing your guacamole server from work etc.
```bash
sudo apt-get install apache2
sudo aptitude install -y libapache2-mod-proxy-html libxml2-dev 

sudo a2enmod
#enter the following
proxy proxy_ajp proxy_http rewrite deflate headers proxy_balancer proxy_connect proxy_html 
#configure the following file as seen below
sudo nano /etc/apache2/sites-available/000-default.conf
```
```xml
<VirtualHost *:80>

# /guacamole settings
ProxyPass /guacamole http://localhost:8080/guacamole
ProxyPassReverse /guacamole http://localhost:8080/guacamole
<Location /guacamole>

    Order allow,deny
    Allow from all
</Location>
</VirtualHost> 
```
 
you should now be able to hit http://localhost/guacamole and get a login page
now we just need to modify the Users


### UPGRADE TO LATEST VERSION //OPTIONAL///

stop services
```bash
service guacd stop
service tomcat7 stop 

wget -O guacamole-0.9.9.war http://downloads.sourceforge.net/project/guacamole/current/binary/guacamole-0.9.9.war
wget -O guacamole-server-0.9.9.tar.gz http://downloads.sourceforge.net/project/guacamole/current/source/guacamole-server-0.9.9.tar.gz


sudo apt-get install make libcairo2-dev libpng12-dev freerdp-x11 
libssh2-1 libvncserver-dev libfreerdp-dev libvorbis-dev libssl0.9.8 
gcc libssh-dev libpulse-dev gcc libossp-uuid-dev

tar -xzf guacamole-server-0.9.9.tar.gz

cd guacamole-server-0.9.9/

./configure --with-init-dir=/etc/init.d
```

if no errors occur, go to next step. If you get errors, youre likely missing a package
```bash
make

make install

update-rc.d guacd defaults
ldconfig 
```
replace old guacamole web app with new
```bash
rm -r /var/lib/tomcat6/webapps/guacamole
cp guacamole-0.9.9.war /var/lib/tomcat6/webapps/guacamole.war
```
start your services
```bash
service guacd start
service tomcat6 restart 
```
you should now be able to access the login screen
