# DmitryMCN_platform
DmitryMCN Platform repository
<details>
<summary>HomeWork №1</summary>

### Настройка локального окружения:
- Установка kubectl
- Запуск кластера minikube (docker драйвер)
- Установка kubernetes-dashboard и k9s
### Задание 1:
> Разберитесь почему все pod в namespace kube-system восстановились после удаления.
- Поды kube-scheduler, kube-controller-manager, etcd, apiserver это static-pods. Они управляются напрямую демоном kubelet на конкретной ноде (kubelet наблюдает за каждым статическим подом и перезапускает его в случае сбоя).
- Под core-dns-\* управляется с помощью ресурса ReplicaSet, цель которого поддерживать стабильный набор реплик пода в любой момент времени.
- Под kube-proxy-\* управляется с помощью ресурса DaemonSet, который гарантирует запуск пода на всех (или некоторых) нодах.
### Задание 2:
> Cоздание Dockerfile, сборка образа, запуск первого пода и т.д.
- Подготовлен Dockerfile, готовый образ запушен в публичный Container Registry (Docker Hub).
- Создан манифест первого пода, добавлен init-контейнер для генерации index.html. Используем kubectl port-forward для проброса и проверяем стартовую страницу.
### Как запустить проект:
- В директории kubernetes-intro выполнить:
<pre>
kubectl apply -f web-pod.yaml
kubectl port-forward --address 0.0.0.0 pod/web-app 8000:8000
</pre>
Для микросервиса frontend
<pre>
kubectl apply -f frontend-pod-healthy.yaml
</pre>
### Как проверить работоспособность:
<pre>kubectl port-forward --address 0.0.0.0 pod/web-app 8000:8000
</pre>
- Перейти по ссылке http://localhost:8000
### Задание со \*:
> Запуск микросервиса frontend, исправление ошибки при старте пода.
- Под frontend не запускается, в логах ошибка `'panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set'`. Добавляем необходимые переменные.
</details>

<details>
<summary>HomeWork №2</summary>

### Настройка локального окружения:
- Запуск кластера kind
### Что было сделано
- Учимся запускать поды с помощью контроллеров k8s. Попробовал в деле ReplicaSet, Deployment и DaemonSet.
Контроллеры используют селекторы по меткам (labels) для создания и управления подами. С помощью ReplicaSet можно быстро увеличить/уменьшить (scale) кол-во необходимых реплик пода.
Контроллер Deployment использует ReplicaSet для более гибкого управления подами. Например он позволяет управлять стратегией развертывания. Можно сделать откат в случае проблем (команда rollout undo).
DaemonSet запускает по одному экземпляру пода на каждой ноде.
- Подготовлен манифест для запуска node-exporter на всех нодах кластера.
> ReplicaSet. Определите, что необходимо добавить в манифест, исправьте его и примените вновь. 
- Не хватает селектора для меток (missing required field "selector" in io.k8s.api.apps.v1.ReplicaSetSpec) и переменных сервиса
> Почему обновление ReplicaSet не повлекло обновление запущенных pod?
- ReplicaSet поддерживает нужное кол-во подов, но не следит за обновлением их конфигурации.
> Найдите способ модернизировать свой DaemonSet таким образом, чтобы Node Exporter был развернут как на master, так и на worker нодах.
- Используем св-во tolerations: operator: "Exists"
### Как запустить проект:
- В директории kubernetes-controllers выполнить:
<pre>
kubectl apply -f frontend-deployment.yaml
kubectl apply -f paymentservice-deployment.yaml
</pre>
### Как проверить работоспособность:
<pre>kubectl get deploy,rs,po</pre>
### Задание со \*:
Запускаем деплой Node Exporter:
<pre>kubectl apply -f node-exporter-daemonset.yaml</pre>
- В манифесте мы сперва создаем namespace "monitoring", service account и role. Роль связывается с service account с помощью объекта rolebinding. Далее создаем daemonset.
- Проверяем, что node-exporter отдает метрики:
<pre>
kubectl port-forward <имя любого pod в DaemonSet>
curl localhost:9100/metrics
</pre>
</details>

<details>
<summary>HomeWork №3</summary>

