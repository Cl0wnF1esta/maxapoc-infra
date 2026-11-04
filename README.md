# infra-lab

Ansible-код, который приводит сервер Ubuntu 24.04 к рабочему состоянию проекта MaxApoc одной командой: базовая настройка и защита, Docker, прокси приложения, доступ для деплоя из CI и собственный стек мониторинга (Prometheus, Alertmanager с оповещениями в Telegram, Grafana). Повторный запуск ничего не меняет (`changed=0`).

Новые версии прокси выкатывает не Ansible, а CI репозитория MaxApoc: слияние в `main` → тесты → образ в GHCR → деплой по digest через ограниченный ключ, с проверкой здоровья и автооткатом. Ansible готовит сервер и файлы запуска; версией владеет деплой.

Состояние, которое нельзя воссоздать из кода (база x-ui, WireGuard, данные соседнего стека мониторинга, версия прокси), ежедневно копируется в зашифрованном виде в S3 хостера, раз в месяц — на управляющую машину; восстановление на чистую ВМ проверено учениями.

Проверено:

- **тестовая среда:** чистая ВМ из снимка → одна команда (6 мин 05 с через HTTP-прокси лаборатории) → первый деплой (~1 мин) → повторный запуск `changed=0`;
- **рабочий сервер:** переведён на этот код с работающего ручного развёртывания (порядок и откат — в [docs/OPERATIONS.md](docs/OPERATIONS.md)); обновляется деплоем из CI (подтверждение ~7 с, простой ~1 с); повторный запуск Ansible `changed=0`.
- **резервные копии:** ежедневно (restic → S3), с оповещением о сбое; копия рабочего сервера восстановлена на чистую ВМ и сверена с живым сервером (порядок — [docs/OPERATIONS.md](docs/OPERATIONS.md), разделы 8–9).

Решения и их причины — в [docs/DECISIONS.md](docs/DECISIONS.md), модель угроз и известные ограничения — в [docs/SECURITY.md](docs/SECURITY.md).

## Что делает плейбук

`ansible/site.yml` сначала обновляет на управляющей машине чистый клон MaxApoc до ветки `main` (оттуда роли берут compose, `gateways.json` и наблюдаемость), затем применяет роли по порядку:

| Роль | Что делает |
|---|---|
| `common` | Проверяет ОС (только Ubuntu 24.04), ставит базовые пакеты, задаёт часовой пояс, создаёт и включает swap-файл |
| `security` | Запрещает вход по паролю и под root (sshd, с проверкой конфига перед применением); ufw: SSH с ограничением частоты, дополнительные порты из списка, «запретить входящие» по умолчанию |
| `docker` | Подключает официальный репозиторий Docker (убирает старый `docker.list`, если Docker ставили по инструкции с сайта), ставит закреплённые версии пакетов и удерживает их от обновления; `daemon.json` с `live-restore` и, при необходимости, прокси для скачивания образов |
| `backup` | restic и sqlite3; репозиторий окружения в S3 (создаётся один раз); скрипт копии (согласованные снимки баз SQLite → `restic backup` → хранение 7/4/6), таймер systemd ежедневно, оповещение в Telegram при сбое, `restic check` по воскресеньям. Идёт до роли прокси: та берёт из копии версию на чистом сервере. Включается переменной `backup_enabled` |
| `maxapoc_proxy` | Создаёт общую Docker-сеть проекта; входит в реестр образов (токен только на чтение); раскладывает compose, `.env`, сертификаты и `gateways.json`; на сервере без `release.env` берёт версию из последней резервной копии окружения (нет копии — начальную версию из инвентаря) и запускает контейнер от непривилегированного пользователя. Версию образа дальше меняет только деплой |
| `maxapoc_deploy_access` | Пользователь деплоя `maxapoc-deploy` с ключом CI (принудительная команда), скрипт-фильтр команд и sudo на один скрипт деплоя. Включается переменной `maxapoc_deploy_access_enabled` |
| `maxapoc_monitoring` | Поднимает стек мониторинга в той же сети: Prometheus (метрики прокси и правила оповещений проекта), Alertmanager (оповещения в Telegram), Grafana (источник данных и панели из файлов проекта); в конце проверяет здоровье всех трёх сервисов по HTTP. Включается переменной `maxapoc_monitoring_enabled` |

