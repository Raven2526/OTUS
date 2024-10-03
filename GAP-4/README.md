#Main VM
Фаза 1.
1. Установка Grafana:
```
yum install grafana -y
```
3. Настройка автозагрузки, запуск демона, проверка статуса:
```
systemctl enable grafana-server
```
```
systemctl start grafana-server
```
```
systemctl status grafana-server
```
5. Настройка порта в firewalld:
```
firewall-cmd --zone=public --add-port=3000/tcp --permanent
```
```
systemctl reload firewalld
```
7. Проверка доступности Grafana:
</br> Firefox --> http://localhost:3000/
8. Настройка источника данных Prometheus:
