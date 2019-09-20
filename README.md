# Homework 1 (kubernetes-intro)


#### Установка Minikube
- Ставим kubectl и Minikube с Virtualbox в Centos + kubectl

```
😄  minikube v1.2.0 on linux (amd64)
🏄  Done! kubectl is now configured to use "minikube"
```

```
$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443
KubeDNS is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

#### Проверка восстановления компонентов

- Удаляем контейнеры: `minikube ssh`, `docker rm -f $(docker ps -a -q)`

Контейнеры запускаются заново, но поды не пересоздаются, если посмотреть kubectl get pods -n kube-system, видно, что время жизни подов не меняется

```
$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-5c98db65d4-2zlxz           1/1     Running   0          14h
coredns-5c98db65d4-jpjh6           1/1     Running   0          14h
etcd-minikube                      1/1     Running   0          14h
kube-addon-manager-minikube        1/1     Running   0          14h
kube-apiserver-minikube            1/1     Running   0          14h
kube-controller-manager-minikube   1/1     Running   0          14h
kube-proxy-8f244                   1/1     Running   0          14h
kube-scheduler-minikube            1/1     Running   0          14h
storage-provisioner                1/1     Running   0          14h
```

- Удаляем поды: `$ kubectl delete pod --all -n kube-system`
```
$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-5c98db65d4-h5l8b           1/1     Running   0          44s
coredns-5c98db65d4-ltz2f           1/1     Running   0          44s
etcd-minikube                      1/1     Running   0          41s
kube-addon-manager-minikube        1/1     Running   0          40s
kube-apiserver-minikube            1/1     Running   0          37s
kube-controller-manager-minikube   1/1     Running   0          42s
kube-proxy-hgg8d                   1/1     Running   0          35s
kube-scheduler-minikube            1/1     Running   0          40s
storage-provisioner                1/1     Running   0          10s
```

```
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

#### Почему все pod в namespace kube-system восстановились после удаления

- coredns и kubernetes-dashboard (аддоны) запущены как Replica Set, если упали, kubelet запускает новые. Сам kubelet работает на хосте, мы его не убивали

```
$ kubectl describe pods coredns-5c98db65d4-h5l8b -n kube-system
Controlled By:        ReplicaSet/coredns-5c98db65d4

$ kubectl describe pods kubernetes-dashboard-7b8ddcb5d6-ldgjb -n kube-system
Controlled By:  ReplicaSet/kubernetes-dashboard-7b8ddcb5d6
```

```
A ReplicaSet’s purpose is to maintain a stable set of replica Pods running at any given time. As such, it is often used to guarantee the availability of a specified number of identical Pods.
```

- kube-proxy запущен как Daemon Set, обеспечивается запущенность на каждой ноде
```
$ kubectl describe pods kube-proxy-hgg8d -n kube-system
Controlled By:        DaemonSet/kube-proxy
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod
```

- addon-manager, etcd, kube-apiserver, kube-controller-manager, kube-scheduler запускает kubelet как Static Pods по манифестам, в моем случае в `/etc/kubernetes/manifests`
```
$ ls -l /etc/kubernetes/manifests/
total 20
-rw-r----- 1 root root 1406 Jul  8 18:46 addon-manager.yaml.tmpl
-rw------- 1 root root 1971 Jul  8 18:46 etcd.yaml
-rw------- 1 root root 2895 Jul  8 18:46 kube-apiserver.yaml
-rw------- 1 root root 2264 Jul  8 18:46 kube-controller-manager.yaml
-rw------- 1 root root  990 Jul  8 18:46 kube-scheduler.yaml
```

Остановленный контейнер будет автоматически рестартован kubelet'ом.
Running kubelet periodically scans the configured directory for changes and adds/removes pods as files appear/disappear in this directory.
Запуск/остановка static pods выполняется добавлением/удалением соответствующих файлов манифестов 


#### Dockerfile
Попробованы разные варианты, с ограничениями по USER и без копирования или монтирования доп. conf файлов нормально подошел только вариант с python


#### Web pod

- Создаем web pod
```
$ kubectl get pods
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          3m30s
```
```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  108s  default-scheduler  Successfully assigned default/web to minikube
  Normal  Pulling    107s  kubelet, minikube  Pulling image "bitnami/nginx"
  Normal  Pulled     96s   kubelet, minikube  Successfully pulled image "bitnami/nginx"
  Normal  Created    96s   kubelet, minikube  Created container web
  Normal  Started    95s   kubelet, minikube  Started container web
```

