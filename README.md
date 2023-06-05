# Syslog, ELK, Logstash
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/syslog-hw16.git`
В текущей директории появится папка с именем репозитория. В данном случае syslog-hw16. Ознакомимся с содержимым:
```
cd syslog-hw16
ls -l
ansible
README.md
Vagrantfile
```
Здесь:
- ansible - папка с плэйбуками
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ:
```
vagrant up
```
Дальнейшие работы будут проведены под пользователем root:
```
vagrant ssh
sudo -i
```
## Настройка сислог агента (Разоб плэйбука)
Настраиваем временную зону, так как мое локальная зона в Бишкеке - берем зону Asia/Bishkek:
```
- name: SYSLOG-CLIENT | Set timezone to Asia/Bishkek
  timezone:
    name: Asia/Bishkek
  tags:
    - timezone-configuration
```
Устанавливаем необходимые пакеты:
```
- name: SYSLOG-CLIENT | Install NGINX, audispd-plugins package
  yum:
    name: 
     - nginx
     - audispd-plugins 
    state: latest
  notify:
    - restart nginx
    - reload auditd
  tags:
    - nginx-package
    - audispd-plugins-package
    - packages
```
Конфигурируем веб-сервер для пересылку логов:
```
- name: SYSLOG-CLIENT | Create NGINX config file from template
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - reload nginx
  tags:
    - nginx-configuration
```
Проводим необходимые настройки rsyslog для пересылки логов:
```
- name: SYSLOG-CLIENT | Append json-template to rsyslog
  copy:
    dest: /etc/rsyslog.d/json-template.conf
    src: json-template.conf
    owner: root
    group: root
    mode: 0644
  notify:
    - reload rsyslog
  tags:
    - rsyslog-configuration

- name: SYSLOG-CLIENT | Configure to send logs
  template:
    src: rsyslog.conf.j2
    dest: /etc/rsyslog.conf
  notify:
    - reload rsyslog
  tags:
    - rsyslog-configuration
```
Настраиваем auditd для пересылки логов:
```
- name: SYSLOG-CLIENT | Configure AUDITD rules
  copy:
    dest: /etc/audit/audit.rules
    src: audit.rules
    owner: root
    group: root
    mode: 0640
  notify:
    - reload auditd
  tags:
    - auditd-configuration

- name: SYSLOG-CLIENT | Configure AUDITD
  copy:
    dest: /etc/audit/auditd.conf
    src: auditd.conf
    owner: root
    group: root
    mode: 0640
  notify:
    - reload auditd
  tags:
    - auditd-configuration

- name: SYSLOG-CLIENT | Configure AUDITD plugin to send auditd logs
  copy:
    dest: /etc/audit/plugins.d/au-remote.conf
    src: au-remote.conf
    owner: root
    group: root
    mode: 0640
  notify:
    - reload auditd
  tags:
    - auditd-configuration

- name: SYSLOG-CLIENT | Configure to send audit logs
  template:
    src: audisp-remote.conf.j2
    dest: /etc/audit/audisp-remote.conf
  notify:
    - reload auditd
  tags:
    - auditd-configuration
```
## Настройка лог сервера (Разоб плэйбука)
Также настраиваем временную зону:
```
- name: LOG-SERVER | set timezone to Asia/Bishkek
  timezone:
    name: Asia/Bishkek
  tags:
    - timezone-configuration
```
Настраиваем rsyslog для приёма логов:
```
- name: LOG-SERVER | Configure RSYSLOG to recieve logs
  copy:
    dest: /etc/rsyslog.conf
    src: rsyslog.conf
    owner: root
    group: root
    mode: 0644
  notify:
    - reload rsyslog
  tags:
    - rsyslog-configuration
```
Настраиваем auditd для приёма логов:
```
- name: LOG-SERVER | Configure auditd to recieve audit logs
  copy:
    dest: /etc/audit/auditd.conf
    src: auditd.conf
    owner: root
    group: root
    mode: 0640
  notify:
    - reload auditd
  tags:
    - auditd-configuration
