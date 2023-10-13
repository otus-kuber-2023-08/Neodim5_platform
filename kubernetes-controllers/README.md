# Neodim5_platform
Neodim5 Platform repository

## KIND
Установил kind по интсрукции
Используя конфигурацию создал кластер kind
```
$ sudo kubectl get nodes
NAME                 STATUS   ROLES           AGE    VERSION
kind-control-plane   Ready    control-plane   2d8h   v1.27.3
kind-worker          Ready    <none>          2d8h   v1.27.3
kind-worker2         Ready    <none>          2d8h   v1.27.3
kind-worker3         Ready    <none>          2d8h   v1.27.3
```

## ReplicaSet

Создал и применил манифест frontend-replicaset.yaml со своими образами и переменными.
В описании ReplicaSet нехватает важной секции 
```
  selector:
    matchLabels:
      app: frontend
```
Увеличил количество реплик сервиса ad-hoc командой:
```
kubectl scale replicaset frontend --replicas=3
```

Все три реплики готовы
```
$ sudo  kubectl get rs frontend
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       2d8h
```

Благодаря контроллеру pod’ы восстанавливаются после их ручного удаления.

Обновил в манифесте версию образа на v0.0.2 и применил новый манифест. Образ pod'ов не изменился.
После удаления подов frontend, повторное применеиz ReplicaSet обновил образ для подов.
Обновление ReplicaSet не повлекло обновление запущенных pod по причине того, что ReplicaSet не умеет рестартовать запущенные поды при обновлении шаблона.

## Deployment

По подобию с микросервисом frontend проделал все манипуляции с микросервисом paymentService.
Создал валидный манифест paymentservice-replicaset.yaml с тремя репликами, разворачивающими из образа (neodim5/hipster-paymentservice) версии v0.0.1.
Написал Deployment манифест для сервиса paymentservice (paymentservice-deployment.yaml)

Теперь после измения версии образа в манифесте paymentservice-deployment.yaml нашего сервиса, после его применение,  поды пересоздались с новыми образами согласно манифеста автоматически (По умолчанию применяется стратегия Rolling Update).

### Deployment | Rollback

Попробовал сделать откат, посмотрев на историю версий нашего Deployment.
```
kubectl rollout history deployment paymentservice

kubectl rollout undo deployment paymentservice --to-revision=1 | kubectl get rs -l app=paymentservice -w
```
## Deployment | Задание со ⭐

С использованием параметров maxSurge и maxUnavailable самостоятельно реализовал два сценария развертывания.
В результате получил два манифеста:
+ paymentservice-deployment-bg.yaml
+ paymentservice-deployment-reverse.yaml

## Probes

Создал манифест frontend-deployment.yaml из которого можно развернуть три реплики pod с тегом образа v0.0.1
Добавил описание readinessProbe.
```
neodim@otus:~/kind_intro$ sudo kubectl describe po frontend | grep Readiness
    Readiness:      http-get http://:8080/_healthz delay=10s timeout=1s period=10s #success=1 #failure=3
    Readiness:      http-get http://:8080/_healthz delay=10s timeout=1s period=10s #success=1 #failure=3
    Readiness:      http-get http://:8080/_healthz delay=10s timeout=1s period=10s #success=1 #failure=3

```
пока readinessProbe для нового pod не станет успешной - Deployment не будет пытаться продолжить обновление.

## DaemonSet

DaemonSet - при его применении на каждом физическом хосте создается по одному экземпляру pod, описанного в
спецификации.

## DaemonSet | Задание со ⭐

+ Нашел в интернете манифест nodeexporter-daemonset.yaml для развертывания DaemonSet с Node Exporter
+ После применения данного DaemonSet и выполнения команды: kubectl port-forward <имя любого pod в DaemonSet> 9100:9100 метрики доступны на localhost: curl localhost:9100/metrics

## DaemonSet | Задание с ⭐️⭐
По умолчанию, pod управляемые DaemonSet на master нодах не разворачиваются. 
Нашел как модернизировать свой DaemonSet таким образом, чтобы Node Exporter был развернут как на master, так и на worker нодах.
