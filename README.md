# Технология непрерывной поставки ПО

## Topic 1: Monitoring with Prometheus

Проверим доступность API сервера метрик:
```bash
kubectl get --raw /apis/metrics.k8s.io/
```

Изучим Загруженность узлов кластера:
```bash
kubectl top node
```

Изучим утилизацию ресурсов модулей:
```bash
kubectl top pods
```

Изучим утилизацию ресурсов модулей во всех пространствах имён:
```bash
kubectl top pods --all-namespaces
```

Изучим утилизацию ресурсов модулей в пространстве имён "kube-system":
```bash
kubectl top pods -n kube-system
```

Создадим тестовый модуль с меткой "pod-with-defaults":
```bash
kubectl run pod-with-defaults --image alpine --restart Never -- /bin/sleep 999999
```

Изучим утилизацию ресурсов модулей с конкретной меткой:
```bash
kubectl top pod -l run=pod-with-defaults
```

Изучим утилизацию ресурсов конкретного модуля:
```bash
kubectl top pod pod-with-defaults
```

Создадим описание тестового модуля с двумя контейнерами:
```bash
vi group-context.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: group-context
spec:
  securityContext:
    fsGroup: 555
    supplementalGroups: [666, 777]
  containers:
  - name: first
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 1111
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  - name: second
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 2222
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  volumes:
  - name: shared-volume
    emptyDir: {}
```

Создадим тестовый модуль с двумя контейнерами:
```bash
kubectl apply -f group-context.yaml
```

Изучим утилизацию ресурсов контейнеров конкретного модуля:
```bash
kubectl top pods group-context --containers
```

Создадим описание тестового модуля с "проверкой живости":
```bash
vi liveness.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness
spec:
  containers:
  - image: linuxacademycontent/kubeserve
    name: kubeserve
    livenessProbe:
      httpGet:
        path: /
        port: 80
```

Создадим тестовый модуль с "проверкой живости":
```bash
kubectl apply -f liveness.yaml
```

Создадим описание двух тестовых модулей с проверкой готовности и сервис для них:
```bash
vi readiness.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
---
apiVersion: v1
kind: Pod
metadata:
  name: nginxpd
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:191
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

Создадим два тестовых модуля с проверкой готовности и сервис для них:
```bash
kubectl apply -f readiness.yaml
```

Изучим созданные тестовые модули и проверим, что только один из двух запущен успешно:
```bash
kubectl get pods
```

Проверим, что у сервиса "nginx" создана одна конечная точка:
```bash
kubectl get ep nginx
```

Исправим ошибку в описании тестового модуля, при запуске которого возникли проблемы:
```bash
kubectl patch pod nginxpd -p '{"spec": {"containers": [{"name": "nginx", "image": "nginx"}]}}'
```

Изучим созданные тестовые модули и проверим, что оба тестовых модуля запущены успешно:
```bash
kubectl get pods
```

Проверим, что у сервиса "nginx" появилась вторая конечная точка:
```bash
kubectl get ep nginx
```

Удалим созданные ресурсы Kubernetes:
```bash
kubectl delete po group-context liveness nginx nginxpd pod-with-defaults
kubectl delete svc nginx
```

#### Задание:
Добавим Liveness Probe в манифесты Deployment и StatefulSet тестового приложения.

Необходимо создать новый экземпляр виртуальной машины для установки Prometheus. Для этого нажмём кнопку "Создать экземпляр". Требуется задать имя виртуальной машины (prometheus) и зону её размещения (us-central1-c). Выбираем тип VM "n1-standart-2". Необходимо будет выбрать также ОС и размер доступного диска. В разделе "Загрузочный диск" нажимаем кнопку "Изменить", выбираем Ubuntu 16.04 LTS и нажимаем кнопку "Выбрать". После этого в разделе "Брандмауэр" ставим флаг "Разрешить трафик HTTP" и нажимаем кнопку "Создать". Всё, ВМ создана. Обычно её развертывание занимает от 1 до 5 минут.

Для демонстрации работы стека Prometheus понадобится открыть несколько дополнительных портов. Для этого сначала протегируем виртуальную машину "prometheus" соответствующим тегом. Перейдём в Web интерфейс Google Cloud Platform и выберем в разделе Ресурсов вкладку Compute Engine. Необходимо открыть экземпляр "prometheus" и нажать кнопку "изменить". В разделе "Теги сети" добавляем тег "prometheus", нажимаем "Enter" и кнопку сохранить. После этого в разделе Ресурсов перейдём на вкладку "Сеть VPC/Правила брандмауэра" и нажмём кнопку "создать правило брандмауэра". В поле название введём "prometheus", в поле "Описание" введём "Open ports for Prometheus", в поле "Теги целевых экземпляров" введём "prometheus", в поле "Диапазон IP-адресов источников" введём "0.0.0.0/0", а в списке "Указанные протоколы и порты" выберем "tcp", укажем порты "9090, 8080, 8081, 8082, 9187, 9093, 3000" и нажмём кнопку "создать". Доступ к Prometheus и связанных с ним приложений открыт.

Переходим на созданную виртуальную машину и скачиваем Prometheus:
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.17.1/prometheus-2.17.1.linux-amd64.tar.gz
```