- Добавляем несуществующий тег
```
Events:
  Type     Reason     Age    From               Message
  ----     ------     ----   ----               -------
  Warning  Failed     4s     kubelet, minikube  Failed to pull image "bitnami/nginx:tag": rpc error: code = Unknown desc = Error response from daemon: manifest for bitnami/nginx:tag not found
```

- Добавляем init контейнер

Строка `['sh', '-c', 'wget -O- https://raw.githubusercontent.com/express42/otus-platformsnippets/master/Module-02/Introduction-to-Kubernetes/wget.sh | sh']` из pdf копируется с ошибкой, без дефиса

- Делаем форвард, проверяем, работает
$ kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
Forwarding from 0.0.0.0:8000 -> 8000
Handling connection for 8000


# Homework 2 (kubernetes-security)


#### Task01
- ServiceAccount bob
- ClusterRoleBinding сервисаккаунта bob и default ClusterRole admin
- ServiceAccount dave без каких-либо ролей

#### Task02
- Namespace prometheus
- ServiceAccount carol
- ClusterRole pod-reader
- ClusterRoleBinding роли pod-reader и всем в system:serviceaccounts:prometheus 

#### Task03
- Namespace dev
- ServiceAccount jane
- ClusterRoleBinding jane и default ClusterRole admin c ограничением на namespace dev
- ServiceAccount ken
- ClusterRoleBinding ken и default ClusterRole view c ограничением на namespace dev


# Homework 3 (kubernetes-networks)

#### Добавление проверок Pod
Добавлены readinessProbe, livenessProbe

#### Почему следующая конфигурация валидна, но не имеет смысла?
```
livenessProbe:
exec:
command:
- 'sh'
- '-c'
- 'ps aux | grep my_web_server_process'
```

`ps aux | grep process` всегда будет находить процесс grep, надо исключить, `ps aux | grep -v grep | grep process`
Но и в этом случае, если это основной процесс и он умер, то умрет контейнер и будет перезапущен и без этой проверки 

#### Бывают ли ситуации, когда она все-таки имеет смысл?
Наверно, если там не один процесс и наличие этого процесса критично


#### Убедитесь, что условия (Conditions) Available и Progressing выполняются (в столбце Status значение true)

```
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
```  

#### Попробуйте разные варианты деплоя с крайними значениями maxSurge и maxUnavailable (оба 0, оба 100%, 0 и 100%)
Все варианты рабочие, кроме strategy 0 0:
```
$ kubectl apply -f web-deploy.yaml
The Deployment "web" is invalid: spec.strategy.rollingUpdate.maxUnavailable: Invalid value: intstr.IntOrString{Type:0, IntVal:0, StrVal:""}: may not be 0 when `maxSurge` is 0
```

#### Создаем Service Cluster IP
```
$ kubectl get services
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP   93m
web-svc-cip   ClusterIP   10.99.127.69   <none>        80/TCP    32s
```

#### Переходим на ipvs


#### Настраиваем MetalLB

```
$ kubectl --namespace metallb-system get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-7757586ff4-c2gvd   1/1     Running   0          104s
pod/speaker-5xnsc                 1/1     Running   0          104s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/speaker   1         1         1       1            1           beta.kubernetes.io/os=linux   104s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           104s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-7757586ff4   1         1         1       104s
```

#### Проверяем
```
{"caller":"service.go:98","event":"ipAllocated","ip":"172.17.255.1","msg":"IP address assigned by controller","service":"default/web-svc-lb","ts":"2019-07-31T16:07:40.691034054Z"}
```
```
$ minikube ip
192.168.99.100
```
`sudo ip route add 172.17.255.0/24 via 192.168.99.100`
`172.17.255.0/24 via 192.168.99.100 dev vboxnet0`

Балансировка по RR:
```
$ curl 172.17.255.1
<html>
<head/>
<body>
<!-- IMAGE BEGINS HERE -->
<font size="-3">
<pre><font color=white>01110100111110111...
...
172.17.0.8      web-58464fb5c9-c4qhp</pre>
```

```
172.17.0.7      web-58464fb5c9-9z6bk</pre>
...
172.17.0.6      web-58464fb5c9-4xfq9</pre>
...
172.17.0.8      web-58464fb5c9-c4qhp</pre>
```

#### Задание со *. Сделайте сервис LoadBalancer, который откроет доступ к CoreDNS снаружи кластера

