# Развертывание в кластере Yandex-облака

Убедитесь что вы получили необходимые доступы и можете зайти в [веб интерфейс Яндекс-облака.](https://console.cloud.yandex.ru/)

Перед началом работы убедитесь что у вас установлены следующие инструменты:

- Инструмент для управления кластерами Kubernetes: [Kubectl](https://kubernetes.io/docs/tasks/tools/). [Иструкция по настройке](https://yadi.sk/i/QDZGXUP7aG3KOw) kubectl для работы в Яндекс-облаке.
- Графический инструмент для управления кластерами Kubernetes - [Lens Desktop](https://k8slens.dev/)

Проверьте корректность настройки следующей командой:

```sh
kubectl cluster-info
```

## Выделенные ресурсы

Для развертывания тестового сайта джанго в Яндекс-облаке Вам должны быть выделены:
(https://sirius-env-registry.website.yandexcloud.net/edu-sleepy-engelbart.html)

- домен вида `edu-sleepy-engelbart.sirius-k8s.dvmn.org`
- отдельный namespace вида `edu-sleepy-engelbart`
- база данных с именем вида `edu-sleepy-engelbart`
- в указанном namespace должен находиться секрет с именем `postgres`, в ключах которого находятся доступы к указанной выше базе данных (`dsn`)

## Настройка переменных окружения Django

Откройте `environment.yaml` и укажите нужные значения для переменных окружения DEBUG, ALLOWED_HOST и SECRET_KEY. Назначение переменных описано в разделе [Переменные окружения](../../../README.md#environments)

Если доступы к базе данных находятся не в секрете с именем `postgres` или ключ называется не `dsn`, то придется внести изменения в манифесты запускающие джанго и ее менеджмент команды в раздел `valueFrom`.

Для доступа к приложению через выданное доменное имя, укажите в манифесте сервиса (`service_django.yaml`) порт на который настроен роутинг ALB.

## Запуск приложения

Регистрируем переменные конфигурации и секреты:

```sh
kubectl --namespace=<выделенный Вам namespace> apply -f environment.yaml
```

При первом запуске произведите настройку базы данных применив все текущие миграции командой:

```sh
kubectl --namespace=<выделенный Вам namespace> apply -f migrate.yaml
```

Чтобы создать суперпользователя внесите `createsuperuser.yaml` необходимые Вам значения

```sh
DJANGO_SUPERUSER_EMAIL: '<e-mail>'
DJANGO_SUPERUSER_USERNAME: '<username>'
DJANGO_SUPERUSER_PASSWORD: '<password>'
```

и примените манифест командой:

```sh
kubectl --namespace=<выделенный Вам namespace> apply -f createsuperuser.yaml
```

Запускаем Deployment django:

```sh
kubectl --namespace=<выделенный Вам namespace> apply -f deployment-django.yaml
```

Запускаем Service django:

```sh
kubectl apply -f kubernetes/service_django.yaml
```

Поздравляем! Приложение должно быть доступно по [выданному Вам адресу](https://edu-sleepy-engelbart.sirius-k8s.dvmn.org)

## Обновление версии приложения

Для запуска другой версии приложения, укажите в манифесте `deployment-django.yaml` образ приложения на dockerhub с нужным Вам тэгом. Затем примените внесенные изменения командой:

```sh
kubectl --namespace=<выделенный Вам namespace> apply -f deployment-django.yaml
```