Разархивируем Prometheus:
```bash
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

Сохраним базовую конфигурацию Prometheus в отдельный файл:
```bash
cp prometheus.yml prometheus-backup.yml
```

Сконфигурируем Prometheus для отслеживания своего состояния:
```bash
echo "" > prometheus.yml
vi prometheus.yml
```

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

Запустим сервер Prometheus с настройками из созданного нами файла конфигураций:
```bash
./prometheus --config.file=prometheus.yml
```

Откроем в браузере URL сервера Prometheus, на котором он предоставляет свои метрики (http://<prometheus-ip>:9090/metrics). Изучим метрики, предоставляемые сервером Prometheus.

Откроем в браузере URL сервера Prometheus, на котором он предоставляет графический интерфейс (http://<prometheus-ip>:9090/graph). Изучим элементы интерфейса Prometheus.
В поле "Expression" введём "prometheus_target_interval_length_seconds" и нажмём кнопку "Execute". Изучим список выведенных метрик.
В поле "Expression" введём "prometheus_target_interval_length_seconds{quantile="0.99"}" (с заданной меткой) и нажмём кнопку "Execute". Изучим вывод.
В поле "Expression" введём "count(prometheus_target_interval_length_seconds)" (посчитаем количество метрик такого типа) и нажмём кнопку "Execute". Изучим вывод.

Нажмём кнопку "Graph". В поле "Expression" введём "rate(prometheus_tsdb_head_chunks_created_total[1m])" (отобразим среднюю скорость увеличения выбранной метрики) и нажмём кнопку "Execute". Изучим сформированный график.

Установим Go для запуска нескольких простых целей мониторинга:
```bash
cd /tmp
wget https://dl.google.com/go/go1.12.linux-amd64.tar.gz
sudo tar -xvf go1.12.linux-amd64.tar.gz
sudo mv go /usr/local
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
go version
```

Запустим несколько приложений для демонстрации настройки мониторинга:
```bash
cd ~/prometheus-2.17.1.linux-amd64/
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random
go get -d
go build
./random -listen-address=:8080 &
./random -listen-address=:8081 &
./random -listen-address=:8082 &
```

Откроем браузер и проверим, что метрики запущенных приложений доступны по HTTP (http://<prometheus-ip>:8080/metrics, http://<prometheus-ip>:8081/metrics, and http://<prometheus-ip>:8082/metrics).

Сконфигурируем Prometheus для сбора метрик с запущенных тестовых приложений:
```bash
cd ~/prometheus-2.17.1.linux-amd64/
echo "" > prometheus.yml
vi prometheus.yml
```

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'test'
```

Запустим сервер Prometheus с настройками из обновлённого файла конфигураций:
```bash
./prometheus --config.file=prometheus.yml
```

Откроем Web интерфейс Prometheus, нажмём на кнопку "Console", в поле "Expression" введём "rpc_durations_seconds" и нажмём кнопку "Execute". Изучим список выведенных метрик (обратим внимание на теги "test" и "production").

В поле "Expression" введём "avg(rate(rpc_durations_seconds_count[5m])) by (job, service)" и нажмём кнопку "Execute". Изучим список выведенных метрик.
Нажмём кнопку "Graph" и изучим графики рассчитанных значений метрик.

Создадим файл с дополнительным правилом сбора метрик в Prometheus:
```bash
vi prometheus.rules.yml
```

