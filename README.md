# devops-DZ12.3-K8S-apps
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
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.
 
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
 
## Ответ
 
#### Проверим установку MicroK8S
 
```bash
root@vkvm:/home/vk# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
vkvm   Ready    <none>   11d   v1.26.3
root@vkvm:/home/vk# kubectl get pod -A
NAMESPACE     NAME                                        READY   STATUS    RESTARTS       AGE
kube-system   kubernetes-dashboard-dc96f9fc-m949s         1/1     Running   3 (2d6h ago)   11d
kube-system   dashboard-metrics-scraper-7bc864c59-7dspr   1/1     Running   3 (2d6h ago)   11d
kube-system   calico-node-tl7rj                           1/1     Running   3 (2d6h ago)   11d
kube-system   calico-kube-controllers-79568db7f8-jk9ck    1/1     Running   3 (2d6h ago)   11d
kube-system   metrics-server-6f754f88d-qlllr              1/1     Running   3 (2d6h ago)   11d
```
 
### Задание 1
 
#### 1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку
 
Создадим `deploy1.yml` для двух контейнеров `nginx` и `multitool`, настройки по умолчанию
 
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy1
  name: deploy1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy1
  template:
    metadata:
      labels:
        app: deploy1
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
        - name: multitool
          image: wbitt/network-multitool
          ports:
            - name: http-8080
              containerPort: 8080
              protocol: TCP
```
 
![Ссылка на deploy1.yml](deploy1.yml)
 
Запустим развёртывание
 
```bash
root@vkvm:/home/vk/12.3# kubectl create -f deploy1.yml
deployment.apps/deploy1 created
```
 
Проверим состояние
 
```bash
root@vkvm:/home/vk/12.3# kubectl get pods
NAME                      READY   STATUS              RESTARTS   AGE
deploy1-988b78786-nddzw   0/2     ContainerCreating   0          21s
root@vkvm:/home/vk/12.3# kubectl get pods
NAME                      READY   STATUS   RESTARTS   AGE
deploy1-988b78786-nddzw   1/2     Error    0          34s
root@vkvm:/home/vk/12.3# kubectl get deployment
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
deploy1   0/1     1            0           82s
```
 
Проверим логи пода
 
```bash
root@vkvm:/home/vk/12.3# kubectl logs --tail=5 --prefix=true --all-containers=true  deploy1-988b78786-nddzw 
[pod/deploy1-988b78786-nddzw/nginx] 2023/04/18 16:26:42 [notice] 1#1: OS: Linux 5.19.0-38-generic
[pod/deploy1-988b78786-nddzw/nginx] 2023/04/18 16:26:42 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 65536:65536
[pod/deploy1-988b78786-nddzw/nginx] 2023/04/18 16:26:42 [notice] 1#1: start worker processes
[pod/deploy1-988b78786-nddzw/nginx] 2023/04/18 16:26:42 [notice] 1#1: start worker process 28
[pod/deploy1-988b78786-nddzw/nginx] 2023/04/18 16:26:42 [notice] 1#1: start worker process 29
[pod/deploy1-988b78786-nddzw/multitool] nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
[pod/deploy1-988b78786-nddzw/multitool] 2023/04/18 16:28:36 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address in use)
[pod/deploy1-988b78786-nddzw/multitool] nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
[pod/deploy1-988b78786-nddzw/multitool] 2023/04/18 16:28:36 [emerg] 1#1: still could not bind()
[pod/deploy1-988b78786-nddzw/multitool] nginx: [emerg] still could not bind()
```
 
Ошибка `bind() to 0.0.0.0:80 failed (98: Address in use)`, потому что контейнер `multitool` не может подключиться к порту `80`, т.к. он уже занят. Поменяем порт через переменные окружения в манифесте.
 
Выключим неработающий манифест
 
```bash
root@vkvm:/home/vk/12.3# kubectl delete -f deploy1.yml
deployment.apps "deploy1" deleted
```
 
Создадим `deploy2.yml` для двух контейнеров  `nginx` и `multitool`, с переменными окружения.
 
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy2
  name: deploy2
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy2
  template:
    metadata:
      labels:
        app: deploy2
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
        - name: multitool
          image: wbitt/network-multitool
          ports:
            - name: http-8080
              containerPort: 8080
              protocol: TCP
          env:
            - name: HTTP_PORT
              value: "8080"
            - name: HTTPS_PORT
              value: "11443"
```
 
