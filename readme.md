# Syncthing Zabbix Template

Шаблон Zabbix для мониторинга Syncthing через REST API.
Полностью нативное решение без внешних скриптов, использующее HTTP Agent и JSONPath preprocessing.

## Возможности

### Системные метрики
- Uptime (время работы)
- Использование памяти (allocated/system)
- Количество goroutines
- Локальный Device ID
- Discovery статус
- Общий входящий/исходящий трафик

### Мониторинг папок (Folders LLD)
- Состояние папки (Idle/Syncing/Scanning/Error)
- Объём данных в папке (global bytes)
- Данные для локальной синхронизации (need bytes/files)
- Ошибки синхронизации (pull errors)
- Процент синхронизации (локальный)
- Время последнего изменения состояния

### Мониторинг удалённых устройств (Remote Devices LLD)
- Статус подключения (connected/paused/disconnected)
- Входящий/исходящий трафик по каждому устройству
- Адрес подключения
- Версия клиента Syncthing
- Тип подключения (tcp-client, tcp-server, relay-client, quic-client и др.)
- Время последнего подключения (last seen)
- Признак локального устройства (isLocal)

### Мониторинг синхронизации пар папка→устройство (Folder-Device completion LLD)

Самая ценная часть шаблона — показывает **реальный статус синхронизации** между каждой парой папка+устройство, аналогично веб-интерфейсу Syncthing:

- **completion %** — процент синхронизации для конкретного устройства
- **need bytes** — объём данных для передачи устройству
- **need items** — количество файлов для синхронизации
- **need deletes** — количество удалений для синхронизации
- **remote state** — состояние устройства (valid/unknown/paused)
- **global bytes** — общий объём данных в папке

## Требования

- **Zabbix**: 7.0 и выше
- **Syncthing**: любая версия с REST API (1.12.0+)
- Доступ к Syncthing GUI (по умолчанию порт 8384)

## Установка

### 1. Импорт шаблона

1. Скачайте файл `template_syncthing.xml`
2. Перейдите в **Data collection → Templates → Import**
3. Выберите файл и импортируйте

### 2. Настройка аутентификации в Syncthing

Создайте API Key в Syncthing:
1. Откройте веб-интерфейс Syncthing (обычно `https://localhost:8384`)
2. **Actions → Settings → GUI**
3. Найдите поле **API Key** и скопируйте ключ (или сгенерируйте новый)

Альтернативно используйте Basic Auth (логин/пароль GUI).

### 3. Применение шаблона к хосту

1. Перейдите в **Configuration → Hosts**
2. Выберите хост, на котором работает Syncthing
3. Во вкладке **Templates** добавьте `Template App Syncthing`
4. Во вкладке **Macros** настройте переменные (см. ниже)

## Макросы шаблона

| Макрос | Значение по умолчанию | Описание |
|--------|----------------------|----------|
| `{$SYNCTHING.URL}` | `https://127.0.0.1:8384` | URL Syncthing GUI с протоколом и портом |
| `{$SYNCTHING.USERNAME}` | *(пусто)* | Логин для Basic Auth (если не используется API Key) |
| `{$SYNCTHING.PASSWORD}` | *(пусто)* | Пароль для Basic Auth (если не используется API Key) |
| `{$SYNCTHING.APIKEY}` | *(пусто)* | API Key (приоритет выше Basic Auth) |
| `{$SYNCTHING.COMPLETION.MIN}` | `100` | Минимальный процент completion для срабатывания алёрта |
| `{$SYNCTHING.FOLDER.NEED_BYTES.MAX}` | `104857600` | Порог алёрта для несинхронизированных данных (байт). 100 MB |
| `{$SYNCTHING.DEVICES.MIN_CONNECTED}` | `1` | Минимальное количество подключённых устройств |

### Примеры настройки макросов на хосте

**Вариант 1: API Key (рекомендуется)**
```
{$SYNCTHING.URL} = https://172.17.0.1:8384
{$SYNCTHING.APIKEY} = your_api_key_here
```

**Вариант 2: Basic Auth**
```
{$SYNCTHING.URL} = https://172.17.0.1:8384
{$SYNCTHING.USERNAME} = admin
{$SYNCTHING.PASSWORD} = your_password
```

## Используемые REST API эндпоинты

Шаблон использует следующие официальные эндпоинты Syncthing REST API:

| Эндпоинт | Метод | Назначение |
|----------|-------|------------|
| `/rest/system/status` | GET | Системный статус, uptime, память, goroutines |
| `/rest/system/connections` | GET | Статус подключений всех устройств, трафик |
| `/rest/config/folders` | GET | Конфигурация всех папок (для LLD) |
| `/rest/config/devices` | GET | Конфигурация всех устройств (для LLD) |
| `/rest/db/status?folder=X` | GET | Статус конкретной папки |
| `/rest/db/completion?device=X&folder=Y` | GET | Процент синхронизации устройства с папкой |

Полная документация: https://docs.syncthing.net/dev/rest.html

## Триггеры (алёрты)

### Общие триггеры

| Триггер | Приоритет | Описание |
|---------|-----------|----------|
| Syncthing: Service is not responding | HIGH | Нет данных от API более 5 минут |
| Syncthing: Service has been restarted | WARNING | Uptime менее 10 минут — вероятно рестарт |
| Syncthing: Too few devices connected | AVERAGE | Подключено меньше `{$SYNCTHING.DEVICES.MIN_CONNECTED}` устройств |

