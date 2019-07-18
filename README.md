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
