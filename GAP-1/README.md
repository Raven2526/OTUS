ПЕРЕД ПРОЧТЕНИЕМ:
В случае необходимости я готов предоставить удалённый доступ (только просмотр) к моим VM для обсуждения текущего решения задания.

#Main VM
Фаза 1.
0. Конфигурирование VM, установка образа CentOS 9.
1. Установка дополнительных пакетов:
	yum install wget tar
2. Установка и запуск chrony для возможности синхронизации времени:
	yum install chrony
	systemctl enable chronyd
	systemctl start chronyd
3. Открытие портов в firewall:
	firewall-cmd --permanent --add-port=9090/tcp --add-port=9093/tcp --add-port=9094/{tcp,udp} --add-port=9100/tcp --add-port=9104/tcp --add-port=9115/tcp --add-port=1947/tcp
	firewall-cmd --runtime-to-permanent
	firewall-cmd --reload
4. Отключение SELinux enforcing:
	setenforce 0
	sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
5. Скачивание и установка Prometheus:
	wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
6. Создание каталогов:
	mkdir /etc/prometheus /var/lib/prometheus
7. Распаковка архива:
	tar -zxf prometheus-*.linux-amd64.tar.gz
8. Распределение файлов по каталогам, очистка "мусора":
	cd prometheus-*.linux-amd64
	cp prometheus promtool /usr/local/bin/
	cp -r console_libraries consoles prometheus.yml /etc/prometheus
	cd .. && rm -rf prometheus-*.linux-amd64/ && rm -f prometheus-*.linux-amd64.tar.gz
9. Создание пользователя:
	useradd --no-create-home --shell /bin/false prometheus
10. Замена владельца файлов и каталогов:
	chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
	chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}
11. Запуск и проверка Prometheus:
	sudo -u prometheus /usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries
12. Настройка службы Prometheus:
	vi /etc/systemd/system/prometheus.service
-->
	[Unit]
	Description=Prometheus Service
	Documentation=https://prometheus.io/docs/introduction/overview/
	After=network.target

	[Service]
	User=prometheus
	Group=prometheus
	Type=simple
	ExecStart=/usr/local/bin/prometheus \
	 --config.file /etc/prometheus/prometheus.yml \
	 --storage.tsdb.path /var/lib/prometheus/ \
	 --web.console.templates=/etc/prometheus/consoles \
	 --web.console.libraries=/etc/prometheus/console_libraries
	ExecReload=/bin/kill -HUP $MAINPID
	Restart=on-failure

	[Install]
	WantedBy=multi-user.target
--
13. Настройка автозапуска, проверка:
	systemctl enable prometheus
	systemctl start prometheus
	systemctl status prometheus
Фаза 2.
1. Скачивание и установка AlertManager:
	wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
2. Создание каталогов:
	mkdir -p /etc/alertmanager /var/lib/prometheus/alertmanager
3. Распаковка архива:
	tar -zxf alertmanager-*.linux-amd64.tar.gz
4. Распределение файлов по каталогам, очситка "мусора:
	cd alertmanager-*.linux-amd64
	cp alertmanager amtool /usr/local/bin/
	cp alertmanager.yml /etc/alertmanager
	cd .. && rm -rf alertmanager-*.linux-amd64/
5. Создание пользователя:
	useradd --no-create-home --shell /bin/false alertmanager
6. Замена владельца файлов и каталогов:
	chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/prometheus/alertmanager
	chown alertmanager:alertmanager /usr/local/bin/{alertmanager,amtool}
7. Настройка службы Alertmanager:
	vi /etc/systemd/system/alertmanager.service
-->
	[Unit]
	Description=Alertmanager Service
	After=network.target

	[Service]
	EnvironmentFile=-/etc/default/alertmanager
	User=alertmanager
	Group=alertmanager
	Type=simple
	ExecStart=/usr/local/bin/alertmanager \
          	--config.file=/etc/alertmanager/alertmanager.yml \
          	--storage.path=/var/lib/prometheus/alertmanager \
          	--cluster.advertise-address=0.0.0.0:9093 \
          	$ALERTMANAGER_OPTS
	ExecReload=/bin/kill -HUP $MAINPID
	Restart=on-failure

	[Install]
	WantedBy=multi-user.target
--
8. Настройка автозапуска, проверка:
	systemctl enable alertmanager
	systemctl start alertmanager
	systemctl status alertmanager

#Node1 VM
Фаза 3.
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
	firewall-cmd --permanent --add-port=9090/tcp --add-port=9093/tcp --add-port=9094/{tcp,udp} --add-port=9100/tcp --add-port=9104/tcp  --add-port=9115/tcp --add-port=1947/tcp
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