![Ссылка на deploy2.yml](deploy2.yml)
 
Запускаем развёртывание
 
```bash
root@vkvm:/home/vk/12.3# kubectl create -f deploy2.yml
deployment.apps/deploy2 created
```
 
Проверяем состояние
 
```bash
root@vkvm:/home/vk/12.3# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
deploy2-747788c6c4-f522c   2/2     Running   0          20s
root@vkvm:/home/vk/12.3# kubectl get deployment
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
deploy2   1/1     1            1           38s
```
 
Проверим логи
 
```bash
root@vkvm:/home/vk/12.3# kubectl logs --tail=10 --all-containers=true --prefix=true deploy2-747788c6c4-f522c
[pod/deploy2-747788c6c4-f522c/nginx] /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
[pod/deploy2-747788c6c4-f522c/nginx] /docker-entrypoint.sh: Configuration complete; ready for start up
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: using the "epoll" event method
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: nginx/1.23.4
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: OS: Linux 5.19.0-38-generic
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 65536:65536
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: start worker processes
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: start worker process 29
[pod/deploy2-747788c6c4-f522c/nginx] 2023/04/18 16:30:45 [notice] 1#1: start worker process 30
[pod/deploy2-747788c6c4-f522c/multitool] The directory /usr/share/nginx/html is not mounted.
[pod/deploy2-747788c6c4-f522c/multitool] Therefore, over-writing the default index.html file with some useful information:
[pod/deploy2-747788c6c4-f522c/multitool] WBITT Network MultiTool (with NGINX) - deploy2-747788c6c4-f522c - 10.1.101.212 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
[pod/deploy2-747788c6c4-f522c/multitool] Replacing default HTTP port (80) with the value specified by the user - (HTTP_PORT: 8080).
[pod/deploy2-747788c6c4-f522c/multitool] Replacing default HTTPS port (443) with the value specified by the user - (HTTPS_PORT: 11443).
```
 
Конфликт портов исчез, под запустился.
 
### 2. После запуска увеличить количество реплик работающего приложения до 2
### 3. Продемонстрировать количество подов до и после масштабирования
 
Проверяем количество подов до масштабирования
 
```bash
root@vkvm:/home/vk/12.3# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
deploy2-747788c6c4-f522c   2/2     Running   0          12m
root@vkvm:/home/vk/12.3# kubectl get deployment
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
deploy2   1/1     1            1           12m13s
root@vkvm:/home/vk/12.3# kubectl get replicaset
NAME                 DESIRED   CURRENT   READY   AGE
deploy2-747788c6c4   1         1         1       12m20s
```
 
Скопируем `deploy2.yml` в `deploy3.yml`, изменим количество реплик до `2` в файле `deploy3.yml`
 
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy2
  name: deploy2
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deploy2
  template:
    metadata:
      labels:
        app: deploy2
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
        - name: multitool
          image: wbitt/network-multitool
          ports:
            - name: http-8080
              containerPort: 8080
              protocol: TCP
          env:
            - name: HTTP_PORT
              value: "8080"
            - name: HTTPS_PORT
              value: "11443"
