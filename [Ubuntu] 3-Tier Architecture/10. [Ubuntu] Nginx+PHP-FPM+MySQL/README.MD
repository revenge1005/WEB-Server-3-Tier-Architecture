## 1. WEB Server - Nginx install and configuration

```
cat <<EOF >> /etc/hosts

192.168.219.100 web
192.168.219.111 was01
192.168.219.112 was02
192.168.219.200 db
EOF
```

```
apt update

apt -y install nginx
```

```
vim /etc/nginx/sites-available/default

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;

    index index.html index.htm index.nginx-debian.html;

    server_name _;

    location / {
            
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        root           /usr/share/nginx/html;
        fastcgi_pass   was01:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

```
systemctl restart nginx
systemctl enable nginx
systemctl status nginx
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
apt-get -y install software-properties-common 

add-apt-repository ppa:ondrej/php
```

```
apt-get install php7.3 -y
```

```
apt-cache search php7.3
```

```
apt-get install php7.3-bcmath php7.3-bz2 php7.3-cgi php7.3-cli php7.3-common php7.3-curl php7.3-dba php7.3-dev php7.3-enchant php7.3-fpm php7.3-gd php7.3-gmp php7.3-imap php7.3-interbase php7.3-intl php7.3-json php7.3-ldap php7.3-mbstring php7.3-mysql php7.3-mysqlnd php7.3-odbc php7.3-opcache php7.3-pgsql php7.3-phpdbg php7.3-pspell php7.3-readline  php7.3-recode php7.3-snmp php7.3-soap php7.3-sqlite3 php7.3-sybase php7.3-tidy php7.3-xml php7.3-xmlrpc php7.3-zip  php7.3-xsl  -y
```

```
vi /etc/php/7.3/fpm/php.ini

date.timezone = Asia/Seoul
```

```
vi /etc/php/7.3/cli/php.ini

date.timezone = Asia/Seoul
```

```
vim /etc/php/7.3/fpm/pool.d/www.conf

listen = 9000
listen.allowed_clients = 192.168.219.101
```

```
systemctl restart php7.3-fpm
systemctl enable php7.3-fpm
systemctl status php7.3-fpm
```

```
mkdir -p /usr/share/nginx/html
cat <<EOF >> /usr/share/nginx/html/info.php
<?php phpinfo(); ?>
EOF
```

<br>

----
 
## 3. DB Server - MySQl (MariaDB) Install

```
apt -y update

apt -y install mariadb-server

sed -i "s/bind-address            = 127.0.0.1/bind-address            = 0.0.0.0/g" /etc/mysql/mariadb.conf.d/50-server.cnf
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

GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'PASSWORD';

exit;
```

<br>

----
 
## 4. WAS Server - DB connection check

```
cat <<EOF >> /usr/share/nginx/html/dbtest.php
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