### Как устроен стек

```
              сеть Docker maxapoc_net (без публикации наружу)
 ┌──────────────────────────────────────────────────────────────────┐
 │  maxapoc-proxy ──:9101/metrics──▶ maxapoc-prometheus ──▶ maxapoc-alertmanager ──▶ Telegram
 │        :8443                          ▲      :9090               :9093
 │                                       │
 │                               maxapoc-grafana :3000
 └──────────────────────────────────────────────────────────────────┘
 Наружу опубликован только порт прокси 8443.
 9090, 9093, 3000 (и временно 9101) — только на локальном адресе сервера, доступ через SSH-туннель.
```

- Prometheus находит прокси по имени `maxapoc-proxy` в общей сети, а не через адрес хоста.
- Сеть создаёт роль `maxapoc_proxy` до запуска любого compose-проекта; оба проекта подключаются к ней как к внешней (`external: true`). Имя сети — одна переменная `maxapoc_network_name`.

### Окружения

| | Тестовая среда (`lab`) | Рабочий сервер (`prod`) |
|---|---|---|
| Инвентарь | `ansible/inventory/lab` — **по умолчанию** | `ansible/inventory/prod` — только явно, `-i inventory/prod` |
| Пароль vault | `lab` | `prod` (отдельный) |
| Стек мониторинга этого репозитория | включён | **выключен** (`maxapoc_monitoring_enabled: false`): памяти сервера не хватает на второй стек, метрики и оповещения MaxApoc обслуживает уже работающий там стек `hobot-metrics` |
| Метрики прокси (`9101`) | на `127.0.0.1` | на адресе WireGuard — для Prometheus `hobot-metrics` |
| HTTP-прокси для загрузок | есть (шаг 7) | нет |
| Деплой новой версии прокси | вручную с управляющей машины (CI до сети лаборатории не достаёт) | CI по слиянию в `main` |
| Ключ деплоя, токен реестра | свои | свои |
| Резервные копии | репозиторий `lab/`, свой пароль | репозиторий `prod/`, свой пароль; ежемесячно копия домой |

## Запуск с нуля

### 1. Требования к управляющей машине

- Ansible с `ansible-core` версии **2.18 или новее**.
  Минимальная версия определяется коллекциями проекта: `community.general` требует ansible-core >= 2.18, `ansible.posix` — >= 2.16, `community.docker` — >= 2.17. Подходит версия, которая удовлетворяет всем требованиям.
- Доступ по SSH к репозиторию MaxApoc на GitHub (тот же ключ, что для `git push`): плейбук сам клонирует и обновляет `~/.cache/infra-lab/APP_REPO` до `main`. Рабочая копия MaxApoc для запуска не нужна.
- Пароли от Ansible Vault (шаг 3).

```bash
ansible --version
```

> Файлы MaxApoc берутся **только из ветки `main`**. Чтобы рабочий сервер получил ровно то, что проверено в тестовой среде, запускайте его с `-e maxapoc_repo_version=<хеш коммита>` (хеш — `git -C ~/.cache/infra-lab/APP_REPO rev-parse HEAD` после запуска тестовой среды). Ручные правки в клоне останавливают запуск.

### 2. Требования к целевому серверу

- ОС: Ubuntu Server 24.04 (на другой ОС плейбук остановится на первой задаче с понятным сообщением).
- Не меньше 1 ГБ памяти. Ориентир по замеру в тестовой среде: прокси ~30 МБ, Prometheus ~20 МБ, Alertmanager ~15 МБ, Grafana ~300 МБ.
- Установлен `openssh-server`.
- **Отдельный пользователь с правами sudo (по паролю), не root.** SSH-ключ управляющей машины добавлен этому пользователю:

```bash
ssh-copy-id -i <путь_к_публичному_ключу> <пользователь>@<адрес_сервера>
```

> Роль `security` запрещает вход под root и по паролю. Если Ansible подключается под root, первый же запуск отрежет ему доступ. Перед запуском проверьте вход **только ключом** под этим пользователем: `ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no <пользователь>@<адрес_сервера> 'id'`.

