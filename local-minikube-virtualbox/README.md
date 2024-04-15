## Развёрытвание с помощью Minikube

Перед началом работы убедитесь что у вас установлены следующие инструменты:

- Гипервизор. Если используете Windows, подойдёт [virtualbox](https://www.virtualbox.org/wiki/Downloads), либо [Hyper-V](https://msdn.microsoft.com/en-us/virtualization/hyperv_on_windows/quick_start/walkthrough_install). Для linux также можно использовать [virtualbox](https://www.virtualbox.org/wiki/Downloads), либо [KVM](https://www.linux-kvm.org/), который более прост и гибок в установке. Обратите внимание, что если вы используете WSL, вам понадобится провести дополнительные манипуляции. [Подробная инструкция.](https://www.virtualizationhowto.com/2021/11/install-minikube-in-wsl-2-with-kubectl-and-helm/)

- Сам [Minicube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)

- Инструмент для управления кластерами Kubernetes: [Kubectl](https://kubernetes.io/docs/tasks/tools/)

### Запускаем кластер Minikube в нашей виртуальной среде

```sh
minikube start
```

Как правило minikube по умолчанию выбирает нужный драйвер, но если возникнут проблемы, можно указать его явно через аргумент, например: `--driver=virtualbox` или `--driver=docker`.

Первый запуск займёт чуть больше времени чем последующие. Проверьте корректность утановки следующей командой:

```sh
kubectl cluster-info
```

## Настройка и запуск Django

Создайте образ контейнера командой

```sh
minikube image build ./backend_main_django/ -t jango_app
```

Убедитесь что он присутствует в списке доступных образов командой:

```sh
minikube image ls
```

Если хотите использовать другой image, откройте файл `kubernetes/deployment-django-ver1.yaml` и измените значение `image: docker.io/library/jango_app:latest` на адрес нужного на docker hub или где-либо еще (возможно придется удалить или закомментировать строку `imagePullPolicy: Never`).

Откройте `kubernetes/environment.yaml` и укажите нужные значения для переменных окружения. Назначение переменных описано в разделе [Переменные окружения](#environments)

Регистрируем переменные конфигурации и секреты:

```sh
kubectl apply -f kubernetes/environment.yaml
```

Запускаем Deployment django:
```sh
kubectl apply -f kubernetes/deployment-django-ver1.yaml
```

Запускаем Service django:
```sh
kubectl apply -f kubernetes/service_django.yaml
```

Для доступа с использованием доменного имени у вас должен быть запущен любой [ингресс контроллер](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). Например для запуска ингресс контроллера [Contour](https://projectcontour.io/) команда запуска будет выглядеть так:

```sh
kubectl.exe apply -f https://projectcontour.io/quickstart/contour.yaml
```

Если ингресс контроллер уже запущен, то  примените манифест с правилами обработки входящих запросов:

```sh
kubectl.exe apply -f kubernetes/ingress-hosts.yaml
```

Создайте периодическую задачу для удаления сессий из БД командой:

```sh
kubectl.exe apply -f kubernetes/clearsession.yaml
```

По умолчанию она запускается каждую минуту (schedule: "* * * * *"). Отредактируйте интервалы [CRON-выражения](https://ru.wikipedia.org/wiki/Cron) согласно стандартной нотации вашим требованиям.

При необходимости применить миграции, используйте команду:

```sh
kubectl.exe apply -f kubernetes/migrate.yaml
```