Поскольку DNS работает по TCP и UDP протоколам:
```
Kubernetes does not currently allow multiprotocol LoadBalancer services. This would normally make it impossible to run services like DNS, because they have to listen on both TCP and UDP. To work around this limitation of Kubernetes with MetalLB, create two services (one for TCP, one for UDP), both with the same pod selector. Then, give them the same sharing key and spec.loadBalancerIP to colocate the TCP and UDP serving ports on the same IP address.
```
Создаем `coredns-svc-lb-udp.yaml` и `coredns-svc-lb-tcp.yaml`

Проверяем:
```
$ kubectl get services --all-namespaces
NAMESPACE              NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                  AGE
kube-system            coredns-svc-lb-tcp          LoadBalancer   10.106.251.223   172.17.255.2   53:30666/TCP             2m35s
kube-system            coredns-svc-lb-udp          LoadBalancer   10.103.8.17      172.17.255.2   53:30365/UDP             95s
```

```
$ nslookup web-svc-cip.default.svc.cluster.local 172.17.255.2
Server:         172.17.255.2
Address:        172.17.255.2#53

Name:   web-svc-cip.default.svc.cluster.local
Address: 10.99.127.69
```

### Ingress
Создаем, проверяем:
```
$ curl 172.17.255.3
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>openresty/1.15.8.1</center>
</body>
</html>
```

Создаем headless service
```
NAMESPACE              NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
default                web-svc                     ClusterIP      None             <none>         80/TCP                       3s
```

Создаем ingress
```
$ kubectl describe ingress web
Name:             web
Namespace:        default
Address:          172.17.255.3
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host  Path  Backends
  ----  ----  --------
  *
        /web   web-svc:8000 (172.17.0.6:8000,172.17.0.7:8000,172.17.0.8:8000)
```

```
$ curl http://172.17.255.3/web/index.html
<html>
<head/>
<body>
<!-- IMAGE BEGINS HERE -->
<font size="-3">
<pre><font color=white>0111010011...
```        

#### Добавьте доступ к kubernetesdashboard через наш Ingress-прокси (сервис должен быть доступен через префикс /dashboard)

- Добавляем `dashboard-ingress.yaml` со ссылкой на сервис kubernetes-dashboard, который работает на `TargetPort:        8443/TCP`
- Предполагаем, что запросов просто /dashboard нет, в url всегда есть доп.параметры,  вырезаем из url  '/dashboard', остальное оставляем как есть. Так меняем nginx rewrite
- С дашбордом работаем по https, надо добавить nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

Проверяем:
```
$ curl -k https://172.17.255.1/dashboard/
<!--
Copyright 2017 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

<!doctype html>
<html>

<head>
  <meta charset="utf-8">
  <title>Kubernetes Dashboard</title>
  <link rel="icon"
        type="image/png"
        href="assets/images/kubernetes-logo.png" />
  <meta name="viewport"
        content="width=device-width">
<link rel="stylesheet" href="styles.74371f6fb857b49820d3.css"></head>

<body>
  <kd-root></kd-root>
<script src="runtime.008b73624e1b2a41f470.js"></script><script src="polyfills-es5.5a9403b72608aff42d15.js" nomodule></script><script src="polyfills.4cb1cc2d75069dcedd9c.js"></script><script src="scripts.b1c7fc483cdf0bfa1025.js"></script><script src="main.111c637f07090984b377.js"></script></body>

</html> 
```

Считаем, что работает 

#### Реализуйте канареечное развертывание (перенаправление части трафика на выделенную группу подов по HTTP заголовку).

Делаем тестовый образ с новой версией `enot/web:1.2`. Приложение на get запрос просто отдает свою версию (чтобы нам понять куда попали), берет ее из переменной окружения, которая выставляется в Dockerfile:
```
FROM python:3.7-alpine
EXPOSE 8000
ENV VERSION 1.2
COPY web.py .
CMD python3 web.py
```
```
import os
from http.server import HTTPServer, BaseHTTPRequestHandler

class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(bytes(os.environ['VERSION'] + '\n', 'UTF-8'))

httpd = HTTPServer(('0.0.0.0', 8000), SimpleHTTPRequestHandler)
httpd.serve_forever()
```

