# Технология непрерывной поставки ПО

## Topic 1: Create Kubernetes Cluster on Google Cloud Platform

1. Открываем в браузере https://cloud.google.com/kubernetes-engine/docs/quickstart
2. Открываем URL "Kubernetes Engine page"
3. Подтверждаем пользовательское соглашение
4. Нажимаем "Создать"
5. Вводим "Название проекта" и нажимаем "Создать"
6. Нажимаем "Активировать пробный период"
7. Выбираем "Страна" и подтверждаем пользовательское соглашение
8. Заполняем информацию о карте. В пробный период (до 1 года) деньги сниматься не будут
9. Нажимаем "Включить оплату"

Открываем консоль Google Cloud Platform и переходим в раздел "Kubernetes Engine". Нажимаем кнопку "Создать кластер". Вводим и выбираем:
Название: test-k8s-cluster
Тип местоположения: Зональный
Зона: europe-north1-a
Версия головного узла: Статическая версия
Статическая версия: 1.14.10-gke.36

Нажимаем кнопку "Создать" и дожидаемся когда кластер будет развёрнут. Нажимаем кнопку "Подключиться", в открывшейся форме копируем команду из раздела "Командная строка". Нажимаем кнопку "Активировать Cloud Shell" и в открывшемся терминале вставляем и выполняем скопированную команду. После этого проверяем доступность кластера:
```bash
kubectl get nodes
kubectl describe node $(kubectl get nodes | grep -v NAME | awk '{print $1}' | head -n 1)
```

## Install Dashboard
Создаём ресурсы информационной панели:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

Получаем авторизационные данные:
```bash
kubectl create serviceaccount dashboard -n default
kubectl create clusterrolebinding dashboard-admin -n default --clusterrole=cluster-admin --serviceaccount=default:dashboard
kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

Запускаем прокси для доступа к информационной панели:
```bash
kubectl proxy -p 8081
```

1. В Cloud Shell нажимаем на кнопку "Посмотреть в браузере", выбираем пункт "Выбрать другой порт и выбираем из списка порт 8081;
2. В открывшейся вкладке заменяем путь после домена верхнего уровня на "/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/" и нажимаем "Enter";
3. Выбираем опцию  "Token", копируем авторизационные данные из терминала, удаляем переносы строк, вставляем в поле "Enter token" и нажимаем "Sign In".

## Topic 2: Understanding Kubernetes Architecture

В Cloud Shell нажимаем на кнопку "Открыть новую вкладку" и переходим в новую вкладку терминала.

Проверим статус компонентов управляющей плоскости Kubernetes:
```bash
kubectl get componentstatus
```

Создадим модуль "nginx" из описания:
```yaml
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

Проверим, что модуль создался:
```bash
kubectl get pods
```

Перейдём в Dashboard Kubernetes и обратим внимание на появившийся под, пространство "Workload Status" и графики метрик использования CPU и RAM (появляются чуть позднее).

Изучим созданный модуль:
```bash
kubectl describe pods nginx
```

Удалим созданный модуль:
```bash
kubectl delete pod nginx
```

Создадим описание конфигурации развёртывания:
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
```

Выведем созданное описание конфигурации развёртывания:
```bash
cat nginx.yaml
```

Создадим развёртывание из этого описания:
```bash
kubectl create -f nginx.yaml
```

Изучим созданное развёртывание:
```bash
kubectl get deployment nginx-deployment -o yaml
```

Перейдём в Dashboard Kubernetes и обратим внимание на появившийся рабочие нагрузки "Deployments", " Pods" и "Replica Sets".

Выведем список созданных модулей с их метками:
```bash
kubectl get pods --show-labels
```

Пометим первый модуль из списка меткой "env=prod":
```bash
kubectl label pods $(kubectl get pods | grep nginx | head -n 1 | awk '{print $1}') env=prod
```

Выведем список созданных модулей с дополнительным столбцом наличия метки "env":
```bash
kubectl get pods -L env
```

Пометим развёртывание "nginx-deployment" аннотацией 'vtb.ru/someannotation="sometext"':
```bash
kubectl annotate deployment nginx-deployment vtb.ru/someannotation="sometext"
```

Изучим модифицированное развёртывание:
```bash
kubectl get deployment nginx-deployment -o yaml
```

Выведем список всех модулей в статусе "Running":
```bash
kubectl get pods --field-selector status.phase=Running
```

Выведем список всех сервисов в пространстве имён "default":
```bash
kubectl get services --field-selector metadata.namespace=default
```

Выведем список всех модулей, соответствующих двум условиям (статус "Running" и пространство имён "default"):
```bash
kubectl get pods --field-selector status.phase=Running,metadata.namespace=default
```

Выведем список всех модулей, не соответствующих обоим условиям (статус "Running" и пространство имён "default"):
```bash
kubectl get pods --field-selector=status.phase!=Running,metadata.namespace!=default
```

Изучим список созданных модулей:
```bash
kubectl get pods -o wide
```

Удалим первый модуль из списка:
```bash
kubectl delete pod $(kubectl get pods | grep nginx | head -n 1 | awk '{print $1}')
```

Проверим, что после удаления модуля запустилась его новая реплика:
```bash
kubectl get pods -o wide
```

Создадим службу с помощью Dashboard Kubernetes. Для этого перейдём в Dashboard Kubernetes и нажмём кнопку "+". На открывшейся форме вставим текстовую область следующий манифест:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  selector:
    app: nginx
```

Нажмём кнопку "Upload" и изучим созданную службу:
```bash
kubectl get services nginx-nodeport
```

Добавляем правило в фаервол Google Cloud Platform для предоставления доступа к сервису:
```bash
gcloud compute firewall-rules create svc-rule --allow=tcp:30080
```

Определяем внешний адрес узла:
```bash
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
```

Проверяем доступность приложений на порте узла:
```bash
curl http://$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{print $1}'):30080
```

Проверим доступность Nginx из браузера. Для этого сгенерируем ссылку на его стартовую страницу:
```bash
echo "Nginx external link: http://$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{print $1}'):30080"
```

Удаляем созданное правило в фаерволе Google Cloud Platform:
```bash
gcloud compute firewall-rules delete svc-rule
```

Изучим список созданных модулей:
```bash
kubectl get pods
```

Развернём модуль для отправки HTTP запросов внутри кластера:
```bash
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: radial/busyboxplus:curl
    args:
    - sleep
    - "1000"
EOF
```

Изучим список созданных модулей:
```bash
kubectl get pods -o wide
```

