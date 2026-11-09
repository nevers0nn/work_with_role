# Домашнее задание «Работа с roles»

Взял playbook из предыдущего задания и разделил его на роли. Для ClickHouse использовал готовую роль AlexeySetevoi версии `1.13`, как указано в задании. Для Vector и LightHouse создал роли командой `ansible-galaxy role init` и перенёс в них задачи и шаблоны из старого playbook.

## Что получилось

В `site.yml` осталось три play, каждый вызывает свою роль:

- ClickHouse — установка сервера, настройка доступа и создание базы `logs`.
- Vector — установка из DEB-пакета и настройка отправки событий в ClickHouse.
- LightHouse — загрузка статики, установка Nginx и настройка сайта.

Создание таблицы `logs.events` оставил в tasks после роли ClickHouse. Перед созданием проверяется, есть ли таблица. В Vector настроил генерацию тестовых JSON-событий раз в 10 секунд, чтобы проверить отправку данных.

Для LightHouse нужен веб-сервер, поэтому Nginx устанавливается в той же роли. Через него открывается интерфейс и проксируются запросы к ClickHouse по пути `/clickhouse/`.

## Файлы проекта

| Файл или каталог | Для чего нужен |
| --- | --- |
| `site.yml` | Основной playbook |
| `requirements.yml` | Адреса и версии ролей |
| `inventory/prod.yml` | Адреса ВМ |
| `group_vars/` | Настройки для каждой группы |
| `ansible.cfg` | Параметры запуска Ansible |
| `.ansible-lint` | Настройки проверки кода |

Роли Vector и LightHouse вынесены в отдельные репозитории. Каталог `roles/` добавил в `.gitignore`, так как зависимости устанавливаются через `ansible-galaxy`.

Inventory оставил из предыдущей работы:

| Группа | IP |
| --- | --- |
| clickhouse | 51.250.66.218 |
| vector | 84.201.157.164 |
| lighthouse | 51.250.14.222 |

## Параметры ролей

Параметры для изменения вынес в `defaults/main.yml`, а имена сервисов — в `vars/main.yml`. Шаблоны находятся в `templates`, перезапуск сервисов выполняется через handlers.

У Vector можно изменить версию пакета, архитектуру, адрес ClickHouse, базу, таблицу и интервал генерации событий:

```yaml
vector_version: "0.44.0"
vector_arch: amd64
vector_clickhouse_endpoint: http://51.250.66.218:8123
vector_clickhouse_database: logs
vector_clickhouse_table: events
vector_demo_interval: 10
```

У LightHouse можно изменить версию статики, каталог установки, порт, имя сайта и адрес ClickHouse:

```yaml
lighthouse_version: master
lighthouse_root: /var/www/lighthouse
lighthouse_port: 80
lighthouse_server_name: _
lighthouse_clickhouse_endpoint: http://51.250.66.218:8123
```

Для ClickHouse указал адрес прослушивания через `clickhouse_listen_host`, разрешённые адреса через `clickhouse_networks_default` и базу через `clickhouse_dbs_custom`. Версию сервера зафиксировал в `clickhouse_version` — `26.10.3.7`. Это версия самого ClickHouse, версия Ansible-роли остаётся `1.13`.

В настройках репозитория использовал существующий ключ с `signed-by`, чтобы не добавлять вторую запись с другими параметрами. Ключ загружается в pre_tasks, а адрес репозитория передаётся параметром роли `clickhouse_repo`. Для старой роли также включил `inject_facts_as_vars` в `ansible.cfg`.

## Запуск

Сначала установить роли:

```bash
ansible-galaxy role install -r requirements.yml -p roles
```

В `requirements.yml` для собственных ролей указаны теги `1.0.0`. Для установки по этому файлу они должны быть опубликованы в репозиториях ролей.

Проверить playbook и выполнить запуск:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff
ansible-playbook site.yml --diff
```

В playbook добавил теги `clickhouse`, `vector` и `lighthouse`. Например, отдельно запустить Vector можно так:

```bash
ansible-playbook site.yml --tags vector --diff
```

К этому моменту ClickHouse уже должен работать, а база и таблица — существовать.

## Проверка результата

Синтаксис playbook проверил через `--syntax-check`. Playbook и свои роли проверил с помощью `ansible-lint`, ошибок не было. Для локальной проверки роли находились в соседних каталогах:

```bash
ansible-lint --offline site.yml ../vector-role ../lighthouse-role
```

Стороннюю роль ClickHouse исключил из lint, её исходники не менял.

Запуск с `--check --diff` прошёл без ошибок. В своих ролях для этого режима пропускаю установку из локального DEB, распаковку, запись конфигурации и запуск сервисов. Эти действия требуют файлов и пакетов, которых при первой установке ещё нет, поэтому дополнительно проверил обычный запуск.

После запуска с `--diff` повторил ту же команду. Результат повторного запуска:

| Узел | changed | failed | unreachable |
| --- | --- | --- | --- |
| clickhouse-01 | 0 | 0 | 0 |
| vector-01 | 0 | 0 | 0 |
| lighthouse-01 | 0 | 0 | 0 |

Повторный запуск ничего не изменил, то есть playbook идемпотентен на этом окружении.

Проверил и сами сервисы: все находятся в состоянии `active`. Команды `vector validate` и `nginx -t` выполнились успешно. LightHouse вернул HTTP 200, а запрос `/clickhouse/ping` — `Ok.`. Во время проверки количество строк в `logs.events` увеличилось с 339 до 340 — события от Vector поступают.

[Открыть LightHouse](http://51.250.14.222/#http://51.250.14.222/clickhouse/)

## Ссылки на репозитории

- [Playbook](https://github.com/nevers0nn/work_with_role)
- [Роль Vector](https://github.com/nevers0nn/vector-role)
- [Роль LightHouse](https://github.com/nevers0nn/lighthouse-role)