- Доступ в интернет к зеркалам Ubuntu, `download.docker.com`, Docker Hub, `ghcr.io` и `api.telegram.org`. Если прямого доступа нет — см. шаг 7.
- Если на сервере уже что-то работает (Docker, контейнеры, правила ufw), первый запуск — это **миграция**, а не установка: см. [docs/OPERATIONS.md](docs/OPERATIONS.md), раздел «Перевод работающего сервера».

### 3. Пароли Ansible Vault

Секреты хранятся в зашифрованных файлах `vault.yml`, у каждого окружения свой пароль (vault ID `lab` и `prod`). Пароли лежат в файлах, пути к которым указаны в `ansible.cfg`:

```ini
# ansible/ansible.cfg
vault_identity_list = lab@./vault-secrets/.lab_vault_pass, prod@./vault-secrets/.prod_vault_pass
```

```bash
mkdir -p ansible/vault-secrets
nano ansible/vault-secrets/.lab_vault_pass      # пароль тестовой среды из менеджера паролей
nano ansible/vault-secrets/.prod_vault_pass     # пароль рабочего сервера
chmod 600 ansible/vault-secrets/.*_vault_pass
```

> Каталог `vault-secrets/` целиком в `.gitignore`. Файлы паролей лежат на диске **открытым текстом**: их защищают только права `0600`. Копии паролей — в менеджере паролей. Потерянный пароль восстановить нельзя: секреты этого окружения придётся выпустить заново.

Смена пароля окружения: сначала перешифровать его файлы новым паролем, и только потом заменить файл пароля:

```bash
ansible-vault rekey --vault-id prod@./vault-secrets/.prod_vault_pass --new-vault-id prod@<файл_с_новым_паролем> <файлы vault.yml прода>
```

### 4. Настройка инвентаря

```
ansible/
├── ansible.cfg                       # inventory = ./inventory/lab
├── site.yml
├── group_vars/
│   └── all.yml                       # общее для всех окружений
└── inventory/
    ├── lab/
    │   ├── hosts
    │   └── host_vars/infralab/       # vars.yml, vault.yml (vault ID lab)
    └── prod/
        ├── hosts
        └── host_vars/maxapoc-prod/   # vars.yml, vault.yml (vault ID prod)
```

**`inventory/<окружение>/hosts`** — группа окружения внутри общей группы `infra`:

```ini
[lab]
infralab ansible_host=<адрес_сервера>

[infra:children]
lab
```

- `infralab` — алиас, по которому Ansible обращается к хосту. Его можно изменить, но вместе с именем папки в `host_vars/`.
- Группу `infra` менять нельзя: на неё ссылается `site.yml` (`hosts: infra`).

**`ansible/group_vars/all.yml`** — общие переменные. Лежит рядом с плейбуком, а не в инвентаре, поэтому одинаково подключается к любому окружению:

```yaml
maxapoc_repo_url: git@github.com:<владелец>/<репозиторий_MaxApoc>.git
maxapoc_repo_clone: <домашний_каталог>/.cache/infra-lab/APP_REPO   # клон, который ведёт сам плейбук
maxapoc_repo_version: main
maxapoc_proxy_src: "{{ maxapoc_repo_clone }}/proxy"
maxapoc_monitoring_stack_src: "{{ maxapoc_repo_clone }}/observability"
backup_s3_endpoint: https://<адрес_S3_хостера>
backup_s3_bucket: <бакет>             # приватный, стандартный класс, с версионированием
backup_s3_region: <регион>
maxapoc_network_name: maxapoc_net    # общая Docker-сеть; нужна обеим ролям, поэтому лежит здесь
proxy_env: {}                        # прокси не используется (см. шаг 7)
```

**`inventory/<окружение>/host_vars/<хост>/`** — переменные хоста. Имя папки совпадает с алиасом из `hosts`. Внутри два файла.

`vars.yml` — обычные переменные:

