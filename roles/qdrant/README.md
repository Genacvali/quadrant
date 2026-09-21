# Ansible Qdrant Role

## Dependencies
* Ansible >= 2.9
* Коллекция `sberdevices_infra.firewalld`
* Целевая ОС: RedHat-based (RHEL / CentOS / Rocky / Alma), systemd

## Role description
Установка и настройка Qdrant Vector Database в кластерном или одиночном режиме.

Бинарник скачивается из Nexus (proxy для GitHub releases), версия и целевая платформа подставляются в URL:

```
{{ qdrant_nexus_base_url }}/{{ qdrant_version_tag }}/qdrant-{{ qdrant_target_triple }}.tar.gz
```

`qdrant_version` задаётся без префикса `v` (например `1.19.1`), тег релиза `v1.19.1` формируется автоматически. Например, для `qdrant_version: "1.19.1"` на x86_64 RedHat:

```
https://nexus.sberdevices.ru/repository/raw_github.com_qdrant_qdrant_releases_proxy/download/v1.19.1/qdrant-x86_64-unknown-linux-gnu.tar.gz
```

Что делает роль при установке (`qdrant_install`):
1. Открывает порты в firewalld (HTTP 6333, gRPC 6334, P2P 6335, metrics 6336).
2. Устанавливает `tar`, `gzip`, `unzip`; создаёт системного пользователя/группу `qdrant` и директории.
3. Проверяет установленную версию (`qdrant --version`). Если она отличается от `qdrant_version` (или задан `qdrant_force_reinstall`) - скачивает архив из Nexus и устанавливает бинарник в `/usr/local/bin/qdrant`.
4. Устанавливает Web-UI из `dist-qdrant.zip` (если архив есть на хосте или задан `qdrant_web_ui_url`) в `{{ qdrant_static_path }}`.
5. Определяет API-ключ: явный из inventory -> из существующего конфига -> генерирует новый (одинаковый на всех узлах кластера).
6. Шаблонизирует конфиг `/etc/qdrant/qdrant.yaml` (0600), env-файл `/etc/qdrant/qdrant.env` (`QDRANT_START_ARGS`) и unit `/etc/systemd/system/qdrant.service`.
7. Запускает сервис. Standalone - просто `qdrant --config-path ...`. Кластер - сначала первый узел (`--uri`), после его готовности (`/healthz`) остальные по одному (`--uri ... --bootstrap <первый узел>`). При смене бинарника/конфига выполняет рестарт.
8. Выводит итоговую сводку (URL, пути, API-ключ).

## Skip and tags
* qdrant_install  - установка Qdrant
* qdrant_update_conf - обновление конфигов Qdrant (с рестартом при изменениях)
* qdrant_wipe - удаление Qdrant

## Configuration examples

### Установка Qdrant Standalone (как в bash-скрипте)
Секция `cluster` в конфиг не пишется, сервис стартует как `qdrant --config-path /etc/qdrant/qdrant.yaml`.

```yaml
# inventory / group_vars
qdrant_desired_action: qdrant_install
qdrant_version: "1.19.1"
qdrant_cluster_enabled: false
```

### Установка Qdrant в кластерном режиме
Первый хост из `ansible_play_batch` становится bootstrap-узлом. API-ключ генерируется на нём и раздаётся остальным.
Все узлы кластера должны попадать в один play (без `serial`), иначе роль не увидит первый узел.

```yaml
# hosts.yml
qdrant:
  hosts:
    qdrant-01:
      ansible_host: 172.19.13.124   # первый в списке - bootstrap-узел
    qdrant-02:
      ansible_host: 172.19.13.123
    qdrant-03:
      ansible_host: 172.19.13.125

# group_vars/qdrant.yml
qdrant_desired_action: qdrant_install
qdrant_version: "1.19.1"
qdrant_cluster_enabled: true
qdrant_http_port: 8033            # при необходимости
qdrant_default_replication_factor: 2
```

Результат на не-первом узле:
```
# /etc/qdrant/qdrant.env
QDRANT_START_ARGS=--config-path /etc/qdrant/qdrant.yaml --uri http://172.19.13.123:6335 --bootstrap http://172.19.13.124:6335
```

