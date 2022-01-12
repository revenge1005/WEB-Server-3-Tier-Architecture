## 1. WEB Server - Apache install and configuration

```
cat <<EOF >> /etc/hosts

192.168.219.100 web
192.168.219.111 was01
192.168.219.112 was02
192.168.219.200 db
EOF
```

```
apt -y install apache2 libapache2-mod-jk
```

```
vim /etc/libapache2-mod-jk/workers.properties

worker.list=tomcat1,tomcat2,loadbalance

worker.tomcat1.type=ajp13
worker.tomcat1.host=192.168.219.111
worker.tomcat1.port=8009
worker.tomcat1.lbfactor=1

worker.tomcat2.type=ajp13
worker.tomcat2.host=192.168.219.112
worker.tomcat2.port=8009
worker.tomcat2.lbfactor=1

worker.loadbalance.type=lb
worker.loadbalance.balance_workers=tomcat1,tomcat2
```

```
vim /etc/apache2/mods-available/httpd-jk.conf

<IfModule jk_module>

    JkWorkersFile /etc/libapache2-mod-jk/workers.properties

    JkLogFile /var/log/apache2/mod_jk.log

    JkLogLevel info

    JkShmFile /var/log/apache2/jk-runtime-status

#    <Location /jk-status>
#        JkMount jk-status
#        Require ip 127.0.0.1
#    </Location>
#    <Location /jk-manager>
#        JkMount jk-manager
#        Require ip 127.0.0.1
#    </Location>

</IfModule>
```

```
vim /etc/apache2/sites-available/000-default.conf

<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        JkMount /*.jsp loadbalance

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

<br>

----

## 2. WAS01/02 Server - JAVA, Tomcat install

```
cat <<EOF >> /etc/hosts

192.168.219.100 web
192.168.219.111 was01
192.168.219.112 was02
192.168.219.200 db
EOF
```

#### (1) JAVA install - https://jdk.java.net/

```
apt -y install wget 

wget https://download.java.net/openjdk/jdk8u41/ri/openjdk-8u41-b04-linux-x64-14_jan_2020.tar.gz

tar xvzf openjdk-8u41-b04-linux-x64-14_jan_2020.tar.gz

mkdir /usr/local/jvm

mv java-se-8u41-ri /usr/local/jvm/
```

```
cat <<EOF >> /etc/profile

JAVA_HOME=/usr/local/jvm/java-se-8u41-ri/
export JAVA_HOME
PATH=\$PATH:\$JAVA_HOME/bin
export PATH
EOF

source /etc/profile
```

```
java -version
```

#### (2) Tomcat install - https://tomcat.apache.org/

```
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.56/bin/apache-tomcat-9.0.56.tar.gz

tar xvzf apache-tomcat-9.0.56.tar.gz

mv apache-tomcat-9.0.56 /usr/local/tomcat9
```

```
cat <<EOF > /usr/lib/systemd/system/tomcat.service
[Unit]
Description=tomcat9
After=network.target syslog.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/local/jvm/java-se-8u41-ri/
User=root
Group=root

ExecStart=/usr/local/tomcat9/bin/startup.sh
ExecStop=/usr/local/tomcat9/bin/shutdown.sh

UMask=0007
RestartSec=10
Restart=always

SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
EOF
```

```
systemctl enable tomcat

systemctl start tomcat
```

```
netstat -anlp | grep 8080
```

<br>

----
 
## 3. WAS01/02 Server - tomcat configure 

```
vim /usr/local/tomcat9/conf/server.xml

# line 115 
<Connector protocol="AJP/1.3"
           address="0.0.0.0"
           secretRequired="false"
           port="8009"
           redirectPort="8443" />

# line 131 "jvmRoute=tomcat1" add
<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat1">

# line 150 - tomcat home directory 
<Host name="localhost"  appBase="webapps"
    unpackWARs="true" autoDeploy="true">

    <Context path="" docBase="source" reloadable="true" />

```

```
mkdir /usr/local/tomcat9/webapps/source
cat << EOF > /usr/local/tomcat9/webapps/source/index.jsp
was01
EOF
```

```
systemctl stop tomcat

systemctl start tomcat

netstat -anlp | grep 8080

netstat -anlp | grep 8009
```

<br>

----
 
## 4. WEB Server - Service Check

```
a2dismod jk

mv /etc/apache2/mods-available/httpd-jk.conf /etc/apache2/mods-available/jk.conf

cd /etc/libapache2-mod-jk/

ln -f -s ../apache2/mods-available/jk.conf httpd-jk.conf

a2enmod jk

apache2ctl configtest
```

```
systemctl restart apache2

cat /var/log/apache2/mod_jk.log
```
![Apache Tomcat 01](https://user-images.githubusercontent.com/42735894/149142966-9838d3e1-78d7-4301-9af9-fe6188296730.PNG)
![Apache Tomcat 02](https://user-images.githubusercontent.com/42735894/149142987-7e18e179-a5e1-44db-bb57-5d9954b1e34d.PNG)