```yaml
ansible_user: <пользователь>                 # пользователь с правами sudo, не root
ansible_ssh_private_key_file: <путь_к_приватному_ключу>
# ansible_port: 22                           # если SSH не на 22

common_swap_size_mb: 512                     # размер swap в мегабайтах

# Дополнительные порты ufw (SSH разрешается ролью всегда, указывать не нужно)
security_ufw_allowed_ports:
  - name: Maxapoc-proxy                      # попадает в комментарий правила ufw
    proto: tcp
    port: 8443
  # - name: <назначение>
  #   proto: tcp | udp
  #   port: "<порт или диапазон 1000:2000 — диапазон в кавычках>"
  #   src: <адрес_источника>                 # необязательно: без src — для всех

# maxapoc_proxy_metrics_bind: <адрес>        # где публиковать метрики прокси (по умолчанию 127.0.0.1)

# Начальная версия прокси — только для сервера без release.env (дальше версией владеет деплой).
# Держать свежей и совместимой с текущим compose.
maxapoc_proxy_image: ghcr.io/<владелец>/maxapoc-proxy@sha256:<digest>
maxapoc_proxy_registry_token: "{{ vault_maxapoc_proxy_registry_token }}"

# Доступ для деплоя из CI: публичная часть ключа (не секрет; подмена видна в истории git)
maxapoc_deploy_access_enabled: true
maxapoc_deploy_pubkey: "ssh-ed25519 AAAA… maxapoc-deploy@<окружение>-ci"

# Резервные копии: свой репозиторий (префикс в бакете) у окружения; что копировать — списки ниже
backup_enabled: true
backup_repo_name: <lab|prod>
backup_s3_access_key: "{{ vault_backup_s3_access_key }}"
backup_s3_secret_key: "{{ vault_backup_s3_secret_key }}"
backup_restic_password: "{{ vault_backup_restic_password }}"
backup_sqlite_dbs:                      # базы SQLite — согласованным снимком
  - { src: /путь/к/базе.db, name: имя-снимка.db }
backup_paths:                           # файлы и каталоги как есть
  - /opt/maxapoc/proxy/vps/release.env
backup_alert_telegram_token: "{{ maxapoc_monitoring_telegram_token }}"
backup_alert_telegram_chat_id: "{{ maxapoc_monitoring_telegram_chat_id }}"
# backup_alert_proxy: http://<прокси>    # если до api.telegram.org нет прямого доступа
# maxapoc_monitoring_enabled: false          # не ставить стек мониторинга на этот хост

# Ссылки на секреты из vault.yml (роли читают эти имена, а не vault_*)
maxapoc_proxy_gh_app_token: "{{ vault_maxapoc_proxy_gh_app_token }}"
maxapoc_proxy_app_key: "{{ vault_maxapoc_proxy_app_key }}"
maxapoc_proxy_llm_odirouter_key: "{{ vault_maxapoc_proxy_llm_odirouter_key }}"
maxapoc_proxy_llm_routeway_key: "{{ vault_maxapoc_proxy_llm_routeway_key }}"
maxapoc_proxy_gh_hub_token: "{{ vault_maxapoc_proxy_gh_hub_token }}"
maxapoc_proxy_server_crt: "{{ vault_maxapoc_proxy_server_crt }}"
maxapoc_proxy_server_key: "{{ vault_maxapoc_proxy_server_key }}"
maxapoc_proxy_ca_crt: "{{ vault_maxapoc_proxy_ca_crt }}"
maxapoc_monitoring_telegram_token: "{{ vault_maxapoc_monitoring_telegram_token }}"
maxapoc_monitoring_telegram_chat_id: "{{ vault_maxapoc_monitoring_telegram_chat_id }}"
maxapoc_monitoring_grafana_admin_password: "{{ vault_maxapoc_monitoring_grafana_admin_password }}"
```

Значения по умолчанию ролей лежат в `roles/<роль>/defaults/main.yml` и при необходимости переопределяются в `vars.yml`:

- `maxapoc_proxy`: папка `/opt/maxapoc/proxy`, UID/GID процесса в контейнере 1000, метрики на `127.0.0.1:9101`, реестр `ghcr.io`, имена репозиториев;
- `maxapoc_deploy_access`: пользователь `maxapoc-deploy`, по умолчанию выключена;
- `maxapoc_monitoring`: включён, папка `/opt/maxapoc/monitoring`, адрес публикации интерфейсов `127.0.0.1`, закреплённые версии образов (Prometheus v3.13.2, Alertmanager v0.28.1, Grafana 13.2.0), лимиты памяти и процессора, срок хранения метрик (90 дней / 2 ГБ), интервал опроса 60 с;
- `docker`: `docker_live_restore: true`, закреплённые версии пакетов.