Фаза 4.
1. Скачивание и установка node_exporter:
	wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
2. Распаковка архива, работа с файлами и каталогами:
	tar -zxf node_exporter-*.linux-amd64.tar.gz
	cd node_exporter-*.linux-amd64
	cp node_exporter /usr/local/bin/
	cd .. && rm -rf node_exporter-*.linux-amd64/ && rm -f node_exporter-*.linux-amd64.tar.gz
3. Создание пользователя:
	useradd --no-create-home --shell /bin/false nodeusr
4. Смена разрешений:
	chown -R nodeusr:nodeusr /usr/local/bin/node_exporter
5. Настройка автозапуска:
	vi /etc/systemd/system/node_exporter.service
-->
	[Unit]
	Description=Node Exporter Service
	After=network.target

	[Service]
	User=nodeusr
	Group=nodeusr
	Type=simple
	ExecStart=/usr/local/bin/node_exporter
	ExecReload=/bin/kill -HUP $MAINPID
	Restart=on-failure

	[Install]
	WantedBy=multi-user.target
--
6. Настройка и проверка автозапуска:
	systemctl enable node_exporter
	systemctl start node_exporter
	systemctl status node_exporter

#Main VM
Фаза 5.
1. Настройка узла node_exporter в Prometheus:
	vi /etc/prometheus/prometheus.yml
-->
	scrape_configs:
  	...
  	- job_name: 'node_exporter_clients'
    	scrape_interval: 5s
    	static_configs:
      	- targets:
          	- 192.168.0.104:9100
--
2. Перезапуск Prometheus:
	systemctl restart prometheus
3. Настройка правила алертинга:
	vi /etc/prometheus/alert.rules.yml
-->
	groups:
	- name: alert.rules
	  rules:
	  - alert: InstanceDown
	    expr: up == 0
	    for: 1m
	    labels:
	      severity: critical
	    annotations:
	      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down
	        for more than 1 minute.'
	      summary: Instance {{ $labels.instance }} down
--
4. Добавление правила в Prometheus:
	vi /etc/prometheus/prometheus.yml
-->
	rule_files:
	  - "alert.rules.yml"
--
5. Перезапуск Prometheus:
	systemctl restart prometheus
#Node1 VM
6. Настройка мониторинга служб VM:
	vi /etc/systemd/system/node_exporter.service
-->
	ExecStart=/usr/local/bin/node_exporter --collector.systemd --collector.systemd.unit-whitelist="(chronyd|mariadb|nginx).service"
--
7. Рестарт служб:
	systemctl daemon-reload
	systemctl restart node_exporter
#Main VM
8. Настройка мониторинга nginx, mariadb, chronyd:
	vi /etc/prometheus/services.rules.yml
-->
	groups:
	- name: services.rules
	  rules:
	    - alert: nginx_service
	      expr: node_systemd_unit_state{name="nginx.service",state="active"} == 0
	      for: 1s
	      annotations:
	        summary: "Instance {{ $labels.instance }} is down"
	        description: "{{ $labels.instance }} of job {{ $labels.job }} is down."
	    - alert: mariadb_service
	      expr: node_systemd_unit_state{name="mariadb.service",state="active"} == 0
	      for: 1s
	      annotations:
	        summary: "Instance {{ $labels.instance }} is down"
	        description: "{{ $labels.instance }} of job {{ $labels.job }} is down."
	    - alert: chronyd_service
	      expr: node_systemd_unit_state{name="chronyd.service",state="active"} == 0
	      for: 1s
	      annotations:
	        summary: "Instance {{ $labels.instance }} is down"
	        description: "{{ $labels.instance }} of job {{ $labels.job }} is down."
--
9. Подключение нового правила:
	vi /etc/prometheus/prometheus.yml
-->
	rule_files:
	  - "alert.rules.yml"
	  - "services.rules.yml"
--
10. Снова перезапуск Prometheus:
	systemctl restart prometheus

Фаза 6.
#Node1 VM
1. Скачивание и установка mysqld_exporter:
	wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
2. Распаковка архива, работа с файлами и каталогами:
	tar -zxf mysqld_exporter-0.15.1.linux-amd64.tar.gz
	cd mysqld_exporter-0.15.1.linux-amd64
	cp mysqld_exporter /usr/local/bin/
	cd .. && rm -rf mysqld_exporter-*.linux-amd64/ && rm -f mysqld_exporter-*.linux-amd64.tar.gz
3. Создание пользователя:
	useradd --no-create-home --shell /bin/false mysqlexporter
4. Смена разрешений:
	chown -R mysqlexporter:mysqlexporter /usr/local/bin/mysqld_exporter
