Фаза 0.
VM #01, VM #02, VM #03:
1. Добавление разрешений Firewall:
	#MariaDB
	firewall-cmd --zone=public --permanent --add-port={3306,4444,4567,4568}/tcp
	firewall-cmd --zone=public --permanent --add-port=4567/udp
	#nginx
	firewall-cmd --zone=public --permanent --add-service={http,https}
	#Prometheus
	firewall-cmd --zone=public --permanent --add-port={9090,9093,9094,9100,9104,9115}/tcp
	firewall-cmd --zone=public --permanent --add-port=9094/udp
	#VictoriaMetrics
	firewall-cmd --zone=public --permanent --add-port=8428/tcp
	#Grafana
	firewall-cmd --zone=public --permanent --add-port=3000/tcp
	#Service reload
	firewall-cmd --reload
2. Установка и запуск MariaDB Server и Galera:
	yum install -y mariadb-server mariadb-server-galera galera
	systemctl enable mariadb.service --now
	systemctl status mariadb.service
3. Выполнение настроек безопасности MariaDB Server:
	mysql_secure_installation

Фаза 1.
VM #01:
1. Настройка конфигурации MariaDB Server на VM #01:
	vi /etc/my.cnf.d/mariadb-server.cnf
-->
[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://192.168.0.112,192.168.0.113,192.168.0.114"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
wsrep_cluster_name="MariaDB_Cluster"
wsrep_node_address="192.168.0.112"
wsrep_node_name="GC01"
bind-address=0.0.0.0
--

VM #02:
2. Настройка конфигурации MariaDB Server на VM #02:
	vi /etc/my.cnf.d/mariadb-server.cnf
-->
wsrep_node_address="192.168.0.113"
wsrep_node_name="GC02"
--

VM #03:
3. Настройка конфигурации MariaDB Server на VM #03:
	vi /etc/my.cnf.d/mariadb-server.cnf
-->
wsrep_node_address="192.168.0.114"
wsrep_node_name="GC03"
--

Фаза 2.
VM #01:
1. Создание кластера Galera:
	galera_new_cluster

VM #02, VM #03:
2. Перезапуск MariaDB на VM #02, VM #03:
	systemctl restart mariadb
	systemctl status mariadb

VM #01:
3. Проверка состояния кластера:
	mysql -u root -p -e "show status like 'wsrep_cluster_size'";
	quit;
4. Создание тестовой БД и тестовых данных:
	mysql -u root -p
	create database tst;
	use tst;
	create table distro(id int,dist varchar(50));
	insert into distro values(1,'Ubuntu');
	insert into distro values(2,'CentOS');
	insert into distro values(3,'Mint');
	select * from tst.distro;
	quit;

VM #02, VM #03:
5. Проверка репликации данных:
	mysql -u root -p
	select * from tst.distro;
	quit;

Фаза 3.
VM #03:
1. Установка nginx:
	yum install nginx -y
2. Раздача прав на Web-каталог:
	chown nginx:nginx /usr/share/nginx/html -R
3. Настройка и проверка автозапуска nginx:
	systemctl enable nginx.service --now
	systemctl status nginx.service
4. Установка PHP-FPM:
	yum install php php-mysqlnd php-fpm php-opcache php-gd php-xml php-mbstring -y
	mkdir /var/run/php
5. Настройка и проверка автозапуска PHP:
	systemctl enable php-fpm.service --now
	systemctl status php-fpm.service
6. Конфигурирование PHP-FPM:
	vi /etc/php-fpm.d/www.conf
-->
user = nginx
group = nginx
--
7. Рестарт PHP-FPM и nginx:
	systemctl reload php-fpm
	systemctl restart nginx php-fpm
8. Установка инструментов разработки:
	yum groupinstall "Development tools" -y
	yum install wget -y
9. Установка WordPress:
	cd /var/www/html
	wget https://wordpress.org/latest.tar.gz
	tar xvzf latest.tar.gz
	rm -fv latest.tar.gz
10. Раздача прав для ./wordpress/:
	chown -R nginx:nginx /var/www/html/wordpress
	chmod -Rf 775 ./wordpress/
11. Настройка nginx vHost:
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
12. Рестарт и проверка nginx:
	systemctl restart nginx.service
	systemctl status nginx.service
13. Настройка БД WordPress:
	mysql -u root
	CREATE DATABASE wordpress;
	CREATE USER 'wordpress'@'localhost' IDENTIFIED BY 'secret';
	GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'localhost';
	FLUSH PRIVILEGES;
	quit;
14. Настройка CMS WordPress:
	cd /var/www/html/wordpress
	vi wp-config.php
-->
...config from the CMS wizzard...
!keep password in safe!
--

Фаза 4.
VM #01:
1. Скачивание Prometheus:
	wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
2. Создание каталогов для Prometheus:
	mkdir /etc/prometheus /var/lib/prometheus
3. Распаковка архива Prometheus:
	tar -zxf prometheus-*.linux-amd64.tar.gz
4. Распределение файлов Prometheus по каталогам, очистка "мусора":
	cd prometheus-*.linux-amd64
	cp prometheus promtool /usr/local/bin/
	cp -r console_libraries consoles prometheus.yml /etc/prometheus
	cd .. && rm -rf prometheus-*.linux-amd64/ && rm -f prometheus-*.linux-amd64.tar.gz
5. Создание пользователя Prometheus:
	useradd --no-create-home --shell /bin/false prometheus
6. Замена владельца файлов и каталогов Prometheus:
	chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
	chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}
