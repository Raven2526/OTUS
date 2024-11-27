#InfluxDB Main
Фаза 1.
1. Обновление ПО "на борту":
	yum update -y 
2. Установка nginx:
	yum install nginx -y
3. Настройка и проверка автозапуска nginx:
	systemctl start nginx
	systemctl enable nginx
	systemctl status nginx
4. Разрешение запросов до Web-сервера
	firewall-cmd --permanent --zone=public --add-service=http
	firewall-cmd --permanent --zone=public --add-service=https
	firewall-cmd --permanent --add-port=9090/tcp --add-port=9093/tcp --add-port=9094/{tcp,udp} --add-port=9100/tcp --add-port=9104/tcp  --add-port=9115/tcp --add-port=1947/tcp --add-port=8086/tcp
	firewall-cmd --runtime-to-permanent
	firewall-cmd --reload
5. Раздача прав на web-каталог:
	chown nginx:nginx /usr/share/nginx/html -R
6. Установка MariaDB Server:
	yum install mariadb-server mariadb -y
7. Настройка и проверка автозапуска MariaDB:
	systemctl start mariadb
	systemctl enable mariadb
	systemctl status mariadb
8. Применение параметров безопасности для MariaDB:
	mysql_secure_installation
9. Установка PHP-FPM:
	yum install php php-mysqlnd php-fpm php-opcache php-gd php-xml php-mbstring -y
	mkdir /var/run/php
10. Настройка и проверка автозапуска PHP:
	systemctl start php-fpm
	systemctl enable php-fpm
	systemctl status php-fpm
11. Конфигурирование PHP-FPM:
	vi /etc/php-fpm.d/www.conf
-->
	user = nginx
	group = nginx
--
12. Рестарт PHP-FPM и nginx:
	systemctl reload php-fpm
	systemctl restart nginx php-fpm
13. Установка инструментов разработки:
	yum groupinstall "Development tools"
	yum install wget
14. Установка WordPress:
	cd /var/www/html
	wget https://wordpress.org/latest.tar.gz
	tar xvzf latest.tar.gz
	rm -v latest.tar.gz
15. Раздача прав для wordpress/
	chown -R nginx:nginx /var/www/html/wordpress
	chmod -Rf 775 ./wordpress/
16. Настройка nginx vHost:
	vi /etc/nginx/conf.d/wordpress.conf
-->
	server {
	listen 80;

	server_name yourdomain.com www.yourdomain.com;
	root /var/www/html/wordpress;
	index index.php index.html index.htm;

	location / {
	try_files $uri $uri/ /index.php?$args;
	}

	location = /favicon.ico {
	log_not_found off;
	access_log off;
	}

	location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
	expires max;
	log_not_found off;
	}

	location = /robots.txt {
	allow all;
	log_not_found off;
	access_log off;
	}

	location ~ \.php$ {
	include /etc/nginx/fastcgi_params;
	fastcgi_pass unix:/run/php-fpm/www.sock;
	fastcgi_index index.php;
	fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	}
	}
--
17. Рестарт и проверка nginx:
	systemctl restart nginx.service
	systemctl status nginx.service
18. Настройка БД WordPress:
	mysql -u root
	CREATE DATABASE wordpress;
	CREATE USER 'wordpress'@'localhost' IDENTIFIED BY 'secret';
	GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'localhost';
	FLUSH PRIVILEGES;
	quit
19. Настройка CMS WordPress:
	cd /var/www/html/wordpress
	vi wp-config.php
-->
	...config from the CMS wizzard...
	!keep password in safe
--

Фаза 2.
1. Установка и запуск InfluxDB:
	wget https://download.influxdata.com/influxdb/releases/influxdb-1.11.8.x86_64.rpm
	yum localinstall influxdb-1.11.8.x86_64.rpm
	systemctl start influxdb
2. Создание БД InfluxDB:
	influx
	create database prom;
	show databases;
3. Создание пользователя:
	influx -execute "create user admin with password 'adminpassword' with all privileges"
	influx -execute "show users"
4. Конфигурирование InfluxDB:
	vi /etc/influxdb/influxdb.conf
-->
	auth-enabled = true
--
	systemctl restart influxdb
	influx -username admin -password adminpassword
	exit
	export INFLUX_USERNAME=admin
	export INFLUX_PASSWORD=adminpassword
	influx
	exit
	curl -G http://localhost:8086/query?pretty=true -u admin:adminpassword --data-urlencode "q=show users"
5. Добавление репозитория Telegraf:
	# influxdata-archive_compat.key GPG fingerprint:
	#     9D53 9D90 D332 8DC7 D6C8 D3B9 D8FF 8E1F 7DF8 B07E
	cat <<EOF | sudo tee /etc/yum.repos.d/influxdata.repo
	[influxdata]
	name = InfluxData Repository - Stable
	baseurl = https://repos.influxdata.com/stable/\$basearch/main
	enabled = 1
	gpgcheck = 1
	gpgkey = https://repos.influxdata.com/influxdata-archive_compat.key
	EOF
6. Установка и настройка Telegraf:
	sudo yum install telegraf
	cd /etc/telegraf
	telegraf -sample-config --output-filter influxdb > telegraf.conf
	telegraf --test
7. Запуск и настройка Telegraf:
	systemctl enable telegraf
	vi /etc/telegraf/telegraf.cond
-->
	[[outputs.influxdb]]
	  urls = ["http://127.0.0.1:8086"]
	  database = "prom"
	  username = "admin"
	  password = "***"
--
	systemctl restart telegraf
8. Установка и запуск Kapacitor:
	wget https://download.influxdata.com/kapacitor/releases/kapacitor-1.7.5-1.x86_64.rpm
	yum localinstall kapacitor-1.7.5-1.x86_64.rpm
9. Конфигурирование Kapacitor:
	vi /etc/kapacitor/kapacitor.conf
-->
	[[influxdb]]
	  enabled = true
	  default = true
	  name = "localhost"
	  urls = ["http://localhost:8086"]
	  username = "admin"
	  password = "***"
--
	systemctl restart kapacitor
10. Установка и запуск Chronograf:
	wget https://download.influxdata.com/chronograf/releases/chronograf-1.10.5.x86_64.rpm
	yum localinstall chronograf-1.10.5.x86_64.rpm
	chronograf
11. Настройка Chronograf:
	http://localhost:8888/
