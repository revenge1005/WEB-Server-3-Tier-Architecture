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
add-apt-repository ppa:ondrej/apache2 -y

apt update

apt -y upgrade

apt -y install apache2 libapache2-mod-jk
```

```
vim /etc/libapache2-mod-jk/workers.properties

worker.list=tomcat1

worker.tomcat1.type=ajp13
worker.tomcat1.host=192.168.219.111
worker.tomcat1.port=8009
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

        JkMount /*.jsp tomcat1

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

<br>

----

## 2. WAS Server - JAVA, Tomcat install

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
 
## 3. WAS Server - tomcat configure 

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
Hello World
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
![U_Apache_tomcat 01](https://user-images.githubusercontent.com/42735894/149080454-42dd2c75-e7c3-4fe7-a4fc-ad63ac6aaa15.PNG)
![U_Apache_tomcat 02](https://user-images.githubusercontent.com/42735894/149080459-996fc386-d18d-4334-b7b3-df7a0d339516.PNG)


<br>

----

## 5. DB Server - MySQl (MariaDB) Install

```
apt -y update

apt -y install mysql-server
```

```
systemctl restart mysql
systemctl enable mysql
systemctl status mysql
```

```
mysql_secure_installation
```

```
mysql -u root -p

create database test;

exit;
```

<br>

----
 
## 6. WAS Server - DB Connectation Check

#### - https://downloads.mysql.com/archives/c-j/

```
wget https://downloads.mysql.com/archives/get/p/3/file/mysql-connector-java-8.0.26.tar.gz

tar xvzf mysql-connector-java-8.0.26.tar.gz

cd mysql-connector-java-8.0.26

cp mysql-connector-java-8.0.26.jar /usr/local/tomcat9/lib/
```

```
systemctl stop tomcat

systemctl start tomcat

netstat -anlp | grep 8080

netstat -anlp | grep 8009
```

```
vim /usr/local/tomcat9/webapps/source/db.jsp

<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" import="java.sql.*"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
 <html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>DB Connection Test</title>
</head>
<body>
        <%
                String DB_URL = "jdbc:mysql://192.168.219.200:3306/test";
                String DB_USER = "root";
                String DB_PASSWORD= "1234";

                Connection conn;
                Statement stmt;

                try {
                        Class.forName("com.mysql.jdbc.Driver");
                        conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
                        stmt = conn.createStatement();
                        conn.close();
                        out.println("MySql jdbc test: connect ok!!");
                } catch(Exception e) {
                        out.println(e.getMessage());
                }
        %>
</body>
</html>
```