7. Запуск и проверка Prometheus:
	sudo -u prometheus /usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries
8. Настройка службы Prometheus:
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
9. Настройка автозапуска, проверка:
	systemctl enable prometheus.service --now
	systemctl status prometheus.service

Фаза 5.
VM #01:
1. Скачивание и установка AlertManager:
	wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
2. Создание каталогов для AlertManager:
	mkdir -p /etc/alertmanager /var/lib/prometheus/alertmanager
3. Распаковка архива AlertManager:
	tar -zxf alertmanager-*.linux-amd64.tar.gz
4. Распределение файлов AlertManager по каталогам, очистка "мусора":
	cd alertmanager-*.linux-amd64
	cp alertmanager amtool /usr/local/bin/
	cp alertmanager.yml /etc/alertmanager
	cd .. && rm -rf alertmanager-*.linux-amd64/
5. Создание пользователя AlertManager:
	useradd --no-create-home --shell /bin/false alertmanager
6. Замена владельца файлов и каталогов AlertManager:
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
8. Настройка автозапуска, проверка AlertManager:
	systemctl enable alertmanager.service --now
	systemctl status alertmanager.service

Фаза 6.
VM #01, VM #02, VM #03:
1. Скачивание и установка Node Exporter:
	wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
2. Распаковка архива, работа с файлами и каталогами Node Exporter:
	tar -zxf node_exporter-*.linux-amd64.tar.gz
	cd node_exporter-*.linux-amd64
	cp node_exporter /usr/local/bin/
	cd .. && rm -rf node_exporter-*.linux-amd64/ && rm -f node_exporter-*.linux-amd64.tar.gz
3. Создание пользователя Node Exporter:
	useradd --no-create-home --shell /bin/false nodeusr
4. Смена разрешений Node Exporter:
	chown -R nodeusr:nodeusr /usr/local/bin/node_exporter
5. Настройка автозапуска Node Exporter:
	vi /etc/systemd/system/node_exporter.service
-->
[Unit]
Description=Node Exporter Service
After=network.target

