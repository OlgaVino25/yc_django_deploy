# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

```shell
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

<a name="env-variables"></a>

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Развёртывание PostgreSQL в кластере

Для работы сайта требуется база данных PostgreSQL. В этом проекте она разворачивается непосредственно в кластере Minikube с помощью Helm-чарта от Bitnami.

**Установка PostgreSQL**

1. Убедитесь, что Helm установлен (инструкция: [helm.sh](https://helm.sh/docs/intro/install/)).
2. Выполните команду для установки PostgreSQL с заданными параметрами:

```bash
# PowerShell (Windows)
helm install my-postgres oci://registry-1.docker.io/bitnamicharts/postgresql `
  --set auth.database=test_k8s `
  --set auth.username=test_k8s `
  --set auth.password=OwOtBep9Frut `
  --set auth.postgresPassword=postgres123 `
  --set persistence.enabled=false

# Linux/macOS
helm install my-postgres oci://registry-1.docker.io/bitnamicharts/postgresql \
  --set auth.database=test_k8s \
  --set auth.username=test_k8s \
  --set auth.password=OwOtBep9Frut \
  --set auth.postgresPassword=postgres123 \
  --set persistence.enabled=false
```

3. Дождитесь запуска пода:

```bash
kubectl get pods -w
```

Под `my-postgres-postgresql-0` должен перейти в состояние `Running`.

4. Проверьте подключение к базе от имени пользователя `test_k8s`:

```bash
kubectl run psql-client --rm -it --restart=Never --image=postgres:12.0-alpine --env="PGPASSWORD=OwOtBep9Frut" -- psql -h my-postgres-postgresql -U test_k8s -d test_k8s
```

Если появится приглашение `test_k8s=#` – всё работает. Выйдите `\q`.

_Примечание: В учебных целях отключено постоянное хранилище (persistence.enabled=false). При перезапуске Minikube данные будут потеряны. Для продакшена следует настроить PersistentVolume._

### Настройка секретов

Перед применением манифестов необходимо создать Secret с конфиденциальными данными. Это можно сделать двумя способами:

_IP Minikube можно узнать командой `minikube ip`_
_Для подключения к PostgreSQL, запущенному на хосте, используйте IP-адрес шлюза Minikube. Его можно узнать командой:_

```bash
minikube ssh "ip route show default | awk '{print $3}'"
```

Эта команда выведет IP шлюза (например, 192.168.49.1 для драйвера Docker или 192.168.59.1 для VirtualBox).
_Примечание для Windows_: если команда не сработает, вы можете зайти в Minikube вручную:

```bash
minikube ssh
```

Затем внутри виртуальной машины выполните:

```bash
ip route show default
```

Найдите строку, начинающуюся с default via, и скопируйте следующий за ней IP-адрес. Выйдите из ssh командой exit.

**Вариант 1 (через файл, не коммитить):**

1. Скопируйте `secrets.yaml.example` и переименуйте в `secrets.yaml` и заполните своими значениями.
2. Примените: `kubectl apply -f secrets.yaml`
3. Убедитесь, что `secrets.yaml` добавлен в `.gitignore`.

**Вариант 2 (через командную строку, без сохранения в файл):**

```bash
kubectl create secret generic django-secrets \
  --from-literal=SECRET_KEY="your-secret-key" \
  --from-literal=DATABASE_URL="postgres://test_k8s:OwOtBep9Frut@my-postgres-postgresql.default.svc.cluster.local:5432/test_k8s" \
  --from-literal=ALLOWED_HOSTS="localhost,127.0.0.1,<minikube-ip>,star-burger.test"

# для powershell

kubectl create secret generic django-secrets `
  --from-literal=SECRET_KEY="your-secret-key" `
  --from-literal=DATABASE_URL="postgres://test_k8s:OwOtBep9Frut@my-postgres-postgresql.default.svc.cluster.local:5432/test_k8s" `
  --from-literal=ALLOWED_HOSTS="localhost,127.0.0.1,<minikube-ip>,star-burger.test"