`vault.yml` — секреты, создаётся зашифрованным с меткой своего окружения:

```bash
ansible-vault create --encrypt-vault-id lab inventory/lab/host_vars/infralab/vault.yml
head -1 inventory/lab/host_vars/infralab/vault.yml     # $ANSIBLE_VAULT;1.2;AES256;lab
```

Нужные переменные (у каждого окружения — свои значения):

| Переменная | Что это |
|---|---|
| `vault_maxapoc_proxy_gh_app_token` | токен GitHub для репозитория приложения (Issues: read/write) |
| `vault_maxapoc_proxy_app_key` | ключ доступа приложения к прокси, **не короче 16 символов** (`openssl rand -hex 24`) |
| `vault_maxapoc_proxy_llm_odirouter_key` | ключ шлюза ODIRouter |
| `vault_maxapoc_proxy_llm_routeway_key` | ключ шлюза Routeway |
| `vault_maxapoc_proxy_gh_hub_token` | токен GitHub для репозитория хаба |
| `vault_maxapoc_proxy_server_crt` | сертификат сервера (шаг 5) |
| `vault_maxapoc_proxy_server_key` | закрытый ключ сервера (шаг 5) |
| `vault_maxapoc_proxy_ca_crt` | сертификат CA (шаг 5) |
| `vault_maxapoc_monitoring_telegram_token` | токен Telegram-бота для оповещений |
| `vault_maxapoc_monitoring_telegram_chat_id` | номер чата для оповещений (у группы — отрицательный); число, без кавычек |
| `vault_maxapoc_monitoring_grafana_admin_password` | пароль пользователя `admin` в Grafana |
| `vault_backup_s3_access_key`, `vault_backup_s3_secret_key` | ключи доступа к бакету S3 |
| `vault_backup_restic_password` | пароль репозитория restic окружения; **копия обязательно в менеджере паролей** — без него копии не расшифровать |
| `vault_maxapoc_proxy_registry_token` | токен GitHub (classic) **только** `read:packages` для скачивания образа; срок 90 дней, у каждого окружения свой |

> Для тестовой среды значения можно выдумать, но они должны проходить те же проверки формата, что и настоящие — иначе прокси не запустится. Для оповещений у тестовой среды **свой бот и свой чат**: токен рабочего бота в тестовую среду не попадает, а учебные алерты не смешиваются с настоящими.

Проверить, что секреты подключаются к хосту, **не выводя их значений**:

```bash
ansible -i inventory/prod maxapoc-prod -m debug -a 'msg="{{ vault_maxapoc_proxy_app_key is defined }}"'   # true
```

> Не используйте для проверки `ansible-inventory --host`: эта команда печатает все переменные хоста, включая расшифрованные секреты.

### 5. Сертификаты прокси

Приложение доверяет только сертификатам, подписанным CA проекта. Сертификат сервера выпускается на его адрес скриптом из репозитория прокси, **в папку вне этого репозитория**:

```bash
<репозиторий_MaxApoc>/proxy/vps/make-certs.sh <адрес_сервера> <папка_вне_репозитория>
```

- `ca.key` остаётся только у владельца в защищённом хранилище. **На сервер и в git не копируется никогда.**
- Содержимое `server.crt`, `server.key` и `ca.crt` вставляется в `vault.yml` многострочными значениями. Строки после `|` сдвинуты на два пробела:

```yaml
vault_maxapoc_proxy_server_key: |
  <содержимое server.key целиком, каждая строка с отступом в 2 пробела>
  <первая и последняя строки — заголовки BEGIN … / END … из файла>
```

У тестовой среды свой CA; ключи и сертификаты рабочего сервера в неё не попадают.

### 6. Установка коллекций

Из папки `ansible/`:

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection list      # коллекции из ansible/collections/ должны быть первыми
```

Коллекции ставятся в `ansible/collections/` (задано в `ansible.cfg`, папка не хранится в git).

### 7. Прокси для тестовой среды (необязательно)

Если у сервера нет прямого доступа к `download.docker.com`, Docker Hub или `api.telegram.org`, а у управляющей машины есть HTTP-прокси, его можно открыть **только для сети лаборатории** через `socat`. Пример службы `/etc/systemd/system/lab-proxy.service` на управляющей машине (прокси на `127.0.0.1:10809`, адрес машины в сети лаборатории `192.168.56.2`):

```ini
[Unit]
Description=Forward lab network to local HTTP proxy
After=network-online.target

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:10809,bind=192.168.56.2,reuseaddr,fork TCP:127.0.0.1:10809
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now lab-proxy
```

В `vars.yml` хоста:

```yaml
proxy_env:
  http_proxy: http://192.168.56.2:10809
  https_proxy: http://192.168.56.2:10809
  no_proxy: localhost,127.0.0.1,192.168.56.0/24