```
 
![Ссылка на deploy3.yml](deploy3.yml)
 
Запустим развёртывание
 
```bash
root@vkvm:/home/vk/12.3# kubectl apply -f deploy3.yml
Warning: resource deployments/deploy2 is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
deployment.apps/deploy2 configured
```
 
Проверим события
 
```bash
root@vkvm:/home/vk/12.3# kubectl get events
LAST SEEN   TYPE      REASON              OBJECT                          MESSAGE
4m37s       Normal    ScalingReplicaSet   deployment/deploy2              Scaled up replica set deploy2-747788c6c4 to 1
4m37s       Normal    SuccessfulCreate    replicaset/deploy2-747788c6c4   Created pod: deploy2-747788c6c4-f522c
4m36s       Normal    Scheduled           pod/deploy2-747788c6c4-f522c    Successfully assigned default/deploy2-747788c6c4-f522c to vkvm
4m35s       Normal    Pulling             pod/deploy2-747788c6c4-f522c    Pulling image "nginx:latest"
4m34s       Normal    Pulled              pod/deploy2-747788c6c4-f522c    Successfully pulled image "nginx:latest" in 1.40035548s (1.400361544s including waiting)
4m34s       Normal    Created             pod/deploy2-747788c6c4-f522c    Created container nginx
4m34s       Normal    Started             pod/deploy2-747788c6c4-f522c    Started container nginx
4m34s       Normal    Pulling             pod/deploy2-747788c6c4-f522c    Pulling image "wbitt/network-multitool"
4m32s       Normal    Pulled              pod/deploy2-747788c6c4-f522c    Successfully pulled image "wbitt/network-multitool" in 1.376839487s (1.376847555s including waiting)
4m32s       Normal    Created             pod/deploy2-747788c6c4-f522c    Created container multitool
4m32s       Normal    Started             pod/deploy2-747788c6c4-f522c    Started container multitool
3m33s       Warning   MissingClusterDNS   node/vkvm                       kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. Falling back to "Default" policy.
48s         Warning   MissingClusterDNS   pod/deploy2-747788c6c4-f522c    pod: "deploy2-747788c6c4-f522c_default(5d36d004-4c0f-4613-bd0a-650190543672)". kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. Falling back to "Default" policy.
47s         Normal    SuccessfulCreate    replicaset/deploy2-747788c6c4   Created pod: deploy2-747788c6c4-2xhpw
47s         Normal    ScalingReplicaSet   deployment/deploy2              Scaled up replica set deploy2-747788c6c4 to 2 from 1
47s         Normal    Scheduled           pod/deploy2-747788c6c4-2xhpw    Successfully assigned default/deploy2-747788c6c4-2xhpw to vkvm
46s         Normal    Pulling             pod/deploy2-747788c6c4-2xhpw    Pulling image "nginx:latest"
45s         Normal    Pulled              pod/deploy2-747788c6c4-2xhpw    Successfully pulled image "nginx:latest" in 1.429402862s (1.429410674s including waiting)
45s         Normal    Created             pod/deploy2-747788c6c4-2xhpw    Created container nginx
44s         Normal    Started             pod/deploy2-747788c6c4-2xhpw    Started container nginx
44s         Normal    Pulling             pod/deploy2-747788c6c4-2xhpw    Pulling image "wbitt/network-multitool"
43s         Normal    Pulled              pod/deploy2-747788c6c4-2xhpw    Successfully pulled image "wbitt/network-multitool" in 1.350900389s (1.350908155s including waiting)
43s         Normal    Created             pod/deploy2-747788c6c4-2xhpw    Created container multitool
43s         Normal    Started             pod/deploy2-747788c6c4-2xhpw    Started container multitool
41s         Warning   MissingClusterDNS   pod/deploy2-747788c6c4-2xhpw    pod: "deploy2-747788c6c4-2xhpw_default(869e8525-7b09-4f16-b690-1281aa1f8c61)". kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. Falling back to "Default" policy.
```
 
Проверим количество подов после масштабирования
 
```bash
root@vkvm:/home/vk/12.3# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
deploy2-747788c6c4-f522c   2/2     Running   0          5m16s
deploy2-747788c6c4-2xhpw   2/2     Running   0          86s
root@vkvm:/home/vk/12.3# kubectl get deployment
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
deploy2   2/2     2            2           5m27s
root@vkvm:/home/vk/12.3# kubectl get replicaset
NAME                 DESIRED   CURRENT   READY   AGE
deploy2-747788c6c4   2         2         2       5m36s
```
 
Убедились, что запустились два пода с двумя контейнерами в каждом.
 
### 4. Создать Service, который обеспечит доступ до реплик приложений из п.1
 
Создадим `service1.yml` с конфигурацией сервиса
 
```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service1
spec:
  selector:
    app: deploy2
  ports:
    - name: nginx-http
      port: 80
      protocol: TCP
      targetPort: 80
    - name: multitool-http
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: multitool-https
      port: 11443
      protocol: TCP
      targetPort: 11443