```

2. Примените манифесты:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

3. Для доступа к сайту в новом терминале:

```bash
minikube service django-service
```

## Развёртывание в Kubernetes с Ingress

**Предварительные требования:**

- Установленный гипервизор или контейнерный движок (например, Docker Desktop или VirtualBox) в зависимости от выбранного драйвера Minikube
- Minikube и kubectl
- PostgreSQL, развёрнутый в кластере (см. раздел "Развёртывание PostgreSQL в кластере").

1. Запустите Minikube

```bash
minikube start
```

2. Включите Ingress-контроллер

```bash
minikube addons enable ingress
```

Дождитесь, пока поды ingress-nginx перейдут в состояние Running (`kubectl get pods -n ingress-nginx`).

3. Создайте секрет с конфиденциальными данными

Скопируйте файл-образец и отредактируйте его:

```bash
cp secrets.yaml.example secrets.yaml
# отредактируйте secrets.yaml, указав свои значения
```

Затем примените секрет:

```bash
kubectl apply -f secrets.yaml
```

_`secrets.yaml` – уже добавлен в `.gitignore`._

4. Примените остальные манифесты

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

5. Запустите миграции базы данных (если база новая или были изменения)
   Для этого используется отдельный Job-манифест `migrate-job.yaml`:

```bash
kubectl apply -f migrate-job.yaml
```

Следите за выполнением:

```bash
kubectl get jobs
kubectl logs job/django-migrate
```

Job автоматически удалится через 10 минут после завершения (благодаря `ttlSecondsAfterFinished`). При желании можно удалить его вручную:

```bash
kubectl delete job django-migrate
```

6. Проверьте созданные ресурсы

```bash
kubectl get pods
kubectl get services    # django-service должен быть типа ClusterIP
kubectl get ingress     # должен быть указан адрес (например, 192.168.49.2)
```

7. Настройте локальный доступ к домену

Узнайте IP-адрес Minikube:

```bash
minikube ip
```

Далее действия зависят от используемого драйвера Minikube:

- **Для драйвера Docker** (по умолчанию на Windows):
  Ingress-контроллер становится доступен через локальный туннель. Выполните в отдельном терминале (не закрывайте его):

```bash
minikube service ingress-nginx-controller -n ingress-nginx
```

Эта команда откроет браузер и покажет локальный адрес вида http://127.0.0.1:xxxxx. Запомните порт (например, 60743).

Отредактируйте файл `C:\Windows\System32\drivers\etc\hosts`.
Для быстрого открытия выполните в PowerShell от имени администратора:

```bash
notepad C:\Windows\System32\drivers\etc\hosts
```

Добавьте в конец файла строку:

```text
127.0.0.1   star-burger.test
```

_Для Linux/macOS: отредактируйте файл /etc/hosts с правами суперпользователя:_

```bash
sudo nano /etc/hosts
```

- **Для драйвера VirtualBox** (или других гипервизоров):
  Ingress-контроллер доступен напрямую по IP Minikube (обычно 192.168.59.xxx).
  В файл hosts добавьте строку в конец файла и замените <IP_Minikube> на значение из minikube ip:

```text
<IP_Minikube>   star-burger.test
```

_Если вы хотите использовать другой домен, отредактируйте поле host в файле ingress.yaml перед применением манифеста._

После изменения hosts сбросьте кэш DNS:

```bash
ipconfig /flushdns
```

8. Откройте сайт

- **Для драйвера Docker:** перейдите по адресу `http://star-burger.test:xxxxx/admin/` (подставьте порт из шага 6).
- **Для драйвера VirtualBox:** перейдите по адресу `http://star-burger.test/admin/` (без порта).

Вы должны увидеть страницу входа в админку Django.

9. Создайте суперпользователя (если нужно)

```bash
kubectl exec -it deploy/django-app -- python manage.py createsuperuser
```

**Примечание:** При использовании драйвера Docker туннель нужно держать открытым всё время работы с сайтом. При каждом перезапуске Minikube порт может меняться – повторяйте шаг 6 для получения актуального адреса. Также при смене драйвера может измениться IP Minikube, и тогда нужно обновить hosts.

## Очистка устаревших сессий Django

Для автоматического удаления устаревших сессий используется CronJob, который запускает команду `python manage.py clearsessions` ежедневно в 02:00. Также для повышения надёжности и автоматической очистки завершённых заданий в манифест `cronjob.yaml` добавлены два параметра:

