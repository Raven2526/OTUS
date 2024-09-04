#Main VM
Фаза 1.
1. Проверка наличия нужных пакетов:
	yum list installed tar wget curl
2. Разрешение порта 8428 в Firewall:
	firewall-cmd --permanent --add-port=8428/tcp
	firewall-cmd --reload
3. Скачивание и установка дистрибутива VictoriaMetrics:
	wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.103.0/victoria-metrics-linux-amd64-v1.103.0.tar.gz
4. Распаковка архива:
	tar zxf victoria-metrics-linux-amd64-*.tar.gz -C /usr/local/bin/
5. Создание служебного пользователя, работа с каталогами:
	useradd -r -c 'VictoriaMetrics TSDB Service' victoriametrics
	mkdir -p /var/lib/victoriametrics /run/victoriametrics
	chown victoriametrics:victoriametrics /var/lib/victoriametrics /run/victoriametrics
6. Создание файла юнита:
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
7. Настройка автозапуска юнита:
	systemctl daemon-reload
	systemctl enable victoriametrics --now
	systemctl status victoriametrics
8. Проверка работоспособности с помощью curl:
	curl 192.168.0.102:8428

Фаза 2.
1. Настройка конфигурации Prometheus на работу с VictoriaMetrics:
	vi /etc/prometheus/prometheus.yml
-->
	global:
	  external_labels:
		site: prod

	remote_write:
	  - url: http://192.168.0.102:8428/api/v1/write
		queue_config:
		  max_samples_per_send: 10000
		  capacity: 20000
		  max_shards: 30
--
2. Перезапуск Promrtheus, проверка работоспособности:
	systemctl restart prometheus
	Firefox --> http://192.168.0.102:8428/vmui/
-->
	node_memory_MemTotal_bytes{site="prod"}
--
3. Указание экспортера для визуализации метрик:
	vi /etc/prometheus/prometheus.yml
-->
	  - job_name: 'node_exporter_clients'
		static_configs:
		  - targets:
			  - 192.168.0.102:8428
--
4. Перезапуск Prometheus, проверка:
	systemctl restart prometheus
	Firefox --> http://192.168.0.102:8428/metrics