### Что было сделано
Созданы service accounts с различными правами в рамках кластера.
### Как запустить проект:
- В директории kubernetes-security выполнить:
<pre>
kubectl apply -f task01/ 
kubectl apply -f task02/
kubectl apply -f task03/
</pre>
### Как проверить работоспособность:
<pre>
kubectl auth can-i get po -n kube-system --as system:serviceaccount:default:bob
kubectl auth can-i create po -n dev --as system:serviceaccount:dev:jane
kubectl auth can-i get po -n dev --as system:serviceaccount:dev:ken
kubectl auth can-i create po -n dev --as system:serviceaccount:dev:ken
</pre>
</details>

<details>
<summary>HomeWork №4</summary>

### Что было сделано
#### Добавление проверок Pod, создание объекта Deployment, добавление сервисов в кластер (ClusterIP), включение режима балансировки IPVS
- Добавляем проверку readinessProbe. Проверяем, что под запустился
<pre>
kubectl apply -f kubernetes-intro/web-pod.yaml --force
kubectl describe pod/web-app
</pre>
<pre>
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True
</pre>
<pre>
Warning  Unhealthy  48s (x60 over 10m)  kubelet, minikube  Readiness probe failed: Get http://172.17.0.3:80/index.html: dial tcp 172.17.0.3:80: connect: connection refused
</pre>
Прод не прошел readiness-пробу
- Добавляем проверку livenessProbe.
<pre>
livenessProbe:
  tcpSocket: { port: 8000 }
</pre>
> Почему следующая конфигурация валидна, но не имеет смысла? 
> 1. livenessProbe:
>   exec:
>     command:
>       - 'sh'
>       - '-c'
>       - 'ps aux | grep my_web_server_process'

Такая проверка всегда будет завершаться с кодом выхода 0, т.к. grep найдет сам процесс grep. Кроме того задача livenessProbe удостовериться, что приложение работает корреектно. Наличие запущенного процесса не всегда гарантирует это.
> 2. Бывают ли ситуации, когда она все-таки имеет смысл?

Для начала нужно убрать grep из вывода: - 'ps aux | grep my_web_server_process | grep -v grep'. Возможно имеет смысл, если это какое-то статичное приложение и достаточно проверять только запущенный процесс.
- Создаем Deployment для пода, смотрим что получилось.
<pre>kubectl describe deployment web</pre>
<pre>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
</pre>
- Исправляем ошибку в манифесте, проверяем состояние Deployment
<pre>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
</pre>
- Пробуем разные варианты maxSurge и maxUnavailable
maxSurge=0 и maxUnavailable=0: The Deployment "web" is invalid: ... - такой вариант является ощибкой.
- Создаем мервис ClusterIP
<pre>
kubectl get svc web-svc-cip -owide
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE    SELECTOR
web-svc-cip   ClusterIP   10.102.59.144   <none>        80/TCP    117s   app=web
</pre>
- Включаем IPVS, проверяем правила iptables.
- Устанавливаем MetalL, создаем ConfigMap
<pre>
kubectl --namespace metallb-system get all
NAME                            READY   STATUS             RESTARTS   AGE
pod/controller-fb659dc8-49x5d   0/1     ErrImagePull       0          51s
pod/speaker-p729h               0/1     ImagePullBackOff   0          51s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/speaker   1         1         0       1            0           beta.kubernetes.io/os=linux   51s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   0/1     1            0           51s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-fb659dc8   1         1         0       51s

echo 'apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
      - name: default
        protocol: layer2
        addresses:
          - "172.17.255.1-172.17.255.255"' > kubernetes-networks/metallb-config.yaml
kubectl apply -f kubernetes-networks/metallb-config.yaml
cp kubernetes-networks/web-svc-{cip,lb}.yaml
</pre>
- Проверяем конфигурацию
<pre>
{"caller":"service.go:114","event":"ipAllocated","ip":"172.17.255.1","msg":"IP address assigned by controller","service":"default/web-svc-lb","ts":"2021-07-17T19:59:16.59910729Z"}
kubectl describe svc web-svc-lb
Name:                     web-svc-lb
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=web
Type:                     LoadBalancer
IP Families:              <none>
IP:                       10.96.56.237
IPs:                      10.96.56.237
LoadBalancer Ingress:     172.17.255.1
Port:                     <unset>  80/TCP
TargetPort:               8000/TCP
NodePort:                 <unset>  30968/TCP
Endpoints:                172.17.0.11:8000,172.17.0.12:8000,172.17.0.13:8000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

