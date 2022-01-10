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
yum -y install httpd
```

```
vim /etc/httpd/conf/httpd.conf

<VirtualHost *:80>
	ProxyRequests Off
	ProxyPreserveHost On
	ProxyErrorOverride on
	ProxyTimeout 600
	# (1) UDS 방식
	#<FilesMatch \.(php|phar)$> 
	#	SetHandler "proxy:unix:/run/php/php-fpm.sock|fcgi://localhost"
	#</FilesMatch>
	# (2) IP 방식
	<FilesMatch \.(php|phar)$>
		SetHandler "proxy:fcgi://was01:9000"
	</FilesMatch>
</VirtualHost>
```

```
systemctl restart httpd
systemctl enable httpd
systemctl status httpd
```

<br>

----
 
## 2. WAS Server - PHP-FPM install and configuration

```
cat <<EOF >> /etc/hosts

192.168.219.100 web
192.168.219.111 was01
192.168.219.112 was02
192.168.219.200 db
EOF
```

```
yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum -y install epel-release yum-utils

yum-config-manager --disable remi-php54  
yum-config-manager --enable remi-php73

yum -y install php php-cli php-fpm php-mysqlnd php-zip php-pdo php-devel php-odbc php-gd php-mcrypt php-mbstring php-curl php-xml php-pear php-bcmath php-json  php-ldap php-fileinfo
```

```
vim /etc/php-fpm.d/www.conf

listen = 9000
listen.allowed_clients = 192.168.219.100
```

```
systemctl restart php-fpm
systemctl enable php-fpm
```

```
mkdir -p /var/www/html
cat <<EOF >> /var/www/html/info.php
<?php phpinfo(); ?>
EOF
```

<br>

----
 
## 3. DB Server - MySQl (MariaDB) Install

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

exit;
```

<br>

----
 
## 4. WAS Server - DB connection check

```
cat <<EOF >> /var/www/html/dbtest.php
WAS01
<?php
    \$host = '192.168.219.200';
    \$user = 'root';
    \$pw = '1234';
    \$dbName = 'test';
    \$mysqli = new mysqli(\$host, \$user, \$pw, \$dbName);
 
    if(\$mysqli){
        echo "MySQL 접속 성공";
    }else{
        echo "MySQL 접속 실패";
    }
?>
EOF
```