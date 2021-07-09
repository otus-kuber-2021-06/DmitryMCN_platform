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
