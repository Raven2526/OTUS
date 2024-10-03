# Main VM
1. Установка Grafana:
```
yum install grafana -y
```
2. Настройка автозагрузки, запуск демона, проверка статуса:
```
systemctl enable grafana-server
```
```
systemctl start grafana-server
```
```
systemctl status grafana-server
```
3. Настройка порта в firewalld:
```
firewall-cmd --zone=public --add-port=3000/tcp --permanent
```
```
systemctl reload firewalld
```
4. Проверка доступности Grafana:
</br> Firefox --> http://localhost:3000/
5. Настройка источника данных Prometheus:
![Conections](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/connections.png)

6. Создание требуемых папок:
![Folders](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/folders.png)

7. Импортированный дашборд для BlackBox Exporter:
![Imported](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/infra.png)

8. Состояние некоторых элементов CMS:
![Apps](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/app.png)

9. Создание правила для алертинга:
![Alert Rule](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/alert_rule.png)

Ключевые моменты:
- Запрос в "Query" точно такой же, как и на панели.
- Параметры "Reduce" и "Threshold" - строгое последнее значение, меньше 1.
- Задание Evaluation-группы, указание периода "Pending".
- Указание текста аннотации для метки алерта.
- Создание отдельной записи в "Contact Points" - параметры настройки алертинга в Telegram аналогичны предыдущему занятию.
- В "Notification Policy" выбранной по умолчанию, требуется выбрать "Default contact point" - новосозданную запись "Telegram".

Получается следующая мило выглядящая панель:
![Panel](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/alerting_panel.png)

В случае недоступности приходит алерт:
<br>![Firing](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/firing.png)

На панели дашборда виден промежуток недоступности, и даже создаётся аннотация:
![Alerting](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/alerting.png)

10. Для DrillDown-дашборда создаётся отдельная панель:
![DrillDown](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/drilldown.png)

11. Если правильно было понято задание, то DrillDown-дашборд решено было делать через свойства "Override":
![Override](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/override.png)

12. Для каждого набора значений из "Query" - $name, $alias, $instance, $job можно задать нужный линк.
13. При этом элементы панели становятся интерактивными: для каждой отдельной метрики можно перейти в конкретную детализацию: 
![ICMP](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/drilldown_icmp.png)

![ya.ru](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/drilldown_ya_ru.png)

14. <b>P. S.</b>Да, можно сделать ещё красивее - через "Variables" свойств дашборда, тогда детализация будет динамическая (${__.name} и т.д.), но силы были уже слабые, в задании конкретно именно это не оговорено ![Smile](https://github.com/Raven2526/OTUS/blob/main/GAP-4/Images/smile_16.png)