```

Кто и как пользуется этим прокси:

| Кто | Как | Зачем |
|---|---|---|
| Задачи Ansible (apt, загрузки, проверки `uri`) | `environment:` плея | установка пакетов, ключей; запросы к `127.0.0.1` идут мимо прокси благодаря `no_proxy` |
| Демон Docker | `/etc/docker/daemon.json` (роль `docker`) | скачивание образов |
| Alertmanager | `http_config.proxy_url` получателя Telegram; адрес берётся из `proxy_env.https_proxy` | отправка оповещений |

Остальные процессы в контейнерах о прокси не знают. Grafana в интернет не ходит: проверка обновлений и отправка статистики отключены.

У рабочего сервера `proxy_env` остаётся пустым, и блок прокси в конфиге Alertmanager не создаётся.

### 8. Запуск плейбука

Запускать **из папки `ansible/`**: Ansible ищет `ansible.cfg` в текущем каталоге, а в нём указаны инвентарь по умолчанию, роли, коллекции и пароли vault.

| | Тестовая среда | Рабочий сервер |
|---|---|---|
| Предварительный просмотр | `ansible-playbook site.yml --check --diff` | `ansible-playbook -i inventory/prod site.yml --check --diff` |
| Применить | `ansible-playbook site.yml` | `ansible-playbook -i inventory/prod site.yml` |

Ansible запросит пароль sudo пользователя на целевом сервере (`BECOME password`). Пароль спрашивается один раз на запуск, поэтому окружения запускаются **по отдельности**.

> Предварительный просмотр — это **прогноз**. Задачи с командами в нём не выполняются, а задача, которая зависит от изменённого файла, сравнивает со старым файлом — например, пересоздание контейнера после нового `.env` не видно (изменений показано меньше); модулям, которым для сравнения нужна сеть, приходится сообщать «изменилось бы» (показано больше), а конфликты между файлами (например, два описания одного репозитория apt) он не видит.

### 9. Проверка

**Плейбук**

- Повторный запуск завершается с `changed=0`.
- Последние задачи роли `maxapoc_monitoring` сами проверяют здоровье стека по HTTP (`/-/ready` у Prometheus и Alertmanager, `/api/health` у Grafana) и ждут ответа до ~50 секунд. Если сервис не поднялся, запуск останавливается с ошибкой.

**Прокси**

- На сервере контейнер прокси в статусе `healthy`, образ из реестра по digest (тот, что в `release.env`), процесс не от root:

```bash
sudo docker inspect maxapoc-proxy --format '{{.Config.Image}} {{.State.Health.Status}}'   # ghcr.io/…@sha256:… healthy
sudo cat /opt/maxapoc/proxy/vps/release.env
sudo docker exec maxapoc-proxy id          # uid=1000
```

- Резервные копии: `systemctl list-timers backup.timer`; `sudo systemctl start backup.service && echo ok`; `sudo /usr/local/sbin/restic-env snapshots`.
- Доступ деплоя (с управляющей машины, ключом окружения): `ssh -q -o IdentitiesOnly=yes -i <ключ_деплоя> maxapoc-deploy@<адрес_сервера> status` → образ и `healthy`; любая другая команда → `denied`.

- С управляющей машины прокси отвечает по TLS **с проверкой сертификата** (ответ `401` означает, что TLS в порядке, а приложение требует ключ):

```bash
curl --noproxy '*' --cacert <папка_с_сертификатами>/ca.crt -I https://<адрес_сервера>:8443/
```

**Docker**

- `live-restore` включён: перезапуск демона Docker не перезапускает контейнеры.

```bash
sudo docker info --format '{{.LiveRestoreEnabled}}'     # true
```

**Мониторинг** (где включён) — через SSH-туннель с управляющей машины (держать открытым в отдельном терминале):

```bash
ssh -N -L 9090:127.0.0.1:9090 -L 9093:127.0.0.1:9093 -L 3000:127.0.0.1:3000 <пользователь>@<адрес_сервера>
```

```bash
# цель maxapoc-proxy в состоянии up
curl --noproxy '*' -s localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"job"|"health"|"lastError"'
# 20 правил оповещений загружено, Alertmanager подключён
curl --noproxy '*' -s localhost:9090/api/v1/rules | python3 -m json.tool | grep -c '"type": "alerting"'
curl --noproxy '*' -s localhost:9090/api/v1/alertmanagers | python3 -m json.tool
```

Если `curl` не может подключиться, сначала проверьте, что туннель открыт и пробрасывает нужный порт: `ss -tlnp | grep ssh`.

- Grafana: `http://localhost:3000`, пользователь `admin`, пароль из vault. В папке **MaxApoc** две панели: «прокси» и «ИИ и хаб».
- Доставка в Telegram: на сервере положить тестовое правило из репозитория MaxApoc в каталог правил и перечитать конфигурацию Prometheus. Через 1–2 минуты придёт 🔴, после удаления файла и повторного перечитывания — 🟢 (через 5–10 минут):

