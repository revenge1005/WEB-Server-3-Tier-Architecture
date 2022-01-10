## 1. WAS Server - JAVA, Tomcat install

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
yum -y install wget 

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
 
## 2. WEB Server - Apache install and configuration

```
cat <<EOF >> /etc/hosts

192.168.219.100 web
192.168.219.111 was01
192.168.219.112 was02
192.168.219.200 db
EOF
```

#### - https://tomcat.apache.org/download-connectors.cgi

```
yum -y install httpd httpd-devel gcc gcc-c++ wget 

wget https://dlcdn.apache.org/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.48-src.tar.gz

tar xvzf tomcat-connectors-1.2.48-src.tar.gz

cd tomcat-connectors-1.2.48-src/native/

./configure --with-apxs=/bin/apxs

make

cd apache-2.0

cp mod_jk.so /usr/lib64/httpd/modules/mod_jk.so

chmod 755 /usr/lib64/httpd/modules/mod_jk.so
```

```
vim /etc/httpd/conf/httpd.conf

# line 58
LoadModule jk_module /usr/lib64/httpd/modules/mod_jk.so

<IfModule jk_module>
    JkWorkersFile /etc/httpd/conf/workers.properties
    JkLogFile /var/log/httpd/mod_jk.log
    JkLogLevel info
    JkLogStampFormat "[%a %b %d %H:%M:%S %Y] "
    JkMount /*.jsp tomcat1
</IfModule>
```

```
cat <<EOF > /etc/httpd/conf/workers.properties
worker.list=tomcat1

worker.tomcat1.type=ajp13
worker.tomcat1.host=192.168.219.111
worker.tomcat1.port=8009
EOF
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
systemctl restart httpd

cat /var/log/httpd/mod_jk.log
```

![01](https://user-images.githubusercontent.com/42735894/148754837-9baa2b90-7bb4-488c-91b1-b7285180b88c.PNG)

![02](https://user-images.githubusercontent.com/42735894/148754844-5e11c276-6d94-475e-9cc0-92f9fc440434.PNG)

<br>

----
 
## 5. DB Server - MySQl (MariaDB) Install

```
yum -y install mariadb-server mariadb
```

```
systemctl restart mariadb.service                      
systemctl enable mariadb.service
systemctl status mariadb.service
```

```
mysql_secure_installation
```

```
mysql -u root -p

create database test;

GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'PASSWORD';

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

![03](https://user-images.githubusercontent.com/42735894/148754900-d6065249-5203-43a9-b1a0-45d45af3a6e2.PNG)