### Установка с TLS
Перед установкой с TLS необходимо сгенерировать сертификаты с помощью vault-agent в `{{ qdrant_tls_dir }}` (по умолчанию `/etc/qdrant/ssl`).

```yaml
qdrant_desired_action: qdrant_install
qdrant_cluster_enabled: true
qdrant_tls_enabled: true
```

### Установка с Web-UI из Nexus
```yaml
qdrant_desired_action: qdrant_install
qdrant_web_ui_url: "https://nexus.sberdevices.ru/<repo>/qdrant-web-ui/<version>/dist-qdrant.zip"
```
Если `qdrant_web_ui_url` не задан, роль ищет архив по пути `qdrant_web_ui_archive` (`/tmp/dist-qdrant.zip`) на целевом хосте; при его отсутствии установка Web-UI пропускается.

### Обновление версии
Достаточно поменять `qdrant_version` в inventory и запустить `qdrant_install` - роль скачает новый бинарник и перезапустит сервис (первый узел, затем остальные по одному).

### Обновление конфигурации
```yaml
qdrant_desired_action: qdrant_update_conf

qdrant_log_level: "DEBUG"
qdrant_max_workers: 4
```

### Удаление Qdrant
```yaml
qdrant_desired_action: qdrant_wipe
```

## Role Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| qdrant_desired_action | Действие с ролью, повторяет значения тегов | nothing | + |
| qdrant_version | Версия Qdrant без `v` (запись с `v` тоже допустима) | 1.19.1 | - |
| qdrant_version_tag | Тег релиза для URL, формируется из `qdrant_version` | v{{ qdrant_version }} | - |
| qdrant_nexus_base_url | Базовый URL Nexus-прокси релизов Qdrant | https://nexus.sberdevices.ru/repository/raw_github.com_qdrant_qdrant_releases_proxy/download | - |
| qdrant_target_triple | Платформа бинарника; по умолчанию по `ansible_architecture` (x86_64 -> `x86_64-unknown-linux-gnu`, aarch64 -> `aarch64-unknown-linux-musl`) | auto | - |
| qdrant_archive_name | Имя архива | qdrant-{{ qdrant_target_triple }}.tar.gz | - |
| qdrant_download_url | Полный URL архива | {{ qdrant_nexus_base_url }}/{{ qdrant_version_tag }}/{{ qdrant_archive_name }} | - |
| qdrant_download_validate_certs | Проверять TLS-сертификат Nexus | true | - |
| qdrant_download_timeout | Таймаут скачивания, сек | 120 | - |
| qdrant_bin_path | Путь установки бинарника | /usr/local/bin/qdrant | - |
| qdrant_tmp_dir | Временная директория на хосте | /tmp/qdrant-install | - |
| qdrant_force_reinstall | Переустановить бинарник даже при совпадении версии | false | - |
| qdrant_user | Пользователь для запуска Qdrant | qdrant | - |
| qdrant_group | Группа для запуска Qdrant | qdrant | - |
| qdrant_data_dir | Основная директория данных | /data/lib/qdrant | - |
| qdrant_storage_path | Директория для хранения данных | {{ qdrant_data_dir }} | - |
| qdrant_snapshots_path | Директория для снепшотов | {{ qdrant_data_dir }}/snapshots | - |
| qdrant_static_path | Директория Web-UI (static content) | {{ qdrant_data_dir }}/static | - |
| qdrant_log_file | Файл лога | {{ qdrant_data_dir }}/qdrant.log | - |
| qdrant_config_dir | Директория конфигурации | /etc/qdrant | - |
| qdrant_config_file | Файл конфигурации | {{ qdrant_config_dir }}/qdrant.yaml | - |
| qdrant_env_file | Env-файл для systemd | {{ qdrant_config_dir }}/qdrant.env | - |
| qdrant_tls_dir | Директория для SSL/TLS сертификатов | {{ qdrant_config_dir }}/ssl | - |
| qdrant_service_file | Путь systemd unit | /etc/systemd/system/qdrant.service | - |
| qdrant_web_ui_enabled | Включить Web-UI | true | - |
| qdrant_web_ui_url | URL архива dist-qdrant.zip (если пуст - не скачивать) | "" | - |
| qdrant_web_ui_archive | Путь архива Web-UI на хосте | /tmp/dist-qdrant.zip | - |
| qdrant_host | Host для Qdrant сервиса | {{ ansible_default_ipv4.address }} | - |
| qdrant_http_port | HTTP API порт | 6333 | - |
| qdrant_grpc_port | gRPC порт | 6334 | - |
| qdrant_p2p_port | P2P порт для кластера | 6335 | - |
| qdrant_metrics_port | Отдельный порт `/metrics` для мониторинга (без API-key) | 6336 | - |
| qdrant_cluster_enabled | Режим: `false` - standalone (без секции `cluster`, без `--uri`), `true` - кластер (`--uri` / `--bootstrap`) | false | - |
| qdrant_tls_enabled | Включение TLS для API | false | - |
| qdrant_p2p_tls_enabled | Включение TLS для P2P коммуникаций | false | - |
| qdrant_tls_cert | Путь к сертификату | {{ qdrant_tls_dir }}/server.crt | - |
| qdrant_tls_key | Путь к ключу | {{ qdrant_tls_dir }}/server.key | - |
| qdrant_tls_ca_cert | Путь к CA сертификату | {{ qdrant_tls_dir }}/server-ca.crt | - |
| qdrant_tls_verify_enabled | Включение проверки клиентских сертификатов | false | - |
| qdrant_max_workers | Максимальное количество воркеров (0 - авто) | 0 | - |
| qdrant_max_search_threads | Максимальное количество потоков поиска (0 - авто) | 0 | - |
| qdrant_request_size_limit_mb | Лимит размера запроса в MB | 32 | - |
| qdrant_log_level | Уровень логирования | INFO | - |
| qdrant_log_format | Формат логов (`text` / `json`) | text | - |
| qdrant_log_to_file | Запись логов в файл | true | - |
| qdrant_snapshots_storage | Тип хранилища для снепшотов | local | - |
| qdrant_wal_capacity_mb | Размер WAL-сегмента, MB | 32 | - |
| qdrant_wal_segments_ahead | Количество WAL-сегментов, создаваемых заранее | 0 | - |
| qdrant_default_replication_factor | Фактор репликации по умолчанию | 1 | - |
| qdrant_default_write_consistency_factor | Фактор согласованности записи | 1 | - |
| qdrant_on_disk_payload | Хранение payload на диске | true | - |
| qdrant_on_disk_vectors | Хранение векторов на диске | null | - |
| qdrant_deleted_threshold | Порог удаления для оптимизации | 0.2 | - |
| qdrant_vacuum_min_vector_number | Минимальное количество векторов для vacuum | 1000 | - |
| qdrant_default_segment_number | Количество сегментов по умолчанию (0 - авто) | 0 | - |
| qdrant_max_segment_size_kb | Максимальный размер сегмента в KB | null | - |
| qdrant_indexing_threshold_kb | Порог индексации в KB | 10000 | - |
| qdrant_flush_interval_sec | Интервал сброса в секундах | 5 | - |
| qdrant_hnsw_m | Параметр M для HNSW индекса | 16 | - |
| qdrant_hnsw_ef_construct | Параметр ef_construct для HNSW | 100 | - |
| qdrant_hnsw_full_scan_threshold_kb | Порог полного сканирования HNSW в KB | 10000 | - |
| qdrant_hnsw_on_disk | HNSW индекс на диске | false | - |
| qdrant_optimizer_cpu_budget | Бюджет CPU для оптимизатора | 0 | - |
| qdrant_async_scorer | Асинхронный скоринг | false | - |
| qdrant_consensus_tick_period_ms | Период тика консенсуса в мс | 100 | - |
| qdrant_consensus_compact_wal_entries | Количество WAL записей для компактификации | 128 | - |
| qdrant_enable_cors | Включение CORS | true | - |
| qdrant_api_key | API ключ; если пуст - берётся из существующего конфига или генерируется | "" | - |
| qdrant_api_key_generate | Генерировать API ключ автоматически, если не задан | true | - |
| qdrant_read_only_api_key | API ключ только для чтения | null | - |
| qdrant_jwt_rbac | Включение JWT RBAC | true | - |
| qdrant_show_api_key | Показать API ключ в итоговом выводе роли | true | - |
| qdrant_telemetry_disabled | Отключение телеметрии | true | - |
| qdrant_healthcheck_retries | Попыток ожидания `/healthz` после старта | 12 | - |
| qdrant_healthcheck_delay | Пауза между попытками, сек | 5 | - |
| qdrant_diag_journal_lines | Строк `journalctl -u qdrant` в выводе при провале health-check | 50 | - |
| qdrant_metrics_enabled | Включение метрик | true | - |
| qdrant_metrics_prefix | Префикс метрик | qdrant_ | - |

