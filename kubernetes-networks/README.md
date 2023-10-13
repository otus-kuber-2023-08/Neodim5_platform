Добавил в описание пода readinessProbe kubernetesintro/web-pod.yml
Запустил под
```
neodim@otus:~/Neodim5_platform/kubernetes-intro$ kubectl get pod/web
NAME   READY   STATUS    RESTARTS   AGE
web    0/1     Running   0          36s

```


Выполнил команду kubectl describe pod/web (вывод объемный, но в нем много интересного)

```
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
  
  
    Warning  Unhealthy  3m33s (x36 over 8m31s)  kubelet            Readiness probe failed: Get "http://10.244.0.57:80/index.html": dial tcp 10.244.0.57:80: connect: connection refused

```
проверка готовности контейнера завершается неудачно. Веб-сервер в контейнере слушает порт 8000 

добавил другой вид проверок: livenessProbe

```
livenessProbe:
 tcpSocket: { port: 8000 }
```

## Вопрос для самопроверки:

+ Почему следующая конфигурация валидна, но не имеет смысла?
```
livenessProbe:
  exec:
    command:
      - 'sh'
      - '-c'
      - 'ps aux | grep my_web_server_process'
```
Данная конфигурация не имеет смысла, потому что совсем не означает, что наш POD (работающий веб сервер) без ошибок отдает веб страницу.

+ Бывают ли ситуации, когда она все-таки имеет смысл?
Возможно, когда требуется проверка работы или проверки доступа до самого POD, без доступа к нему из вне.

## Создание Deployment

Создал Deployment, который упростит обновление конфигурации пода и управление группами подов.
kubernetes-networks/web-deploy.yaml

Поскольку не исправил ReadinessProbe , то поды, входящие в Deployment, не переходят в состояние Ready из-за неуспешной проверки.
Исправил порт в readinessProbe на порт 8000.

```
neodim@otus:~/Neodim5_platform/kubernetes-networks$ kubectl apply -f web-deploy.yaml
deployment.apps/web configured

neodim@otus:~/Neodim5_platform/kubernetes-networks$ kubectl get po | grep web
web-88c76dfff-29d7x               1/1     Running   0              27m
web-88c76dfff-9tc7h               1/1     Running   0              20m
web-88c76dfff-9z6rh               1/1     Running   0              20m
```

Проверил состояние Deployment
```
neodim@otus:~/Neodim5_platform/kubernetes-networks$ kubectl describe deploy/web
~~~
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
~~~
```
Добавил в манифест ( web-deploy.yaml ) блок strategy
```
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 100%
```


Установил kubespy
```
neodim@otus:~$ kubespy trace deploy web
[ADDED apps/v1/Deployment]  default/web
    Rolling out Deployment revision 1
    ✅ Deployment is currently available
    ✅ Rollout successful: new ReplicaSet marked 'available'

ROLLOUT STATUS:
- [Current rollout | Revision 1] [ADDED]  default/web-88c76dfff
    ✅ ReplicaSet is available [3 Pods available of a 3 minimum]
       - [Ready] web-88c76dfff-9tc7h
       - [Ready] web-88c76dfff-9z6rh
       - [Ready] web-88c76dfff-29d7x

```

Попробал разные варианты деплоя с крайними значениями maxSurge и maxUnavailable (оба 0, оба 100%, 0 и 100%)
 + оба 0
   ```
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 100%
        maxSurge: 0
   ```

   ```
   neodim@otus:~/Neodim5_platform/kubernetes-networks$ k apply -f web-deploy.yaml
   The Deployment "web" is invalid: spec.strategy.rollingUpdate.maxUnavailable: Invalid value: intstr.IntOrString{Type:0, IntVal:0, StrVal:""}: may not be 0 when `maxSurge` is 0
   ```

оба значения не могут быть одновременно равны 0

  + maxUnavailable 100% maxSurge 0


    ```
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 100%
        maxSurge: 0
    ```

  удаление всех 3-х старых подов и затем создание трех новых


  + Оба 100%


   ```
   strategy:
     type: RollingUpdate
     rollingUpdate:
       maxUnavailable: 100%
       maxSurge: 100%
   ```

   Одновременное удаление 3-х старых и создание 3-х новых подов


## Создание Service

Для того, чтобы наше приложение было доступно внутри кластера (а тем
более - снаружи), нам потребуется объект типа Service . Начнем с самого
распространенного типа сервисов - ClusterIP .

создал манифест для сервиса в папке kubernetes-networks -  web-svc-cip.yaml.

