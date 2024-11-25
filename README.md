# Домашнее задание к занятию «Запуск приложений в K8S»

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hw-3
spec:
  selector:
    matchLabels:
      app: netology
  replicas: 1
  template:
    metadata:
      labels:
        app: netology
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "1180"
```

Запуск:

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl apply -n default -f task1-deployment.yaml
deployment.apps/hw-3 created
```

Проверка:

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl get deployments -n default
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
hw-3   1/1     1            1           5m15s
```

2. После запуска увеличить количество реплик работающего приложения до 2.

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl scale deployment --replicas=2 -n default hw-3
deployment.apps/hw-3 scaled
user@k8s:/opt/hw_k8s_3$ microk8s kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
hw-3   2/2     2            2           14m
```

3. Продемонстрировать количество подов до и после масштабирования.

До:

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
hw-3-55d6c9d747-cbzng   2/2     Running   0          11m
```

После:

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
hw-3-55d6c9d747-cbzng   2/2     Running   0          14m
hw-3-55d6c9d747-gmj4k   2/2     Running   0          79s
```

4. Создать Service, который обеспечит доступ до реплик приложений из п.1.

```
apiVersion: v1
kind: Service
metadata:
  name: hw-3-service
  namespace: default
spec:
  selector:
    app: hw-3
  ports:
    - protocol: TCP
      name: nginx
      port: 80
      targetPort: 80
    - protocol: TCP
      name: multitool
      port: 8080
      targetPort: 1180
```

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl  apply -f task1-service.yaml
service/hw-3-service created
user@k8s:/opt/hw_k8s_3$ microk8s kubectl -n default get svc
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
hw-3-service   ClusterIP   10.152.183.147   <none>        80/TCP,8080/TCP   2m35s
kubernetes     ClusterIP   10.152.183.1     <none>        443/TCP           5d19h

```

5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

```
apiVersion: v1
kind: Pod
metadata:
   name: multitool
   namespace: default
spec:
   containers:
     - name: multitool
       image: wbitt/network-multitool
       ports:
        - containerPort: 8080
```

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl apply -f multitool.yaml
pod/multitool created
user@k8s:/opt/hw_k8s_3$ microk8s kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
hw-3-55d6c9d747-cbzng   2/2     Running   0          39m
hw-3-55d6c9d747-gmj4k   2/2     Running   0          26m
multitool               1/1     Running   0          6s
```

Проверка:

```
user@k8s:/opt/hw_k8s_3$ microk8s kubectl exec -n default hw-3 -- curl hw-3-service:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0   506k      0 --:--:-- --:--:-- --:--:--  600k
user@k8s:/opt/hw_k8s_3$ microk8s kubectl exec -n default hw-3 -- curl hw-3-service:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   138  100   138    0     0  15081      0 --:--:-- --:--:-- --:--:-- 15333
WBITT Network MultiTool (with NGINX) - hw-3-565c67c86c-f9gj7 - 10.1.77.43 - HTTP: 1180 , HTTPS: 443 . (Formerly praqma/network-multitool)
```

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------