## Требования к сертификатам

При `qdrant_tls_enabled: true` необходимо разместить сертификаты в `{{ qdrant_tls_dir }}` (по умолчанию `/etc/qdrant/ssl/`):

```
/etc/qdrant/ssl/
├── server.crt        # Серверный сертификат
├── server.key        # Приватный ключ сервера
└── server-ca.crt     # CA сертификат
```

## Directory Structure

```
/usr/local/bin/qdrant                 # бинарник
/etc/qdrant/
├── qdrant.yaml                       # основной конфиг (0600, qdrant:qdrant)
├── qdrant.env                        # env для systemd (QDRANT_START_ARGS)
└── ssl/                              # сертификаты (опционально)
/etc/systemd/system/qdrant.service    # unit
/data/lib/qdrant/                     # storage_path
├── snapshots/                        # снепшоты
├── static/                           # Web-UI
└── qdrant.log                        # лог
```

## Standalone vs Cluster

| | `qdrant_cluster_enabled: false` | `qdrant_cluster_enabled: true` |
|---|---|---|
| Секция `cluster` в `qdrant.yaml` | отсутствует | `enabled: true`, `p2p.port`, `consensus` |
| `QDRANT_START_ARGS` (первый узел) | `--config-path <cfg>` | `--config-path <cfg> --uri http://<ip>:6335` |
| `QDRANT_START_ARGS` (остальные) | - | `--config-path <cfg> --uri http://<own_ip>:6335 --bootstrap http://<first_ip>:6335` |
| Порядок запуска | один хост | первый узел -> `/healthz` -> остальные по одному |
| Проверка | `/healthz` | `/healthz` + `/cluster` |

