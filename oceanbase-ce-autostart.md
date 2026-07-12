# OceanBase CE: автоподъём кластера после перезагрузки серверов

Проверено на: OceanBase CE 4.4.2.1, OBD 4.3.0, AlmaLinux 9 (SELinux enforcing), кластер задеплоен через obd.

## Проблема

OBD не устанавливает systemd-юниты. После ребута хоста observer, obshell, obproxy и ob-configserver никто не поднимает. Официальные варианты:

| Путь | Что даёт | Когда использовать |
|---|---|---|
| OCP + ocp-agent | Агент как системный сервис супервизирует observer | Продакшен, «блессед» решение |
| all-in-one `oceanbase.service` | Юнит-обёртка вокруг `obd cluster start` | Демо на одном хосте |
| Свои systemd-юниты | Полный контроль | CE-кластер без OCP (наш случай) |

Итоговая схема — двухуровневая:

1. **Per-node юнит `observer.service`** на каждом OB-узле — узел сам возвращается в кластер после своего ребута, независимо от управляющего сервера.
2. **Юнит-обёртка `ob-lab.service`** на управляющем хосте (где obd) — после его ребута поднимает configserver, добирает obshell/obproxy и подстраховывает observer'ы.

Оба уровня идемпотентны: повторный запуск поверх живого процесса безопасен (observer проверяет pid-файл и выходит).

## Шаг 0. Предпосылки и грабли

### ob-configserver — отдельным деплоем

Если configserver задеплоен как компонент кластера, `obd cluster destroy` этого кластера снесёт его вместе с регистрациями всех остальных кластеров. Выносим в отдельный деплой:

```yaml
# ~/obconfigserver.yaml
user:
  username: admin
  port: 22
ob-configserver:
  servers:
    - 192.168.55.200
  global:
    listen_port: 8080
    server_ip: 0.0.0.0
    home_path: /home/admin/ob-configserver
    vip_address: 192.168.55.200
    vip_port: 8080
    storage:
      database_type: sqlite3
```

```bash
obd cluster deploy obcfg -c ~/obconfigserver.yaml
obd cluster start obcfg
```

### Симлинки OBD → копии

OBD по умолчанию (`install mode: ln`) кладёт в `home_path/bin` и `home_path/lib` **симлинки** на `/home/admin/.obd/repository/...`. Для systemd это фатально: SELinux проверяет контекст цели симлинка, а домашние каталоги (`user_home_t`) из init-домена не исполняются → `status=203/EXEC Permission denied`.

Заменяем симлинки копиями (работающий процесс не пострадает — он держит inode открытого файла):

```bash
cd /data/obc442/oceanbase/bin
for f in *; do [ -L "$f" ] && cp --remove-destination "$(readlink -f "$f")" "$f"; done
cd /data/obc442/oceanbase/lib
for f in *; do [ -L "$f" ] && cp --remove-destination "$(readlink -f "$f")" "$f"; done
```

Для новых установок сразу использовать `export OBD_REPO_INSTALL_MODE=cp` перед deploy — тогда этот шаг не нужен.

**Внимание:** `obd cluster upgrade` пересоздаст симлинки. После апгрейда повторить cp + restorecon.

## Шаг 1. SELinux-контексты

Каталоги на data-диске обычно `unlabeled_t` или `default_t` — systemd не сможет ни исполнить бинарник, ни пройти по пути. Нужен пакет с semanage:

```bash
sudo dnf install -y policycoreutils-python-utils
```

Правила (порядок критичен — см. ниже):

```bash
sudo semanage fcontext -a -t var_lib_t "/data(/.*)?"
sudo semanage fcontext -a -t bin_t     "/data/obc442/oceanbase/bin(/.*)?"
sudo semanage fcontext -a -t lib_t     "/data/obc442/oceanbase/lib(/.*)?"
sudo restorecon -R /data

# проверка — оба должны отдать bin_t:
matchpathcon /data/obc442/oceanbase/bin/observer
ls -lZ /data/obc442/oceanbase/bin/observer
```

**Грабля с порядком правил:** в `file_contexts.local` при пересечении шаблонов выигрывает **последнее добавленное** правило, а не более специфичное. Если общее правило `/data(/.*)?` добавлено ПОСЛЕ правил для bin/lib — оно их перекроет, observer получит `var_lib_t` и systemd снова упадёт с 203. Лечится передобавлением:

```bash
sudo semanage fcontext -d "/data/obc442/oceanbase/bin(/.*)?"
sudo semanage fcontext -d "/data/obc442/oceanbase/lib(/.*)?"
sudo semanage fcontext -a -t bin_t "/data/obc442/oceanbase/bin(/.*)?"
sudo semanage fcontext -a -t lib_t "/data/obc442/oceanbase/lib(/.*)?"
sudo restorecon -Rv /data/obc442/oceanbase/bin /data/obc442/oceanbase/lib
```

Альтернативы semanage: `chcon` (есть в coreutils, но метки не переживут полный relabel ФС) или permissive-режим SELinux (для изолированной лабы допустимо, для гайда/прода — нет).

## Шаг 2. Per-node юнит observer.service

На каждом OB-узле:

