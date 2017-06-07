# Разворачивание проекта на продакшене

## Переменные окружения

### Общие

| Название | По умолчанию | Описание |
| ------ | ------ | ------ |
| DATABASE_NAME | cian_content_development | Имя бд |
| DATABASE_HOST | localhost:1433 | Хост бд или ip, можно указать порт через `:` |
| DATABASE_USER | cian | Пользователь бд |
| DATABASE_PASSWORD | c1i2A3n4 | Пароль к пользователю |
| CIAN_API_URL | http://master.dev.cian.ru | Базой урл к API cian |
| CIAN_API_KEY | c94263c7-b1ca-464c-9c5d-7ae3c1085c9e | `X-PassKey` для работы с пользователями |
| CIAN_API_MESSAGER_KEY | 553D30AA-1BD7-457D-AF4C-0EBC4CE82E6C | `X-PassKey` для рассылки сообщений |
| CELERY_BROKER_URL | amqp://test:test@192.168.16.22:5672// | Брокер для celery |
| BASE_URL | http://beta2.frontend-content.stage.cian.ru | Хост для формирования ссылок в рассылки и при переходах в админке |

### api

| Название | Значение | Описание |
| ------ | ------ | ------ |
| DJANGO_SETTINGS_MODULE | cian_content.settings.prod | python модуль. Указывает какие модуля и настройки необходимо применять |
| SWIFT_AUTH_TOKEN_DURATION | 3600 | время жизни авторизационного токена для cdn |


### admin

| Название | Значение | Описание |
| ------ | ------ | ------ |
| PREVIEW_URL | Пример: `http://master.frontend-content.dev2.cian.ru/preview` | Ссылка на страницу где будет отображено превью записи с админки. |
| DJANGO_SETTINGS_MODULE | cian_content.settings.admin_prod | python модуль. Указывает какие модуля и настройки необходимо применять |
| ADMIN_DEBUG | f | Убираем дебаг мод с админки |


Для беты конфиги могут быть проставлены не все, т.к. по умолчанию высталены правильно. Нужно учитывать это при переносе конфигов на прод.

## Конфиг в nomad

1. backend-content - api контентного раздела. Можно стартовать много машин. Служит воркером для celery.
1. backend-content-admin - админка контентного раздела. Можно стартовать много машин.
1. backend-content-crontab - машина для запуска крон задач. Необходимо запускать **ТОЛЬКО** 1 машину, что бы не порождать крон задачи. Так же служит воркеров для celery.

## Работа с зависимостями

Все зависимости ставятся с помощью pip3 и **их установка уже настроена в докере**

```sh
RUN pip3 install -r /srv/backend-content/requirements/prod.txt
```

Список окружений для установки разных зависимостей

1. `requirements/dev.txt`
1. `requirements/prod.txt`
1. `requirements/test.txt`

## Разворачивание базы данных и перенос данных с `realty.dmir.ru`

Для запуска команд используется django менеджер команд.

Креды БД должны быть переданы как переменные окружения в момент **запуска проекта и выполнения команд**:
например:

```sh
export DATABASE_NAME="cian_content_development"
export DATABASE_HOST="localhost:1433"
export DATABASE_USER="cian"
export DATABASE_PASSWORD="c1i2A3n4"
```

или

```sh
DATABASE_HOST=YOUR_HOST DATABASE_NAME=YOUR_DATABASE_NAME DATABASE_USER=YOUR_DATABASE_USERNAME DATABASE_PASSWORD=YOUR_DATABASE_PASSWORD python manage.py [command]
```

### Создание и настройка БД.

Создание пользователя:

```sql
create login your_login with password = 'your_password';
GO
```

Создание БД и логина:

```sql
CREATE DATABASE your_database;
GO

use your_database;
GO

create USER your_user for login your_login;
GO

ALTER ROLE [db_owner] ADD MEMBER your_user;
GO
```

Для продакшн БД пользователю могут быть выданы иные привелегии, но обязательно должны быть права на создание\изменение\удаление таблиц.

***Все последующие команды лучше выполнять на удаленной машине с развернутым проектом, при этом необходимо указывать креды БД.***

### Запуск миграций.

Запуск миграций производиться путем выполнения команд:

 1. `[db_credentials] python manage.py migrate` - API
 1. `[db_credentials] DJANGO_SETTINGS_MODULE=cian_content.settings.admin python manage.py migrate` - ADMIN

При запуске миграций административной панели произойдет ошибка и миграции упадут. Ошибка связана с попыткой изменения констреинта. Для ее устранения необходимо посмотреть имя констреинта (в консоле), подключиться к серверу БД и выполнить команду (в консоли tsql):
`alter table [auth_user] drop constraint [your_constraint]`
После необходимо повторну запустить миграции административной панели.
`[credentials] DJANGO_SETTINGS_MODULE=cian_content.settings.admin python manage.py migrate`
(Не забыть указать креды БД)

### Экспорт данных

#### Изначально необходимо перенести данные из БД `mSites`. Для этого необходимо выполнить комманду

Необходимо руками удалить таблицу `dbo.content_journal_similar_journals` если таковая имеется. (Этап 1.2-1.3)

`[db_credentials] python manage.py exportdata`

Экспорт может занять более часа (в зависимости от скорости работы БД).
В случае прерывания работы скрипта необходимо повторно запустить его.

#### Перенос количества просмотров

`[db_credentials] python manage.py exportnumviews`

Скрипт может принимать 2 (опциональных) аргумента:

    - `--articles_num_views path/to/file.csv` - количество просмотров статей
    - `--news_num_views path/to/file.csv` - количество просмотров новостей

    Файлы должны иметь формат:
    [id статьи/новости], [количество просмотров]
    Пример:

    ID, NumViews

    32019,618911

    36323,191537

    33366,98598 ..

    По-умолчанию скрипт ищет файлы в директории '/dumps' в корне проекта. Файлы должны называться `articles_num_views.csv` и `news_num_views.csv` соответственно.

#### Переименование регионов

`[db_credentials] python manage.py rename_regions`

#### Перенос подкатегорий (Этап 1.2-1.3)

`[db_credentials] python manage.py exportsubcategories`

#### Первоначальная генерация тегов (Этап 1.2-1.3)

`[db_credentials] python manage.py generatetags`

#### Первоначальная генерация блока читать по теме (Этап 1.2-1.3)

`[db_credentials] python manage.py generatesimilarjournals`

### Миграция картинок

Миграция проходит в несколько этапов.
При возникновении повторных обрывов в соединении скрипт может завершить свое действия

Для полного переноса необходимо выполнить последовательно команды:

Перенос картинок из `realty.dmir.ru` на `cdn` по полю image в таблице journal. Может выполнятся до 12ч.

```sh
python manage.py exportimages
```

Вытягиваем картинки из content таблицы journal и переносим с `realty.dmir.ru` на `cdn`. Меняет абсолютные ссылки картинок на `http://archive-cdn.cian.site/realty` + `путь к картинки в контейнере`. Может выполнятся до 10ч.

```sh
python manage.py exportimagescontent
```


Кеширование превью картинок. Делает 2 варианта превью картинок и сохраняет в CACHE папку в контейнере. Может выполнятся до 24ч.

```sh
DJANGO_SETTINGS_MODULE=cian_content.settings.tools python manage.py imagegenerator
```

Если картинка уже имеется на cdn, то повторного переноса с `realty.dmir.ru` не происходит. За счет этого скрипт может выполнится в разы быстрее.
Подготовка превью занимает минимум 3ч, т.к. пытается получить все превью картинок, дабы убедится что все картинки на месте.