```
 
![Ссылка на service1.yml](service1.yml)
 
Запустим развёртывание сервисов
 
```bash
root@vkvm:/home/vk/12.3# kubectl apply -f service1.yml
service/service1 created
```
 
Проверим состояние сервисов
 
```bash
root@vkvm:/home/vk/12.3# kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                     AGE
kubernetes   ClusterIP   10.152.183.1    <none>        443/TCP                     11d
service1     ClusterIP   10.152.183.96   <none>        80/TCP,8080/TCP,11443/TCP   18s
```
 
Проверим forwarding портов до сервиса
 
```bash
root@vkvm:/home/vk/12.3# kubectl port-forward service/service1 :80
Forwarding from 127.0.0.1:42505 -> 80
Forwarding from [::1]:42505 -> 80

vk@vkvm:~$ curl --silent -i 127.0.0.1:42505 | grep Server
Server: nginx/1.23.4

root@vkvm:/home/vk/12.3# kubectl port-forward service/service1 :8080
Forwarding from 127.0.0.1:33729 -> 8080
Forwarding from [::1]:33729 -> 8080

vk@vkvm:~$ curl --silent -i 127.0.0.1:33729 | grep Server
Server: nginx/1.20.2
```
 
### 5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1
 
Создадим файл `pod1.yml` с конфигурацией пода
 
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    app: pod1
spec:
  containers:
    - name: multitool
      image: wbitt/network-multitool
      ports:
        - name: http-1080
          containerPort: 1080
          protocol: TCP
      env:
        - name: HTTP_PORT
          value: "1080"
        - name: HTTPS_PORT
          value: "10443"
```
 
![Ссылка на pod1.yml](pod1.yml)
 
Запустим развёртывание
 
```bash
root@vkvm:/home/vk/12.3# kubectl apply -f pod1.yml
pod/pod1 created
```
 
Проверим состояние подов
 
```bash
root@vkvm:/home/vk/12.3# kubectl get pod -o wide
NAME                       READY   STATUS    RESTARTS   AGE    IP             NODE   NOMINATED NODE   READINESS GATES
deploy2-747788c6c4-f522c   2/2     Running   0          12m    10.1.101.212   vkvm   <none>           <none>
deploy2-747788c6c4-2xhpw   2/2     Running   0          9m9s   10.1.101.213   vkvm   <none>           <none>
pod1                       1/1     Running   0          18s    10.1.101.214   vkvm   <none>           <none>
```
 
Запустим `curl` из пода
 
```bash
root@vkvm:/home/vk# kubectl exec pod1 -- curl --silent -i 10.1.101.212:80 
HTTP/1.1 200 OK
Server: nginx/1.23.4
Date: Tue, 18 Apr 2023 16:46:55 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 28 Mar 2023 15:01:54 GMT
Connection: keep-alive
ETag: "64230162-267"
Accept-Ranges: bytes

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
root@vkvm:/home/vk# kubectl exec pod1 -- curl --silent -i 10.1.101.213:8080 
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Tue, 18 Apr 2023 16:46:17 GMT
Content-Type: text/html
Content-Length: 145
Last-Modified: Tue, 18 Apr 2023 16:34:36 GMT
Connection: keep-alive
ETag: "643ec69c-91"
Accept-Ranges: bytes

WBITT Network MultiTool (with NGINX) - deploy2-747788c6c4-2xhpw - 10.1.101.213 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```
 
Вывод - сервис опубликовал порты, nginx вместе с multitool отвечают на запросы по этим портам.
 
Удалим развёрнутые ресурсы
 
```bash
root@vkvm:/home/vk/12.3kubectl delete -f deploy3.yml -f service1.yml -f pod1.ymlml
deployment.apps "deploy2" deleted
service "service1" deleted
pod "pod1" deleted
```
 
## Задание 2
 
### 1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения
 
Включим поддержку DNS в microk8s
 
```bash
root@vkvm:/home/vk/12.3# microk8s enable dns
Infer repository core for addon dns
Enabling DNS
Using host configuration from /run/systemd/resolve/resolv.conf
Applying manifest
serviceaccount/coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
clusterrole.rbac.authorization.k8s.io/coredns created
clusterrolebinding.rbac.authorization.k8s.io/coredns created
Restarting kubelet
DNS is enabled
```
 
Проверим отсутствие подов и сервисов
 
