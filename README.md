### Документация по развертываннию проекта (Этап 1.1)

##### Создание и настройка БД.

Креды БД должны быть переданы как переменные окружения в момент запуска проекта:
1. `DATABASE_NAME` default = `cian_content_development`
1. `DATABASE_HOST` default = `localhost:1433`
1. `DATABASE_USER` default = `cian`
1. `DATABASE_PASSWORD` default = `c1i2A3n4`

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

##### Запуск миграций.

###### Все последующие команды лучше выполнять на удаленной машине с развернутым проектом, при этом необходимо указывать креды БД.

`DATABASE_HOST=YOUR_HOST DATABASE_NAME=YOUR_DATABASE_NAME DATABASE_USER=YOUR_DATABASE_USERNAME   DATABASE_PASSWORD=YOUR_DATABASE_PASSWORD python manage.py [command]`

Запуск миграций производиться путем выполнения команд:
 1. `[db_credentials] python manage.py migrate` - API
 2. `[db_credentials] DJANGO_SETTINGS_MODULE=cian_content.settings.admin python manage.py migrate` - ADMIN

При запуске миграций административной панели произойдет ошибка и миграции упадут. Ошибка связана с попыткой изменения констреинта. Для ее устранения необходимо посмотреть имя констреинта (в консоле), подключиться к серверу БД и выполнить команду (в консоли tsql):
`alter table [auth_user] drop constraint [your_constraint]`
После необходимо повторну запустить миграции административной панели.
`[credentials] DJANGO_SETTINGS_MODULE=cian_content.settings.admin python manage.py migrate`
(Не забыть указать креды БД)

##### Экспорт данных.

###### Все последующие команды лучше выполнять на удаленной машине с развернутым проектом, при этом необходимо указывать креды БД.

`DATABASE_HOST=YOUR_HOST DATABASE_NAME=YOUR_DATABASE_NAME DATABASE_USER=YOUR_DATABASE_USERNAME   DATABASE_PASSWORD=YOUR_DATABASE_PASSWORD python manage.py [command]`

 1. Изначально необходимо перенести данные из БД `mSites`. Для этого необходимо выполнить комманду:
`[db_credentials] python manage.py exportdata`
Экспорт может занять более часа (в зависимости от скорости работы БД).
В случае прерывания работы скрипта необходимо повторно запустить его.

 1. Перенос количества просмотров. 
 `[db_credentials] python manage.py exportdata`
Скрипт может принимать 2 (опциональных) аргумента:
    - `--articles_num_views path/to/file.csv` - количество просмотров статей
    - `--news_num_views path/to/file.csv` - количество просмотров новостей

    Файлы должны иметь формат:
    [id статьи/новости], [количество просмотров]
    Пример:
    32019,618911
    36323,191537
    33366,98598 ...



