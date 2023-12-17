# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## <a id='run-with-docker-compose'>Как запустить dev-версию через `docker compose`:</a>

Запустите базу данных и сайт:

```sh
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```sh
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

## <a id='environments>Переменные окружения</a>

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Развёрытвание с помощью Minikube

Перед началом работы убедитесь что у вас установлены следующие инструменты:

- Гипервизор. Если используете Windows, подойдёт [virtualbox](https://www.virtualbox.org/wiki/Downloads), либо [Hyper-V](https://msdn.microsoft.com/en-us/virtualization/hyperv_on_windows/quick_start/walkthrough_install). Для linux также можно использовать [virtualbox](https://www.virtualbox.org/wiki/Downloads), либо [KVM](https://www.linux-kvm.org/), который более прост и гибок в установке. Обратите внимание, что если вы используете WSL, вам понадобится провести дополнительные манипуляции. [Подробная инструкция.](https://www.virtualizationhowto.com/2021/11/install-minikube-in-wsl-2-with-kubectl-and-helm/)

- Сам [Minicube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)

- Инструмент для управления кластерами Kubernetes: [Kubectl](https://kubernetes.io/docs/tasks/tools/)

### Запускаем кластер Minikube в нашей виртуальной среде:

```sh
minikube start
```

Как правило minikube по умолчанию выбирает нужный драйвер, но если возникнут проблемы, можно указать его явно через аргумент, например: `--vm-driver=virtualbox`.

Первый запуск займёт чуть больше времени чем последующие. Проверьте корректность утановки следующей командой:

```sh
kubectl cluster-info
```

## Настройка и запуск Django
При развёртывании будет использоваться image с Django созданный при [развертывании с помощью `docker compose`](#run-with-docker-compose). Название образа должно быть `docker.io/library/nginx:latest`. Убедитесь что он присутствует в списке доступных образов командой:

```sh
minikube image ls
```

Если хотите использовать свой image, откройте файл `kubernetes/django-service.yml` и измените значение `image: docker.io/library/jango_app:latest` на адрес нужного на docker hub или где-либо еще (возможно придется удалить или закомментировать строку `imagePullPolicy: Never`).

Откройте `cubernetes/configmap.yml` и укажите нужные значения для Django. Назначение переменных описано в разделе [Переменные окружения](#environment)

Регистрируем конфиг и поднимаем Django:

```sh
kubectl apply -f kubernetes/configmap.yml
kubectl apply -f kubernetes/django-service.yml
```

Обратите внимание что в одном файле `kubernetes/django-service.yml` создаётся сразу Deployment и Service для нашего приложения.