```bash
root@vkvm:/home/vk/12.3# kubectl get pod
No resources found in default namespace.
root@vkvm:/home/vk/12.3# kubectl get service
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.152.183.1   <none>        443/TCP   11d
```
 
### 2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox
 
Создадим `deploy4.yml` с конфигурацией развёртывания. Основной контейнер `nginx`, init контейнер `busybox`.
Busybox будет пытаться имя по DNS раз в 5 сек.
 
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy4
  name: deploy4
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy4
  template:
    metadata:
      labels:
        app: deploy4
    spec:
     containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
      initContainers:
        - name: busybox
          image: busybox:latest
          command: ['sh', '-c', 'until nslookup service2.default.svc.cluster.local; do echo Waiting for service2...; sleep 5; done;']
```
 
![Ссылка на deploy4.yml](deploy4.yml)
 
Запустим развёртывание 
```bash
root@vkvm:/home/vk/12.3# kubectl apply -f deploy4.yml
deployment.apps/deployment-4 created
```
 
Проверим состояние подов до создания сервиса
 
```bash
    kubectl get pod -o wide
 
    NAME                           READY   STATUS     RESTARTS   AGE   IP          NODE            NOMINATED NODE   READINESS GATES
    deployment-4-99b9c99d6-wvp4b   0/1     Init:0/1   0          34s   10.1.2.94   12-kubernetes   <none>           <none>
```
 
Проверим логи пода
 
```bash
root@vkvm:/home/vk/12.3# kubectl logs --tail=10 --all-containers=true --prefix=true deployment-4-99f7d49b9-8s5ql
[pod/deployment-4-99f7d49b9-8s5ql/busybox] 
[pod/deployment-4-99f7d49b9-8s5ql/busybox] Waiting for service2...
[pod/deployment-4-99f7d49b9-8s5ql/busybox] Server:		10.152.183.10
[pod/deployment-4-99f7d49b9-8s5ql/busybox] Address:	10.152.183.10:53
[pod/deployment-4-99f7d49b9-8s5ql/busybox] 
[pod/deployment-4-99f7d49b9-8s5ql/busybox] ** server can't find service2.default.svc.cluster.local: NXDOMAIN
[pod/deployment-4-99f7d49b9-8s5ql/busybox] 
[pod/deployment-4-99f7d49b9-8s5ql/busybox] ** server can't find service2.default.svc.cluster.local: NXDOMAIN
[pod/deployment-4-99f7d49b9-8s5ql/busybox] 
[pod/deployment-4-99f7d49b9-8s5ql/busybox] Waiting for service2...
Error from server (BadRequest): container "nginx" in pod "deployment-4-99f7d49b9-8s5ql" is waiting to start: PodInitializing
```
 
Из-за того, что init контейнер busybox не может разрешить DNS имя, основной контейнер nginx не запускается.
 
### 3. Создать и запустить Service. Убедиться, что Init запустился
 
Создадим файл `service2.yml` с конфигурацией сервиса
 
```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service2
spec:
  selector:
    app: deploy4
  ports:
    - name: nginx-http
      port: 80
      protocol: TCP
      targetPort: 80
```
 
![Ссылка на service2.yml](service2.yml)
 
Запустим развёртывание сервиса
 
```bash
root@vkvm:/home/vk/12.3# kubectl apply -f service2.yml
service/service2 created
```
 
Проверим список сервисов
 
```bash
root@vkvm:/home/vk/12.3# kubectl get service
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP   11d
service2     ClusterIP   10.152.183.193   <none>        80/TCP    20s
```
 
### 4. Продемонстрировать состояние пода до и после запуска сервиса
 
Проверим состояние подов после создания сервиса
 
```bash
root@vkvm:/home/vk/12.3# kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP             NODE   NOMINATED NODE   READINESS GATES
deployment-4-99f7d49b9-8s5ql   1/1     Running   0          3m52s   10.1.101.217   vkvm   <none>           <none>
```
 
При создании сервиса создалось DNS имя, далее init контейнер busybox успешно выполнился, далее запустился основной контейнер nginx
 
Удалим развёрнутые ресурсы
 
```bash
root@vkvm:/home/vk/12.3# kubectl delete -f deploy4.yml -f service2.yml
deployment.apps "deployment-4" deleted
service "service2" deleted
```