```bash
sudo install -o root -g root -m 0644 maxapoc-test-alerts.yml /opt/maxapoc/monitoring/prometheus/rules/
sudo docker kill -s HUP maxapoc-prometheus
# после проверки — обязательно удалить: роль сама файл не уберёт
sudo rm /opt/maxapoc/monitoring/prometheus/rules/maxapoc-test-alerts.yml
sudo docker kill -s HUP maxapoc-prometheus
```

- Время реакции (замер в тестовой среде): о падении прокси оповещение приходит примерно через 3,5 минуты (опрос раз в минуту + `for: 2m` в правиле + 30 секунд группировки).

### 10. Деплой из CI (рабочий сервер)

Настраивается один раз, после первого запуска плейбука (он создаёт пользователя деплоя):

- Ключ деплоя окружения: `ssh-keygen -t ed25519 -f ~/.ssh/maxapoc_deploy_<окружение> -C "maxapoc-deploy@<окружение>-ci" -N ""`; публичная часть — в `maxapoc_deploy_pubkey`.
- GitHub → Settings → Environments → **`production`**: правило веток — только `main`; секреты окружения (не репозитория): `DEPLOY_SSH_KEY` (приватная часть ключа), `DEPLOY_HOST` (адрес сервера), `DEPLOY_KNOWN_HOSTS` (строка ключа хоста):

```bash
ssh -q <пользователь>@<адрес_сервера> cat /etc/ssh/ssh_host_ed25519_key.pub | awk -v h=<адрес_сервера> '{print h, $1, $2}' > /tmp/known_hosts_line
ssh-keygen -lf /tmp/known_hosts_line          # отпечаток должен совпасть с записью в ~/.ssh/known_hosts
```

Подключение проверяется ручным запуском workflow `Proxy` с ветки `main` (задача `deploy-check` выводит образ и состояние). Порядок выкладки, коды результата и откат — [docs/OPERATIONS.md](docs/OPERATIONS.md), разделы 5–7.

## Смена секретов

Секрет меняется только в `vault.yml` своего окружения с последующим запуском плейбука.

- **Токен Telegram-бота:** `ansible-vault edit …/vault.yml` → запуск плейбука. Файл токена обновится, Alertmanager перезапустится сам.
- **Пароль Grafana:** `ansible-vault edit …/vault.yml` → запуск плейбука (контейнер Grafana должен быть запущен). Пароль задаётся из файла только при первом запуске Grafana; при последующих изменениях роль сама меняет его командой `grafana cli admin reset-admin-password`.

Остальные процедуры (секреты прокси, потеря пароля vault, работы с Docker на рабочем сервере) — в [docs/OPERATIONS.md](docs/OPERATIONS.md).

## Безопасность

Кратко; подробно — в [docs/SECURITY.md](docs/SECURITY.md).