Изучим список созданных служб:
```bash
kubectl get services
```

Вызовем из модуля "busybox" службу "nginx-nodeport" по HTTP:
```bash
kubectl exec busybox -- curl $(kubectl get services | grep nginx-nodeport | awk '{print $3}'):80
```

Создадим Deployment с помощью Dashboard Kubernetes. Для этого перейдём в Dashboard Kubernetes и нажмём кнопку "+". На открывшейся форме перейдём в подраздел "Create from form" и заполним поля следующим образом:
App name: redis
Container image: redis:alpine
Number of pods: 1
Service: None

Нажимаем кнопку "Deploy" и проверяем, что Deployment успешно создан.

Удаляем созданные ресурсы:
```bash
kubectl delete pod busybox
kubectl delete svc nginx-nodeport
kubectl delete deployment nginx-deployment
kubectl delete deployment redis
```

#### Задание:
Развернуть WordPress (https://hub.docker.com/_/wordpress) в Kubernetes. Создать для него сервис с типом NodePort, правило на Firewall и проверить его доступность в браузере. Можно сделать одним из двух способов:
- развернуть Deployment с помощью Dashboard Kubernetes, указав нужный образ;
- написать манифесты с Pod или Deployment и Service и применить их с помощью kubectl или Dashboard Kubernetes.

## Topic 3: Explore Kubernetes Cluster

Изучим  секреты, созданные в пространстве имён "default":
```bash
kubectl get secrets
```

Создадим тестовое пространство имён:
```bash
kubectl create ns my-ns
```

Запустим в тестовом пространстве имён модуль с proxy к API Kubernetes:
```bash
kubectl run test --image=chadmcrowell/kubectl-proxy -n my-ns
```

Проверим, что модуль с proxy к API Kubernetes в тестовом пространстве имён успешно запущен:
```bash
kubectl get pods -n my-ns
```

Откроем терминал модуля с proxy к API Kubernetes в тестовом пространстве имён:
```bash
kubectl exec -it $(kubectl get pods -n my-ns | grep test | awk '{print $1}') -n my-ns sh
```

Проверим, что API Kubernetes доступно на localhost:
```bash
curl localhost:8001/api/v1/namespaces/my-ns/services
```

Изучим содержимое токена, смонтированного в контейнер модуля с proxy к API Kubernetes:
```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
exit
```

Изучим созданные сервисные аккаунты:
```bash
kubectl get serviceaccounts -n my-ns
```

Удалим созданное пространство имён:
```bash
kubectl delete ns my-ns
```

Создадим тестовый модуль для проверки работы кластера:
```bash
kubectl run nginx --image=nginx
```

Проверим, что тестовый модуль успешно запущен:
```bash
kubectl get pods
```

Перенаправим порт 80 тестового модуля на локальный порт 8082:
```bash
kubectl port-forward $(kubectl get pods | grep nginx | awk '{print $1}') 8082:80
```

Откроем ещё одну вкладку терминала и проверим, что тестовый модуль доступен на локальном порту 8082:
```bash
curl --head http://127.0.0.1:8082
```

Вернёмся на предыдущую вкладку и изучим логи тестового модуля:
```bash
kubectl logs $(kubectl get pods | grep nginx | awk '{print $1}')
```

Проверим, что мы можем подключаться к тестовому модулю и выполнять внутри него команды:
```bash
kubectl exec -it $(kubectl get po | grep nginx | awk '{print $1}') -- nginx -v
```

Создадим сервис для перенаправления трафика с одного из портов узлов кластера на тестовый модуль:
```bash
kubectl expose pod nginx --port 80 --type NodePort
```

Изучим созданный сервис:
```bash
kubectl get services nginx -o json
```

Добавляем правило в фаервол Google Cloud Platform для предоставления доступа к сервису:
```bash
gcloud compute firewall-rules create svc-rule --allow=tcp:$(kubectl get services nginx -o jsonpath='{.spec.ports[].nodePort}')
```

Проверяем доступность приложений на порте узла:
```bash
curl http://$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{print $1}'):$(kubectl get services nginx -o jsonpath='{.spec.ports[].nodePort}')
```

Проверим доступность Nginx из браузера. Для этого сгенерируем ссылку на его стартовую страницу:
```bash
echo "Nginx external link: http://$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{print $1}'):$(kubectl get services nginx -o jsonpath='{.spec.ports[].nodePort}')"
```

Удаляем созданное правило в фаерволе Google Cloud Platform:
```bash
gcloud compute firewall-rules delete svc-rule
```

Проверим, что все узлы кластера готовы к работе:
```bash
kubectl get nodes
```

Изучим описание узлов:
```bash
kubectl describe nodes
```

Изучим описание созданных модулей:
```bash
kubectl describe pods
```

## Topic 4: Cluster Communications

Развернём модуль для тестирования коммуникаций внутри кластера:
```bash
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: radial/busyboxplus:curl
    args:
    - sleep
    - "1000"
EOF
```

Попробуем пропинговать созданный сервис по IP адресу:
```bash
kubectl exec busybox -- ping -w 5 $(kubectl get services nginx -o jsonpath="{.spec.clusterIP}")
```

Изучим доступные сервисы:
```bash
kubectl get services
```

Изучим доступные конечные точки:
```bash
kubectl get endpoints
```

Заходим по ssh на узел, на котором поднят pod Nginx:
```bash
gcloud compute ssh $(kubectl get nodes -o wide | grep $(kubectl get po nginx -o jsonpath="{.status.hostIP}") | awk '{print $1}')
Enter passphrase
```

Найдём правила iptables, созданные для работы сервисов "nginx*":
```bash
sudo iptables-save | grep KUBE | grep nginx
exit
```

Изучим доступные сервисы:
```bash
kubectl get services
```

Создадим описание сервиса, предоставляющего доступ к тестовому модудю через внешний балансировщик нагрузки:
```bash
vi nginx-loadbalancer.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    run: nginx
```

Создадим сервис из описания:
```bash
kubectl create -f nginx-loadbalancer.yaml
```

Проверим, что сервису с типом LoadBalancer создан назначен внешний IP адрес:
```bash
kubectl get services -f
```

Проверим доступность тестового модуля через балансировщик нагрузки:
```bash
curl http://$(kubectl get services nginx-loadbalancer -o jsonpath="{.status.loadBalancer.ingress[].ip}")
```

Откроем внешний IP адрес сервиса с типом LoadBalancer в браузере и проверим доступность Nginx извне.

Создадим ещё один тестовый модуль:
```bash
kubectl create deployment test --image=chadmcrowell/kubeserve2
```

Изучим созданные конфигурации развёртывания:
```bash
kubectl get deployment
```

Смасштабируем количество реплик созданного тестовго модуля до 2:
```bash
kubectl scale deployment/test --replicas=2
```

Изучим распределение тестовых модулей по узлам кластера:
```bash
kubectl get pods -o wide
```

Создадим сервис, предоставляющего доступ к новому тестовому модудю через внешний балансировщик нагрузки:
```bash
kubectl expose deployment test --port 80 --target-port 8080 --type LoadBalancer
```

Проверим, что сервис с типом LoadBalancer создан и доступен по отдельному IP адресу:
```bash
kubectl get services -f
```

Проверим доступность нового тестового модуля через балансировщик нагрузки и распределение запросов по экземплярам:
```bash
curl http://$(kubectl get services test -o jsonpath="{.status.loadBalancer.ingress[].ip}")
curl http://$(kubectl get services test -o jsonpath="{.status.loadBalancer.ingress[].ip}")
```

Изучим конфигурацию созданного сервиса более подробно:
```bash
kubectl get services test -o yaml
```

Изучим аннотацию созданного сервиса:
```bash
kubectl describe services test
```

Добавим аннотацию для созданного сервиса чтобы обеспечить вызов локальных экземпляров модулей в случае если запрос пришёл на один из узлов, на которых размещены его экземпляры:
```bash
kubectl annotate service test externalTrafficPolicy=Local
```

Создадим Ingress Controller:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```

Создадим сервис для Ingress Controller'а:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
```

Проверим, что Ingress Controller успешно запущен:
```bash
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
```

Создадим описание правил распределения трафика для Ingress Controller'а:
```bash
vi service-ingress.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: service-ingress
spec:
  rules:
  - host: test.example.com
    http:
      paths:
      - backend:
          serviceName: test
          servicePort: 80
  - host: nginx.example.com
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
```

Создадим правила распределения трафика для Ingress Controller'а: из описания:
```bash
kubectl apply -f service-ingress.yaml
```

Изучим созданные правила распределения трафика для Ingress Controller'а:
```bash
kubectl describe ingress
```

Протестируем перенаправление трафика с помощью Ingress Controller'а в зависимости от указания хоста:
```bash
curl --header "Host: test.example.com" http://34.120.24.46
curl --header "Host: nginx.example.com" http://34.120.24.46
```

Изучим модули управления кластером:
```bash
kubectl get pods -n kube-system
```

Изучим конфигурации развёртывания модулей управления кластером:
```bash
kubectl get deployments -n kube-system
```

Изучим сервисы модулей управления кластером:
```bash
kubectl get services -n kube-system
```

Изучим конфигурацию DNS тестового модуля busybox:
```bash
kubectl exec -it busybox -- cat /etc/resolv.conf
```

Выведем информацию о DNS записях хоста с именем kubernetes:
```bash
kubectl exec -it busybox -- nslookup kubernetes
```

Выведем информацию о DNS записях хоста с именем тествого модуля по умолчанию (надо заменить точки на тире!!!):
```bash
kubectl exec -ti busybox -- nslookup $(kubectl get po busybox -o jsonpath='{.status.podIP}' | sed 's/\./-/g').default.pod.cluster.local
```

Выведем информацию о DNS записях хоста с именем сервиса kube-dns:
```bash
kubectl exec -it busybox -- nslookup kube-dns.kube-system.svc.cluster.local
```

Изучим логи модуля DNS сервера:
```bash
kubectl logs $(kubectl get po -n kube-system | grep kube-dns | head -1 | awk '{print $1}') -c kubedns -n kube-system
```

Создадим описание сервиса без кластерного IP адреса для тестового модуля:
```bash
vi kube-headless.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-headless
spec:
  clusterIP: None
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubserve2
```

Создадим сервис без кластерного IP адреса для тестового модуля из описания:
```bash
kubectl create -f kube-headless.yaml
```

Изучим созданный сервис:
```bash
kubectl get svc kube-headless -o yaml
```

Удалим созданные артефакты:
```bash
kubectl delete deployments test
kubectl delete svc nginx nginx-loadbalancer test kube-headless
kubectl delete po busybox nginx test
kubectl delete ingress service-ingress
```

#### Задание:
Создать ingress для тестового приложения.

## Topic 5: Pod Scheduling within the Kubernetes Cluster

Пометим первый узел меткой "availability-zone=zone1":
```bash
kubectl label node $(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}') availability-zone=zone1
```

Пометим второй узел меткой "share-type=dedicated":
```bash
kubectl label node $(kubectl get nodes | grep -v NAME | sed -n 2p | awk '{print $1}') share-type=dedicated
```

Создадим описание конфигурации развёртывания тестового модуля с 5 репликами и заданными правилами распределения по узлам:
```bash
vi pref-deployment.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: pref
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: pref
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: availability-zone
                operator: In
                values:
                - zone1
          - weight: 20
            preference:
              matchExpressions:
              - key: share-type
                operator: In
                values:
                - dedicated
      containers:
      - args:
        - sleep
        - "99999"
        image: busybox
        name: main
```

Создадим конфигурацию развёртывания тестового модуля с 5 репликами и заданными правилами распределения по узлам:
```bash
kubectl create -f pref-deployment.yaml
```

Изучим созданную конфигурацию развёртывания:
```bash
kubectl get deployments
```

Изучим распределение созданных реплик тестового модуля:
```bash
kubectl get pods -o wide
```

Удаляем конфигурацию развёртывания нового тестового модуля:
```bash
kubectl delete -f pref-deployment.yaml
```

Изучим доступные на узлах ресурсы:
```bash
kubectl describe nodes
```

Создадим тестовый модуль с указанием запроса ресурсов и узла::
```bash
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod1
spec:
  nodeSelector:
    kubernetes.io/hostname: "$(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}')"
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: pod1
    resources:
      requests:
        cpu: 500m
        memory: 20Mi
EOF
```

Изучим созданный модуль:
```bash
kubectl get pods -o wide
```

Создадим тестовый модуль с указанием узла и запроса ресурсов, превышающего доступные ресурсы на узле:
```bash
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod2
spec:
  nodeSelector:
    kubernetes.io/hostname: "$(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}')"
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: pod2
    resources:
      requests:
        cpu: 400m
        memory: 20Mi
EOF
```

Изучим созданный модуль:
```bash
kubectl get pods -o wide
```

Изучим описание созданного модуля:
```bash
kubectl describe pods resource-pod2
```

Изучим описание узла, на котором мы пытаемся развернуть новый тестовый модуль:
```bash
kubectl describe nodes $(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}')
```

Удалим первый созданный тестовый модуль:
```bash
kubectl delete pods resource-pod1
```

Проверим, что новый тестовый модуль успешно запустился:
```bash
kubectl get pods -o wide -w
```

Создадим описание тестового модуля с указанием узла и лимита выделения ресурсов:
```bash
vi limited-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      limits:
        cpu: 400m
        memory: 20Mi
```

Создадим тестовый модуль с указанием узла и лимита выделения ресурсов:
```bash
kubectl create -f limited-pod.yaml
```

Проверим, что новый тестовый модуль успешно запустился:
```bash
kubectl get pods -o wide -w
```

Изучим текущую утилизацию ресурсов нового тестовго модуля:
```bash
kubectl exec -it limited-pod top
```

Изучим текущую утилизацию ресурсов нового тестовго модуля:
```bash
kubectl delete pods limited-pod resource-pod2
```

Изучим созданные модули в пространстве имён "kube-system":
```bash
kubectl get pods -n kube-system -o wide
```

Изучим удалим один из модулей "kube-proxy-*":
```bash
kubectl delete pods $(kubectl get pods -n kube-system | grep kube-proxy | head -1 | awk '{print $1}') -n kube-system
```

Проверим, что был создан новый модуль "kube-proxy-*":
```bash
kubectl get pods -n kube-system -o wide
```

Пометим первый узел меткой "disk=ssd":
```bash
kubectl label node $(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}') disk=ssd
```

Создадим описание набора демонов, который работает на узлах с меткой "disk: ssd":
```bash
vi ssd-monitor.yaml
```

```yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: main
        image: linuxacademycontent/ssd-monitor
```

Создадим набор демонов, который работает на узлах с меткой "disk: ssd":
```bash
kubectl create -f ssd-monitor.yaml
```

Проверим, что на первом узле был создан новый модуль "ssd-monitor-*":
```bash
kubectl get pods -o wide
```

Пометим второй узел меткой "disk=ssd":
```bash
kubectl label node $(kubectl get nodes | grep -v NAME | sed -n 2p | awk '{print $1}') disk=ssd
```

Проверим, что на втором узеле был создан новый модуль "ssd-monitor-*":
```bash
kubectl get pods -o wide
```

Удалим метку "disk=ssd" со второго узла:
```bash
kubectl label node $(kubectl get nodes | grep -v NAME | sed -n 2p | awk '{print $1}') disk-
```

Проверим, что новый модуль "ssd-monitor-*" удалён со второго узла:
```bash
kubectl get pods -o wide
```

Изменим метку первого узла на "disk=hdd":
```bash
kubectl label node $(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}') disk=hdd --overwrite
```

Изучим метки первого узла:
```bash
kubectl get nodes $(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}') --show-labels
```

Проверим, что все новые модули "ssd-monitor-*" удалены:
```bash
kubectl get pods -o wide
```

Удалим набор демонов "ssd-monitor":
```bash
kubectl delete daemonset ssd-monitor
```

Снова создадим несколько тестовых модулей для демонстрации событий планировщика:
```bash
kubectl create -f busybox.yaml
kubectl create -f resource-pod1.yaml
```

Изучим события в пространстве имён "default":
```bash
kubectl get events
```

Удалим все созданные модули в пространстве имён "default":
```bash
kubectl delete pods --all
```

Удалим все созданные модули в пространстве имён "default":
```bash
kubectl get events -w
```

#### Задание:
Для тестового приложения задать лимиты и реквесты.

## Topic 6: Deploying Applications in the Kubernetes Cluster

Создадим описание конфигурации развёртывания тестового модуля:
```bash
vi kubeserve-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeserve
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubeserve
  template:
    metadata:
      name: kubeserve
      labels:
        app: kubeserve
    spec:
      containers:
      - image: linuxacademycontent/kubeserve:v1
        name: app
```

Создадим конфигурацию развёртывания тестового модуля:
```bash
kubectl create -f kubeserve-deployment.yaml --record
```

Изучим статус развёртывания созданной конфигурации:
```bash
kubectl rollout status deployments kubeserve
```

Изучим созданный набор реплик:
```bash
kubectl get replicasets
```

Смасштабируем количество реплик тестового модуля до 5:
```bash
kubectl scale deployment kubeserve --replicas=5
```

Изучим созданные модули:
```bash
kubectl get pods
```

Создадим сервис для предоставления доступа к репликам конфигурации развёртывания на одном из портов рабочих узлов:
```bash
kubectl expose deployment kubeserve --port 80 --target-port 80 --type NodePort
```

Установим количество секунд, после прохождения которых развёрнутый модуль считается готовым к эксплуатации:
```bash
kubectl patch deployment kubeserve -p '{"spec": {"minReadySeconds": 10}}'
```

Изменим версию образа на "linuxacademycontent/kubeserve:v2":
```bash
vi kubeserve-deployment.yaml
```

```yaml
...
      containers:
      - image: linuxacademycontent/kubeserve:v2
        name: app
```

Обновим конфигурацию развёртывания тестового модуля:
```bash
kubectl apply -f kubeserve-deployment.yaml
```

Изменим обратно версию образа на "linuxacademycontent/kubeserve:v1":
```bash
vi kubeserve-deployment.yaml
```

```yaml
...
      containers:
      - image: linuxacademycontent/kubeserve:v1
        name: app
```

Заменим конфигурацию развёртывания тестового модуля:
```bash
kubectl replace -f kubeserve-deployment.yaml
```

Определим порт узла, на котором предоставлен доступ к репликам тестового модуля:
```bash
kubectl get svc kubeserve -o jsonpath='{.spec.ports[0].nodePort}'
```

Добавляем правило в фаервол Google Cloud Platform для предоставления доступа к сервису:
```bash
gcloud compute firewall-rules create svc-rule --allow=tcp:$(kubectl get svc kubeserve -o jsonpath='{.spec.ports[0].nodePort}')
```

### Следующая команда выполняется в отдельном терминале

Запустим цикл вызова тестового модуля по HTTP:
```bash
while true; do curl http://$(kubectl get nodes $(kubectl get nodes | grep -v NAME | sed -n 1p | awk '{print $1}') -o jsonpath='{.status.addresses[?(@.type=="ExternalIP")].address}'):$(kubectl get svc kubeserve -o jsonpath='{.spec.ports[0].nodePort}'); done
```

Обновим образ в конфигурации развёртывания тестового модуля до версии "linuxacademycontent/kubeserve:v2" с указанием номера ревизии:
```bash
kubectl set image deployments/kubeserve app=linuxacademycontent/kubeserve:v2 --v 6
```

Изучим описание активного набора реплик:
```bash
kubectl describe replicasets $(kubectl get rs | grep -v NAME | grep -v 0 | awk '{print $1}')
```

Обновим образ в конфигурации развёртывания тестового модуля до версии "linuxacademycontent/kubeserve:v3", содержащей дефект:
```bash
kubectl set image deployment kubeserve app=linuxacademycontent/kubeserve:v3
```

Откатим изменения конфигурации развёртывания:
```bash
kubectl rollout undo deployments kubeserve
```

Изучим историю развёртываний тестового модуля:
```bash
kubectl rollout history deployment kubeserve
```

Откатим конфигурацию развёртывания до первоначальной версии:
```bash
kubectl rollout undo deployment kubeserve --to-revision=3
```

Остановим развёртывание для тестирования работоспособности новой версии:
```bash
kubectl rollout pause deployment kubeserve
```

Продолжим развёртывание после завершения тестирования:
```bash
kubectl rollout resume deployment kubeserve
```

Создадим описание конфигурации развёртывания тестового модуля с проверкой живости:
```bash
vi kubeserve-deployment-readiness.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeserve
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubeserve
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      name: kubeserve
      labels:
        app: kubeserve
    spec:
      containers:
      - image: linuxacademycontent/kubeserve:v3
        name: app
        readinessProbe:
          periodSeconds: 1
          httpGet:
            path: /
            port: 80
```

Создадим конфигурацию развёртывания тестового модуля:
```bash
kubectl apply -f kubeserve-deployment-readiness.yaml
```

Изучим статус развёртывания новой версии тестового модуля тестового модуля:
```bash
kubectl rollout status deployment kubeserve
```

Изучим описание конфигурацию развёртывания тестового модуля:
```bash
kubectl describe deployment
```

Изучим описание неактивной реплики тестового модуля:
```bash
kubectl describe pod $(kubectl get pods | grep 0/1 | awk '{print $1}')
```

Удаляем созданное правило в фаерволе Google Cloud Platform:
```bash
gcloud compute firewall-rules delete svc-rule
```

Создадим словарь конфигурации с двумя записями:
```bash
kubectl create configmap appconfig --from-literal=key1=value1 --from-literal=key2=value2
```

Изучим описание созданного словаря конфигурации:
```bash
kubectl get configmap appconfig -o yaml
```

Создадим описание тестового модуля, использующего словарь конфигурации для определения переменной окружения:
```bash
vi configmap-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: app-container
    image: busybox:1.28
    command: ['sh', '-c', "echo $(MY_VAR) && sleep 3600"]
    env:
    - name: MY_VAR
      valueFrom:
        configMapKeyRef:
          name: appconfig
          key: key1
```

Создадим тестовый модуль, использующий словарь конфигурации для определения переменной окружения:
```bash
kubectl apply -f configmap-pod.yaml
```

Изучим логи созданного тестового модуля:
```bash
kubectl logs configmap-pod
```

Создадим описание тестового модуля, использующего словарь конфигурации в качестве постоянного тома:
```bash
vi configmap-volume-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: app-container
    image: busybox
    command: ['sh', '-c', "echo $(MY_VAR) && sleep 3600"]
    volumeMounts:
      - name: configmapvolume
        mountPath: /etc/config
  volumes:
    - name: configmapvolume
      configMap:
        name: appconfig
```

Создадим тестовый модуль, использующий словарь конфигурации в качестве постоянного тома:
```bash
kubectl apply -f configmap-volume-pod.yaml
```

Изучим директорию тестового модуля, в которую был смонтирован словарь конфигурации в качестве постоянного тома:
```bash
kubectl exec configmap-volume-pod -- ls /etc/config
```

Изучим один из файлов тестового модуля, смонтированный из словаря конфигурации:
```bash
kubectl exec configmap-volume-pod -- cat /etc/config/key1
```

Создадим описание секрета с двумя объектами типа "stringData":
```bash
vi appsecret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: appsecret
stringData:
  cert: value
  key: value
```

Создадим секрет с двумя объектами типа "stringData":
```bash
kubectl apply -f appsecret.yaml
```

Создадим описание тестового модуля, использующего одно из значений секрета для определения переменной окружения:
```bash
vi secret-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app-container
    image: busybox
    command: ['sh', '-c', "echo Hello, Kubernetes! && sleep 3600"]
    env:
    - name: MY_CERT
      valueFrom:
        secretKeyRef:
          name: appsecret
          key: cert
```

Создадим тестовый модуль, использующий одно из значений секрета для определения переменной окружения:
```bash
kubectl apply -f secret-pod.yaml
```

Откроем сессию терминала внутри тестового модуля, использующего одно из значений секрета для определения переменной окружения:
```bash
kubectl exec -it secret-pod -- sh
```

Выведем значение заданной переменной окружения:
```bash
echo $MY_CERT
```

Создадим описание тестового модуля, использующего секрет в качестве постоянного тома:
```bash
vi secret-volume-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app-container
    image: busybox
    command: ['sh', '-c', "echo $(MY_VAR) && sleep 3600"]
    volumeMounts:
      - name: secretvolume
        mountPath: /etc/certs
  volumes:
    - name: secretvolume
      secret:
        secretName: appsecret
```

Создадим тестовый модуль, использующий секрет в качестве постоянного тома:
```bash
kubectl apply -f secret-volume-pod.yaml
```

Изучим директорию тестового модуля, в которую был смонтирован секрет в качестве постоянного тома:
```bash
kubectl exec secret-volume-pod -- ls /etc/certs
```

Удалим конфигурацию развёртывания и сервис тестового модуля, а также отдельно созданные тестовые модули:
```bash
kubectl delete pods --all
kubectl delete deployment kubeserve
kubectl delete svc kubeserve
```

Создадим описание набора реплик с 3 репликами тестового модуля:
```bash
vi replicaset.yaml
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myreplicaset
  labels:
    app: app
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: main
        image: linuxacademycontent/kubeserve
```

Создадим набор реплик с 3 репликами тестового модуля:
```bash
kubectl apply -f replicaset.yaml
```

Изучим созданный набор реплик:
```bash
kubectl get replicasets
```

Изучим созданные тестовые модули:
```bash
kubectl get pods
```

Создадим описание тестового модуля, содержащего метку, аналогичную набору реплик:
```bash
vi pod-replica.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    tier: frontend
spec:
  containers:
  - name: main
    image: linuxacademycontent/kubeserve
```

Создадим тестовый модуль, содержащий метку, аналогичную набору реплик:
```bash
kubectl apply -f pod-replica.yaml
```

Проверим, что 4-й тестовый модуль, содержащий метку, аналогичную набору реплик, удалён:
```bash
kubectl get pods -w
```

Удалим созданный набор реплик тестовых модулей:
```bash
kubectl delete replicaset myreplicaset
```

#### Задание:
Создать секрет со строкой подключения к БД и передать его в виде переменной окружения в тестовое приложение.

## Topic 7: Managing Data in the Kubernetes Cluster

Создаём диск в облаке для размещения постоянного тома:
```bash
gcloud container clusters list
gcloud compute disks create --size=1GiB --zone=$(gcloud container clusters list | grep -v NAME | awk '{print $2}') mongodb
```

Создадим yaml описание для создания постоянного тома:
```bash
vi pv-01.yml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv00001
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  gcePersistentDisk:
    pdName: mongodb
    fsType: ext4
```

Создадим постоянный том из описанного шаблона:
```bash
kubectl create -f pv-01.yml
```

Изучим созданный постоянный том:
```bash
kubectl get pv
```

Удалим постоянный том:
```bash
kubectl delete pv pv00001
```

Создадим описание тестового модуля с использованием постоянного тома сервера:
```bash
vi mongodb-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  volumes:
  - name: mongodb-data
    gcePersistentDisk:
      pdName: mongodb
      fsType: ext4
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
      readOnly: false
    ports:
    - containerPort: 27017
      protocol: TCP
```

Создадим тестовый модуль с использованием постоянного тома NFS сервера:
```bash
kubectl apply -f mongodb-pod.yaml
```

Изучим созданный тестовый модуль и определим узел, на котором он размещён:
```bash
kubectl get pods -o wide
```

Сохраняем данные в созданном тестовом модуле:
```bash
kubectl exec -it mongodb mongo
> use mystore
> db.foo.insert({name: 'foo'})
> db.foo.find()
> exit
```

Удаляем созданный тестовый модуль:
```bash
kubectl delete pod mongodb
```

Создадим тестовый модуль с использованием постоянного тома сервера заново:
```bash
kubectl apply -f mongodb-pod.yaml
```

Проверяем, что тестовый модуль размещён на другом узле (отличном от узла, на котором он был размещён в первый раз):
```bash
kubectl get pods -o wide
```

### (!!!) Следующая команда выполняется только в случае если узлы при размещении тестового модуля в первый и второй раз совпали

Перемещаем тестовый модуль на другой узел:
```bash
kubectl drain $(kubectl get pods mongodb -o jsonpath='{.spec.nodeName}') --ignore-daemonsets --force
kubectl apply -f mongodb-pod.yaml
kubectl uncordon $(kubectl get nodes | grep Disabled | awk '{print $1}')
```

Проверяем, что данные, сохранённые в созданном тестовом модуле, не утрачены после его удаления и пересоздания:
```bash
kubectl exec -it mongodb mongo
> use mystore
> db.foo.find()
> exit
```

Удалим тестовый модуль:
```bash
kubectl delete -f mongodb-pod.yaml
```

Создадим yaml описание для создания постоянного тома для MongoDB:
```bash
vi mongodb-persistentvolume.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity: 
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    pdName: mongodb
    fsType: ext4
```

Создадим постоянный том  для MongoDB из описанного шаблона:
```bash
kubectl create -f mongodb-persistentvolume.yaml
```

Изучим созданный постоянный том:
```bash
kubectl get persistentvolumes
```

Создадим yaml описание для создания запроса на постоянный том для MongoDB:
```bash
vi mongodb-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc 
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

Создадим запрос на постоянный том для MongoDB из описанного шаблона:
```bash
kubectl create -f mongodb-pvc.yaml
```

Изучим созданный запрос на постоянный том:
```bash
kubectl get pvc
```

Проверим, что постоянный том mongodb-pv привязан к созданному запросу на постоянный том:
```bash
kubectl get pv
```

Создадим yaml описание для создания тестового модуля, использующего запрос на постоянный том для MongoDB:
```bash
vi mongo-pvc-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

Создадим тестовый модуль, использующий запрос на постоянный том для MongoDB:
```bash
kubectl create -f mongo-pvc-pod.yaml
```

Проверяем, что данные, сохранённые в постоянном томе, также доступны при использовании запроса на постоянный том:
```bash
kubectl exec -it mongodb mongo
> use mystore
> db.foo.find()
> exit
```

Изучим созданный постоянный том:
```bash
kubectl describe pv mongodb-pv
```

Изучим созданный запрос на постоянный том:
```bash
kubectl describe pvc mongodb-pvc
```

Удалим созданный запрос на постоянный том:
```bash
kubectl delete pvc mongodb-pvc
```

Изучим текущий статус удалённого запроса на постоянный том:
```bash
kubectl get pvc
```

Проверяем, что данные, сохранённые в постоянном томе, также доступны после удаления запроса на постоянный том:
```bash
kubectl exec -it mongodb mongo
> use mystore
> db.foo.find()
> exit
```

Удалим созданный тестовый модуль:
```bash
kubectl delete pod mongodb
```

Проверим, что запрос на постоянный том окончательно удалён:
```bash
kubectl get pvc
```

Проверим, что статус постоянного тома изменён:
```bash
kubectl get pv
```

Удалим созданный запрос на постоянный том:
```bash
kubectl delete pv mongodb-pv
```

Создадим описание набора тестовых модулей из двух реплик, сохраняющих состояние:
```bash
vi statefulset.yaml
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

Создадим набор тестовых модулей из двух реплик, сохраняющих состояние:
```bash
kubectl apply -f statefulset.yaml
```

Изучим созданный набор тестовых модулей из двух реплик, сохраняющих состояние:
```bash
kubectl get statefulsets
```

Изучим описание созданного набора тестовых модулей из двух реплик, сохраняющих состояние:
```bash
kubectl describe statefulsets
```

Проверим, что реплики создались:
```bash
kubectl get po
```

Изучим созданные запросы на постоянный том:
```bash
kubectl get pvc
```

Удалим набор тестовых модулей из двух реплик, сохраняющих состояние и его запросы на постоянный том:
```bash
kubectl delete statefulsets web
kubectl delete pvc www-web-1 www-web-0
```

Удалим созданный постоянный диск:
```bash
gcloud compute disks delete --zone=$(gcloud container clusters list | grep -v NAME | awk '{print $2}') mongodb
```

#### Задание:
Развернуть Redis в виде StatefulSet с томом в директории, в которой хранятся его данные. Сохранить данные в Redis, удалить под Redis и проверить, что после запуска нового пода данные вновь доступны.

## Topic 8: Securing the Kubernetes Cluster

Изучим имеющиеся сервисные аккаунты пространства имён "default":
```bash
kubectl get serviceaccounts
```

Создадим в пространстве имён "default" сервисный аккаунт "gitlab":
```bash
kubectl create serviceaccount gitlab
```

Изучим созданный сервисный аккаунт:
```bash
kubectl get sa
```

Изучим YAML описание созданного сервисного аккаунта:
```bash
kubectl get serviceaccounts gitlab -o yaml
```

Изучим секретный токен, автоматически привязанный к созданному сервисному аккаунту:
```bash
kubectl get secret $(kubectl get serviceaccounts gitlab -o jsonpath='{.secrets[].name}')
```

Создадим yaml описание для создания тестового модуля, запускаемого под сервисным аккаунтом "gitlab":
```bash
vi busybox-sa.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  serviceAccountName: gitlab
  containers:
  - image: busybox:1.28.4
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

Создадим тестовый модуль, запускаемый под сервисным аккаунтом "jenkins":
```bash
kubectl create -f busybox-sa.yaml
```

Проверим, что тестовый модуль успешно создан:
```bash
kubectl get po
```

Проверим, что тестовый модуль успешно создан:
```bash
kubectl get po busybox -o yaml
```

Изучим конфигурацию kubectl:
```bash
kubectl config view
```

Выведем конфигурацию kubectl напрямую из конфигурационного файла:
```bash
cat ~/.kube/config
```

Создаём файл, содержащий certificate-authority-data нашего кластера Kubernetes:
```bash
cat ~/.kube/config | grep certificate-authority-data | awk '{print $2}' | base64 -d > ca.cert
```

Временно дадим права администратора кластера анонимным пользователям:
```bash
kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous
```

Добавим нового пользователя в конфигурацию kubectl:
```bash
kubectl config set-credentials testuser --username=testuser --password=password
```

Зададим настройки кластера для конфигурации kubectl:
```bash
kubectl config set-cluster kubernetes --server=$(cat ~/.kube/config | grep server | awk '{print $2}') --certificate-authority=ca.cert --embed-certs=true
```

Зададим контекст для конфигурации kubectl:
```bash
kubectl config set-context kubernetes --cluster=kubernetes --user=testuser --namespace=default
```

Используем заданный контекст для конфигурации kubectl:
```bash
kubectl config use-context kubernetes
```

Проверим, что kubectl сконфигурирован корректно:
```bash
kubectl get nodes
```

Удалим ранее созданный тестовый модуль:
```bash
kubectl delete po busybox
```

Удалим ранее созданную привязку кластерной роли:
```bash
kubectl delete clusterrolebinding cluster-system-anonymous
```

Проверим, что доступ анонимным пользователям заблокирован:
```bash
kubectl get nodes
```

Используем основной контекст для конфигурации kubectl:
```bash
kubectl config use-context $(cat ~/.kube/config | grep name | grep gke | head -1 | awk '{print $2}')
```

Создадим новое пространство имён:
```bash
kubectl create ns web
```

Создадим yaml описание для роли в пространстве имён "web", позволяющей просматривать список сервисов данного пространства имён:
```bash
vi role.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: web
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["services"]
```

Создадим роль в пространстве имён "web", позволяющую просматривать список сервисов данного пространства имён:
```bash
kubectl create -f role.yaml
```

Создадим привязку сервисного аккаунта "web:default" к созданной роли "service-reader":
```bash
kubectl create rolebinding test --role=service-reader --serviceaccount=web:default -n web
```

Запустим локальный прокси сервер Kubernetes API для проверки доступности вывода списка сервисов в пространсте имён "web" (доступно без создания роли и привязки ролей):
```bash
kubectl proxy
```

#### Следующая команда выполняется в отдельной вкладке Cloud Shell

Запросим список сервисов пространства имён "web":
```bash
curl localhost:8001/api/v1/namespaces/web/services
```

Создадим кластерную роль для просмотра списка постоянных томов в кластере:
```bash
kubectl create clusterrole persistent-volume-reader --verb=get,list --resource=persistentvolumes
```

Создадим привязку сервисного аккаунта "web:default" к кластерной роли "persistent-volume-reader" для просмотра списка постоянных томов в кластере:
```bash
kubectl create clusterrolebinding pv-test --clusterrole=persistent-volume-reader --serviceaccount=web:default
```

Создадим yaml описание для тестового модуля в пространстве имён "web", имеющего возможность обращаться к API Kubernetes:
```bash
vi curl-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curlpod
  namespace: web
spec:
  containers:
  - image: tutum/curl
    command: ["sleep", "9999999"]
    name: main
  - image: linuxacademycontent/kubectl-proxy
    name: proxy
  restartPolicy: Always
```

Создадим тестовый модуль в пространстве имён "web", имеющий возможность обращаться к API Kubernetes:
```bash
kubectl apply -f curl-pod.yaml
```

Изучим созданный тестовый модуль:
```bash
kubectl get pods -n web
```

Создадим SSH сессию к тестовому модулю:
```bash
kubectl exec -it curlpod -n web -- sh
```

Отправим запрос к API Kubernetes для вывода списка постоянных томов кластера:
```bash
curl localhost:8001/api/v1/persistentvolumes
```

Удалим созданное пространство имён:
```bash
kubectl delete ns web
```

#### Задание:
Создать отдельные Service Account для redis и тестового приложения и добавить в их Deployment и StatefulSet использование данных аккаунтов вместо аккаунтов по умолчанию.

## Topic 9: GitLab Integration

Создадим сервисного пользователя для доступа к реестру образов Docker GitLab из Kubernetes. Для этого переходим в Web интерфейс GitLab с административными правами и нажимаем на кнопку "Admin Area", для создания пользователя нажимаем на кнопку "New User", в поле "Name" вписываем полное имя нового пользователя "Kubernetes", в поле "Username" вписываем имя нового пользователя в рамках Gitlab "kubernetes", в поле "Email" вписываем вымышленный почтовый ящик нового пользователя "k8s@gitlab.local" и нажимаем на кнопку "Create user".

Требуется задать пароль вручную. Для этого нажимаем на кнопку "Edit" и в полях "Password" и "Password confirmation" вписываем временный пароль "Q!W@e3r4" и нажимаем на кнопку "Save changes". После этого копируем ссылку на GitLab и открываем её в другом браузере/вкладке в режиме инкогнито. Вводим логин "kubernetes" и пароль "Q!W@e3r4" и переходим на форму смены временного пароля. Вводим в поле "Current password" текущий пароль "Q!W@e3r4", в поля "New password" и "Password confirmation" вводим новый пароль "!QAZ2wsx" и нажимаем на кнопку "Set new password". После этого мы оказываемся на странице нового пользователя в GitLab (возможно придётся пройти процедуру авторизации заново).

В данный момент новому пользователю не даны права на работу с созданной нами группой репозиториев и реестром образов Docker. Добавим его в группу "cicd". Для этого вернёмся на вкладку GitLab под пользователем "root", нажмём на кнопку "Groups" и выберем пункт "Your groups". В списке групп выберем "cicd", перейдём на вкладку "Members", в поле "GitLab member or Email address" выберем пользователя "Kubernetes", в поле выбора роли пользователя "Choose a role permission" выберем "Developer" и нажмём кнопку "Invite". Вернёмся на вкладку GitLab под пользователем "kubernetes" и обновим страницу. В списке проектов появилась группа "cicd" и проекты "Test Project" и "Compose Test". Откроем проект "Test Project", перейдём в подраздел "Packages/Container Registry" и проверим, что просмотр созданных образов Docker доступен.

Проверим с помощью Cloud Shell доступность интегрированного в GitLab реестра образов Docker от созданного пользователя (kubernetes/!QAZ2wsx):
```bash
docker login <gitlab-instance-ip-address>.xip.io:5050
```

Создадим секрет с параметрами аутентификации для загрузки образов из реестра GitLab:
```bash
kubectl create secret docker-registry gitlab --docker-server=https://<gitlab-instance-ip-address>.xip.io:5050 --docker-username=kubernetes --docker-password='!QAZ2wsx' --docker-email=k8s@gitlab.local
```

Создадим описание тестового модуля, использующего образ, зугруженный в реестр образов Docker:
```bash
cat << EOF | tee test-project-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-project-pod
  labels:
    app: test-project
spec:
  containers:
    - name: main
      image: 34.71.172.16.xip.io:5050/cicd/test-project:10
      imagePullPolicy: Always
      ports:
      - containerPort: 80
  imagePullSecrets:
  - name: gitlab
EOF
```

Создадим тестовый модуль, использующий образ, зугруженный в реестр образов Docker:
```bash
kubectl apply -f test-project-pod.yaml
```

Проверим, что тестовый модуль успешно создан:
```bash
kubectl get pods
```

Создадим сервис для обеспечения доступности развёрнутого приложения внутри кластера:
```bash
kubectl expose pods test-project-pod
```

Проверим что сервис для тестового пода успешно создан:
```bash
kubectl get svc
```

Развернём модуль для отправки HTTP запросов внутри кластера:
```bash
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: radial/busyboxplus:curl
    args:
    - sleep
    - "1000"
EOF
```

Изучим список созданных модулей:
```bash
kubectl get pods -o wide
```

Проверим доступность развёрнутого приложения, вызвав из модуля "busybox" службу "test-project-pod" по HTTP:
```bash
kubectl exec busybox -- curl test-project-pod:80
```

Удалим созданные ресурсы:
```bash
kubectl delete po test-project-pod
```

Изучим имеющиеся секреты в проекте "default":
```bash
kubectl get secrets
```

Изучим тестовый модуль на наличие смонтированных секретов:
```bash
kubectl describe pods pod-with-defaults
```

Изучим секрет, смонтированный в тестовый модуль:
```bash
kubectl describe secret $(kubectl get secrets | grep default | awk '{print $1}')
```

Создадим ключ для генерации SSL cертификата:
```bash
openssl genrsa -out https.key 2048
```

Сгенерируем SSL cертификат:
```bash
openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.example.com
```

Создадим дополнительный пустой файл для создания секрета:
```bash
touch file
```

Создадим секрет на основе созданных файлов для монтирования в тестовый модуль с HTTP-сервером, поддерживающим SSL шифрование:
```bash
kubectl create secret generic example-https --from-file=https.key --from-file=https.cert --from-file=file
```

Изучим созданный секрет:
```bash
kubectl get secrets example-https -o yaml
```

Создадим описание словаря конфигурации с конфигурационным файлом Nginx для монтирования в тестовый модуль с HTTP-сервером, поддерживающим SSL шифрование:
```bash
vi my-nginx-config.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  my-nginx-config.conf: |
    server {
        listen              80;
        listen              443 ssl;
        server_name         www.example.com;
        ssl_certificate     certs/https.cert;
        ssl_certificate_key certs/https.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

    }
  sleep-interval: |
    25
```

Создадим словарь конфигурации с конфигурационным файлом Nginx для монтирования в тестовый модуль с HTTP-сервером, поддерживающим SSL шифрование:
```bash
kubectl apply -f my-nginx-config.yaml
```

Изучим созданный словарь конфигурации:
```bash
kubectl describe configmap config
```

Создадим описание тестового модуля с HTTP-сервером и приложением для динамической генерации текста для отображения:
```bash
vi example-https.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-https
spec:
  containers:
  - image: linuxacademycontent/fortune
    name: html-web
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: config
          key: sleep-interval
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: config
      items:
      - key: my-nginx-config.conf
        path: https.conf
  - name: certs
    secret:
      secretName: example-https
```

Создадим тестовый модуль с HTTP-сервером и приложением для динамической генерации текста для отображения:
```bash
kubectl apply -f example-https.yaml
```

Проверим, что тестовый модуль успешно создан:
```bash
kubectl get pods
```

Изучим файловые системы, смонтированные в тестовый модуль, и найдём среди них файловую систему с смонтированным секретом:
```bash
kubectl exec example-https -c web-server -- mount | grep certs
```

Осуществим перенаправление порта "443" тестового модуля на порт "8443" текущего узла:
```bash
kubectl port-forward example-https 8443:443 &
```

### Следующая команда выполняется в дополнительном окне терминала на узле master

Проверим доступность тестового модуля по протоколу HTTPS на порту "8443" текущего узла:
```bash
curl https://localhost:8443 -k
```

Удалим созданные тестовые модули:
```bash
kubectl delete pods example-https
```