```yaml
groups:
- name: example
  rules:
  - record: job_service:rpc_durations_seconds_count:avg_rate5m
    expr: avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

Сконфигурируем Prometheus для сбора метрик с помощью созданного файла с правилом:
```bash
echo "" > prometheus.yml
vi prometheus.yml
```

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # Evaluate rules every 15 seconds.

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  external_labels:
    monitor: 'monitor'

rule_files:
  - 'prometheus.rules.yml'

scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'test'
```

Запустим сервер Prometheus с настройками из обновлённого файла конфигураций:
```bash
./prometheus --config.file=prometheus.yml
```

Откроем Web интерфейс Prometheus, нажмём на кнопку "Console", в поле "Expression" введём "job_service:rpc_durations_seconds_count:avg_rate5m" и нажмём кнопку "Execute".
Нажмём кнопку "Graph" и изучим графики рассчитанных значений метрики.

Остановим запущенные процессы для генерации метрик
```bash
ps -aux | grep random | grep -v grep | awk '{print $2}' | xargs kill -9
```

Установим PostgreSQL на виртуальную машину:
```bash
sudo apt-get install wget ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt-get update
sudo apt-get install -y postgresql postgresql-contrib
sudo service postgresql enable
sudo service postgresql start
```

Настроим использование PostgreSQL из под пользователя "exporter":
```bash
sudo useradd exporter -m
sudo passwd exporter
sudo su - postgres
psql
postgres-# CREATE ROLE exporter WITH LOGIN CREATEDB ENCRYPTED PASSWORD 'exporter';
postgres-# \q
su - exporter
createdb exporter
psql
exporter-# \list
exporter-# \q
```

Установим и запустим Prometheus PostgreSQL Exporter:
```bash
go get github.com/wrouesnel/postgres_exporter
cd ${GOPATH-$HOME/go}/src/github.com/wrouesnel/postgres_exporter
go run mage.go binary
export DATA_SOURCE_NAME="postgresql://exporter:exporter@localhost:5432/exporter"
./postgres_exporter
nohup ./postgres_exporter > exporter.out 2>&1 &
```