5. Создание пользователя БД MySQL:
	mysql -u root -p
	CREATE USER 'mysqlexporter'@'localhost' IDENTIFIED BY 'StrongPassword';
	GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqlexporter'@'localhost';
	FLUSH PRIVILEGES;
	quit;
	vi /etc/.mysqld_exporter.cnf
-->
	[client]
	user=mysqlexporter
	password=StrongPassword
--
	chown mysqlexporter:mysqlexporter /etc/.mysqld_exporter.cnf
6. Настройка автозапуска:
	vi /etc/systemd/system/mysql_exporter.service
-->
	[Unit]
	Description=Prometheus MySQL Exporter
	After=network.target
	User=mysqlexporter
	Group=mysqlexporter

	[Service]
	Type=simple
	Restart=always
	ExecStart=/usr/local/bin/mysqld_exporter \
	--config.my-cnf /etc/.mysqld_exporter.cnf \
	--collect.global_status \
	--collect.info_schema.innodb_metrics \
	--collect.auto_increment.columns \
	--collect.info_schema.processlist \
	--collect.binlog_size \
	--collect.info_schema.tablestats \
	--collect.global_variables \
	--collect.info_schema.query_response_time \
	--collect.info_schema.userstats \
	--collect.info_schema.tables \
	--collect.perf_schema.tablelocks \
	--collect.perf_schema.file_events \
	--collect.perf_schema.eventswaits \
	--collect.perf_schema.indexiowaits \
	--collect.perf_schema.tableiowaits \
	--collect.slave_status \
	--web.listen-address=192.168.0.104:9104

	[Install]
	WantedBy=multi-user.target
--
7. Настройка и проверка автозапуска:
	systemctl daemon-reload
	systemctl enable mysql_exporter
	systemctl start mysql_exporter
	systemctl status mysql_exporter
#Main VM
8. Добавление правила в Prometheus:
	vi /etc/prometheus/prometheus.yml
-->
	scrape_configs:
	  - job_name: "mysqldb"
	    scrape_interval: 5s
	    static_configs:
	      - targets:
		- 192.168.0.104.9104
	        labels:
	          alias: wordpress
--
9.Снова перезапуск Promeheus:
	systemctl restart prometheus
#Node1 VM
1. Скачивание и установка blackbox_exporter:
	wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
2. Распаковка архива, работа с файлами и каталогами:
	tar -zxf blackbox_exporter-0.25.0.linux-amd64.tar.gz
	cd blackbox_exporter-0.25.0.linux-amd64
	cp blackbox_exporter /usr/local/bin/
	cd .. && rm -rf blackbox_exporter-*.linux-amd64/ && rm -f blackbox_exporter-*.linux-amd64.tar.gz
	mkdir /opt/blackbox
3. Создание пользователя:
	useradd --no-create-home --shell /bin/false blackbox_exporter
4. Смена разрешений:
	chown -R blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
5. Настройка конфигурации систменого юнита blackbox_exporter:
	vi /etc/systemd/system/blackbox_exporter.service
-->
	[Unit]
	Description=Blackbox Exporter
	Documentation=https://github.com/prometheus/blackbox_exporter
	Wants=network-online.target
	After=network-online.target

	[Service]
	User=blackbox_exporter
	Group=blackbox_exporter
	Restart=on-failure
	RestartSec=5
	Type=simple
	AmbientCapabilities=CAP_NET_RAW
	ExecStart=/usr/local/bin/blackbox_exporter \
	    --config.file="/opt/blackbox/blackbox.yml" \
	    --web.listen-address="127.0.0.1:9115"
	ExecReload=/bin/kill -HUP $MAINPID
	
	[Install]
	WantedBy=multi-user.target
--
6. Перечитывание списка юнитов:
	systemctl daemon-reload
7. Настройка конфигурации blackbox_exporter:
	vi /opt/blackbox/blackbox.yml
-->
	modules:
	  http_2xx:
		prober: http
		timeout: 5s
		http:
		  valid_status_codes: []
		  method: GET
		  preferred_ip_protocol: ip4
		  fail_if_ssl: false
		  fail_if_not_ssl: true
		  tls_config:
			insecure_skip_verify: true
	  http_post_2xx:
		prober: http
		http:
		  method: POST
	  tcp_connect:
		prober: tcp
		timeout: 5s
		tcp:
		  preferred_ip_protocol: "ip4"
	  ssh_banner:
		prober: tcp
		tcp:
		  query_response:
		  - expect: "^SSH-2.0-"
	  icmp:
		prober: icmp
		timeout: 5s
		icmp:
		  preferred_ip_protocol: "ip4"
		  ip_protocol_fallback: false