### Триггеры папок (для каждой папки)

| Триггер | Приоритет | Описание |
|---------|-----------|----------|
| Folder is in ERROR state | HIGH | Папка в состоянии ошибки |
| Folder has pull errors | WARNING | Есть файлы с ошибками синхронизации |
| Folder needs too much data sync | AVERAGE | need_bytes > `{$SYNCTHING.FOLDER.NEED_BYTES.MAX}` в течение 10 минут |
| Folder is not idle for 1 hour | WARNING | Папка не в idle состоянии более часа |

### Триггеры устройств (для каждого устройства)

| Триггер | Приоритет | Описание |
|---------|-----------|----------|
| Device is disconnected | AVERAGE | Устройство отключено и не приостановлено |
| Device is paused | INFO | Устройство приостановлено |

### Триггеры синхронизации (для каждой пары папка→устройство)

| Триггер | Приоритет | Описание |
|---------|-----------|----------|
| Folder → Device out of sync | WARNING | completion % < `{$SYNCTHING.COMPLETION.MIN}` |
| Folder → Device needs too much data | AVERAGE | need_bytes > `{$SYNCTHING.FOLDER.NEED_BYTES.MAX}` |
| Folder → Device offline and out of sync | HIGH | remote_state = "unknown" И completion < 100% |

## Архитектура решения

```
┌─────────────────────────────────────────────────────────┐
│                    Zabbix Server / Proxy                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Master items (HTTP Agent):                             │
│  ├── /rest/system/status        → system metrics        │
│  ├── /rest/system/connections   → device connections    │
│  ├── /rest/config/folders       → folders LLD           │
│  └── /rest/config/devices       → devices LLD           │
│                                                         │
│  Dependent items (JSONPATH/JS preprocessing):           │
│  ├── Uptime, Memory, Goroutines, DeviceID               │
│  ├── Total in/out bytes                                 │
│  ├── Devices connected/disconnected/paused counts       │
│  └── Folders/Devices total counts                       │
│                                                         │
│  LLD rules:                                             │
│  ├── Folders discovery → per-folder items               │
│  │   └── /rest/db/status?folder=X                       │
│  │       ├── state, need_bytes, pull_errors             │
│  │       └── sync_pct, global_bytes                     │
│  │                                                      │
│  ├── Remote Devices discovery → per-device items        │
│  │   └── (from /rest/system/connections)                │
│  │       ├── connected, paused, in/out_bytes            │
│  │       └── address, client_version, type, last_seen   │
│  │                                                      │
│  └── Folder-Device completion discovery                 │
│      └── /rest/db/completion?device=X&folder=Y          │
│          ├── completion %, need_bytes, need_items       │
│          ├── need_deletes, remote_state                 │
│          └── global_bytes                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Графики

Шаблон создаёт следующие типы графиков:

- **Папки**: Sync status (процент синхронизации)
- **Folder-Device**: Sync progress (completion %), Need bytes

## Кастомизация

### Отключение алёртов для выключенных устройств

Если не хотите получать алёрты когда устройство просто выключено:

```
{$SYNCTHING.COMPLETION.MIN} = 0
```

### Увеличение интервалов опроса

По умолчанию:
- Мастер-элементы системы: **1m**
- Мастер-элементы конфигурации: **5m**
- Статусы папок и completion: **1m**

Измените `delay` в соответствующих item'ах при необходимости.

## Известные особенности

1. **Самоподписанные сертификаты**: шаблон настроен с `verify_peer=NO` и `verify_host=NO` для работы с самоподписанными сертификатами Syncthing.

2. **Нулевая дата Go**: значение `0001-01-01T00:00:00Z` в поле `last_seen` заменяется на `Never` для удобства.

3. **Локальное устройство**: ваше локальное устройство всегда будет показывать 100% completion — это ожидаемое поведение.

4. **API Key приоритетнее**: если заполнен `{$SYNCTHING.APIKEY}`, Basic Auth игнорируется.

## Тестирование API вручную

Проверьте доступность API через curl:

```bash
# Системный статус
curl -sk -H "X-API-Key: YOUR_KEY" "https://172.17.0.1:8384/rest/system/status"

# Список папок
curl -sk -H "X-API-Key: YOUR_KEY" "https://172.17.0.1:8384/rest/config/folders"

# Completion для конкретного устройства
curl -sk -H "X-API-Key: YOUR_KEY" \
  "https://172.17.0.1:8384/rest/db/completion?device=DEVICE_ID&folder=FOLDER_ID"
```

## Полезные ссылки

- Официальная документация Syncthing: https://docs.syncthing.net/
- Syncthing REST API: https://docs.syncthing.net/dev/rest.html
- Официальный сайт Syncthing: https://syncthing.net/
- Официальный сайт Zabbix: https://www.zabbix.com/
- Документация Zabbix: https://www.zabbix.com/documentation/current/en/

## Авторы

- **Создание шаблона, отладка, документация**: Qwen3.7-Max & TerAnYu
- **Идея и тестирование**: TerAnYu
- **Реализация Zabbix-шаблона**: Qwen3.7-Max (AI assistant, Alibaba Cloud)

## Лицензия

MIT License

Шаблон предоставляется "как есть" без каких-либо гарантий. Используйте на свой страх и риск.