[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.systemd --collector.systemd.unit-whitelist="(chronyd|mariadb|nginx).service"
#nginx только на VM #03
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
--
6. Настройка и проверка автозапуска Node Exporter:
	systemctl enable node_exporter.service --now
	systemctl status node_exporter.service

Фаза 7.
VM #01, VM #02, VM #03:
1. Скачивание и установка MySQL Exporter:
	wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
2. Распаковка архива, работа с файлами и каталогами MySQL Exporter:
	tar -zxf mysqld_exporter-0.15.1.linux-amd64.tar.gz
	cd mysqld_exporter-0.15.1.linux-amd64
	cp mysqld_exporter /usr/local/bin/
	cd .. && rm -rf mysqld_exporter-*.linux-amd64/ && rm -f mysqld_exporter-*.linux-amd64.tar.gz
3. Создание пользователя MySQL Exporter:
	useradd --no-create-home --shell /bin/false mysqlexporter
4. Смена разрешений для MySQL Exporter:
	chown -R mysqlexporter:mysqlexporter /usr/local/bin/mysqld_exporter
5. Создание пользователя БД MySQL:
	#Сделать требуется только 1 раз на любой VM, т. к. у нас Galera Cluster
	mysql -u root -p
	CREATE USER 'mysqlexporter'@'localhost' IDENTIFIED BY 'StrongPassword';
	GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqlexporter'@'localhost';
	FLUSH PRIVILEGES;
	quit;
6. Настройка конфигурации MySQL Exporter:
	vi /etc/.mysqld_exporter.cnf
-->
[client]
user=mysqlexporter
password=StrongPassword
host=127.0.0.1
--
7. Раздача прав на файл с доступами к БД:
	chown mysqlexporter:mysqlexporter /etc/.mysqld_exporter.cnf
8. Настройка автозапуска MySQL Exporter:
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
--web.listen-address=192.168.0.112:9104
#Подставить нужный IP

[Install]
WantedBy=multi-user.target
--
9. Настройка и проверка автозапуска MySQL Exporter:
	systemctl enable mysql_exporter.service --now
	systemctl status mysql_exporter.service

Фаза 8.
VM #03:
1. Скачивание и установка BlackBox Exporter:
	wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
2. Распаковка архива, работа с файлами и каталогами BlackBox Exporter:
	tar -zxf blackbox_exporter-0.25.0.linux-amd64.tar.gz
	cd blackbox_exporter-0.25.0.linux-amd64
	cp blackbox_exporter /usr/local/bin/
	cd .. && rm -rf blackbox_exporter-*.linux-amd64/ && rm -f blackbox_exporter-*.linux-amd64.tar.gz
	mkdir /opt/blackbox
3. Создание пользователя BlackBox Exporter:
	useradd --no-create-home --shell /bin/false blackbox_exporter
4. Смена разрешений папки BlackBox Exporter:
	chown -R blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
5. Настройка конфигурации систменого юнита BlackBox Exporter:
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
    --web.listen-address="192.168.0.114:9115"
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
--
6. Настройка конфигурации BlackBox Exporter:
	vi /opt/blackbox/blackbox.yml
-->
modules:
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_status_codes: [200,302,301,304,401,403] 
      method: GET
  http_post_2xx:
    prober: http
    timeout: 10s
    http:
      valid_status_codes: [200,302,301,304,401,403]
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  grpc:
    prober: grpc
    grpc:
      tls: true
      preferred_ip_protocol: "ip4"
  grpc_plain:
    prober: grpc
    grpc:
      tls: false
      service: "service1"
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
      - send: "SSH-2.0-blackbox-ssh-check"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
--
7. Настройка автозапуска, проверка BlackBox Exporter:
	systemctl enable blackbox_exporter.service --now
	systemctl status blackbox_exporter.service

Фаза 9.
VM #01:
1. Настройка конфигурации Prometheus:
	vi /etc/prometheus/prometheus.yml
2. Создание правил алертинга для Prometheus:
	vi /etc/prometheus/alert.rules.yml
	vi /etc/prometheus/warning.rules.yml
	vi /etc/prometheus/critical.rules.yml
	vi /etc/prometheus/services.rules.yml
3. Настройка алертов в AlertManager:
	vi /etc/alertmanager/alertmanager.yml
4. Рестарт Prometheus, AlertManager:
	systemctl restart prometheus.service
	systemctl restart alertmanager.service
	systemctl status prometheus.service
	systemctl status alertmanager.service

Фаза 10.
VM #02:
1. Скачивание VictoriaMetrics:
	wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.103.0/victoria-metrics-linux-amd64-v1.103.0.tar.gz
2. Распаковка архива, работа с файлами:
	tar zxf victoria-metrics-linux-amd64-*.tar.gz -C /usr/local/bin/
	rm -rf victoria-metrics-linux-amd64-*.tar.gz
3. Создание служебного пользователя, работа с каталогами:
	useradd -r -c 'VictoriaMetrics TSDB Service' victoriametrics
	mkdir -p /var/lib/victoriametrics /run/victoriametrics
	chown victoriametrics:victoriametrics /var/lib/victoriametrics /run/victoriametrics
4. Создание файла юнита:
	vi /etc/systemd/system/victoriametrics.service
-->
[Unit]
Description=VictoriaMetrics
After=network.target

[Service]
Type=simple
User=victoriametrics
PIDFile=/run/victoriametrics/victoriametrics.pid
ExecStart=/usr/local/bin/victoria-metrics-prod -storageDataPath /var/lib/victoriametrics -retentionPeriod 14d
ExecStop=/bin/kill -s SIGTERM $MAINPID
StartLimitBurst=5
StartLimitInterval=0
Restart=on-failure
RestartSec=1

[Install]
WantedBy=multi-user.target
--
5. Настройка автозапуска юнита:
	systemctl daemon-reload
	systemctl enable victoriametrics --now
	systemctl status victoriametrics
6. Проверка работоспособности с помощью curl:
	curl 192.168.0.113:8428

Фаза 11.
VM #01:
1. Настройка конфигурации Prometheus на работу с VictoriaMetrics:
	vi /etc/prometheus/prometheus.yml
-->
remote_write:
  - url: http://192.168.0.113:8428/api/v1/write
    queue_config:
      max_samples_per_send: 10000
      capacity: 20000
      max_shards: 30
--
2. Перезапуск Prometheus, проверка работоспособности VictoriaMetrics:
	systemctl restart prometheus
	Firefox --> http://192.168.0.113:8428/vmui/
	Firefox --> node_memory_MemTotal_bytes
	Firefox --> http://192.168.0.113:8428/metrics
Фаза 12.
VM #02:
1. Скачивание, распаковака и установка Grafana:
	yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-10.1.0-1.x86_64.rpm
2. Настройка конфигурации Grafana:
	vi /etc/grafana/grafana.ini
-->
[server]
protocol = http
http_addr = 0.0.0.0
http_port = 3000
[auth.anonymous]
enabled = true
--
3. Настройка автозагрузки Grafana, запуск демона, проверка статуса:
	systemctl enable grafana-server --now
	systemctl status grafana-server
4. Проверка доступности Grafana:
	Firefox --> http://192.168.0.113:3000/