```
## Настройка ELK сервера (Пошаговая инструкция)
Переходим в elk-сервер:
```
$ vagrant ssh elk
```
Устанавливаем Elasticsearch с официальнго сайта:
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
apt install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-8.x.list
apt update && apt install elasticsearch
systemctl daemon-reload
```
В процессе установки будет выведен пароль elastic, при необходимости можно сбросить пароль следующей командой:
```
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```
Настраиваем Elasticsearch для прослушки внутренного IP-адреса:
```
$ vi /etc/elasticsearch/elasticsearch.yml
network.host: 192.168.50.20
```
Включаем сервис в автозагрузку и запускаем сервис:
```
systemctl --now enable elasticsearch
```
Далее проводим установку Logstash:
```
apt install logstash
```
Настраиваем Logstash, создаем конфигурационный файл:
```
$ vi /etc/logstash/conf.d/logstash.conf
# This input block will listen on port 10514 for logs to come in.
# host should be an IP on the Logstash server.
# codec => "json" indicates that we expect the lines we're receiving to be in JSON format
# type => "rsyslog" is an optional identifier to help identify messaging streams in the pipeline.
input {
  udp {
    host => "192.168.50.20"
    port => 10514
    codec => "json"
    type => "rsyslog"
  }
}
# This is an empty filter block.  You can later add other filters here to further process
# your log lines
filter { }
# This output block will send all events of type "rsyslog" to Elasticsearch at the configured
# host and port into daily indices of the pattern, "rsyslog-YYYY.MM.DD"
output {
  if [type] == "rsyslog" {
    elasticsearch {
      hosts => [ "https://192.168.50.20:9200" ]     # веб-портал elasticsearch
      ssl_certificate_verification => false
      user => elastic
      password => password_from_installation        # Сюда вставляем пароль, который был дан при установки elasticsearch
    }
  }
}
```
Включаем сервис в автозагрузку и запускаем сервис Logstash:
```
systemctl daemon-reload
systemctl --now enable logstash.service
```
Проверяем:
```
$ curl -XGET -u elastic:password 'https://192.168.50.20:9200/_all/_search?q=*&pretty' -k 
{
  "took" : 148,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 652,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : ".ds-logs-generic-default-2023.06.05-000001",
        "_id" : "au4Ai4gB5stVim62wNg8",
        "_score" : 1.0,
        "_source" : {
          "@version" : "1",
          "message" : "(/etc/cron.hourly) starting 0anacron",
          "severity" : "notice",
          "sysloghost" : "web",
          "procid" : "18368",
          "programname" : "run-parts",
          "facility" : "cron",
          "@timestamp" : "2023-06-05T10:01:01.823563Z",
          "type" : "rsyslog",
          "event" : {
            "original" : "{\"@timestamp\":\"2023-06-05T16:01:01.823563+06:00\",\"@version\":\"1\",\"message\":\"(\\/etc\\/cron.hourly) starting 0anacron\",\"sysloghost\":\"web\",\"severity\":\"notice\",\"facility\":\"cron\",\"programname\":\"run-parts\",\"procid\":\"18368\"}\n"
          },
          "host" : {
            "ip" : "192.168.50.10"
          },
          "data_stream" : {
            "type" : "logs",
            "dataset" : "generic",
            "namespace" : "default"
          }
        }
      },
      {
        "_index" : ".ds-logs-generic-default-2023.06.05-000001",
        "_id" : "a-4Ai4gB5stVim62wNg8",
        "_score" : 1.0,
        "_source" : {
          "@version" : "1",
          "message" : "(root) CMD (run-parts /etc/cron.hourly)",
          "severity" : "info",
          "sysloghost" : "web",
          "procid" : "18368",
          "programname" : "CROND",
          "facility" : "cron",
          "@timestamp" : "2023-06-05T10:01:01.800611Z",
          "type" : "rsyslog",
          "event" : {
            "original" : "{\"@timestamp\":\"2023-06-05T16:01:01.800611+06:00\",\"@version\":\"1\",\"message\":\"(root) CMD (run-parts \\/etc\\/cron.hourly)\",\"sysloghost\":\"web\",\"severity\":\"info\",\"facility\":\"cron\",\"programname\":\"CROND\",\"procid\":\"18368\"}\n"
          },
          "host" : {
            "ip" : "192.168.50.10"
          },
          "data_stream" : {
            "type" : "logs",
            "dataset" : "generic",
            "namespace" : "default"
          }
        }
      },
```