- Создаем деплоймет для тестовой версии: `web-canary-deploy.yaml`, там меняем метку, версию образа
- Создаем сервис: `web-canary-svc-headless.yaml`, селектор с новой меткой
- Создаем ingress: `web-canary-ingress.yaml` для нового сервиса, добавляем:
```
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-by-header: "canary"
nginx.ingress.kubernetes.io/ssl-redirect: "false"
```
Контроллер `Compares an Ingress of a potential alternative backend's rules with each existing server and finds matching host + path pairs`, не находит host и не может сделать мерж (ожидаемое поведение):
```
E0801 15:53:39.906603       6 controller.go:1258] cannot merge alternative backend default-web-canary-svc-8000 into hostname  that does not exist
```
Поэтому добавляем host в ingress rules

- Проверяем

Попадаем на web версии 1.1:
```
$ curl -H "Host: web.com" http://172.17.255.3/web
<html>
<head/>
<body>
<!-- IMAGE BEGINS HERE -->
<font size="-3">
<pre><font color=white>01110100111110111...
```

Попадаем на web версии 1.2:
```
$ curl -H "canary: always" -H "Host: web.com" http://172.17.255.3/web
1.2
```

### Homework 4 (kubernetes-volumes)


#### Запускаем kind

Не забываем добавить своего юзера в группу docker, иначе с kind без sudo могут возникнуть проблемы  
Нужные образы потребовалось сначала загрузить в кластер
`kind load docker-image minio/minio:RELEASE.2019-07-10T00-34-56Z`

#### Создаем minio

Проверяем

```
$ kubectl get statefulsets
NAME    READY   AGE
minio   1/1     73m

$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          74m

$ kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-minio-0   Bound    pvc-6bb57663-5e64-40f6-91e9-754b2eb53de7   10Gi       RWO            standard       23m

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-6bb57663-5e64-40f6-91e9-754b2eb53de7   10Gi       RWO            Delete           Bound    default/data-minio-0   standard                23m
```

#### Задание со *. Поместите данные в secret и настройте конфигурацию на их использование

Кодируем MINIO_ACCESS_KEY и MINIO_SECRET_KEY в base64:
```
$ echo -n 'minio' | base64
bWluaW8=
$ echo -n 'minio123' | base64
bWluaW8xMjM=
```

Создаем secret:
```
apiVersion: v1
kind: Secret
metadata:
  name: minio-secret
type: Opaque
data:
  access-key: bWluaW8=
  secret-key: bWluaW8xMjM=
```

```
$ kubectl get secret minio-secret -o yaml
apiVersion: v1
data:
  access-key: bWluaW8=
  secret-key: bWluaW8xMjM=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"access-key":"bWluaW8=","secret-key":"bWluaW8xMjM="},"kind":"Secret","metadata":{"annotations":{},"name":"minio-secret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2019-08-02T16:10:52Z"
  name: minio-secret
  namespace: default
  resourceVersion: "5711"
  selfLink: /api/v1/namespaces/default/secrets/minio-secret
  uid: 0bd1fa1b-3a76-496e-9e95-5ba5ac4c3f1b
type: Opaque
```

Меняем в StatefulSet:
```
        env:
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: access-key
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: secret-key   
```

```
$ kubectl describe statefulset minio
...
    Environment:
      MINIO_ACCESS_KEY:  <set to the key 'access-key' in secret 'minio-secret'>  Optional: false
      MINIO_SECRET_KEY:  <set to the key 'secret-key' in secret 'minio-secret'>  Optional: false
```

### Homework 5 (kubernetes-storage)

#### Создать кластер
`$ kind create cluster --config cluster.yaml`

#### Установить драйвер
```
$ git clone https://github.com/kubernetes-csi/csi-driver-host-path.git
$ csi-driver-host-path/deploy/kubernetes-1.13/deploy-hostpath.sh
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
csi-hostpath-attacher-0      1/1     Running   0          75s
csi-hostpath-provisioner-0   1/1     Running   0          73s
csi-hostpath-resizer-0       1/1     Running   0          73s
csi-hostpath-snapshotter-0   1/1     Running   0          72s
csi-hostpath-socat-0         1/1     Running   0          72s
csi-hostpathplugin-0         3/3     Running   0          74s

$ kubectl get sc
NAME                 PROVISIONER               AGE
csi-hostpath-sc      hostpath.csi.k8s.io       94s
standard (default)   kubernetes.io/host-path   43m
```

#### Создать StorageClass и PVC

```
$ kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
storage-pvc   Bound    pvc-63d46088-b6a8-42d0-89aa-aae89baa0e90   1Gi        RWO            csi-hostpath-sc   3s

$ kubectl get volumesnapshotclass
NAME                     AGE
csi-hostpath-snapclass   25m
```

#### Проверить снепшоты

