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

## Как запустить в кластере Kubernetes с помощью Minikube

Установите [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/). Установите и запустите [minikube](https://minikube.sigs.k8s.io/docs/):

```bash
minikube start
```

Создайте образ Django-приложения в кластере:

```bash
minikube image build -t django_app backend_main_django/
```

Задайте переменные окружения в файле конфигурации `kubernetes/django-app-configmap.yml`:

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-app-configmap
data:
  ALLOWED_HOSTS: '*'
  DATABASE_URL: postgres://test_k8s:OwOtBep9Frut@10.0.2.2:6666/test_k8s
  DEBUG: 'False'
  SECRET_KEY: your-secret-key
```

Создайте `ConfigMap`:

```bash
kubectl apply -f kubernetes/django-app-configmap.yml
```

Запустите дейплой:

```bash
kubectl apply -f kubernetes/django-app-deployment.yml
```

### Изменение ConfigMap

При внесении изменений в `kubernetes/django-app-configmap.yml` необходимо пересоздать ConfigMap и перезапустить деплой:

```bash
kubectl apply -f kubernetes/django-app-configmap.yml
kubectl rollout restart deployment
```

### Настройка Ingress

В `/etc/host` добавьте строку:

```txt
your-minikube-ip star-burger.test
```

Чтобы узнать IP-адрес minikube:

```bash
minikube ip
```

Установите расширение для `minikube`:

```bash
minikube addons enable ingress
```

Запустите манифест:

```bash
kubectl apply -f kubernetes/django-app-ingress.yml
```

### Настройка удаления сессий

Задать регулярность удаления пользовательских сессий можно в файле `django-clear-sessions-cronjob.yml`:

```yaml
schedule: "0 0 1 * *"
```

После этого запустите команду в работу:

```bash
kubectl apply -f kubernetes/django-app-clearsessions-cronjob.yml
```

### База данных и миграции

Базу данных PosgreSQL лучше развернуть вне кластера, но при необходимости можно сделать это и в Kubernetes, воспользовавшись [Helm](https://helm.sh/):

```bash
brew install helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install postgres-1 bitnami/postgresql
```

Для запуска миграции базы данных используйте:

```bash
kubectl apply -f kubernetes/django-app-migrate-job.yml
```

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