sudo ip route add 172.17.255.0/24 via 192.168.49.2
curl http://172.17.255.1
</pre>
- Создаем сервис типа LoadBalancer для доступа к CoreDNS снаружи кластера
<pre>
kubectl get svc -n kube-system
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                  AGE
dns-tcp-svc   LoadBalancer   10.103.171.229   172.17.255.5   53:30242/TCP             30s
dns-udp-svc   LoadBalancer   10.99.88.169     172.17.255.5   53:32288/UDP             35s
kube-dns      ClusterIP      10.96.0.10       <none>         53/UDP,53/TCP,9153/TCP   15d

nslookup web.default.cluster.local 172.17.255.5
</pre>
- Применяем манифест для ingress-nginx, создаем манифест для сервиса с типом LoadBalancer. Приверяем ip, назначенный ему MetalLB. Пробуем Curl.
<pre>
kubectl apply -f kubernetes-networks/nginx-lb.yaml
kubectl get svc ingress-nginx -n ingress-nginx
</pre>
> ingress-nginx   LoadBalancer   10.97.41.211   172.17.255.2   80:32741/TCP,443:32715/TCP   5m8s
<pre>
Проверяем: curl http://172.17.255.2
Видим, что nginx отвечает.
</pre>
- Создание Headless-сервиса. Проверяем, что ClusterIP для сервиса web-svc не назначен.
kubectl apply -f  kubernetes-networks/web-svc-headless.yaml
kubectl get svc web-svc

web-svc   ClusterIP   None         <none>        80/TCP    87s

- Настроим наш ingress-прокси, создав манифест с ресурсом Ingress.
При этом есть ряд проблем:
<pre>
Warning: networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
Error from server (InternalError): error when creating "kubernetes-networks/web-ingress.yaml": Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": an error on the server ("") has prevented the request from succeeding
</pre>
Изменил версию апи и структуру манифеста под networking.k8s.io/v1. Также удалил ValidatingWebhookConfiguration, как самое простое решение проблемы
<pre>
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
</pre>
Проверяем наш созданный ingress/web
<pre>
kubectl describe ingress/web
Name:             web
Namespace:        default
Address:          192.168.49.2
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /web   web-svc:8000 (172.17.0.11:8000,172.17.0.12:8000,172.17.0.13:8000)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age                    From                      Message
  ----    ------  ----                   ----                      -------
  Normal  Sync    8m59s (x2 over 9m24s)  nginx-ingress-controller  Scheduled for sync
</pre>
При этом есть ошибка `default-http-backend:80 (<error: endpoints "default-http-backend" not found>)`
Добавляем defaultBackend
<pre>
defaultBackend:
  service:
    name: web-svc
    port:
      number: 80
</pre>
- Создаем сервис и ingress для Dashboard
- Создаем канареечное развертывание с помощью ingress-nginx
Проверяем:
<pre>
curl -k https://172.17.255.1/canary/
curl -H 'test-header: true' -k https://172.17.255.1/canary/
</pre>
</details>

<details>
<summary>HomeWork №5</summary>

### Что было сделано
#### Добавлены манифесты для StatefulSet с Minio. Авторизационные данные перемещены в secrets.
> Поместите данные в secrets и настройте конфигурацию на их использование.
- Кодируем переменные в base64, редактируем и применяем манифесты.
<pre>
echo -n 'minio' | base64
echo -n 'minio123' | base64
kubectl apply -f minio-secret.yaml
kubectl apply -f minio-statefulset.yaml
</pre>
</details>

<details>
<summary>HomeWork №6</summary>

### Что было сделано
#### * Создан k8s кластер в GCP.
#### * Установлены helm чарты nginx-ingress, cert-manager, chartmuseum, harbor.
#### * Создание helm чарта для приложения hipster-shop, шаблонизация, helm-secrets.
#### * Работа с утилитой Kubecfg.
#### * Инструмент Kustomize.

### Cert-manager

- Устанавливаем helm chart cert-manager. Проверяем, что все работает корректно по инструкции https://cert-manager.io/docs/installation/verify/
<pre>
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
kubectl apply -f kubernetes-templating/cert-manager/test-resources.yaml
kubectl describe certificate -n cert-manager-test
</pre>
<pre>
Events:
 Normal  Issuing    18s   cert-manager  The certificate has been successfully issued
