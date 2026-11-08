# Практика 4. Миграция Mattermost с MySQL на PostgreSQL

Было: Mattermost 9.5 + MySQL 8.0 (`legacy/docker-compose.yml`)
Стало: Mattermost 12.0.1 + PostgreSQL 15 (`docker-compose.prod.yml`)

## Структура

```
legacy/docker-compose.yml        старый стек
docker-compose.prod.yml          новый стек
migration.load                   конфиг pgloader (пароли подставляются в пайплайне)
.github/workflows/migration.yml  пайплайн миграции
```

Оба compose-файла используют одно имя проекта (`name: mattermost`), одну сеть `mattermost_network`
и один том `mattermost_data`. За счет этого новый Mattermost видит старые вложения и аватары,
а `--remove-orphans` при деплое удаляет контейнер MySQL.

## Запуск legacy

В `legacy/` создать `.env` по образцу `legacy/.env.example` и выполнить:

```
cd legacy
docker compose up -d
```

Mattermost на http://localhost:8065: создать админа, команду, приватный канал и написать несколько сообщений.

## Пайплайн

В Settings → Secrets and variables → Actions добавить секреты:

- `DB_MYSQL_ROOT_PASSWORD` (тот же, что в `legacy/.env`)
- `DB_POSTGRES_PASSWORD`

Раннер self-hosted (Linux или WSL, с docker). Пайплайн запускается пушем в ветку `migration`.

Шаги:

1. Проверка флага `~/.mattermost_migration_completed`. Если он есть, все шаги миграции пропускаются
   и выполняется только деплой и проверка.
2. Остановка старого Mattermost и бэкап MySQL в `~/mattermost_backups/`.
3. Запуск PostgreSQL.
4. Удаление схемы, оставшейся от прошлого (упавшего) запуска, и создание чистой схемы версии 9.5
   командой `mattermost db migrate`.
5. Перенос данных образом `mattermost/mattermost-pgloader` (настройки взяты из шаблона migration-assist).
6. Сравнение количества строк в Users, Teams, Channels, Posts.
7. Установка флага.
8. `docker compose up -d --remove-orphans`, Mattermost 12 сам доводит схему до своей версии.
9. Проверка `/api/v4/system/ping`.

## Почему не просто `pgloader --with "include drop"`

Пробовал как в шаблоне: данные переносятся, но pgloader сам создает таблицы и делает это не так,
как нужно Mattermost (нет enum-типов `channel_type` и т.д.). Mattermost 12 после этого не стартует:

```
pq: type "channel_type" does not exist
```

Поэтому схему создает Mattermost, а pgloader переносит только данные. Повторный запуск не дает дублей,
потому что перед переносом схема удаляется целиком (`DROP SCHEMA ... CASCADE`). Это делает то же самое,
что и `include drop`.

## Проверка

```
docker ps
```

Должны работать `mattermost` (12.0.1) и `mattermost-postgres`, контейнера `mattermost-mysql` нет.
Второй запуск пайплайна проходит быстро, шаги миграции помечены как skipped.
