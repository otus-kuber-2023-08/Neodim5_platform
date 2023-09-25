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

##Вопрос для самопроверки:

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

##Создание Deployment

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

