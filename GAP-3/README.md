#Main VM
1. Установка и предварительная настройка AlertManager уже была описана в GAP-1:
	https://raw.githubusercontent.com/Raven2526/OTUS/refs/heads/main/GAP-1/README.md
2. Настройка алертов в AlertManager:
	vi /etc/alertmanager/alertmanager.yml
-->
	global:
	  resolve_timeout: 10s
	  telegram_api_url: "https://api.telegram.org"
	route:
	  receiver: "blackhole"
	  group_wait: 10s
	  group_interval: 10s
	  repeat_interval: 5m
	  group_by: ['alertname', 'severity']
	  routes:
	  - receiver: 'otus_critical_alert_bot'
		matchers:
		- severity = "critical"
		- notification = "otus_critical_alert_bot"
	  - receiver: 'otus_warning_alert_bot'
		matchers:
		- severity = "warning"
		- notification = "otus_warning_alert_bot" 
	receivers:
	  - name: 'otus_critical_alert_bot'
		telegram_configs:
		- chat_id: -1002162782026
		  parse_mode: "HTML"
		  bot_token: '675616****:AAF_PNmB7eMEo7w87OwkTTtWUPd1XSvAPeo'
	  - name: 'otus_warning_alert_bot'
		telegram_configs:
		- chat_id: -1002401599329
		  parse_mode: "HTML"
		  bot_token: '742651****:AAEsSH0i7N0j7MIqVqzn2RU_LLSUup-6BRM'
	  - name: 'blackhole'
3. Создание правил для Prometheus:
	vi /etc/prometheus/critical.rules.yml
-->
	groups:
	- name: critical
	  rules:
		- alert: mysql_exporter_status
		  expr: up{job="mysql_exporter"} == 0
		  for: 1m
		  labels:
			severity: critical
			notification: otus_critical_alert_bot
		  annotations:
			summary: "Job {{ $labels.job }} is down"
			description: "Job {{ $labels.job }} is down for the last 1m."
--
	vi /etc/prometheus/warning.rules.yml
-->	groups:
	- name: warning
	  rules:
	  - alert: node_exporter_status
		expr: up{job="node_exporter"} == 0
		for: 1m
		labels:
		  severity: warning
		  notification: otus_warning_alert_bot
		annotations:
		  description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."
		  summary: Instance {{ $labels.instance }} down
--
4. Включение новых правил в конфигурацию Prometheus и активация алертинга:
	vi /etc/prometheus/prometheus.yml
-->
	rule_files:
	  - "critical.rules.yml"
	  - "warning.rules.yml"

	alerting:
	  alertmanagers:
	  - static_configs:
		- targets:
		  - 192.168.0.102:9093
--
5. Рестарт Prometheus, AlertManager:
	systemctl restart prometheus; systemctl restart alertmanager