--
8. Настройка автозапуска, проверка:
	systemctl enable --now blackbox_exporter
	systemctl status blackbox_exporter
#Main VM
9. Настройка blackbox_exporter в Prometheus:
	vi /etc/prometheus/prometheus.yml
-->
	  - job_name: 'Blackbox-ICMP'
		scrape_interval: 5m
		metrics_path: /probe
		params:
		  module: [icmp]
		file_sd_configs:
		  - files:
			- /etc/prometheus/targets.d/blackbox-icmp.yml
		relabel_configs:
		  - source_labels: [__address__]
			target_label: __param_target
		  - source_labels: [__param_target]
			target_label: instance
		  - target_label: __address__
			replacement: 192.168.0.104:9115

	  - job_name: 'Blackbox-TCP'
		scrape_interval: 5m
		metrics_path: /probe
		params:
		  module: [tcp_connect]
		file_sd_configs:
		  - files:
			- /etc/prometheus/targets.d/blackbox-tcp.yml
		relabel_configs:
		  - source_labels: [__address__]
			target_label: __param_target
		  - source_labels: [__param_target]
			target_label: instance
		  - target_label: __address__
			replacement: 192.168.0.104:9115

	  - job_name: 'Blackbox-HTTP'
		scrape_interval: 5m
		metrics_path: /probe
		params:
		  module: [http_2xx]
		file_sd_configs:
		  - files:
			- /etc/prometheus/targets.d/blackbox-http.yml
		relabel_configs:
		  - source_labels: [__address__]
			target_label: __param_target
		  - source_labels: [__param_target]
			target_label: instance
		  - target_label: __address__
			replacement: 192.168.0.104:9115

  	- job_name: 'SSL-blackbox-http_2xx'
    		scrape_interval: 5s
    		metrics_path: /probe
    		params:
	      	  module: [http_2xx]
	    	file_sd_configs:
	     	  - files:
	     	  	- /etc/prometheus/targets.d/blackbox-http.yml
	    	relabel_configs:
	      	- source_labels: [__address__]
	        	target_label: __param_target
	      	- source_labels: [__param_target]
	        	target_label: instance
	      	- target_label: __address__
	        	replacement: 192.168.0.104:9115
--
10. Создание папки для конфигураций:
	mkdir /etc/prometheus/targets.d
	chown -R prometheus:prometheus /etc/prometheus/targets.d
11. Создание файла со списком хостов для ICMP-проверки:
	vi /etc/prometheus/targets.d/blackbox-icmp.yml
-->
	- targets:
	  - 192.168.0.104
	  labels:
	    alias: ping
--
12. Создание файла со списком хостов для проверки TCP-портов:
	vi /etc/prometheus/targets.d/blackbox-tcp.yml
-->
	- targets:
	  - 192.168.0.104:1947
	  labels:
	    alias: tcp-connect
--
13. Создание файла со списком хостов для HTTP-проверки, валидности SSL-сертификатов:
	vi /etc/prometheus/targets.d/blackbox-http.yml
-->
	- targets:
	  - ya.ru
	  labels:
	    alias: http
--
14. Перезапуск Prometheus:
	systemctl restart prometheus
	systemctl status prometheus

Фаза 7.
#Main VM
#Node
1. Перенастройка nginx:
	systemctl enable nginx
	firewall-cmd --add-service=http
	firewall-cmd --add-service=https
	firewall-cmd --runtime-to-permanent
2. Создание SSL-сертификата:
	mkdir /etc/ssl/private
	chmod 700 /etc/ssl/private
	openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
	openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
3. Настройка виртуального хоста:
	vi /etc/nginx/conf.d/ssl.conf
-->
	server {
	listen 443 http2 ssl;
	listen [::]:443 http2 ssl;
	server_name 192.168.0.104;
	ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
	ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
	ssl_dhparam /etc/ssl/certs/dhparam.pem;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_prefer_server_ciphers on;
	ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
	ssl_ecdh_curve secp384r1;
	ssl_session_cache shared:SSL:10m;
	ssl_session_tickets off;
	ssl_stapling on;
	ssl_stapling_verify on;
	resolver 8.8.8.8 8.8.4.4 valid=300s;
	resolver_timeout 5s;
	add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
	add_header X-Frame-Options DENY;
	add_header X-Content-Type-Options nosniff;
	root /usr/share/nginx/html;
	location / {
	}
	error_page 404 /404.html;
	location = /404.html {
	}
	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
	}
--
4. Настройка редиректа с HTTP на HTTPS:
	vi /etc/nginx/default.d/ssl-redirect.conf
-->
	return 301 https://$host$request_uri/;
--
5. Обновление настроек Nginx:
	nginx -t
	systemctl restart nginx