```
apiVersion: v1
kind: Service
metadata:
  name: web-svc-cip
spec:
  selector:
    app: web
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

```
neodim@otus:~/Neodim5_platform/kubernetes-networks$ kubectl get services
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
web-svc-cip      ClusterIP      10.104.146.198   <none>        80/TCP           17d
```

Сделал curl http://10.104.146.198/index.html

```
neodim@otus:~$ minikube ssh
Last login: Fri Oct 13 07:14:47 2023 from 192.168.49.1
docker@minikube:~$ sudo -i
root@minikube:~# curl http://10.104.146.198/index.html
<html>
<head/>
<body>
<!-- IMAGE BEGINS HERE -->
<!-- IMAGE ENDS HERE -->
<!-- TEST PHRASE -->
<h3>Mountpoints</h3>
<pre>overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/AZX3EUAEGZFIBAEBPJKOIB4Q2A:/var/lib/docker/overlay2/l/43ZRDMNA3UJ7XVEENJVYWYHWHV,upperdir=/var/lib/docker/overlay2/66979ba2f734f824d7a6a1fb35a1407965c4638801c24f0b551b7eefe1c8c2e8/diff,workdir=/var/lib/docker/overlay2/66979ba2f734f824d7a6a1fb35a1407965c4638801c24f0b551b7eefe1c8c2e8/work,xino=off)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
sysfs on /sys type sysfs (ro,nosuid,nodev,noexec,relatime)
tmpfs on /sys/fs/cgroup type tmpfs (rw,nosuid,nodev,noexec,relatime,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (ro,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (ro,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/perf_event type cgroup (ro,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (ro,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/pids type cgroup (ro,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (ro,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/rdma type cgroup (ro,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/blkio type cgroup (ro,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/devices type cgroup (ro,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (ro,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/memory type cgroup (ro,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/cpuset type cgroup (ro,nosuid,nodev,noexec,relatime,cpuset)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
/dev/mapper/ubuntu--vg-ubuntu--lv on /app type ext4 (rw,relatime)
/dev/mapper/ubuntu--vg-ubuntu--lv on /dev/termination-log type ext4 (rw,relatime)
/dev/mapper/ubuntu--vg-ubuntu--lv on /etc/resolv.conf type ext4 (rw,relatime)
/dev/mapper/ubuntu--vg-ubuntu--lv on /etc/hostname type ext4 (rw,relatime)
/dev/mapper/ubuntu--vg-ubuntu--lv on /etc/hosts type ext4 (rw,relatime)
shm on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k)
tmpfs on /var/run/secrets/kubernetes.io/serviceaccount type tmpfs (ro,relatime,size=4013596k)
proc on /proc/bus type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/fs type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/irq type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sys type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sysrq-trigger type proc (ro,nosuid,nodev,noexec,relatime)
tmpfs on /proc/acpi type tmpfs (ro,relatime)
tmpfs on /proc/kcore type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/keys type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/timer_list type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/sched_debug type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/scsi type tmpfs (ro,relatime)
tmpfs on /sys/firmware type tmpfs (ro,relatime)</pre>
<h3>Environment</h3>
<pre>export BALANCED_PORT='tcp://10.97.97.75:8080'
export BALANCED_PORT_8080_TCP='tcp://10.97.97.75:8080'
export BALANCED_PORT_8080_TCP_ADDR='10.97.97.75'
export BALANCED_PORT_8080_TCP_PORT='8080'
export BALANCED_PORT_8080_TCP_PROTO='tcp'
export BALANCED_SERVICE_HOST='10.97.97.75'
export BALANCED_SERVICE_PORT='8080'
export HELLO_MINIKUBE_PORT='tcp://10.106.192.17:8080'
export HELLO_MINIKUBE_PORT_8080_TCP='tcp://10.106.192.17:8080'
export HELLO_MINIKUBE_PORT_8080_TCP_ADDR='10.106.192.17'
export HELLO_MINIKUBE_PORT_8080_TCP_PORT='8080'
export HELLO_MINIKUBE_PORT_8080_TCP_PROTO='tcp'
export HELLO_MINIKUBE_SERVICE_HOST='10.106.192.17'
export HELLO_MINIKUBE_SERVICE_PORT='8080'
export HOME='/root'
export HOSTNAME='web-88c76dfff-9tc7h'
export KUBERNETES_PORT='tcp://10.96.0.1:443'
export KUBERNETES_PORT_443_TCP='tcp://10.96.0.1:443'
export KUBERNETES_PORT_443_TCP_ADDR='10.96.0.1'
export KUBERNETES_PORT_443_TCP_PORT='443'
export KUBERNETES_PORT_443_TCP_PROTO='tcp'
export KUBERNETES_SERVICE_HOST='10.96.0.1'
export KUBERNETES_SERVICE_PORT='443'
export KUBERNETES_SERVICE_PORT_HTTPS='443'
export KUBIA_PORT='tcp://10.104.214.194:80'
export KUBIA_PORT_80_TCP='tcp://10.104.214.194:80'
export KUBIA_PORT_80_TCP_ADDR='10.104.214.194'
export KUBIA_PORT_80_TCP_PORT='80'
export KUBIA_PORT_80_TCP_PROTO='tcp'
export KUBIA_SERVICE_HOST='10.104.214.194'
export KUBIA_SERVICE_PORT='80'
export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
export PWD='/'
export SHLVL='2'</pre>
<h3>Memory info</h3>
<pre>              total        used        free      shared  buff/cache   available
Mem:           3920        2197         135          27        1587        1422
Swap:          3931          58        3873</pre>
<h3>DNS resolvers info</h3>
<pre>nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local lan
options ndots:5</pre>
<h3>Static hosts info</h3>
<pre># Kubernetes-managed hosts file.
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
fe00::0 ip6-mcastprefix
fe00::1 ip6-allnodes
fe00::2 ip6-allrouters
10.244.0.61     web-88c76dfff-9tc7h</pre>
</body>
</html>

```

+ Сделал ping 10.104.146.198 - пинга нет
  ```
  root@minikube:~# ping 10.104.214.194
  PING 10.104.214.194 (10.104.214.194) 56(84) bytes of data.
  ^C
  --- 10.104.214.194 ping statistics ---
  4 packets transmitted, 0 received, 100% packet loss, time 3079ms

  ```

+ Сделал arp -an , ip addr show - нигде нет ClusterIP

  ```
  root@minikube:~# arp -an
  -bash: arp: command not found

  root@minikube:~# ip addr show
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  2: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
      link/ether 02:42:ea:43:7c:9d brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
         valid_lft forever preferred_lft forever
  3: bridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
      link/ether fe:7e:c2:54:78:79 brd ff:ff:ff:ff:ff:ff
      inet 10.244.0.1/16 brd 10.244.255.255 scope global bridge
         valid_lft forever preferred_lft forever
  4: veth09b207ed@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether 02:fe:fb:6f:b2:7e brd ff:ff:ff:ff:ff:ff link-netnsid 1
  5: vethe904f0fa@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether 56:75:74:48:81:60 brd ff:ff:ff:ff:ff:ff link-netnsid 2
  6: veth12575e84@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether a6:77:1a:db:48:23 brd ff:ff:ff:ff:ff:ff link-netnsid 3
  7: veth70c15e0e@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether da:66:3f:b5:f7:35 brd ff:ff:ff:ff:ff:ff link-netnsid 4
  8: vethc180fb81@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether 5a:f5:74:7d:a4:d6 brd ff:ff:ff:ff:ff:ff link-netnsid 5
  9: veth01c4fdcf@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether b6:bd:d8:f0:7d:1a brd ff:ff:ff:ff:ff:ff link-netnsid 6
  10: vethb068d62e@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether be:e4:de:84:ab:39 brd ff:ff:ff:ff:ff:ff link-netnsid 7
  11: veth0a5d0d65@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether 3e:0e:ab:3a:9d:21 brd ff:ff:ff:ff:ff:ff link-netnsid 8
  12: veth7d5eb048@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether ae:18:0d:c7:97:6b brd ff:ff:ff:ff:ff:ff link-netnsid 9
  13: veth9c3621eb@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether 1e:51:b0:e5:ae:c5 brd ff:ff:ff:ff:ff:ff link-netnsid 10
  14: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
      link/ether 02:42:c0:a8:31:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 192.168.49.2/24 brd 192.168.49.255 scope global eth0
         valid_lft forever preferred_lft forever
  15: veth36911a0f@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master bridge state UP group default
      link/ether a6:b0:9a:ff:bc:85 brd ff:ff:ff:ff:ff:ff link-netnsid 11

  ```


+ Сделал iptables --list -nv -t nat

```
root@minikube:~# iptables --list -nv -t nat
Chain PREROUTING (policy ACCEPT 7 packets, 420 bytes)
 pkts bytes target     prot opt in     out     source               destination
   34  2166 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
    1    84 DOCKER_OUTPUT  all  --  *      *       0.0.0.0/0            192.168.49.1
   24  1440 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
    9   540 CNI-HOSTPORT-DNAT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 8 packets, 480 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 5721 packets, 343K bytes)
 pkts bytes target     prot opt in     out     source               destination
 5886  353K KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
    9   654 DOCKER_OUTPUT  all  --  *      *       0.0.0.0/0            192.168.49.1
 2314  139K DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL
 2968  178K CNI-HOSTPORT-DNAT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 5721 packets, 343K bytes)
 pkts bytes target     prot opt in     out     source               destination
 5886  353K KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
 5869  352K CNI-HOSTPORT-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* CNI portfwd requiring masquerade */
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
    0     0 DOCKER_POSTROUTING  all  --  *      *       0.0.0.0/0            192.168.49.1
    0     0 CNI-73368b0da5da5d8832111404  all  --  *      *       10.244.0.84          0.0.0.0/0            /* name: "bridge" id: "7dd1a9e3dc9fe5604902ff2c6f6dc81f658c3f0dd5c75461c4e881ab6d6951e8" */
    0     0 CNI-d5503afc80dc31006a400cff  all  --  *      *       10.244.0.85          0.0.0.0/0            /* name: "bridge" id: "64df66c5acfb17fbb46a263a4148ea6a8925dc5832f4635f0d460a4b079334d4" */
    1    60 CNI-f5ffcae3218f8c41164a3156  all  --  *      *       10.244.0.87          0.0.0.0/0            /* name: "bridge" id: "7f19c44d3a22cc8b16b04c36a420e326b8ded07e5de4d1128d151c91053f43d2" */
    0     0 CNI-8bcd9c6c8dec31567cf15547  all  --  *      *       10.244.0.86          0.0.0.0/0            /* name: "bridge" id: "20974be256a5f3c8af88257650b30bcc157b784f88aacd6b935f97b1ede9629e" */
    0     0 CNI-6ffcad103a656a215bdc3397  all  --  *      *       10.244.0.88          0.0.0.0/0            /* name: "bridge" id: "925cc6d72a5093f959796c47bf7646fdb94f5f4cfe74e7b91a55f40752ad7b50" */
    0     0 CNI-04be4f3c82c4f68cc0112b0c  all  --  *      *       10.244.0.89          0.0.0.0/0            /* name: "bridge" id: "341d3b7cf8f9791adfa621c039c315fcd262bec2273a04faf83717d7619aa607" */
    1    83 CNI-7238fcdf5ec94e8bd68e01da  all  --  *      *       10.244.0.90          0.0.0.0/0            /* name: "bridge" id: "911f22bc2cc9737a7f0f290bb474a34dd0e329254bfe6df961d9c148baacfd09" */
    1    83 CNI-e6a171fd4b5cf448fc2e7ef4  all  --  *      *       10.244.0.92          0.0.0.0/0            /* name: "bridge" id: "64127b9e950734e4e5e6acd62947dd8fde69d2daf0d5cbcdc2ed8a9e0f1e315f" */
    0     0 CNI-5d93e062b449575084054cc5  all  --  *      *       10.244.0.93          0.0.0.0/0            /* name: "bridge" id: "7086745daa78d61d3c8d6f329e0d4b608265110eba3a5bbf48f0074be241787a" */
    1    83 CNI-5204c55a933810b16b6c262f  all  --  *      *       10.244.0.91          0.0.0.0/0            /* name: "bridge" id: "81c028f73b1776e71b5ead39e407449a3ad239b31b1439677f2f4d6dc60c75e3" */
    1    60 CNI-01f52f2192ca5fb49c978e7a  all  --  *      *       10.244.0.94          0.0.0.0/0            /* name: "bridge" id: "4b6e2dfa9280acc3c6eb386792c5f53290cd192dba20237fc38b0f2c9ab7412b" */

Chain CNI-01f52f2192ca5fb49c978e7a (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "4b6e2dfa9280acc3c6eb386792c5f53290cd192dba20237fc38b0f2c9ab7412b" */
    1    60 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "4b6e2dfa9280acc3c6eb386792c5f53290cd192dba20237fc38b0f2c9ab7412b" */

Chain CNI-04be4f3c82c4f68cc0112b0c (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "341d3b7cf8f9791adfa621c039c315fcd262bec2273a04faf83717d7619aa607" */
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "341d3b7cf8f9791adfa621c039c315fcd262bec2273a04faf83717d7619aa607" */

Chain CNI-5204c55a933810b16b6c262f (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "81c028f73b1776e71b5ead39e407449a3ad239b31b1439677f2f4d6dc60c75e3" */
    1    83 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "81c028f73b1776e71b5ead39e407449a3ad239b31b1439677f2f4d6dc60c75e3" */

Chain CNI-5d93e062b449575084054cc5 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "7086745daa78d61d3c8d6f329e0d4b608265110eba3a5bbf48f0074be241787a" */
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "7086745daa78d61d3c8d6f329e0d4b608265110eba3a5bbf48f0074be241787a" */

Chain CNI-6ffcad103a656a215bdc3397 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "925cc6d72a5093f959796c47bf7646fdb94f5f4cfe74e7b91a55f40752ad7b50" */
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "925cc6d72a5093f959796c47bf7646fdb94f5f4cfe74e7b91a55f40752ad7b50" */

Chain CNI-7238fcdf5ec94e8bd68e01da (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "911f22bc2cc9737a7f0f290bb474a34dd0e329254bfe6df961d9c148baacfd09" */
    1    83 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "911f22bc2cc9737a7f0f290bb474a34dd0e329254bfe6df961d9c148baacfd09" */

Chain CNI-73368b0da5da5d8832111404 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "7dd1a9e3dc9fe5604902ff2c6f6dc81f658c3f0dd5c75461c4e881ab6d6951e8" */
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "7dd1a9e3dc9fe5604902ff2c6f6dc81f658c3f0dd5c75461c4e881ab6d6951e8" */

Chain CNI-8bcd9c6c8dec31567cf15547 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "20974be256a5f3c8af88257650b30bcc157b784f88aacd6b935f97b1ede9629e" */
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "20974be256a5f3c8af88257650b30bcc157b784f88aacd6b935f97b1ede9629e" */

Chain CNI-DN-5d93e062b449575084054 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       10.244.0.0/16        0.0.0.0/0            tcp dpt:80
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       127.0.0.1            0.0.0.0/0            tcp dpt:80
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:10.244.0.93:80
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       10.244.0.0/16        0.0.0.0/0            tcp dpt:443
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       127.0.0.1            0.0.0.0/0            tcp dpt:443
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443 to:10.244.0.93:443

Chain CNI-HOSTPORT-DNAT (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 CNI-DN-5d93e062b449575084054  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* dnat name: "bridge" id: "7086745daa78d61d3c8d6f329e0d4b608265110eba3a5bbf48f0074be241787a" */ multiport dports 80,443

Chain CNI-HOSTPORT-MASQ (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match 0x2000/0x2000

Chain CNI-HOSTPORT-SETMARK (4 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* CNI portfwd masquerade mark */ MARK or 0x2000

Chain CNI-d5503afc80dc31006a400cff (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "64df66c5acfb17fbb46a263a4148ea6a8925dc5832f4635f0d460a4b079334d4" */
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "64df66c5acfb17fbb46a263a4148ea6a8925dc5832f4635f0d460a4b079334d4" */

Chain CNI-e6a171fd4b5cf448fc2e7ef4 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "64127b9e950734e4e5e6acd62947dd8fde69d2daf0d5cbcdc2ed8a9e0f1e315f" */
    1    83 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "64127b9e950734e4e5e6acd62947dd8fde69d2daf0d5cbcdc2ed8a9e0f1e315f" */

Chain CNI-f5ffcae3218f8c41164a3156 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "7f19c44d3a22cc8b16b04c36a420e326b8ded07e5de4d1128d151c91053f43d2" */
    1    60 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "7f19c44d3a22cc8b16b04c36a420e326b8ded07e5de4d1128d151c91053f43d2" */

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER_OUTPUT (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            192.168.49.1         tcp dpt:53 to:127.0.0.11:45811
   10   738 DNAT       udp  --  *      *       0.0.0.0/0            192.168.49.1         udp dpt:53 to:127.0.0.11:54942

Chain DOCKER_POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 SNAT       tcp  --  *      *       127.0.0.11           0.0.0.0/0            tcp spt:45811 to:192.168.49.1:53
    0     0 SNAT       udp  --  *      *       127.0.0.11           0.0.0.0/0            udp spt:54942 to:192.168.49.1:53

Chain KUBE-EXT-CG5I4G2RS3ZVWGLK (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade traffic for ingress-nginx/ingress-nginx-controller:http external destinations */
    0     0 KUBE-SVC-CG5I4G2RS3ZVWGLK  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain KUBE-EXT-EDNDUDH2C75GIR6O (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade traffic for ingress-nginx/ingress-nginx-controller:https external destinations */
    0     0 KUBE-SVC-EDNDUDH2C75GIR6O  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain KUBE-EXT-JTZOBYERWP2AKFAC (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade traffic for default/balanced external destinations */
    0     0 KUBE-SVC-JTZOBYERWP2AKFAC  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain KUBE-EXT-MFJHED5Y2WHWJ6HX (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade traffic for default/hello-minikube external destinations */
    0     0 KUBE-SVC-MFJHED5Y2WHWJ6HX  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain KUBE-KUBELET-CANARY (0 references)
 pkts bytes target     prot opt in     out     source               destination

Chain KUBE-MARK-MASQ (34 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-EXT-CG5I4G2RS3ZVWGLK  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:http */ tcp dpt:31090
    0     0 KUBE-EXT-JTZOBYERWP2AKFAC  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/balanced */ tcp dpt:31715
    0     0 KUBE-EXT-EDNDUDH2C75GIR6O  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:https */ tcp dpt:31371
    0     0 KUBE-EXT-MFJHED5Y2WHWJ6HX  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/hello-minikube */ tcp dpt:30881

Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination
 5721  343K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match ! 0x4000/0x4000
    2   120 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK xor 0x4000
    2   120 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ random-fully

Chain KUBE-PROXY-CANARY (0 references)
 pkts bytes target     prot opt in     out     source               destination

Chain KUBE-SEP-3YGF6NQAASH74MR3 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.93          0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller-admission:https-webhook */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller-admission:https-webhook */ tcp to:10.244.0.93:8443

Chain KUBE-SEP-52JMIFDGEAJCXDSX (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.89          0.0.0.0/0            /* kubernetes-dashboard/dashboard-metrics-scraper */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/dashboard-metrics-scraper */ tcp to:10.244.0.89:8000

Chain KUBE-SEP-6MHCCEU3MDPYWZAY (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.87          0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ tcp to:10.244.0.87:53

Chain KUBE-SEP-6WHBMPQHNC3XRLVX (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.93          0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:https */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:https */ tcp to:10.244.0.93:443

Chain KUBE-SEP-BRQBNTRUL2PB3MWA (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.87          0.0.0.0/0            /* kube-system/kube-dns:metrics */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics */ tcp to:10.244.0.87:9153

Chain KUBE-SEP-BSSUJNUPH4MV3436 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.91          0.0.0.0/0            /* default/web-svc-cip */
    1    60 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ tcp to:10.244.0.91:8000

Chain KUBE-SEP-CDBP7NGOHNJJU4OC (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.84          0.0.0.0/0            /* default/balanced */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/balanced */ tcp to:10.244.0.84:8080

Chain KUBE-SEP-CYJE3JM7TSZFQCI4 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.90          0.0.0.0/0            /* default/web-svc-cip */
    1    60 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ tcp to:10.244.0.90:8000

Chain KUBE-SEP-HRVJBNWSGEBVJWGG (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.92          0.0.0.0/0            /* default/web-svc-cip */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ tcp to:10.244.0.92:8000

Chain KUBE-SEP-KUYMYUNOEROJEKGI (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.94          0.0.0.0/0            /* kubernetes-dashboard/kubernetes-dashboard */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/kubernetes-dashboard */ tcp to:10.244.0.94:9090

Chain KUBE-SEP-L5OR7NER5SK3XH6F (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.93          0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:http */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:http */ tcp to:10.244.0.93:80

Chain KUBE-SEP-LI34KE3T2TGPGXAY (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.88          0.0.0.0/0            /* default/kubia */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubia */ tcp to:10.244.0.88:8080

Chain KUBE-SEP-PSDVXZCOGHGWDEJK (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.86          0.0.0.0/0            /* default/hello-minikube */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/hello-minikube */ tcp to:10.244.0.86:8080

Chain KUBE-SEP-RLQQ3ENOLGJ5OBB6 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.85          0.0.0.0/0            /* kube-system/metrics-server:https */
    6   360 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/metrics-server:https */ tcp to:10.244.0.85:4443

Chain KUBE-SEP-VPILYQBSPPXYB66K (1 references)
 pkts bytes target     prot opt in     out     source               destination
    1    60 KUBE-MARK-MASQ  all  --  *      *       192.168.49.2         0.0.0.0/0            /* default/kubernetes:https */
    6   360 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */ tcp to:192.168.49.2:8443

Chain KUBE-SEP-W7Y6XSEUY34M5DB2 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.87          0.0.0.0/0            /* kube-system/kube-dns:dns */
    0     0 DNAT       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ udp to:10.244.0.87:53

Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination
    1    60 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-Z6GDYMWE5TV2NNJN  tcp  --  *      *       0.0.0.0/0            10.101.0.40          /* kubernetes-dashboard/dashboard-metrics-scraper cluster IP */ tcp dpt:8000
    2   120 KUBE-SVC-6CZTMAROCN3AQODZ  tcp  --  *      *       0.0.0.0/0            10.104.146.198       /* default/web-svc-cip cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-CG5I4G2RS3ZVWGLK  tcp  --  *      *       0.0.0.0/0            10.107.15.226        /* ingress-nginx/ingress-nginx-controller:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SVC-JTZOBYERWP2AKFAC  tcp  --  *      *       0.0.0.0/0            10.97.97.75          /* default/balanced cluster IP */ tcp dpt:8080
    0     0 KUBE-SVC-EDNDUDH2C75GIR6O  tcp  --  *      *       0.0.0.0/0            10.107.15.226        /* ingress-nginx/ingress-nginx-controller:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-EZYNCFY2F7N6OQA2  tcp  --  *      *       0.0.0.0/0            10.110.68.233        /* ingress-nginx/ingress-nginx-controller-admission:https-webhook cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-MFJHED5Y2WHWJ6HX  tcp  --  *      *       0.0.0.0/0            10.106.192.17        /* default/hello-minikube cluster IP */ tcp dpt:8080
    0     0 KUBE-SVC-Z4ANX4WAEWEBLCTM  tcp  --  *      *       0.0.0.0/0            10.98.89.139         /* kube-system/metrics-server:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-XCCRWSFD2BRL7CRI  tcp  --  *      *       0.0.0.0/0            10.104.214.194       /* default/kubia cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-CEZPIJSAUFW5MYPQ  tcp  --  *      *       0.0.0.0/0            10.105.213.191       /* kubernetes-dashboard/kubernetes-dashboard cluster IP */ tcp dpt:80
 2896  174K KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

Chain KUBE-SVC-6CZTMAROCN3AQODZ (1 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.104.146.198       /* default/web-svc-cip cluster IP */ tcp dpt:80
    1    60 KUBE-SEP-CYJE3JM7TSZFQCI4  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip -> 10.244.0.90:8000 */ statistic mode random probability 0.33333333349
    1    60 KUBE-SEP-BSSUJNUPH4MV3436  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip -> 10.244.0.91:8000 */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-HRVJBNWSGEBVJWGG  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip -> 10.244.0.92:8000 */

Chain KUBE-SVC-CEZPIJSAUFW5MYPQ (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.105.213.191       /* kubernetes-dashboard/kubernetes-dashboard cluster IP */ tcp dpt:80
    0     0 KUBE-SEP-KUYMYUNOEROJEKGI  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/kubernetes-dashboard -> 10.244.0.94:9090 */

Chain KUBE-SVC-CG5I4G2RS3ZVWGLK (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.107.15.226        /* ingress-nginx/ingress-nginx-controller:http cluster IP */ tcp dpt:80
    0     0 KUBE-SEP-L5OR7NER5SK3XH6F  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:http -> 10.244.0.93:80 */

Chain KUBE-SVC-EDNDUDH2C75GIR6O (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.107.15.226        /* ingress-nginx/ingress-nginx-controller:https cluster IP */ tcp dpt:443
    0     0 KUBE-SEP-6WHBMPQHNC3XRLVX  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller:https -> 10.244.0.93:443 */

Chain KUBE-SVC-ERIFXISQEP7F7OF4 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SEP-6MHCCEU3MDPYWZAY  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp -> 10.244.0.87:53 */

Chain KUBE-SVC-EZYNCFY2F7N6OQA2 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.110.68.233        /* ingress-nginx/ingress-nginx-controller-admission:https-webhook cluster IP */ tcp dpt:443
    0     0 KUBE-SEP-3YGF6NQAASH74MR3  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ingress-nginx/ingress-nginx-controller-admission:https-webhook -> 10.244.0.93:8443 */

Chain KUBE-SVC-JD5MR3NA4I4DYORP (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SEP-BRQBNTRUL2PB3MWA  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics -> 10.244.0.87:9153 */

Chain KUBE-SVC-JTZOBYERWP2AKFAC (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.97.97.75          /* default/balanced cluster IP */ tcp dpt:8080
    0     0 KUBE-SEP-CDBP7NGOHNJJU4OC  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/balanced -> 10.244.0.84:8080 */

Chain KUBE-SVC-MFJHED5Y2WHWJ6HX (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.106.192.17        /* default/hello-minikube cluster IP */ tcp dpt:8080
    0     0 KUBE-SEP-PSDVXZCOGHGWDEJK  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/hello-minikube -> 10.244.0.86:8080 */

Chain KUBE-SVC-NPX46M4PTMTKRN6Y (1 references)
 pkts bytes target     prot opt in     out     source               destination
    1    60 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    6   360 KUBE-SEP-VPILYQBSPPXYB66K  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https -> 192.168.49.2:8443 */

Chain KUBE-SVC-TCOU7JCQXEZGVUNU (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  udp  --  *      *      !10.244.0.0/16        10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SEP-W7Y6XSEUY34M5DB2  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns -> 10.244.0.87:53 */

Chain KUBE-SVC-XCCRWSFD2BRL7CRI (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.104.214.194       /* default/kubia cluster IP */ tcp dpt:80
    0     0 KUBE-SEP-LI34KE3T2TGPGXAY  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubia -> 10.244.0.88:8080 */

Chain KUBE-SVC-Z4ANX4WAEWEBLCTM (1 references)
 pkts bytes target     prot opt in     out     source               destination
    6   360 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.98.89.139         /* kube-system/metrics-server:https cluster IP */ tcp dpt:443
    6   360 KUBE-SEP-RLQQ3ENOLGJ5OBB6  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/metrics-server:https -> 10.244.0.85:4443 */

Chain KUBE-SVC-Z6GDYMWE5TV2NNJN (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.101.0.40          /* kubernetes-dashboard/dashboard-metrics-scraper cluster IP */ tcp dpt:8000
    0     0 KUBE-SEP-52JMIFDGEAJCXDSX  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/dashboard-metrics-scraper -> 10.244.0.89:8000 */
```

```
root@minikube:~# iptables --list -nv -t nat | grep 10.104.214.194
    0     0 KUBE-SVC-XCCRWSFD2BRL7CRI  tcp  --  *      *       0.0.0.0/0            10.104.214.194       /* default/kubia cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.104.214.194       /* default/kubia cluster IP */ tcp dpt:80

```


## Включение IPVS

Включил IPVS для kube-proxy , исправив ConfigMap (конфигурация Pod, хранящаяся в кластере)

файле конфигурации kube-proxy строку mode: "" исправил на "ipvs" и добавил параметр strictARP: true.


Теперь удалим Pod с kube-proxy , чтобы применить новую конфигурацию 
```
kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
```

После успешного рестарта kube-proxy проверим, что получилось.

Выполним команду iptables --list -nv -t nat в ВМ Minikube

```
neodim@otus:~$ minikube ssh
Last login: Fri Oct 13 07:23:14 2023 from 192.168.49.1
docker@minikube:~$ sudo -i
root@minikube:~# iptables --list -nv -t nat | grep 10.104.214.194
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.104.214.194       /* default/kubia cluster IP */ tcp dpt:80
root@minikube:~#

```
Что-то поменялось, но старые цепочки на месте (хотя у них теперь 0 references) 

Полностью очистим все правила iptables

Создадим в ВМ с Minikube файл /tmp/iptables.cleanup
```
 *nat
 -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
 COMMIT
 *filter
 COMMIT
 *mangle
 COMMIT
```

Применим конфигурацию: iptables-restore /tmp/iptables.cleanup

Как посмотреть конфигурацию IPVS?
Можно использовать встроенную поддержку IPVS в Kubernetes. Для этого выполните команду kubectl get service -o yaml.

```
neodim@otus:~$ kubectl get service -o yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: "2023-09-08T13:45:25Z"
    labels:
      app: balanced
    name: balanced
    namespace: default
    resourceVersion: "1814"
    uid: 9cf85c3a-6546-4492-aa6f-9ba9c7d9b804
  spec:
    allocateLoadBalancerNodePorts: true
    clusterIP: 10.97.97.75
    clusterIPs:
    - 10.97.97.75
    externalTrafficPolicy: Cluster
    internalTrafficPolicy: Cluster
    ipFamilies:
    - IPv4
    ipFamilyPolicy: SingleStack
    ports:
    - nodePort: 31715
      port: 8080
      protocol: TCP
      targetPort: 8080
    selector:
      app: balanced
    sessionAffinity: None
    type: LoadBalancer
  status:
    loadBalancer: {}
- apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: "2023-09-08T13:33:01Z"
    labels:
      app: hello-minikube
    name: hello-minikube
    namespace: default
    resourceVersion: "902"
    uid: d8c2a6ae-4abf-41db-a6b2-8e873da67401
  spec:
    clusterIP: 10.106.192.17
    clusterIPs:
    - 10.106.192.17
    externalTrafficPolicy: Cluster
    internalTrafficPolicy: Cluster
    ipFamilies:
    - IPv4
    ipFamilyPolicy: SingleStack
    ports:
    - nodePort: 30881
      port: 8080
      protocol: TCP
      targetPort: 8080
    selector:
      app: hello-minikube
    sessionAffinity: None
    type: NodePort
  status:
    loadBalancer: {}
- apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: "2023-09-08T13:25:23Z"
    labels:
      component: apiserver
      provider: kubernetes
    name: kubernetes
    namespace: default
    resourceVersion: "231"
    uid: 32273a73-4df5-4b1f-979c-7e3e6a283723
  spec:
    clusterIP: 10.96.0.1
    clusterIPs:
    - 10.96.0.1
    internalTrafficPolicy: Cluster
    ipFamilies:
    - IPv4
    ipFamilyPolicy: SingleStack
    ports:
    - name: https
      port: 443
      protocol: TCP
      targetPort: 8443
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"kubia","namespace":"default"},"spec":{"ports":[{"port":80,"targetPort":8080}],"selector":{"app":"kubia"}}}
    creationTimestamp: "2023-09-08T17:16:36Z"
    name: kubia
    namespace: default
    resourceVersion: "12599"
    uid: 8ff8f91e-cd97-4715-8d7c-73233237049b
  spec:
    clusterIP: 10.104.214.194
    clusterIPs:
    - 10.104.214.194
    internalTrafficPolicy: Cluster
    ipFamilies:
    - IPv4
    ipFamilyPolicy: SingleStack
    ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
    selector:
      app: kubia
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"web-svc-cip","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":8000}],"selector":{"app":"web"},"type":"ClusterIP"}}
    creationTimestamp: "2023-09-25T17:18:27Z"
    name: web-svc-cip
    namespace: default
    resourceVersion: "80571"
    uid: 8dc5e407-feaf-4ee9-af7d-d6a164f731e6
  spec:
    clusterIP: 10.104.146.198
    clusterIPs:
    - 10.104.146.198
    internalTrafficPolicy: Cluster
    ipFamilies:
    - IPv4
    ipFamilyPolicy: SingleStack
    ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
    selector:
      app: web
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
kind: List
metadata:
  resourceVersion: ""

```

Теперь сделаем ping кластерного IP

```
root@minikube:~# ping 10.104.214.194
PING 10.104.214.194 (10.104.214.194) 56(84) bytes of data.
From 10.104.214.194 icmp_seq=1 Destination Port Unreachable
From 10.104.214.194 icmp_seq=2 Destination Port Unreachable
From 10.104.214.194 icmp_seq=3 Destination Port Unreachable
From 10.104.214.194 icmp_seq=4 Destination Port Unreachable
From 10.104.214.194 icmp_seq=5 Destination Port Unreachable
From 10.104.214.194 icmp_seq=6 Destination Port Unreachable
From 10.104.214.194 icmp_seq=7 Destination Port Unreachable
^C
--- 10.104.214.194 ping statistics ---
7 packets transmitted, 0 received, +7 errors, 100% packet loss, time 6128ms
```

Этот IP теперь есть на интерфейсе kube-ipvs0
```
root@minikube:~# ip addr show kube-ipvs0
16: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether 4e:ad:ce:22:58:c1 brd ff:ff:ff:ff:ff:ff
    inet 10.97.97.75/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.10/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.105.213.191/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.101.0.40/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.106.192.17/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.110.68.233/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.1/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.104.214.194/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.104.146.198/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.107.15.226/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.98.89.139/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```

## Установка MetalLB

MetalLB позволяет запустить внутри кластера L4-балансировщик, который будет принимать извне запросы к сервисам и раскидывать их между подами

Проверим, что были созданы нужные объекты

```
kubectl --namespace metallb-system get all

NAME                              READY   STATUS              RESTARTS   AGE
pod/controller-64f57db87d-pmm94   0/1     ImagePullBackOff    0          2m24s
pod/speaker-c8hlb                 0/1     ContainerCreating   0          2m24s

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.111.193.1   <none>        443/TCP   2m24s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   1         1         0       1            0           kubernetes.io/os=linux   2m24s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   0/1     1            0           2m24s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-64f57db87d   1         1         0       2m24s
```

настроим балансировщик с помощью ConfigMap

Создал манифест metallb-config.yaml в папке kubernetesnetworks:

```
apiVersion: v1
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
      - "172.17.255.1-172.17.255.255"
```

kubectl apply -f metallb-config.yaml

Сделал копию файла web-svc-cip.yaml в web-svc-lb.yaml.
Изменил имя сервиса и его тип на LoadBalancer.

LB так и не заработал...





