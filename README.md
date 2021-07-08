<details>
<summary>
HomeWork №1</summary>

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