</pre>
> Изучите cert-manager, и определите, что еще требуется установить для корректной работы
Смотрим документацию:
> The first thing you’ll need to configure after you’ve installed cert-manager is an issuer which you can then use to issue certificates.
Создаем ACME issuer Let's Encrypt. Сделаем его глобальным для кластера (kind: ClusterIssuer).
<pre>
kubectl apply -f kubernetes-templating/cert-manager/cluster-issuer.yaml
</pre>

### Chartmuseum

- Устанавливаем helm chart chartmuseum. Проверим, что release chartmuseum установился
<pre>
helm upgrade --install chartmuseum stable/chartmuseum --wait --namespace=chartmuseum -f kubernetes-templating/chartmuseum/values.yaml
helm ls -n chartmuseum
</pre>
Проверяем в браузере - сертификат валиден.
![CHARTMUSEUM-CERT](https://github.com/otus-kuber-2021-06/DmitryMCN_platform/blob/kubernetes-templating/kubernetes-templating/chartmuseum-cert.png?raw=true)

Далее нужно выполнить ряд действий в GCP:

Создаем Service account (см .https://cloud.google.com/docs/authentication/getting-started)
> export GOOGLE_APPLICATION_CREDENTIALS=$HOME/sustained-vial-321511-74daedfac94f.json

Создаем google storage bucket (см. https://cloud.google.com/storage/docs/creating-buckets)

Качаем бинарь chartmuseum и запускаем
<pre>
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum
chartmuseum --debug --port=8080   --storage="google"   --storage-google-bucket="my-gcs-chartmuseum-bucket"   --storage-google-prefix=""

2021-08-01T19:55:10.962+0300	DEBUG	Fetching chart list from storage	{"repo": ""}
2021-08-01T19:55:11.164+0300	DEBUG	No change detected between cache and storage	{"repo": ""}
2021-08-01T19:55:11.164+0300	INFO	Starting ChartMuseum	{"port": 8080}
</pre>

Далее устанавливаем плагин helm-push https://github.com/chartmuseum/helm-push. Добавляем репозиторий
<pre>
helm repo add chartmuseum http://localhost:8080
</pre>

Пушим тестовый чарт
<pre>
helm push frontend/ chartmuseum
Pushing frontend-0.1.0.tgz to chartmuseum...
Done.
</pre>

Проверяем, что чарт доступен в репозитории
<pre>
curl http://localhost:8080/api/charts
{"frontend":[{"name":"frontend","version":"0.1.0","description":"A Helm chart for Kubernetes","apiVersion":"v2","appVersion":"1.16.0","type":"application","urls":["charts/frontend-0.1.0.tgz"],"created":"2021-08-01T16:57:36.577Z","digest":"844d3afde2d58b3534e1b85d9ab26059d33a69ad65c44a415dda41d6b5479eda"}]}
</pre>

Устанавливаем чарт в кластер
<pre>
helm install frontend chartmuseum/frontend --create-namespace -n dev
W0802 12:28:35.421029 3175311 warnings.go:70] networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
W0802 12:28:37.557695 3175311 warnings.go:70] networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME: frontend
LAST DEPLOYED: Mon Aug  2 12:28:34 2021
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE: None

kubectl get po -n dev
NAME                         READY   STATUS    RESTARTS   AGE
front-end-7b8bcd59cb-hlhnv   1/1     Running   0          37s
</pre>

Удаляем чарт и ns
<pre>
helm uninstall frontend -n dev && kubectl delete ns dev
</pre>

### Harbor

- Устанавливаем helm chart harbor.
<pre>
helm repo add harbor https://helm.goharbor.io
kubectl create ns harbor
helm install harbor harbor/harbor --wait --namespace=harbor -f kubernetes-templating/harbor/values.yaml

NAME: harbor
LAST DEPLOYED: Sat Aug  7 16:04:12 2021
NAMESPACE: harbor
STATUS: deployed
REVISION: 1
TEST SUITE: None
</pre>

Проверяем в браузере - сертификат валиден.
![HARBOR-CERT](https://github.com/otus-kuber-2021-06/DmitryMCN_platform/blob/kubernetes-templating/kubernetes-templating/harbor-cert.jpeg?raw=true)

#### * Создание helm чарта для приложения hipster-shop, шаблонизация, helm-secrets.
- Создаем свой helm chart. Устанавливаем и проверяем работу UI.
<pre>
kubectl port-forward -n hipster-shop frontend-5c6dcc58c-mkm4v 8080:8080
</pre>

- Устанавливаем плагин helm-secrets, добавляем секрет для приложения frontend
<pre>
helm plugin install https://github.com/jkroepke/helm-secrets --version v3.8.2
helm secrets upgrade --install frontend kubernetes-templating/frontend --namespace hipster-shop -f kubernetes-templating/frontend/values.yaml -f kubernetes-templating/frontend/secrets.yaml
kubectl get secret secret -n hipster-shop -o yaml
</pre>

> Поместите все получившиеся helm chart's в ваш установленный harbor в публичный проект.
Добавляем registry, пушим чарты
<pre>
helm registry login harbor.34.66.149.53.nip.io

helm chart save kubernetes-templating/frontend/ harbor.34.66.149.53.nip.io/templating/frontend:0.0.1
ref:     harbor.34.66.149.53.nip.io/templating/frontend:0.0.1
digest:  eccb93986c0c9a63b7fe0a5449ab56d564697fa379f5392a1e8e16ab2e230c9b
size:    2.7 KiB
name:    frontend
version: 0.1.0
0.0.1: saved

helm chart push harbor.34.66.149.53.nip.io/templating/frontend:0.0.1
The push refers to repository [harbor.34.66.149.53.nip.io/templating/frontend]
ref:     harbor.34.66.149.53.nip.io/templating/frontend:0.0.1
digest:  a8b6da66d26c14f51778663a59e95e33b6475cc4b67f3e10ebc07c06dbe3ec4c
size:    2.7 KiB
name:    frontend
version: 0.1.0
0.0.1: pushed to remote (1 layer, 2.7 KiB total)
</pre>
Тоже самое для hipster-shop. Теперь чарты доступны в хранилище.

![HARBOR](https://github.com/otus-kuber-2021-06/DmitryMCN_platform/blob/kubernetes-templating/kubernetes-templating/harbor.png?raw=true)

#### Работа с утилитой Kubecfg.

Проверим, что манифесты генерируются корректно и установим их:
<pre>
kubecfg show services.jsonnet
kubecfg update kubernetes-templating/kubecfg/services.jsonnet --namespace hipster-shop 
INFO  Validating deployments paymentservice
INFO  validate object "apps/v1, Kind=Deployment"
INFO  Validating services paymentservice
INFO  validate object "/v1, Kind=Service"
INFO  Validating deployments shippingservice
INFO  validate object "apps/v1, Kind=Deployment"
INFO  Validating services shippingservice
INFO  validate object "/v1, Kind=Service"
INFO  Fetching schemas for 4 resources
INFO  Creating services paymentservice
INFO  Creating services shippingservice
INFO  Creating deployments paymentservice
INFO  Creating deployments shippingservice
</pre>

#### Инструмент Kustomize

Проверка усстановки на выбранное окружение. В данном случае dev:
<pre>
kubectl apply --dry-run=server -k kubernetes-templating/kustomize/overrides/hipster-shop/
service/dev-adservice created (server dry run)
deployment.apps/dev-adservice created (server dry run)
</pre>

</details>

<details>
<summary>HomeWork №7</summary>

### Что было сделано
#### * Созданы CRD и СR.
#### * Запущен оператор, который работает с данным СR.
#### * Сделан деплой оператора в k8s.

- Применяем манифест Custom Resource Definitions.
<pre>
$ kubectl apply -f deploy/crd.yml
The CustomResourceDefinition "mysqls.otus.homework" is invalid: spec.versions[0].schema.openAPIV3Schema: Required value: schemas are required
</pre>
Получаем ошибку. Манифест устарел, в доументации указано:
> With apiextensions.k8s.io/v1 the definition of a structural schema is mandatory for CustomResourceDefinitions

Добавляем схему, деплоим повторно.

> Задание: добавьте описание обязательный полей в CustomResourceDefinition. 

Добавляем через блок required.

> Вопрос: почему объект создался, хотя мы создали CR, до того, как запустили контроллер?

Работает reconciliation loop: контроллер периодически получает актуальное состояние и сравнивает его с желаемым

> Добавьте в README вывод комманды kubectl get jobs (там должны быть успешно выполненные backup и restore job)
<pre>
$ kubectl get jobs
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           2s         2m11s
restore-mysql-instance-job   1/1           20s        103s
</pre>

> Приложите вывод при запущенном MySQL:
<pre>
$ export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[\*].metadata.name}")
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
</pre>

> Задание со \* (1). Исправить контроллер, чтобы он писал в status subresource

Добавлен return для функции mysql_on_create

<pre>
$ kubectl describe mysqls.otus.homework mysql-instance | grep -A 2 Status
Status:
  mysql_on_create:
    Message:  mysql-instance created with restore-job
</pre>

> Задание со \* (2).
> Добавить в контроллер логику обработки изменений CR. Например, реализовать смену пароля от MySQL. Показать, что код работает

Пример:
<pre>
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
</pre>
Меняем пароль в CR MySQL
<pre>
$ kubectl patch ms --type=merge mysql-instance -p '{"spec":{"password":"otuspassword555"}}'
mysql.otus.homework/mysql-instance patched
</pre>
Делаем еще один select со старым паролем.
<pre>
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
command terminated with exit code 1
</pre>
И теперь с новым
<pre>
$ kubectl exec -it $MYSQLPOD -- mysql -potuspassword555 -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
</pre>
> Объяснить, что он делает

Оператор использует фреймворк Kopf. Добавляем обработчик для обновления нашего кастомного ресурса: <b>@kopf.on.update('otus.homework', 'v1', 'mysqls')</b>. (см. https://kopf.readthedocs.io/en/latest/walkthrough/updates). В функции необходимые параметры получаем из дополнительных агрументов ** kwargs. Если пароль не менялся (например в CR менялось что-то другое), то выходим по return. Если пароль изменился забираем список подов в данном ns фильтруя по label_selector. В цикле идем по нужным подам и меняем пароль с помощью функции stream из модуля kubernetes
@kopf.on.update('otus.homework', 'v1', 'mysqls')
<pre>
# Функция, которая будет запускаться при изменении объектов тип MySQL
# Умеет менять только пароль для всех подов CustomResource
def mysql_on_update(spec, status, namespace, logger, **kwargs):
    name = kwargs['name']
    try:
        old_password = kwargs['old']['spec']['password']
        new_password = kwargs['new']['spec']['password']
        if new_password == old_password:
            return
    except kopf.PermanentError("Password must be set"):
        return
    api = kubernetes.client.CoreV1Api()
    pods = api.list_namespaced_pod(namespace=namespace, label_selector='app='+name).items
    for pod in pods:
        name = pod.metadata.name
        exec_command = [
            '/bin/sh',
            '-c',
            'mysql -p'+old_password+' -e "ALTER USER \'root\'@\'localhost\' IDENTIFIED BY \''+new_password+'\'"']
        resp = stream(api.connect_get_namespaced_pod_exec,
                      name,
                      namespace,
                      command=exec_command,
                      stderr=True, stdin=False,
                      stdout=True, tty=False)
        print("Response: " + resp)
</pre>
</details>

<details>
<summary>HomeWork №8</summary>

### Что было сделано
#### Собран образ nginx с модулем stub_status. Созданы манифесты deployment, service и servicemonitor для nginx и nginx exporter.
- Ставим prometheus-operator через  kubectl apply из офф. репозитория
<pre>
kubectl apply -f https://github.com/prometheus-operator/prometheus-operator/blob/master/bundle.yaml?raw=true
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
serviceaccount/prometheus-operator created
service/prometheus-operator created
</pre>

- Создаем образ nginx, в конфиге задействуем stub_status.
<pre>
cd kubernetes-monitoring/nginx/
docker build -t nginx_stub:0.0.1 .
</pre>

- Собираем контейнер с nginx exporter.
<pre>
git clone https://github.com/nginxinc/nginx-prometheus-exporter
make container
</pre>

- Используем его как sidecar для пода c nginx. Применяем манифесты, проверям, что метрики доступны.
<pre>
cd kubernetes-monitoring/temlates/
kubectl apply -f nginx_stub_deploy.yaml 
kubectl apply -f nginx_stub_svc.yaml 
kubectl apply -f nginx_stub_servicemonitor.yaml
kubectl exec -ti nginx-7d88fb5c6-r2zdt -- /bin/bash
curl http://localhost:9113/metrics
</pre>
</details>

<details>
<summary>HomeWork №9</summary>

### Что было сделано
#### * Создан кластер в GKE.
#### * Установлен hipstershop, Развернут EFK стек.
#### * Установлены helm-чарты cert-manager, nginx-ingress, kube-prometheus-stack
#### * Созданы дашборды с визулизацией
#### * https://kibana.104.154.89.14.nip.io/
#### * https://prometheus.104.154.89.14.nip.io/
#### * https://grafana.104.154.89.14.nip.io/

- Создал кластер в GCP с 4 нодами (1 нода default-pool, 3 ноды infra-pool)
В default-pool рвзвернут hipstershop, все остальное работает в infra-pool.
- Ставим helm-чарт cert-manager.
<pre>
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.5.4
kubectl apply -f kubernetes-logging/cert-manager/cluster-issuer.yaml
</pre>

- И nginx-ingress
<pre>
helm upgrade --install nginx-ingress stable/nginx-ingress --wait --namespace=nginx-ingress -f nginx-ingress.values.yaml
</pre>

- Дальше ставим EFK
<pre>
helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability -f elasticsearch.values.yaml
</pre>
Получаем ошибку
> Error: UPGRADE FAILED: template: elasticsearch/templates/statefulset.yaml:298:27: executing "elasticsearch/templates/statefulset.yaml" at <.Values.roles.master>: can't evaluate field master in type interface {}

Исправляем (https://stackoverflow.com/questions/67837291/elasticsearch-installation-in-helm-fails-with-statefulset-error)

Поды с эластиком работают на нодах инфры
<pre>
kubectl get pods -o wide -n observability -l app=elasticsearch-master
NAME                     READY   STATUS    RESTARTS   AGE     IP         NODE                                     NOMINATED NODE   READINESS GATES
elasticsearch-master-0   1/1     Running   0          2d23h   10.4.1.6   gke-cluster-1-infra-pool-2aeade7d-s91l   <none>           <none>
elasticsearch-master-1   1/1     Running   0          2d23h   10.4.3.5   gke-cluster-1-infra-pool-2aeade7d-d1wj   <none>           <none>
elasticsearch-master-2   1/1     Running   0          5d2h    10.4.2.6   gke-cluster-1-infra-pool-2aeade7d-88l8   <none>           <none>
</pre>

> Причину можно найти в логах pod с Fluent Bit, он пытается обработать JSON, отдаваемый приложением, и находит там дублирующиеся поля time и timestamp

Не понял, о какой проблеме речь. В логах пода с Fluent Bit ошибок не увидел. Указаный issue давно закрыт, возможно проблема уже исправлена.

- Задание со \*. Предложите более элегантное решение в контексте конкретной ситуации и реализуйте его

Решение предложено в одном из коментариев к issue https://github.com/fluent/fluent-bit/issues/628

> If you are in Kubernetes, use the option Merge_Log_Key in your Kubernetes filter, so your unpacked data will be under that new key, avoiding duplicates if any:

Столкнулся с проблемой парсинга. В лог попадал не корректный json вида: "log": "2021-10-16T11:59:48.051Z stdout F {\"context\":{". Проблема описана тут https://github.com/helm/charts/issues/10424
Добавил новый parser для fluent-bit

- Далее ставим kube-prometheus-stack и elasticsearch-exporter:
<pre>
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace=observability
helm upgrade --install elasticsearch-exporter stable/elasticsearch-exporter --set es.uri=http://elasticsearch-master:9200 --set serviceMonitor.enabled=true --namespace=observability
</pre>

Достаем дефолтный пароль на grafana:
<pre>
kubectl get secret kube-prometheus-stack-grafana -n observability -o jsonpath='{.data.admin-password}' | base64 --decode
</pre>

По умолчанию prometheus-оператор не поддерживает annotation-based discovery of services. И данные в дашборде эластика я не увидел. Добавил ключи:

> serviceMonitorSelectorNilUsesHelmValues: false
> podMonitorSelectorNilUsesHelmValues: false

- Обновляем чарт:
<pre>
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace=observability -f prometheus.values.yaml
</pre>

- Создал дэшборд с визуализацией в Kibana, выгрузил его в export.ndjson
- Установил loki, добавив его как additionalDataSources для Графаны. Создал дэшборд в grafana с графами и логами nginx-ingress, выгрузил в nginx-ingress.json.
</details>
