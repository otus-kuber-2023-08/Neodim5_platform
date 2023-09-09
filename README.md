# Neodim5_platform
Neodim5 Platform repository

### Kubernetes-Intro "Знакомство с Kubernetes, основные понятия и архитектура"
* Настроен Github actions для запуска тестов
* Развернут локальный кластер kubernetes посредством minikube
* Установлена актуальная версия kubectl
* Разобрался почему все pod в namespace kube-system восстановились после удаления. Описание в PR.
* Создан докерфайл запускающий web-сервер (nginx)
* Собрал из Dockerfile образ контейнера и поместил его в публичный Container Registry Docker Hub (neodim5/kube-intro)
* Создан под через web-pod.yaml
* Добавил в pod init контейнер, генерирующий страницу index.html
* Проверил работоспособность web сервера с помощью kubectl port-forward
* Hipster Shop frontend
* Соберал собственный образ для frontend (используя готовый Dockerfile)
* Поместите собранный образ на Docker Hub (neodim5/hipster-frontend)
* Попробовал использовать ad-hoc режим и возможности Kubectl для создания ресурсов
* Выяснил причину, по которой pod frontend находится в статусе Error. Описание в PR.
* Разобраны декларативный и императивный подходы
