# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск сайта в Minikube

### Окружение 

На локальном компьютере необходимо установить:

- [VirtualBox](https://virtualbox.org)
- [Minikube](https://minikube.sigs.k8s.io)
- [Kubernetes kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)

Видео [Поднятия простого Локального k8s Cluster на Windows](https://www.youtube.com/watch?v=WAIrMmCQ3hE&list=PLg5SS_4L6LYvN1RqaVesof8KAf-02fJSi&index=3&ab_channel=ADV-IT)

В каталоге `kubernetes` создайте конфигурационный файл `django-config.yaml`, содержащий в себе настройки переменных окружения (пример конфига `django-congig-example.yaml`)

### Запуск в Minikube

Запустите Minikube:
```shell
minikube start
```

Настройте терминал для использования Docker Daemon Minikube:
```shell
eval $(minikube docker-env)
```

Соберите образ Django:
```shell
docker-compose build web
```

Включите встроенный Ingress-контроллер:
```shell
minikube addons enable ingress
```

Примените Ingress-правило:
```shell
kubectl apply -f kubernetes/starburger-ingress.yaml
```

Добавьте запись в файл hosts:

- Найдите IP-адрес Minikube:
```shell
minikube ip
```

Затем добавьте этот IP в файл `hosts` (<IP minikube> starburger.test).
- На Linux и macOS, файл hosts обычно находится в /etc/hosts.
- На Windows, файл hosts обычно находится в C:\Windows\System32\Drivers\etc\hosts.

Примените конфигурационный файл:
```shell
kubectl apply -f kubernetes/django-config.yaml
```

Запустите deployment:
```shell
kubectl apply -f kubernetes/django-deployment.yaml
```

Для очистки сессий, запустите django-clearsessions:
```shell
kubectl apply -f kubernetes/django-clearsessions.yaml
```

### Подключение базы данных

Установите инструмент для управления приложениями Kubernetes [Helm](https://helm.sh)

Добавьте Helm Chart Repository:
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Создайте файл `postgres-config.yaml` с конфигурацией базы данных (пример конфигурационного файла `postgres-config-example.yaml`)

Установите PostgreSQL, используя созданный конфигурационный файл:
```shell
helm install postgres bitnami/postgresql -f postgres-config.yaml
```

В файле `django-config` измените переменную `DATABASE_URL`, поменяв `HOST` базы данных

Примените миграции к базе данных:
```shell
kubectl apply -f kubernetes/django-job.yaml
```

Сайт будет доступен по ссылке [http://starburger.test](http://starburger.test)

## Цели проекта
Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org).