- Секреты хранятся только в зашифрованных `vault.yml`, у тестовой среды и рабочего сервера разные пароли; задачи, которые с ними работают, помечены `no_log`, поэтому секреты не попадают в вывод, в том числе при `--diff`.
- Вход на сервер — только по ключу и не под root; ufw запрещает входящие по умолчанию.
- Токен Telegram и пароль Grafana передаются в контейнеры **файлами** (права `0400`, владелец — пользователь процесса в контейнере: 65534 у Alertmanager, 472 у Grafana), а не переменными окружения: их нет в `docker inspect` и в окружении процессов.
- Прокси работает в контейнере от пользователя 1000, а не от root.
- Интерфейсы мониторинга (Prometheus, Alertmanager, Grafana) опубликованы только на `127.0.0.1` сервера; доступ — через SSH-туннель. Метрики прокси Prometheus получает внутри сети Docker.
- Порты, которые публикует Docker, **обходят ufw**: доступ к ним ограничивается только адресом публикации (`-p <адрес>:<порт_хоста>:<порт_контейнера>`). Порт без адреса открыт всем, независимо от правил файрвола.
- Пользователь сервера не входит в группу `docker`: доступ к Docker равен доступу root.
- Деплой из CI идёт под отдельным пользователем: ключ с принудительной командой, белый список команд (`status`, `deploy <digest>`), sudo на один скрипт; секреты деплоя — в окружении GitHub `production` (только `main`); CI не знает секретов приложения.
- Резервные копии шифруются restic на сервере до отправки в S3; пароль репозитория и ключи S3 — в vault окружения (на сервере `0400`/`0600`, root).
- Логи запусков (`*.log`) не хранятся в git.

## Известные ограничения

- Роль `security` только **добавляет** правила ufw: порт, удалённый из списка, остаётся открытым до ручного удаления правила.
- Роли только **добавляют** файлы: правило оповещений или панель, удалённые из списка, остаются на сервере до ручного удаления.
- Размер swap применяется только при создании файла; изменение `common_swap_size_mb` для существующего файла не действует.
- Версии Docker закреплены и удерживаются от обновления (`apt-mark hold`). Обновление — осознанное: поднять версии в `roles/docker/defaults/main.yml` и разрешить изменение удержанных пакетов в задаче установки.
- Изменение `daemon.json` перезапускает демон Docker; при включённом `live-restore` контейнеры продолжают работать. На сервере, где `live-restore` ещё не включён, его можно включить без остановки контейнеров (reload) — см. [docs/OPERATIONS.md](docs/OPERATIONS.md), раздел 4.
- Развёртывание зависит от внешних источников (зеркала Ubuntu, репозиторий Docker, Docker Hub). При их недоступности сетевые задачи повторяются несколько раз, затем запуск останавливается.
- Версию прокси Ansible не меняет: `release.env` он создаёт только на сервере, где его нет — из последней резервной копии (версия на момент копии, ≤ суток), а без копии — из начальной версии в инвентаре.
- Резервная копия — раз в сутки (потеря данных при аварии — до суток); копия домой — вручную раз в месяц. История Prometheus копируется на ходу, последние часы могут быть неполными.
- Восстановление при **новом** адресе сервера требует нового сертификата прокси и выпуска приложения — копии от этого не спасают; порядок — [docs/OPERATIONS.md](docs/OPERATIONS.md), раздел 9.
- Compose-файл прокси доставляет Ansible, а образ — деплой: изменение, которое требует и того и другого (новая переменная), выпускается в два шага (сначала compose и `.env`, затем код).
- При недоступном реестре новую версию выкатить нельзя (сборки на сервере нет); аварийный порядок — [docs/OPERATIONS.md](docs/OPERATIONS.md), раздел 7.
- Пароль Grafana, изменённый через интерфейс, плейбук не обнаружит (файл пароля не меняется). Источник истины — vault.
- Если Alertmanager не может достучаться до Telegram, в его лог попадает адрес запроса **вместе с токеном бота** (токен — часть адреса Bot API). Лог доступен root на сервере.
- Порт метрик прокси `9101` публикуется на адрес `METRICS_BIND` для мониторинга `hobot-metrics` на рабочем сервере. Будет убран, когда рабочий сервер перейдёт на стек мониторинга этого репозитория.
- Без подключённого репозитория хаба часть панелей «ИИ и хаб» пустая, а правило `MaxapocHubSnapshotMissing` срабатывает.