- `startingDeadlineSeconds: 100` — если задание не было запущено в запланированное время (например, из-за недоступности кластера), у него есть 100 секунд, чтобы запуститься. После этого оно считается пропущенным, и ошибка не возникает.
- `ttlSecondsAfterFinished: 600` — завершённое задание (Job) и все его поды автоматически удаляются через 10 минут после завершения, что предотвращает накопление мусора в кластере.

1. Примените манифест:

```bash
kubectl apply -f cronjob.yaml
```

2. Проверьте создание CronJob:

```bash
kubectl get cronjobs
```

3. Для немедленного запуска (например, для теста) создайте Job вручную:

```bash
kubectl create job --from=cronjob/django-clearsessions-cronjob django-clearsessions-manual
```

4. Убедитесь, что Job выполнилась успешно:

```bash
kubectl get jobs
kubectl logs <pod-name>
```

При желании можно изменить расписание, отредактировав поле `schedule` в `cronjob.yaml`.

## Деплой тестового Nginx в dev-окружение (yc-sirius-dev)

Этот раздел описывает развёртывание простого пода с Nginx в вашем облачном namespace и его подключение через уже существующий домен.

**Перед началом** замените в командах и конфигах следующие плейсхолдеры на свои значения:

- `<namespace>` — ваш namespace в кластере (например, `edu-ivan-ivanov`).
- `<domain>` — ваш домен (например, `edu-ivan-ivanov.yc-sirius-dev.pelid.team`).
- `<service-name>` — имя вашего сервиса (в примере `test-nginx-service`).

1. Применить манифест пода

```bash
kubectl apply -f deploy/yc-sirius/<namespace>/test-nginx-pod.yaml
```

Проверьте, что под запустился:

```bash
kubectl get pods -n <namespace>
```

2. Создать Service для доступа к поду

```bash
kubectl apply -f deploy/yc-sirius/<namespace>/test-nginx-service.yaml
```

Убедитесь, что сервис получил эндпоинт (IP пода):

```bash
kubectl describe service test-nginx-service -n <namespace>
```

*В выводе должна быть строка Endpoints: <IP*пода>:80 (фактический IP вашего пода).\_

3. Локальная проверка через port-forward
   Если нужно проверить работу пода локально, выполните:

```bash
# используйте свободный порт, например 8081
kubectl port-forward -n <namespace> pod/test-nginx 8081:80
```

_После этого откройте браузер по адресу http://localhost:8081 — должна открыться страница Nginx. Если и этот порт занят, пробуйте последовательно 8082, 8083 и т.д._

4. Доступ через домен (через main-nginx)
   В кластере уже запущен `main-nginx`, который обслуживает ваш домен `<domain>`. Чтобы тестовый под стал доступен по пути `/test`, отредактируйте ConfigMap `main-nginx-config`:

```bash
kubectl edit configmap main-nginx-config -n <namespace>
```

Добавьте в секцию `server` новый `location`:

```nginx
location /test/ {
    proxy_pass http://<service-name>.<namespace>.svc.cluster.local:80/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

Сохраните и закройте редактор. Затем перезапустите deployment `main-nginx`, чтобы конфигурация применилась:

```bash
kubectl rollout restart deployment main-nginx -n <namespace>
```

5. Проверка
   Откройте в браузере:
   `https://<domain>/test/`

Вы должны увидеть страницу Nginx. Обратите внимание на HTTPS — он работает автоматически благодаря настройкам кластера.

6. Просмотр логов и отладка

- Логи вашего пода: `kubectl logs -n <namespace> test-nginx`
- Логи main-nginx (если что-то не работает): `kubectl logs -n <namespace> deployment/main-nginx`

## Подготовка секрета для подключения к БД

В кластере уже создан секрет `postgres`, содержащий все необходимые данные (хост, порт, имя БД, пользователя, пароль и корневой сертификат). Для использования в поде достаточно:

- Смонтировать ключ `root.crt` в контейнер с правами 0600.
- Указать переменные окружения, используя `valueFrom.secretKeyRef`.

Пример пода с автоматическим монтированием сертификата см. в `deploy/yc-sirius/edu-olga-vinokurova/psql-client-automount.yaml`.
