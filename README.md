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
####
- Устанавливаем helm chart cert-manager. Проверяем, что все работает корректно по инструкции https://cert-manager.io/docs/installation/verify/
<pre>
kubectl apply -f cert-manager/test-resources.yaml
kubectl describe certificate -n cert-manager-test
</pre>
<pre>
...
Events:
...
 Normal  Issuing    18s   cert-manager  The certificate has been successfully issued
</pre>
> Изучите cert-manager, и определите, что еще требуется установить для корректной работы
Смотрим документацию:
> The first thing you’ll need to configure after you’ve installed cert-manager is an issuer which you can then use to issue certificates.
Создаем ACME issuer Let's Encrypt. Сделаем его глобальным для кластера (kind: ClusterIssuer).
<pre>
kubectl apply -f cert-manager/cluster-issuer.yaml
</pre>
- Устанавливаем helm chart chartmuseum. Проверим, что release chartmuseum установился
<pre>
helm ls -n chartmuseum
</pre>
Проверяем в браузере - сертификат валиден.

Создаем Service account https://cloud.google.com/docs/authentication/getting-started

export GOOGLE_APPLICATION_CREDENTIALS=$HOME/sustained-vial-321511-74daedfac94f.json
	pip3 install --upgrade google-cloud-storage

Создаем google storage bucket https://cloud.google.com/storage/docs/creating-buckets
Качаем бинарь chartmuseum и запускаем локально
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum
chartmuseum --debug --port=8080   --storage="google"   --storage-google-bucket="my-gcs-chartmuseum-bucket"   --storage-google-prefix=""

2021-08-01T19:55:10.962+0300	DEBUG	Fetching chart list from storage	{"repo": ""}
2021-08-01T19:55:11.164+0300	DEBUG	No change detected between cache and storage	{"repo": ""}
2021-08-01T19:55:11.164+0300	INFO	Starting ChartMuseum	{"port": 8080}

Устанавливаем плагин helm-push https://github.com/chartmuseum/helm-push

Пушим тестовый чарт
helm push frontend/ chartmuseum
Pushing frontend-0.1.0.tgz to chartmuseum...
Done.

Проверяем, что чарт доступен в репозитории
curl http://localhost:8080/api/charts
{"frontend":[{"name":"frontend","version":"0.1.0","description":"A Helm chart for Kubernetes","apiVersion":"v2","appVersion":"1.16.0","type":"application","urls":["charts/frontend-0.1.0.tgz"],"created":"2021-08-01T16:57:36.577Z","digest":"844d3afde2d58b3534e1b85d9ab26059d33a69ad65c44a415dda41d6b5479eda"}]}

Устанавливаем чарт в кластер


</details>
