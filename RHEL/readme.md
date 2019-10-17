# CentOS 7
These instructions are written based on the following guides:  
[How to Install MySQL 5.7 on CentOS/RHEL 7/6, Fedora 27/26/25](https://tecadmin.net/install-mysql-5-7-centos-rhel/)  
[How to Install Latest MySQL 5.7.21 on RHEL/CentOS 7](https://dinfratechsource.com/2018/11/10/how-to-install-latest-mysql-5-7-21-on-rhel-centos-7/)  
[How To Install Nginx on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-centos-7)  

# Install required software
```sh
yum install epel-release
yum localinstall https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
yum update
yum install java-1.8.0-openjdk-devel
yum install mysql-community-server
yum install wget
```

# MySQL

## Get temporary MySQL Root password
```sh
service mysqld start
grep 'A temporary password' /var/log/mysqld.log |tail -1
```

## Configure/secure installation
```sh
/usr/bin/mysql_secure_installation
```

## Create database and user for XWiki

```sh
mysql -u root -p -e "create database xwiki default character set utf8 collate utf8_bin"
mysql -u root -p -e "grant all privileges on *.* to xwiki@localhost identified by 'SOMETHING74f3H3r3?'"
```
## Fine-tune for XWiki

We will need to allow larger packets for situations where we upload large documents.
First, run: `vi /etc/my.cnf`
and add the following line:

```
max_allowed_packet = 512M
```

# Tomcat 9

## Create user for Tomcat

```sh
groupadd tomcat
useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat
```

## Create the necessary folders

```sh
mkdir /opt/xwiki
mkdir /opt/tomcat/9.0.26
```

## Download & Unpack/Symlink Tomcat 

```sh
wget http://apache.mirrors.spacedump.net/tomcat/tomcat-9/v9.0.26/bin/apache-tomcat-9.0.26.tar.gz 
tar xzvf apache-tomcat-9.0.26.tar.gz -C /opt/tomcat/9.0.26 --strip-components=1
ln -s /opt/tomcat/9.0.26 /opt/tomcat/latest
```

## Configure

### Tomcat environment
Now we need to edit the file where we'll set our environment-settings:  
`vi /opt/tomcat/latest/bin/setenv.sh`

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

### Configure Tomcat to use UTF-8 encoding

You'll need to edit `conf/server` and ensure that the connector for port 8080 has this option: `URIEncoding="UTF-8"`  

Example on how it could look:  
```xml
<Connector port="8080" maxHttpHeaderSize="8192"
    maxThreads="150" minSpareThreads="25" maxSpareThreads="75"
    enableLookups="false" redirectPort="8443" acceptCount="100"
    connectionTimeout="20000" disableUploadTimeout="true"
    URIEncoding="UTF-8"/>
```

### Set correct permissions

```sh
chown -RH tomcat: /opt/tomcat/latest/ /opt/xwiki
chmod +x /opt/tomcat/latest/bin/*.sh
```

## Set up as a systemd service

The command `alternatives --config java` will give us the path to our Java-installation.
Remove the ending `/jre/bin/java/` and use that as the variable `JAVA_HOME`

In CentOS 7, this is the path I got for java-1.8.0-openjdk-devel: `/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b10-1.el7_7.x86_64`

`vi /etc/systemd/system/tomcat.service`

```sh
[Unit]
Description="Apache Tomcat Web Application Container"
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b10-1.el7_7.x86_64"
Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat/latest"
Environment="CATALINA_BASE=/opt/tomcat/latest"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"
Environment="JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:///dev/urandom"
ExecStart="/opt/tomcat/latest/bin/startup.sh"
ExecStop="/opt/tomcat/latest/bin/shutdown.sh"

UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

After saving `/etc/systemd/system/tomcat.service`, run the following commands to start the service and to allow access to the service.  

```sh
systemctl daemon-reload
systemctl start tomcat
systemctl enable tomcat
systemctl status tomcat
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

### manager & host-manager

Edit `/opt/tomcat/webapps/manager/META-INF/context.xml`
and `/opt/tomcat/webapps/host-manager/META-INF/context.xml`
to allow your computer (as the admin) to access Tomcat's manager and host-manager-application:

`allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.100\.\d+" />`

The example above will give access to the host-manager and manager-applications of Tomcat from any IP that starts with `192.168.100.X`, so modify it to suit your needs.

**TODO:** Describe how to create users for **manager** & **host-manager**

### Add support for MySQL

```sh
cd /opt/tomcat/latest/lib/
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.48.tar.gz
tar xzf mysql-connector-java-5.1.48.tar.gz 
mv mysql-connector-java-5.1.48/*.jar ./
rm -rf mysql-connector-java-5.1.48/ mysql-connector-java-5.1.48.tar.gz
chown tomcat:tomcat mysql*
```
# NginX

We'll need to create a configuration file for NginX to act as a reverse proxy for our website:
`vi /etc/nginx/conf.d/tomcat.conf`

```
## Expires map based upon HTTP Response Header Content-Type
#    map $sent_http_content_type $expires
#{
#       default                 off;
#       text/html               epoch;
#       text/css                1h;
#       text/javascript         1h;
#       application/javascript  1h;
#       ~image/                 1m;
#}

## Expires map based upon request_uri (everything after hostname)
map $request_uri $expires {
    default off;
    ~*\.(ogg|ogv|svg|svgz|eot|otf|woff|mp4|ttf|rss|atom|js|jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf)(\?|$) 1h;
    ~*\.(css) 0m;
}
expires $expires;

server {
    listen       80;
    server_name  wiki.DOMAIN.TLD wiki; 
    charset utf-8;
    client_max_body_size 64M;

    # Normally root should not be accessed, however, root should not serve files that might compromise the security of your server.
    root /var/www/html;

    location /
    {
        # All "root" requests will have /xwiki appended AND redirected to wiki.kibino.local
        rewrite ^ $scheme://$server_name/xwiki$request_uri? permanent;
    }

    location ^~ /xwiki
    {
       # If path starts with /xwiki - then redirect to backend: XWiki application in Tomcat
       # Read more about proxy_pass: http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass
       proxy_pass              http://localhost:8080;
       proxy_cache             off;
       proxy_set_header        X-Real-IP $remote_addr;
       proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header        Host $http_host;
       proxy_set_header        X-Forwarded-Proto $scheme;
       expires                 $expires;
    }
}
```

Since CentOS7 uses SELinux, you will also need to allow nginx to act as a reverse proxy by running this command:  

```sh
setsebool -P httpd_can_network_connect true
```

# XWiki

## Download

```sh
wget http://nexus.xwiki.org/nexus/content/groups/public/org/xwiki/platform/xwiki-platform-distribution-war/10.11.9/xwiki-platform-distribution-war-10.11.9.war --directory-prefix=/opt/tomcat/latest/
```

## Rename and move 

```sh
mv xwiki-platform-distribution-war-10.11.9.war /opt/tomcat/latest/webapps/xwiki.war
```

## Unpack/expand the war-file

```sh
systemctl restart tomcat
```

This will cause a high load on the server while it unpacks the war into `/opt/tomcat/latest/webapps/xwiki`
As soon as you can reach the tomcat-session on [the server through http](http://your-server-name:8080), it's time to stop Tomcat

```sh
systemctl stop tomcat
```

**Then, we'll remove the .war-file so Tomcat won't try to unpack/expand it and thereby overwriting our configuration-files**

```sh
rm /opt/tomcat/latest/webapps/xwiki.war
```

#### /opt/tomcat/webapps/xwiki/WEB-INF/hibernate.cfg.xml

**FIX ME!**
 
 
#### /opt/tomcat/webapps/xwiki/WEB-INF/xwiki.properties 

These are the settings necessary to edit before we can continue with the actual installation of XWiki. 
Since we have defined `xwiki.data.dir` in `setenv.sh`, this setting can be left commented out,  
I've left this note of the setting here to show a different way of handling it in case you don't want the setting to be globally known throughout the Tomcat-server.

```
environment.permanentDirectory=/opt/xwiki/ 
```

#### /opt/tomcat/webapps/xwiki/WEB-INF/xwiki.cfg 

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
We also want to set the "correct" url so the cookies will be correct, since XWiki won't know it's behind a reverseproxy by default. This is done by adding this line to xwiki.cfg
```
xwiki.home=http://wiki.DOMAIN.TLD/
```

  
#### XWiki installation-process 
  
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

Date=$(date +"%Y.%m.%d-%H.%M.%S")
BackupFolder="/opt/backup"
MySQLBackup="$BackupFolder/mysql/xwiki_db_backup-$Date.sql"
FilesBackup="$BackupFolder/files/xwiki_files_backup-$Date.tar.gz"
Host="localhost"
User="xwiki"
Pass="xwiki"
Db="xwiki"
Options="--add-drop-database --max_allowed_packet=1G --comments --dump-date --log-error=$BackupFolder/Error_$myDate.log"

###################################
# Make a SQL-dump and compress it #
###################################

if mysqldump --host=$Host --password=$Pass --user=$User --databases $Db $Options > $MySQLBackup; then
        if tar -zcf $MySQLBackup.tar.gz $MySQLBackup ; then
               rm -rf $MySQLBackup
        else
               echo "The compression of the sql-dump was unsuccessful" >> $BackupFolder/Error_$Date.log
        fi
else
        echo "The mysqldump was unsuccessful!" >> $BackupFolder/Error_$Date.log
fi


################
# Backup files #
################

if ! tar -czf $FilesBackup /opt/xwiki /opt/tomcat/latest/webapps/ /opt/tomcat/latest/work/; then
        echo "The backup was unsuccessful!" >> $BackupFolder/Error_$Date.log;
fi


#####################
# Clean old backups #
#####################

find $BackupFolder -daystart -mtime +28 -type f -name "*.tar.gz" -print0 | xargs -0 -r rm
find $BackupFolder -daystart -mtime +7 -type f -name "*.sql" -print0 | xargs -0 -r rm
find $BackupFolder -daystart -mtime +90 -type f -name "*.log" -print0 | xargs -0 -r rm
```


Add the following line to `/etc/crontab` so our scripts runs daily at 01:00 (AM)

```
0 1 * * *   root    /opt/backup/backup.sh > /dev/null 2>&1
```

# Firewall
If you've followed all steps until now, you already should have a firewall configured and up and running.
You might choose to restrict direct access to the tomcat-session (on port 8080) but that's up to you.