# XWiki-MySQL
Guide to setup XWiki using MySQL as database, Tomcat as servlet-container, OpenJDK-8 and nginx as a reverse proxy


# INTRO
The first goal of this project is to create a guide for how to setup XWiki on Ubuntu 18.04 using Tomcat, OpenJDK, MySQL and with nginx as a reverse-proxy in a easy to maintain manner.
The second goal is to add how to do it on FreeBSD/FreeNAS.
And the third is to try and make a script that can do all of this automatically, as long as there is a "recipe" for the chosen OS.


# Ubuntu 18.04

# Configure date & time 

` sudo dpkg-reconfigure tzdata`

# Install Java Runtime Environment 

As of 2018-11-22, XWiki doesn't support Java 9+ so we need to install a Java 8 Runtime. 
For licensing-reasons, the instructions below describe how to install the OpenJDK.

```sh
 sudo add-apt-repository universe 
 sudo apt-get update 
 sudo apt-get install openjdk-8-jre-headless
```


# Install XWIKI from repositories 

[Installation using Debian (.DEB) packages](https://www.xwiki.org/xwiki/bin/view/Documentation/AdminGuide/Installation/InstallationViaAPT/)  
The description in the link above doesn't work in 18.04.1 LTS, the packages aren't compatible yet (2018-11-22) for Bionic Beaver. 
As such, installation must be done manually. 
**Even if the repository would work, the part about nginx still has to be done manually**

# Manual installation 

## MySQL server

 
```sh
 sudo apt-get install mysql-server-5.7 
 sudo mysql -u root -e "create database xwiki default character set utf8 collate utf8_bin" 
 sudo mysql -u root -e "grant all privileges on *.* to xwiki@localhost identified by 'xwiki'" 
 sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Edit `mysqld.cnf` so the line with max_allowed_packet looks like this: 

`max_allowed_packet      = 512M `

## Nginx 

```sh
 sudo apt-get install nginx 
 sudo ufw allow http 
 sudo ufw allow https
```

Now we need to configure nginx to work as a reverse-proxy so any users trying to access the server on port 80 (default HTTP) will behind the curtains reach our Tomcat/XWiki on port 8080 

```
server { 
    listen       80; 
    server_name  wiki.server.local wiki; 

    # Normally root should not be accessed, however, root should not serve files that might compromise the security of your server. 

    root /var/www/html; 

    location / { 
        # All "root" requests will have /xwiki appended AND redirected to mydomain.com 
        rewrite ^ $scheme://$server_name/xwiki$request_uri? permanent; 
    } 

    location ^~ /xwiki { 
       # If path starts with /xwiki - then redirect to backend: XWiki application in Tomcat 
       # Read more about proxy_pass: http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass 

       proxy_pass http://localhost:8080/xwiki; 
       proxy_set_header        X-Real-IP $remote_addr; 
       proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for; 
       proxy_set_header        Host $http_host; 
       proxy_set_header        X-Forwarded-Proto $scheme; 
       expires                           2h; 
    } 
}
```

## Tomcat9 

https://linuxize.com/post/how-to-install-tomcat-9-on-ubuntu-18-04/  
https://www.linuxtechi.com/install-apache-tomcat-9-ubuntu-18-04-16-04-server/  
https://www.xwiki.org/xwiki/bin/view/Documentation/AdminGuide/Installation/InstallationWAR/  
https://www.xwiki.org/xwiki/bin/view/Documentation/AdminGuide/Installation/InstallationWAR/InstallationTomcat/  
https://www.xwiki.org/xwiki/bin/view/Documentation/AdminGuide/Installation/InstallationWAR/InstallationMySQL/  

 
### 1: Install Tomcat9 


Run the following commands: 

```sh
 sudo groupadd tomcat
 sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat 
 sudo mkdir /opt/tomcat 
 sudo mkdir /opt/xwiki 
 sudo tar xzvf apache-tomcat-9.0.13.tar.gz -C /opt/tomcat --strip-components=1 
 sudo chgrp -R tomcat /opt/tomcat/ /opt/xwiki/ 
 sudo chmod -R g+r /opt/tomcat/conf 
 sudo chmod g+x /opt/tomcat/conf 
 sudo chown -R tomcat /opt/tomcat/webapps/ /opt/tomcat/work/ /opt/tomcat/temp/ /opt/tomcat/logs/ /opt/xwiki/
```
 

We need to create our `/opt/tomcat/conf/setenv.sh` that will contain our settings for Tomcat, for this we will use nano: 

` sudo nano /opt/tomcat/conf/setenv.sh`

```sh
#! /bin/bash 

# Better garbage-collection 
export CATALINA_OPTS="$CATALINA_OPTS -XX:+UseParallelGC" 

# Instead of 1/6th or up to 192MB of physical memory for Minimum HeapSize 
export CATALINA_OPTS="$CATALINA_OPTS -Xms512M" 

# Instead of 1/4th of physical memory for Maximum HeapSize 
export CATALINA_OPTS="$CATALINA_OPTS -Xmx1536M" 

# Start the jvm with a hint that it's a server 
export CATALINA_OPTS="$CATALINA_OPTS -server" 

# Headless mode 
export JAVA_OPTS="${JAVA_OPTS} -Djava.awt.headless=true" 

# Allow \ and / in page-name 
export CATALINA_OPTS="$CATALINA_OPTS -Dorg.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH=true" 
export CATALINA_OPTS="$CATALINA_OPTS -Dorg.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH=true" 

# Set permanent directory 
export CATALINA_OPTS="$CATALINA_OPTS -Dxwiki.data.dir=/opt/xwiki/"
```

 
The command `update-java-alternatives -l` will give us the path to our JAVA-installation, use this and ensure that `JAVA_HOME` points to that path. 

```sh
 update-java-alternatives -l 
 sudo nano /etc/systemd/system/tomcat.service
```

Remember to verify JAVA_HOME path before saving the contents below! 

```
[Unit] 
Description=Apache Tomcat Web Application Container 
After=network.target 

[Service] 
Type=forking 

Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid 
Environment=CATALINA_HOME=/opt/tomcat 
Environment=CATALINA_BASE=/opt/tomcat 
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC' 
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom' 
ExecStart=/opt/tomcat/bin/startup.sh 
ExecStop=/opt/tomcat/bin/shutdown.sh 

User=tomcat 
Group=tomcat 
UMask=0007 
RestartSec=10 
Restart=always 

[Install] 
WantedBy=multi-user.target
```


After saving the file above, run the following commands to start the service and to allow access to the service. 

```sh
 sudo systemctl daemon-reload 
 sudo systemctl start tomcat 
 sudo systemctl enable tomcat 
 sudo systemctl status tomcat 
 sudo ufw allow 8080/tcp
```
 
### 2 Configure Tomcat 

Edit `/opt/tomcat/webapps/manager/META-INF/context.xml`  
and `/opt/tomcat/webapps/host-manager/META-INF/context.xml`  
to allow your computer (as the admin) to access Tomcat's manager and host-manager-application: 

`allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.100\.\d+" /> `

The example above will give access to the host-manager and manager-applications of Tomcat from any IP that starts with **192.168.100.**, so modify it to suit your needs.


### 3 Install MySQL Connector 

We need to aquire a MySQL Connector so Java/Tomcat/XWiki can access the MySQL-database.  
As of 2018-11-22, __MySQL 8.0 is NOT compatible with any version of XWiki__ so we will be using 5.1.47.  
Since we will need to traverse the folders that we don't have permissions to, we will need to switch to a superuser.  
Either use `su` or `sudo -i` to temporarily switch to a user with superuser-permissions.  
When done, type `exit` to return to your regular "non-privileged" user.
  
```sh
 sudo -i
 cd /opt/tomcat/lib/
 wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz
 tar xzf mysql-connector-java-5.1.47.tar.gz 
 mv mysql-connector-java-5.1.47/*.jar /opt/tomcat/lib/
 exit
```

## XWiki 
 

Download the .WAR into the folder `/opt/tomcat/webapps`  
` sudo wget http://nexus.xwiki.org/nexus/content/groups/public/org/xwiki/platform/xwiki-platform-distribution-war/9.11.8/xwiki-platform-distribution-war-9.11.8.war --directory-prefix=/opt/tomcat/ `

Then we will rename it and put it in the webapps folder so it can be automatically expanded (unpacked)  
` sudo mv xwiki-platform-distribution-war-9.11.8.war /opt/tomcat/webapps/xwiki.war `

After this, we'll need to restart the Tomcat-service so it will actually expand our .war-file  
` sudo systemctl restart tomcat `

This will incur a heavy load on the server for a short while (should be less then 2 minutes) as it unpacks the package into a folder of similiar name as the file.  
If you've followed my instructions so far, it should end up being:
```sh
/opt/tomcat/webapps/xwiki/
```
 
When this is done and you can visit any of the Tomcat pages ( http://SERVER:8080/manager/ http://SERVER:8080/host-manager/ ) it should have finished deploying/unpacking our .war-file, and thus it's time to remove the .war-file 
```sh
sudo rm -rf /opt/tomcat/webapps/xwiki.war
```

At this point, it's time to start configuring XWiki itself as all requirements should be in place.
 

### /opt/tomcat/webapps/xwiki/WEB-INF/xwiki.properties 

These are the settings necessary to edit before we can continue with the actual installation of XWiki. 
Since we have defined `xwiki.data.dir` in `setenv.sh`, this setting can be left commented out,  
I've left this note of the setting here to show a different way of handling it in case you don't want the setting to be globally known throughout the Tomcat-server.

```
environment.permanentDirectory=/opt/xwiki/ 
```

### /opt/tomcat/webapps/xwiki/WEB-INF/xwiki.cfg 

We need to edit these two lines so we aren't using the default keys (out of security-reasons).  
```
xwiki.authentication.validationKey=totototototototototototototototo 
xwiki.authentication.encryptionKey=titititititititititititititititi
```
  
We also want to modify how attachments are stored, in later versions of XWiki (10.2 and later), the default is to store attachments as files directly on the drive. 
```
#-# [Since 9.0RC1] The default document content recycle bin storage. Default is hibernate. 
#-# This property is only taken into account when deleting a document and has no effect on already deleted documents. 
xwiki.store.recyclebin.content.hint=file 

#-# The attachment storage. [Since 3.4M1] default is hibernate. 
xwiki.store.attachment.hint=file 

#-# The attachment versioning storage. Use 'void' to disable attachment versioning. [Since 3.4M1] default is hibernate. 
xwiki.store.attachment.versioning.hint=file 

#-# [Since 9.9RC1] The default attachment content recycle bin storage. Default is hibernate. 
#-# This property is only taken into account when deleting an attachment and has no effect on already deleted documents. 
xwiki.store.attachment.recyclebin.content.hint=file
```
  
### XWiki installation-process 
  
1. Begin by opening [http://SERVER](http://SERVER) (if nginx isn't working, you should be able to reach it by using [http://SERVER:8080/xwiki/](http://SERVER:8080/xwiki/) instead)  
1. Create your XWiki user
1. Select XWiki Standard Flavor for installation and install it 
  
  
### XWiki database optimization 
  
After-install tuneup of MySQL database 
```sh
sudo mysql -u root -e "create index xwl_value on xwikilargestrings (xwl_value(50)); create index xwd_parent on xwikidoc (xwd_parent(50)); create index xwd_class_xml on xwikidoc (xwd_class_xml(20)); create index ase_page_date on  activitystream_events (ase_page, ase_date); create index xda_docid1 on xwikiattrecyclebin (xda_docid); create index ase_param1 on activitystream_events (ase_param1(200)); create index ase_param2 on activitystream_events (ase_param2(200)); create index ase_param3 on activitystream_events (ase_param3(200)); create index ase_param4 on activitystream_events (ase_param4(200)); create index ase_param5 on activitystream_events (ase_param5(200));" xwiki
```
 
# Backup-management 
  
[Backup/Restore](https://www.xwiki.org/xwiki/bin/view/Documentation/AdminGuide/Backup)  
Configure the script below to run on a daily basis through a cron-job 
  
```sh
#!/bin/bash 
############################### 
# global variables for script # 
############################### 

myDate=$(date +"%Y.%m.%d-%H.%M.%S") 
myBackupFolder="/opt/backups/mysql" 
myBackupFile="$myBackupFolder/xwiki_db_backup-$myDate.sql" 
myHost="localhost" 
myUser="xwiki" 
myPass="xwiki" 
myDb="xwiki" 
myOptions="--add-drop-database --max_allowed_packet=1G --comments --dump-date --log-error=$myBackupFolder/$myDate.log" 

################################### 
# Make a SQL-dump and compress it # 
################################### 

if mysqldump --host=$myHost --password=$myPass --user=$myUser --databases $myDb $myOptions > $myBackupFile; then 
        if tar -zcf $myBackupFile.tar.gz $myBackupFile ; then 
               rm -rf $myBackupFile 
        else 
               echo "The compression of the sql-dump was unsuccessful" >> $myBackupFolder/$myDate.log 
        fi 
else 
        echo "The mysqldump was unsuccessful!" >> $myBackupFolder/$myDate.log 
fi 
 
##################### 
# Clean old backups # 
##################### 
 
find $myBackupFolder -daystart -mtime +28 -type f -name "*.tar.gz" -print0 | xargs -0 -r rm 
find $myBackupFolder -daystart -mtime +7 -type f -name "*.sql" -print0 | xargs -0 -r rm 
find $myBackupFolder -daystart -mtime +90 -type f -name "*.log" -print0 | xargs -0 -r rm
```


Add the following line to `/etc/crontab` so our scripts runs daily at 01:00 (AM)

```
0 1 * * *   root    /opt/backup/backup.sh > /dev/null 2>&1
```
  
# Firewall 
  
Now that things work, it's time to start securing it. By now, all that's left to allow should be SSH for remote control:  
```sh
 sudo ufw allow ssh 
 sudo ufw status
```
  
This should return with `Status: inactive`, so we now need to activate the firewall by running: 
` sudo ufw enable `
  
If you run the status-command again, it should look like this: 
```sh
root@xwiki-server:/# ufw status 
Status: active 
 

To               Action      From 

--               ------      ---- 

8080/tcp         ALLOW       Anywhere 
22/tcp           ALLOW       Anywhere 
80/tcp           ALLOW       Anywhere 
443/tcp          ALLOW       Anywhere 
8080 (v6)        ALLOW       Anywhere (v6) 
22/tcp (v6)      ALLOW       Anywhere (v6) 
80/tcp (v6)      ALLOW       Anywhere (v6) 
443/tcp (v6)     ALLOW       Anywhere (v6)
```
  
There really isn't any good reason to leave port 8080/tcp entirely open, even if you don't expose the port to the internet.
A better solution would be to only allow access from the IP/subnet where the admins reside and localhost (so nginx can access it), but I'll leave that up to you to decide for yourself.

Here are two links that can tell you more about ufw
[How To Set Up a Firewall with UFW on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04)
[UFW - Ubuntu Community](https://help.ubuntu.com/community/UFW)
  
----
  
# FreeBSD 11.2 / FreeNAS 11.2
## FreeNAS and Java
  
For Java to work in our Jail, we'll need to add a couple of Pre-Init scripts in FreeNAS since we can't run the two commands the installation of openjdk8 will ask us to do, nor will it work editing the jails fstab.
In FreeBSD, you should just follow the instructions below (shown after the installation of openjdk8)
```
======================================================================

This OpenJDK implementation requires fdescfs(5) mounted on /dev/fd and
procfs(5) mounted on /proc.

If you have not done it yet, please do the following:

        mount -t fdescfs fdesc /dev/fd
        mount -t procfs proc /proc

To make it permanent, you need the following lines in /etc/fstab:

        fdesc   /dev/fd         fdescfs         rw      0       0
        proc    /proc           procfs          rw      0       0

======================================================================
```
  
### Tasks - Init/Shutdown scripts
#### fdescfs
  
|||
|:-|:-|
|**Type**|command|
|**Command**|`mount -t fdescfs null /mnt/iocage/jails/XWiki/root/dev/fd`|
|**When**|preinit|
|**Enabled**|yes
  
#### procfs
  
|||
|:-|:-|
|**Type**|command|
|**Command**|`mount -t procfs proc null /mnt/iocage/jails/XWiki/root/proc`|
|**When**|preinit|
|**Enabled**|yes
  
**Note** For these scripts to work, we will need to reboot the server before continuing!
  
## FreeBSD/FreeNAS install of packages
```sh
 pkg update
 pkg install openjdk8
 pkg install tomcat9
 pkg install nginx
```
  
## MySQL

```sh
pkg install mysql-connector-java-5.1.47 mysql57-server
```  

After installing our mysql57-server, we need to enable the service and start it.
```sh
printf '\n# Enable MySQL server\nmysql_enable="YES"\n' >> /etc/rc.conf
service mysql-server start
```

When it starts for the first time, it will create a file containing our MySQL-root users password, so we now need to take a look at `/root/.mysql_secret` and write down the password for later use.
```sh
cat /root/.mysql_secret
```

Just to make sure our password is correct, let's give it a try!
```sh
mysql --user=root --password="YOURPASSWORDHERE" --host=localhost
```

## PostgreSQL

**STUB-article** To be done at a later time!
  
## Tomcat

[Install Tomcat 8 In FreeBSD 10/10.1](https://www.unixmen.com/install-tomcat-7-freebsd-9-3/)  

Before we do anything, we will create our `setenv.sh` containing our settings to make Tomcat work better with XWiki.

```sh
vi /usr/local/apache-tomcat-9/bin/setenv.sh
```

In this file, add the following lines:

```
#! /bin/bash 

# Better garbage-collection 
export CATALINA_OPTS="$CATALINA_OPTS -XX:+UseParallelGC" 

# Instead of 1/6th or up to 192MB of physical memory for Minimum HeapSize 
export CATALINA_OPTS="$CATALINA_OPTS -Xms512M" 

# Instead of 1/4th of physical memory for Maximum HeapSize 
export CATALINA_OPTS="$CATALINA_OPTS -Xmx1536M" 

# Start the jvm with a hint that it's a server 
export CATALINA_OPTS="$CATALINA_OPTS -server" 

# Headless mode 
export JAVA_OPTS="${JAVA_OPTS} -Djava.awt.headless=true" 

# Allow \ and / in page-name 
export CATALINA_OPTS="$CATALINA_OPTS -Dorg.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH=true" 
export CATALINA_OPTS="$CATALINA_OPTS -Dorg.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH=true" 

# Set permanent directory 
export CATALINA_OPTS="$CATALINA_OPTS -Dxwiki.data.dir=/usr/local/opt/xwiki/"
```

Then we'll change it's permissions:
```sh
chmod 755 /usr/local/apache-tomcat-9.0/bin/setenv.sh
```

And now we should take the time to create our permanent storage-folder for XWiki that we've defined in the startup-script for Tomcat.

```sh
mkdir -p /usr/local/opt/xwiki/
chown -R root:wheel /usr/local/opt/xwiki/
```

Before we start Tomcat for the first time, we'll configure `host-manager` and `manager` so we can login and manage our Tomcat-servlet using it's webGUI.
To do so, we need to edit these two files:
```sh 
/usr/local/apache-tomcat-9.0/webapps/manager/META-INF/context.xml
/usr/local/apache-tomcat-9.0/webapps/host-manager/META-INF/context.xml
```

Open these two files and edit the line `allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />` so your IP-address is allowed to connect.
If you want to allow all IPs starting with 192.168.100. for example, it could look like this:
```sh
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.100\.\d+" />
```

After this, we want to edit `tomcat-users.xml` so we can login to the two different webGUIs, modify this line as you see fit, but add it before `</tomcat-users>`

```sh
<user username="tomcat-admin" password="MYSECRETPASSWORD" roles="admin-gui,manager-gui,manager-status"/>
```

Verify that the config-file looks correct, and then you can delete the backup-file
```sh
rm -rf /usr/local/apache-tomcat-9.0/conf/tomcat-users.xml.bak
```

Now we're ready to startup Tomcat!

```sh
cd /usr/local/apache-tomcat-9.0/bin
./startup.sh
cd /usr/local/apache-tomcat-9.0/webapps/
curl http://nexus.xwiki.org/nexus/content/groups/public/org/xwiki/platform/xwiki-platform-distribution-war/9.11.8/xwiki-platform-distribution-war-9.11.8.war --output xwiki.war
```

After this, we will need to wait for a moment until Tomcat has autoexpanded/deployed (unpacked) the war.  
The unpacked war will exist in `/usr/local/apache-tomcat-9.0/webapps/xwiki/` when it's done.  
So before we continue, we should change the ownership of the folder so it follows the other folders from the install of tomcat-9.0  

```sh
chown -R root:wheel /usr/local/apache-tomcat-9.0/webapps/xwiki
```