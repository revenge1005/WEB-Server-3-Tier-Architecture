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
cat <<EOF > /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/\$basearch/
gpgcheck=0
enabled=1
EOF
```

```
yum install -y nginx
```

```
vim /etc/nginx/conf.d/default.conf

upstream myServer{
    server was01:9000 max_fails=3 fail_timeout=10s;
    server was02:9000 max_fails=3 fail_timeout=10s;
}
server {
    listen       80;
    server_name  localhost;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
    location ~ \.php$ {
        root           /usr/share/nginx/html;
        fastcgi_pass   myServer;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

```
systemctl enable nginx
systemctl restart nginx
systemctl status nginx
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
yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum -y install epel-release yum-utils

yum-config-manager --disable remi-php54  
yum-config-manager --enable remi-php73

yum -y install php php-cli php-fpm php-mysqlnd php-zip php-pdo php-devel php-odbc php-gd php-mcrypt php-mbstring php-curl php-xml php-pear php-bcmath php-json  php-ldap php-fileinfo
```

```
vim /etc/php-fpm.d/www.conf

listen = 9000
listen.allowed_clients = 192.168.219.101
```

```
systemctl restart php-fpm
systemctl enable php-fpm
```

```
mkdir -p /usr/share/nginx/html
cat <<EOF >> /usr/share/nginx/html/info.php
<?php phpinfo(); ?>
EOF
```