Записываем тестовые данные

`root@storage-pod:/# echo test > /data/data`

Создаем снепшот
```
$ kubectl get volumesnapshot
NAME               AGE
storage-snapshot   26s
```

Удаляем pvc, pod
Создаем pvc из снепшота, создаем под

Все на месте
```
root@storage-pod:/# cat /data/data
test
```

### Homework 6 (kubernetes-debug)


#### kubectl debug

Устанавливаем по инструкции, версия 0.1.1

При запуске в daemonset используется старая версия агента, `image: aylei/debug-agent:0.0.1`, в ней нет ptrace и admin capbilities для debug контейнера, поэтому strace не работает.
Добавлены в версии v0.1.1: `v0.1.1 90ee6bf Add ptrace and admin capbilities for debug container`
`CapAdd:      strslice.StrSlice([]string{"SYS_PTRACE", "SYS_ADMIN"}),`
Можно поменять в daemonset манифесте на новую версию, а можно просто работать в --agentless режиме, который не использует daemonset - так и сделал
`kubectl debug POD_NAME --agentless`

#### iptables-tailer
Выполнялось на ВМ на baremetal, Centos7, calico, 2 воркера

Устанавливаем iptables-tailer

```
git clone https://github.com/piontec/netperf-operator.git
cd netperf-operator
kubectl create -f deploy/crd.yaml
kubectl create -f deploy/rbac.yaml
kubectl create -f deploy/operator.yaml
```

`$ kubectl apply -f cr.yaml`

```
$ kubectl describe netperf.app.example.com/example
Name:         example
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"app.example.com/v1alpha1","kind":"Netperf","metadata":{"annotations":{},"name":"example","namespace":"default"}}
API Version:  app.example.com/v1alpha1
Kind:         Netperf
Metadata:
  Creation Timestamp:  2019-09-20T12:02:20Z
  Generation:          4
  Resource Version:    5194034
  Self Link:           /apis/app.example.com/v1alpha1/namespaces/default/netperfs/example
  UID:                 f2bee90a-dbf7-4f07-9cfd-c4d16674833a
Spec:
  Client Node:
  Server Node:
Status:
  Client Pod:          netperf-client-c4d16674833a
  Server Pod:          netperf-server-c4d16674833a
  Speed Bits Per Sec:  103.63
  Status:              Done
Events:                <none>
```

Добавляем сетевую политику
```
$ kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-03/Debugging/netperf-calico-policy.yaml
networkpolicy.crd.projectcalico.org/netperf-calico-policy created
```
В логах ноды Kubernetes появились сообщения об отброшенных пакетах

Устанавливаем iptables-tailer.
Собрал образ по инструкции в репозитории, нашел пример daemonset в demo директории, сравнил с `iptables-tailer.yaml` из сниппета, в итоге просто создал из `iptables-tailer.yaml`)
Но `"JOURNAL_DIRECTORY" = "/var/log/journal"` на centos7 просто так работать не будет, сделал следующее:
```
mkdir /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

Создаем ServiceAccount, ClusterRole, ClusterRoleBinding
`calico-packet`, `JOURNAL_DIRECTORY`, `C-Go` - уже есть
Запускаем тест:
```
default         4m6s        Warning   PacketDrop                pod/netperf-client-d95dc03b0048                 Packet dropped when sending traffic to server (10.0.235.168)
```

#### Задание со ⭐

#### Поправьте манифест DaemonSet из репозитория, чтобы в логах отображались имена Podов, а не их IP-адреса
POD_IDENTIFIER = "name"
ip остался, но добавилось имя пода
```
default       50s         Warning   PacketDrop         pod/netperf-client-d5fcdf35bef7   Packet dropped when sending traffic to netperf-server-d5fcdf35bef7 (10.0.235.180)
```

#### Исправьте ошибку в нашей сетевой политике, чтобы Netperf снова начал работать

Можно сделать селекторы для ingress и egress netperf-type == "client" и netperf-type == "server", остальное оставить как есть, но вроде можно сократить описание:
```
spec:
  order: 10
  selector: netperf-type == "server"
  ingress:
    - action: Allow
      source:
        selector: netperf-type == "client"
    - action: Log
    - action: Deny
```
Drop пакетов нет, тест проходит:
```
Status:
  Client Pod:          netperf-client-40f3ea396501
  Server Pod:          netperf-server-40f3ea396501
  Speed Bits Per Sec:  105.82
  Status:              Done
Events:                <none>
```  