Откроем URL Prometheus PostgreSQL Exporter (http://<prometheus-ip>:9187/metrics) в браузере и проверим, что метрики БД доступны по HTTP.

Сконфигурируем Prometheus для сбора метрик с Prometheus PostgreSQL Exporter:
```bash
cd ~/prometheus-2.17.1.linux-amd64/
echo "" > prometheus.yml
vi prometheus.yml
```

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'PostgreSQL'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9187']
```

Запустим сервер Prometheus с настройками из созданного нами файла конфигураций:
```bash
./prometheus --config.file=prometheus.yml
```

Откроем Web интерфейс Prometheus, нажмём на кнопку "Console", в поле "Expression" введём "pg_up" и нажмём кнопку "Execute". Изучим список выведенных метрик.
Нажмём кнопку "Graph" и изучим график доступности БД.

Установим Alert Manager:
```bash
sudo adduser --no-create-home --disabled-login --shell /bin/false --gecos "Alertmanager User" alertmanager
sudo mkdir /etc/alertmanager
sudo mkdir /etc/alertmanager/template
sudo mkdir -p /var/lib/alertmanager/data
sudo touch /etc/alertmanager/alertmanager.yml
sudo chown -R alertmanager:alertmanager /etc/alertmanager
sudo chown -R alertmanager:alertmanager /var/lib/alertmanager
wget https://github.com/prometheus/alertmanager/releases/download/v0.20.0/alertmanager-0.20.0.linux-amd64.tar.gz
tar xvzf alertmanager-0.20.0.linux-amd64.tar.gz
sudo cp alertmanager-0.20.0.linux-amd64/alertmanager /usr/local/bin/
sudo cp alertmanager-0.20.0.linux-amd64/amtool /usr/local/bin/
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/amtool
sudo vi /etc/alertmanager/alertmanager.yml
```

```yaml
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@example.org'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'

templates:
- '/etc/alertmanager/template/*.tmpl'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 1m
  repeat_interval: 3h
  receiver: team-X-mails
  routes:
  - match:
      job: "PostgreSQL"
    receiver: team-X-mails

receivers:
- name: 'team-X-mails'
  email_configs:
  - to: 'team-X+alerts@example.org'

inhibit_rules:
- source_match:
    severity: 'page'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  # CAUTION: 
  #   If all label names listed in `equal` are missing 
  #   from both the source and target alerts,
  #   the inhibition rule will apply!
  equal: ['alertname', 'cluster', 'service']
```

Создадим описание сервиса для Alert Manager:
```bash
sudo vi /etc/systemd/system/alertmanager.service
```

```ini
[Unit]
Description=Prometheus Alertmanager Service
Wants=network-online.target
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
    --config.file /etc/alertmanager/alertmanager.yml \
    --storage.path /var/lib/alertmanager/data
Restart=always

[Install]
WantedBy=multi-user.target
```

Запустим Alert Manager:
```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
rm alertmanager-0.20.0.linux-amd64.tar.gz
rm -rf alertmanager-0.20.0.linux-amd64
```

Откроем Web интерфейс Alert Manager (http://<prometheus-ip>:9093/#/alerts) и проверим, что он доступен из браузера. Изучим компоненты интерфейса более подробно.

Сконфигурируем Prometheus для использования развёрнутого Alert Manager:
```bash
echo "" > prometheus.yml
vi prometheus.yml
```

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'monitor'

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - localhost:9093

rule_files:
  - 'prometheus.rules.yml'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'PostgreSQL'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9187']
```

Создадим правило Prometheus для генерации алерта:
```bash
echo "" > prometheus.rules.yml
vi prometheus.rules.yml
```


```yaml
groups:
- name: postgresql-down
  rules:

  - alert: more-numbackends
    expr: pg_stat_database_numbackends > 1
    for: 15s
    labels:
      severity: page
    annotations:
      summary: "On instance {{ $labels.instance }} numbackends more than 1"
      description: "On {{ $labels.instance }} instance of {{ $labels.job }} numbackends has been more than 1"
```

Запустим Prometheus в виде демона:
```bash
nohup ./prometheus --config.file=prometheus.yml > prometheus.out 2>&1 &
```

Запустим фреймворк для тестирования PostgeSQL для создания алерта:
```bash
su - exporter
pgbench -i
pgbench -T 120 exporter
exit
```

Перейдём в веб интерфейс Prometheus и проверим, что созданный алерт перешёл в статус "Firing". Откроем Web интерфейс Alert Manager и проверим, что появился алерт.

Настроим отправку уведомлений из Alert Manager в Slack. Для этого откроем сайт https://slack.com/ и нажмём кнопку "GET STARTED". Нажмём кнопку "+ Create a Slack Workspace", введём адрес своей электронной почты и нажмём кнопку "Confirm". В поле "What's the name of your company or team?" введём "<your-last-name>-alerting", в поле "What’s a project your team is working on?" введём "Prometheus", нажмём на ссылку "skip for now" и кнопку "See Your Channel in Slack". Перейдём по ссылке на приложение "Incoming WebHooks" для Slack (https://slack.com/apps/A0F7XDUAZ-incoming-webhooks) и добавим приложение к каналу в Slack:
- нажмём на кнопку "Add to Slack"
- выберем в списке "Post to Channel" занчение "#prometheus"
- нажмём на кнопку "Add Incoming WebHook integretion"
- скопируем ссылку на "Webhook URL": https://hooks.slack.com/services/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

Настроим отправку уведомлений в Slack в конфигурации Alert Manager:
```bash
sudo rm /etc/alertmanager/alertmanager.yml
sudo vi /etc/alertmanager/alertmanager.yml
```

```yaml
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@example.org'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'

templates:
- '/etc/alertmanager/template/*.tmpl'


route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 1m
  repeat_interval: 3h
  receiver: slack-channel
  routes:
  - match:
      job: "PostgreSQL"
    receiver: slack-channel

receivers:
- name: slack-channel
  slack_configs:
  - api_url: <slack-webhook-url>
    channel: #prometheus
    icon_url: https://avatars3.githubusercontent.com/u/3380462
    send_resolved: true
    title: '{{ template "custom_title" . }}'
    text: '{{ template "custom_slack_message" . }}'

inhibit_rules:
- source_match:
    severity: 'page'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  # CAUTION: 
  #   If all label names listed in `equal` are missing 
  #   from both the source and target alerts,
  #   the inhibition rule will apply!
  equal: ['alertname', 'cluster', 'service']
```

Добавим шаблон для отправки уведомлений:
```bash
vi /etc/alertmanager/template/notifications.tmpl
```

```go
{{ define "__single_message_title" }}{{ range .Alerts.Firing }}{{ .Labels.alertname }} @ {{ .Annotations.identifier }}{{ end }}{{ range .Alerts.Resolved }}{{ .Labels.alertname }} @ {{ .Annotations.identifier }}{{ end }}{{ end }}
{{ define "custom_title" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ if or (and (eq (len .Alerts.Firing) 1) (eq (len .Alerts.Resolved) 0)) (and (eq (len .Alerts.Firing) 0) (eq (len .Alerts.Resolved) 1)) }}{{ template "__single_message_title" . }}{{ end }}{{ end }}
{{ define "custom_slack_message" }}
{{ if or (and (eq (len .Alerts.Firing) 1) (eq (len .Alerts.Resolved) 0)) (and (eq (len .Alerts.Firing) 0) (eq (len .Alerts.Resolved) 1)) }}
{{ range .Alerts.Firing }}{{ .Annotations.description }}{{ end }}{{ range .Alerts.Resolved }}{{ .Annotations.description }}{{ end }}
{{ else }}
{{ if gt (len .Alerts.Firing) 0 }}
*Alerts Firing:*
{{ range .Alerts.Firing }}- {{ .Annotations.identifier }}: {{ .Annotations.description }}
{{ end }}{{ end }}
{{ if gt (len .Alerts.Resolved) 0 }}
*Alerts Resolved:*
{{ range .Alerts.Resolved }}- {{ .Annotations.identifier }}: {{ .Annotations.description }}
{{ end }}{{ end }}
{{ end }}
{{ end }}
```

Перезапустим Alert Manager:
```bash
sudo systemctl stop alertmanager
sudo systemctl start alertmanager
```

Запустим фреймворк для тестирования PostgeSQL для создания алерта:
```bash
su - exporter
pgbench -T 120 exporter
exit
```

Откроем в Slack канал "#prometheus" и проверим, что уведомление пришло.

Настроим предоставление метрик в собственном приложении на Golang. Для этого клонируем с GitHub пример тестового приложения и установим нужные зависимости:
```bash
cd ..
git clone https://github.com/jenoOvchi/hackeru-devops-go-simple.git
cd hackeru-devops-go-simple/
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promauto
go get github.com/prometheus/client_golang/prometheus/promhttp
```

Добавим в приложение нужную зависимость и Handler:
```bash
vi hello.go
```

```go
...
import (
    "fmt"
    "net/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)
...
func main() {
    http.HandleFunc("/", HelloWorld)
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":3000", nil)
}
...
```

Соберём и запустим приложение:
```bash
go build -o hello
./hello
```

Откроем в браузере адрес приложения (http://<prometheus-ip>:3000/metrics) и проверим, что его метрики доступны. Несколько раз открываем страницу http://<prometheus-ip>:3000 и изучаем метрику promhttp_metric_handler_requests_total{code="200"}.

Добавим в код приложения сбор и предоставление отдельной метрики:
```bash
vi hello.go
```

```go
...
import (
    "fmt"
    "time"
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)
...
func recordMetrics() {
        go func() {
                for {
                        opsProcessed.Inc()
                        time.Sleep(2 * time.Second)
                }
        }()
}

var (
        opsProcessed = promauto.NewCounter(prometheus.CounterOpts{
                Name: "myapp_processed_ops_total",
                Help: "The total number of processed events",
        })
)
...
func main() {
    recordMetrics()
    http.HandleFunc("/", HelloWorld)
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":3000", nil)
}
...
```

Соберём и запустим приложение:
```bash
go build -o hello
./hello
```

Откроем в браузере адрес приложения (http://<prometheus-ip>:3000/metrics) и проверим, что его метрики доступны. Несколько раз открываем страницу http://<prometheus-ip>:3000 и изучаем метрику myapp_processed_ops_total.

#### Задание:
Добавить в тестовое приложение предоставление метрик по HTTP, добавить пользовательскую метрику, запустить его с помощью Docker Compose и настроить для него сбор метрик с помощью Prometheus (сбор метрик с помощью PostgreSQL оставить):

Установим Grafana:
```bash
sudo wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install -y grafana=6.3.5
```

Запустим сервис Grafana:
```bash
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Открываем в браузере Web интерфейс Grafana http://<prometheus-ip>:3000 и авторизуемся (admin/admin). Изучаем элементы интерфейса Grafana и разделы меню.

Нажимаем на кнопку "Configurations/Data Sources", "Add data source" и добавляем новый источник данных:
- Type: Prometheus
- Name: Prometheus
- Url: http://localhost:9090
- Access: Server

Нажимаем на кнопку "Save & Test".

Открываем официальный сайт Grafana (https://grafana.com), выбираем раздел "Dashboards" и находим в поиске Dashboard "PostgreSQL Database". 

Добавим информационную панель. Для этого нажмём на кнопку "+/Import" и в поле "Grafana.com Dashboard" вводим номер дэшборда (9628). В списке "DS_PROMETHEUS" выбираем "Prometheus" и нажимаем "Import".

Нажмём на один из графиков, выберем пункт "edit" и изучим способ сортировки данных.

Добавим график на загруженный дашборд.

#### Задание:
Добавить дэшборд для мониторинга Go приложений.

## Topic 2: Logging with ELK Stack

Перейдём по SSH на один из узлов кластера Kubernetes и изучим директорию, в которой Kubelet хранит логи контейнеров:
```bash
ls /var/log/containers
```

Создадим описание тестового модуля, пишущего логи параллельно в два файла:
```bash
vi twolog.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

Создадим тестовый модуль, пишущий логи параллельно в два файла:
```bash
kubectl apply -f twolog.yaml
```

Изучим папку с файлами логов, созданных в тестовом модуле:
```bash
kubectl exec counter -- ls /var/log
```

Создадим описание тестового модуля, пишущего логи параллельно в два файла и имеющего два sidecar контейнера, читающих логи в стандартный поток вывода:
```bash
vi counter-with-sidecars.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

Создадим тестовый модуль, пишущий логи параллельно в два файла и имеющий два sidecar контейнера, читающих логи в стандартный поток вывода:
```bash
kubectl apply -f counter-with-sidecars.yaml
```

Изучим логи первого sidecar контейнера:
```bash
kubectl logs counter count-log-1
```

Изучим логи второго sidecar контейнера:
```bash
kubectl logs counter count-log-2
```

Создадим описание конфигурации развёртывания тестового модуля:
```bash
vi nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

Создадим развёртывание тестового модуля из этого описания:
```bash
kubectl apply -f nginx.yaml
```

Изучим логи созданного тестового модуля:
```bash
kubectl logs $(kubectl get po | grep nginx | head -n 1 | awk '{print $1}')
```

Изучим логи одного из контейнеров ранее созданного тестового модуля:
```bash
kubectl logs counter -c count-log-1
```

Изучим логи всех контейнеров ранее созданного тестового модуля:
```bash
kubectl logs counter --all-containers=true
```

Изучим логи всех модулей с определённой меткой:
```bash
kubectl logs -l app=nginx
```

Изучим логи ранее завершённого контейнера созданного тестового модуля (если такой есть):
```bash
kubectl logs -p -c nginx $(kubectl get po | grep nginx | head -n 1 | awk '{print $1}')
```

Изучим логи одного из контейнеров ранее созданного тестового модуля, активировав режим слежения:
```bash
kubectl logs -f -c count-log-1 counter
```

Выведем определённое количество логов созданного тестового модуля:
```bash
kubectl logs --tail=20 $(kubectl get po | grep nginx | head -n 1 | awk '{print $1}')
```

Выведем логи созданного тестового модуля за определённый период:
```bash
kubectl logs --since=1h $(kubectl get po | grep nginx | head -n 1 | awk '{print $1}')
```

Выведем логи контейнеров с определённым именем заданной конфигурации развёртывания:
```bash
kubectl logs deployment/nginx-deployment -c nginx
```

Сохраним логи одного из контейнеров ранее созданного тестового модуля в файл:
```bash
kubectl logs counter -c count-log-1 > count.log
```

Удалим созданные ресурсы Kubernetes:
```bash
kubectl delete deployment nginx-deployment
kubectl delete po counter
```

#### Задание:
Добавить логирование запросов в приложение bookapp, запустить обновлённую версию в Kubernetes, сделать запрос к приложению и изучить логи пода.

Необходимо создать новый экземпляр виртуальной машины для установки ELK Stack. Для этого нажмём кнопку "Создать экземпляр". Требуется задать имя виртуальной машины (elastic) и зону её размещения (us-central1-c). Выбираем тип VM "n1-standart-2". Необходимо будет выбрать также ОС и размер доступного диска. В разделе "Загрузочный диск" нажимаем кнопку "Изменить", выбираем Ubuntu 16.04 LTS и нажимаем кнопку "Выбрать". После этого в разделе "Брандмауэр" ставим флаг "Разрешить трафик HTTP" и нажимаем кнопку "Создать". Всё, ВМ создана. Обычно её развертывание занимает от 1 до 5 минут.

Для демонстрации работы стека ELK понадобится открыть несколько дополнительных портов. Для этого сначала протегируем виртуальную машину "elastic" соответствующим тегом. Перейдём в Web интерфейс Google Cloud Platform и выберем в разделе Ресурсов вкладку Compute Engine. Необходимо открыть экземпляр "elastic" и нажать кнопку "изменить". В разделе "Теги сети" добавляем тег "elastic", нажимаем "Enter" и кнопку сохранить. После этого в разделе Ресурсов перейдём на вкладку "Сеть VPC/Правила брандмауэра" и нажмём кнопку "создать правило брандмауэра". В поле название введём "elastic", в поле "Описание" введём "Open ports for ELK", в поле "Теги целевых экземпляров" введём "elastic", в поле "Диапазон IP-адресов источников" введём "0.0.0.0/0", а в списке "Указанные протоколы и порты" выберем "tcp", укажем порты "9090, 8080, 8081, 8082, 9187, 9093, 3000" и нажмём кнопку "создать". Доступ к ELK открыт.

Добавим нужные ключи и установим необходимые пакеты:
```bash
sudo wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
sudo apt-get update && sudo apt-get install -y openjdk-8-jre apt-transport-https wget nginx
```

Проверим версию Java:
```bash
java -version
```

Создаем пользователя и пароль для доступа к Kibana:
```bash
sudo echo "admin:`openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.users
```

Конфигурируем Nginx для доступа к Kibana:
```bash
sudo vi /etc/nginx/sites-available/kibana
```

```json
server {
listen 80;

server_name suricata.server;

auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/htpasswd.users;

location / {
proxy_pass http://localhost:5601;
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection 'upgrade';
proxy_set_header Host $host;
proxy_cache_bypass $http_upgrade;
}
}
```

Обновляем конфигурацию Nginx:
```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/kibana /etc/nginx/sites-enabled/kibana
```

Перезапускаем Nginx:
```bash
sudo systemctl restart nginx
```

Установим и запустим Elastic Search:
```bash
sudo apt-get install -y elasticsearch
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```

Проверим, что Elastic Search запущен:
```bash
curl -X GET "localhost:9200/"
```

Установим и запустим Kibana:
```bash
sudo apt-get install -y kibana
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
sudo systemctl start kibana.service
```

Открываем в браузере URL Kibana (http://<elk-vm-ip>/) и авторизуемся (admin/!QAZ2wsx) и изучаем интерфейс Kibana.

Установим Go для запуска тестового приложения:
```bash
cd /tmp
wget https://dl.google.com/go/go1.12.linux-amd64.tar.gz
sudo tar -xvf go1.12.linux-amd64.tar.gz
sudo mv go /usr/local
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
go version
```

Создадим директорию для тестового приложения:
```bash
mkdir testapp
cd testapp
```

Создадим тестовое приложение на go для демонстрации логирования:
```bash
vi testapp.go
```

```go
package main

import (
    "log"
)

func main() {
    log.Print("Logging in Go!")
}
```

Соберём тестовое приложение:
```bash
go build testapp.go
```

Запустим тестовое приложение и изучим выведенный лог:
```bash
./testapp
```

Изменим тестовое приложение для демонстрации записи журнала в файл:
```bash
vi testapp.go
```

```go
package main

import (
    "log"
    "os"
)

func main() {
    file, err := os.OpenFile("info.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal(err)
    }

    defer file.Close()

    log.SetOutput(file)
    log.Print("Logging to a file in Go!")
}
```

Соберём тестовое приложение:
```bash
go build testapp.go
```

Запустим тестовое приложение и изучим созданный файл журнала:
```bash
./testapp
ls
cat info.log
```

Установим фреймворк для журналирования:
```bash
go get "github.com/Sirupsen/logrus"
```

Изучим различные уровни логирования, доступные в рамках фреймворка:
```go
log.Debug("Useful debugging information.")
log.Info("Something noteworthy happened!")
log.Warn("You should probably take a look at this.")
log.Error("Something failed but I'm not quitting.")
// Calls os.Exit(1) after logging
log.Fatal("Bye.")
// Calls panic() after logging
log.Panic("I'm bailing.")
```

Изменим тестовое приложение для демонстрации работы фреймворка:
```bash
vi testapp.go
```

```go
package main

import (
    "os"
    log "github.com/Sirupsen/logrus"
)

func main() {
    file, err := os.OpenFile("info.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal(err)
    }

    defer file.Close()

    log.SetOutput(file)
    log.Print("Logging to a file in Go!")
}
```

Соберём тестовое приложение:
```bash
go build testapp.go
```

Запустим тестовое приложение и изучим созданный файл журнала:
```bash
./testapp
ls
cat info.log
```

Изменим тестовое приложение для демонстрации работы различных уровней логирования:
```bash
vi testapp.go
```

```go
package main

import (
    "os"

    log "github.com/Sirupsen/logrus"
)

func main() {
    file, err := os.OpenFile("info.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal(err)
    }

    defer file.Close()

    log.SetOutput(file)
    log.SetFormatter(&log.JSONFormatter{})
    log.SetLevel(log.WarnLevel)

    log.WithFields(log.Fields{
        "animal": "walrus",
        "size":   10,
    }).Info("A group of walrus emerges from the ocean")

    log.WithFields(log.Fields{
        "omg":    true,
        "number": 122,
    }).Warn("The group's number increased tremendously!")

    log.WithFields(log.Fields{
        "omg":    true,
        "number": 100,
    }).Fatal("The ice breaks!")
}
```

Соберём тестовое приложение:
```bash
go build testapp.go
```

Запустим тестовое приложение и изучим созданный файл журнала:
```bash
./testapp
ls
cat info.log
```

Создадим директорию для логов и сделаем её владельцем текущего пользователя:
```bash
sudo mkdir /var/log/testapp
sudo chown $(whoami):$(whoami) /var/log/testapp
```

Изменим тестовое приложение для демонстрации работы различных уровней логирования:
```bash
vi testapp.go
```

```go
package main

import (
    "os"

    log "github.com/Sirupsen/logrus"
)

func main() {
    file, err := os.OpenFile("/var/log/testapp/info.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal(err)
    }

    defer file.Close()

    log.SetOutput(file)
    log.SetFormatter(&log.JSONFormatter{})
    log.SetLevel(log.WarnLevel)

    log.WithFields(log.Fields{
        "animal": "walrus",
        "size":   10,
    }).Info("A group of walrus emerges from the ocean")

    log.WithFields(log.Fields{
        "omg":    true,
        "number": 122,
    }).Warn("The group's number increased tremendously!")

    log.WithFields(log.Fields{
        "omg":    true,
        "number": 100,
    }).Fatal("The ice breaks!")
}
```

Соберём тестовое приложение:
```bash
go build testapp.go
```

Запустим тестовое приложение и изучим созданный файл журнала:
```bash
./testapp
cat /var/log/testapp/info.log
```

Создадим скрипт для непрерывного запуска данного приложения:
```bash
vi testapp-loop.bash
```

```bash
#!/bin/bash
while true; do
  ./testapp
  sleep 1
done
```

Сделаем созданный скрипт исполняемым:
```bash
chmod +x testapp-loop.bash
```

Запустим созданный скрипт:
```bash
nohup /home/vagrant/testapp/testapp-loop.bash > testapp-loop.out 2>&1 &
```

Изучим созданный лог:
```bash
tail -f /var/log/testapp/info.log
```

Установим и запустим filebeat:
```bash
sudo apt-get install -y filebeat
sudo systemctl daemon-reload
sudo systemctl enable filebeat
sudo systemctl start filebeat.service
```

Настроим filebeat:
```bash
sudo vi /etc/filebeat/filebeat.yml
```

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/testapp/info.log
```

Перезапустим filebeat:
```bash
sudo systemctl restart filebeat.service
```

Открываем Web интерфейс Kibana, создаём индекс и переходим в раздел Discovery. Изучаем логи, созданные тестовым приложением.