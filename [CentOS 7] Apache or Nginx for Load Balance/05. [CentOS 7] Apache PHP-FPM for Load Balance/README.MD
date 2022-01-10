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
	<Proxy balancer://mycluster>
		BalancerMember "fcgi://was01:9000" route=1 
		BalancerMember "fcgi://was02:9000" route=2 
		#BalancerMember "fcgi://was01:9000" route=1 enablereuse=On max=5
		#BalancerMember "fcgi://was02:9000" route=2 enablereuse=On max=5
		# enablereuse 옵션을 통해서 WAS와 DB 간의 Connection Pool과 유사하게 Apache와 PHP-FPM 간의 미리 연결을 확보하여 Connection을 재사용 할수 있다.
		# enablereuse 옵션 사용시 사전에 검토나 고려가 필요한 부분이 있다.
		# enablereuse 옵션 미사용 때보다 php-fpm 에서의 프로세스 확보가 더 필요해 오히려 늦어지거나 저하 현상이 발생할 수 있다.
		ProxySet lbmethod=bytraffic
		# -> Load Balance 방법 : byrequests (요청별 분배), bytraffic (byte 트래픽 가중치 분배), bybussyness( 보류중 요청 분배)
		ProxySet stickysession=ROUTEID
	</Proxy>
	# (1)
	<FilesMatch \.php$>
		SetHandler proxy:balancer://mycluster
	</FilesMatch>
	# (2) 
	# ProxyPass /Websource/path/ balancer://mycluster/
	# ProxyPassReverse /Websource/path/ balancer://mycluster/
</VirtualHost>
```

```
systemctl restart httpd
systemctl enable httpd
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
listen.allowed_clients = web
```

```
systemctl restart php-fpm
systemctl enable php-fpm
systemctl status php-fpm
```

```
mkdir -p /var/www/html
cat <<EOF >> /var/www/html/info.php

<?php phpinfo(); ?>
EOF
```