Первый узел - первый хост в `ansible_play_batch`. API-ключ единый для всех узлов.

## API Endpoints

- HTTP API: `http(s)://<host>:6333`
- Web-UI: `http(s)://<host>:6333/dashboard`
- gRPC API: `<host>:6334`
- Cluster P2P: `<host>:6335`
- Metrics (без API-key): `http://<host>:6336/metrics`
- Health Check: `http(s)://<host>:6333/healthz`

## Additional Features

- Интеграция с Firewalld для настройки правил брандмауэра
- Отдельный порт метрик для систем мониторинга
- Автогенерация и сохранение API-ключа между запусками
- Обновление версии сменой `qdrant_version`

## Changes and Releases

Release 1.2.0
* `qdrant_version` задаётся без префикса `v` (`1.19.1`); тег для URL - `qdrant_version_tag`
* Секция `cluster` в конфиге пишется только при `qdrant_cluster_enabled: true`
* Запуск через `QDRANT_START_ARGS` в env-файле: standalone без `--uri`, кластер с `--uri` / `--bootstrap`
* При провале health-check роль выводит `systemctl status qdrant` и `journalctl -u qdrant` (`qdrant_diag_journal_lines`)

Release 1.1.0
* Скачивание бинарника из Nexus `raw_github.com_qdrant_qdrant_releases_proxy` с подстановкой версии и платформы
* Установка/обновление бинарника по сравнению версий (`qdrant --version`)
* Пути и структура по образцу bash-скрипта: `/data/lib/qdrant`, `/etc/qdrant`, env-файл, `--uri`
* Web-UI из `dist-qdrant.zip` (`service.static_content_dir`)
* Автогенерация API-ключа, `jwt_rbac`
* Отдельный `metrics_port: 6336`, открыт в firewalld
* Исправлен несуществующий handler в `update_conf`
* Удалена интеграция с filebeat

Release 1.0.0-rc1

## Maintainers
* vdbaranov@sberdevices.ru