```ini
# /etc/systemd/system/observer.service
[Unit]
Description=OceanBase observer
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
User=admin
Environment=LD_LIBRARY_PATH=/data/obc442/oceanbase/lib
WorkingDirectory=/data/obc442/oceanbase
ExecStart=/data/obc442/oceanbase/bin/observer -p 2881
PIDFile=/data/obc442/oceanbase/run/observer.pid
Restart=on-failure
RestartSec=10
LimitNOFILE=655350
LimitSTACK=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable observer.service
```

Ключевые моменты:

- **`Type=forking` + `PIDFile`** — observer демонизируется сам (родитель выходит с кодом 0), PIDFile обязателен, иначе systemd теряет main PID.
- **`Restart=on-failure`, не always** — иначе systemd будет воскрешать observer после штатного `obd cluster stop`. За автостарт при boot отвечает `enable`, а не `Restart`.
- **`LimitSTACK=infinity`** — заодно закрывает предупреждение OBD-1007 (stack size).
- Команда запуска идентична тому, что делает obd (проверено через `obd display-trace`): `cd home_path && LD_LIBRARY_PATH=lib ./bin/observer -p 2881`. Конфиг observer читает сам из `etc/observer.config.bin`.
- Юнит НЕ покрывает obshell — кластеру для работы он не нужен, его донастроит obd при следующем `cluster start`.

Если observer уже работает (запущен вручную/через obd), юнит просто enable без start — он подхватит управление после следующего ребута. Для немедленной передачи под systemd: `kill $(cat run/observer.pid)`, затем `systemctl start observer.service` (учитывать состояние кворума! см. Шаг 4).

## Шаг 3. Юнит-обёртка на управляющем хосте

```ini
# /etc/systemd/system/ob-lab.service
[Unit]
Description=OceanBase lab clusters autostart via OBD
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=costa
RemainAfterExit=yes
# ждём ssh-доступности OB-узлов (VM могут подниматься дольше)
ExecStartPre=/usr/bin/bash -c 'for i in $(seq 1 60); do ssh -o ConnectTimeout=3 -o BatchMode=yes admin@192.168.55.202 true && ssh -o ConnectTimeout=3 -o BatchMode=yes admin@192.168.55.205 true && exit 0; sleep 5; done; exit 1'
ExecStart=/usr/bin/bash -lc 'obd cluster start obcfg'
ExecStart=/usr/bin/bash -lc 'obd cluster start obc442'
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target
```

Требования: ssh-ключи пользователя obd к admin@узлам должны работать в batch-режиме (без passphrase или с агентом); `bash -lc` нужен для PATH до obd.

Конфликтов с per-node юнитами нет: если observer уже поднят своим юнитом, obd увидит живой pid и пропустит запуск.

## Шаг 4. Приёмочный тест

Полный ребут OB-узла (не kill процесса — именно ребут VM, он проверяет всю цепочку boot → SELinux → systemd → observer):

```bash
# на узле после ребута
systemctl status observer.service --no-pager | head -6   # active (running), Main PID найден
pgrep -a observer
```

```sql
-- с любого клиента
SELECT SVR_IP, STATUS, START_SERVICE_TIME FROM oceanbase.DBA_OB_SERVERS;
-- узел ACTIVE

SELECT TENANT_ID, LS_ID, SVR_IP, ROLE, IN_SYNC, PAXOS_REPLICA_NUM
FROM oceanbase.GV$OB_LOG_STAT ORDER BY 1,2,3;
-- все реплики IN_SYNC=YES, у каждого LS есть LEADER
```

**Про кворум при тестах:** в конфигурации из 2 зон (F@zone1, F@zone2) мажорити = 2 — ребут любого узла замораживает LS с member_list 2/2 до его возврата. Планировать тесты соответственно; полноценная отказоустойчивость начинается с 3 зон.

## Диагностика типовых отказов

| Симптом | Причина | Лечение |
|---|---|---|
| `status=203/EXEC Permission denied` | SELinux: бинарник не `bin_t` (симлинк в home, unlabeled, перекрытое правило) | Шаги 0–1; `matchpathcon` для проверки |
| Юнит failed, но процесс жив, затем умирает | systemd убивает cgroup при рестарт-цикле | Исправить причину failed; `systemctl reset-failed` |
| `New main PID does not exist` / timeout | Нет PIDFile при Type=forking | Добавить PIDFile |
| Observer стартует и сразу умирает, в логе `load_ssl_config fail` | Включён SSL, на узле нет wallet (типично после scale_out) | Скопировать/выпустить сертификаты в `home_path/wallet` |
| Observer воскресает после `obd cluster stop` | `Restart=always` | Использовать `Restart=on-failure` |
| Узел ACTIVE не становится, лог молчит | Процесс убит рестарт-циклом systemd раньше инициализации | `journalctl -u observer.service`, разорвать цикл stop+reset-failed |

Ключевые команды диагностики:

```bash
sudo journalctl -u observer.service -b --no-pager   # журнал юнита с текущего boot
sudo ausearch -m avc -ts recent                     # SELinux denials
matchpathcon <путь>                                 # какой контекст положен по политике
ls -lZ <путь>                                       # какой контекст фактически
obd display-trace <trace_id>                        # что именно делал obd
```
