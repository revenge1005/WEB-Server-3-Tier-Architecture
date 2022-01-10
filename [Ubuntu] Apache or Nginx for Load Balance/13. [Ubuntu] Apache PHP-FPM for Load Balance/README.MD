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

apt -y install apache2 \
libapache2-mod-fcgid libapache2-mod-proxy-uwsgi \
libapache2-mod-xforward openssl
```

```
vim /etc/apache2/sites-available/000-default.conf

<VirtualHost *:80>

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        <Proxy "balancer://myappcluster/">
                BalancerMember "fcgi://was01:9000" route=1
                BalancerMember "fcgi://was02:9000" route=2
                ProxySet stickysession=ROUTEID
        </Proxy>

        <FilesMatch "\.php$">
                SetHandler "proxy:balancer://myappcluster/"
        </FilesMatch>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

```
a2enmod fcgid 
a2enmod proxy_fcgi
a2enmod proxy_balancer
a2enmod lbmethod_byrequests
```

```
systemctl restart apache2
systemctl enable apache2
systemctl status apache2
```

<br>

----
 
## 2. WAS01/02 Server - PHP-FPM install and configuration

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

apt update
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
vim /etc/php-fpm.d/www.conf

listen = 9000
listen.allowed_clients = 192.168.219.101
```

```
systemctl restart php7.3-fpm
systemctl enable php7.3-fpm
systemctl status php7.3-fpm
```

```
mkdir -p /var/www/html
cat <<EOF >> /var/www/html/info.php

<?php phpinfo(); ?>
EOF
```