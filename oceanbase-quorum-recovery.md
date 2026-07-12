# OceanBase CE: оживление выжившего сервера после потери кворума и возврат узлов

Проверено: CE 4.4.2.1, потеря узла(ов) с кворумом. Выживший в примерах: 192.168.55.205.

## 1. Получить ob_admin (в системе его нет — только в utils-RPM)

Версия RPM должна **точно совпадать** с установленным oceanbase-ce (версия + release-суффикс):

```bash
# на машине с интернетом (суффикс подогнать под свой observer):
curl -LO https://mirrors.aliyun.com/oceanbase/community/stable/el/8/x86_64/oceanbase-ce-utils-4.4.2.1-101000022026050611.el8.x86_64.rpm

# распаковать без установки:
rpm2cpio oceanbase-ce-utils-4.4.2.1-*.rpm | cpio -idmv ./usr/bin/ob_admin

# доставить на выживший узел:
scp ./usr/bin/ob_admin admin@192.168.55.205:/home/admin/
ssh admin@192.168.55.205 'chmod +x /home/admin/ob_admin'
```

## 2. Оживить sys — ВСЛЕПУЮ, первым, до любых запросов

Без кворума sys подключение может не работать вовсе или все словарные запросы висят —
поэтому список LS заранее не получить. Sys LS всегда `1:1`, его форсим первым.

На выжившем узле (observer должен работать), **loopback обязателен** (127.0.0.1 обходит ussl):

```bash
cd /home/admin
export OB_ADMIN_LOG_DIR=~/.ob_admin_log

./ob_admin -h127.0.0.1 -p2882 force_set_ls_as_single_replica 1:1
```

Вывод ob_admin шумный, в конце `Session has timed out` — **не ошибка**, команда доходит.
Проверка (подключение напрямую к узлу, не через прокси):

```bash
obclient -h127.0.0.1 -P2881 -uroot@sys -p'<пароль>' -Doceanbase \
  --init-command="SET SESSION ob_query_timeout=5000000" -e \
"SELECT tenant_id, ls_id, role, paxos_replica_num FROM __all_virtual_log_stat WHERE tenant_id=1;
SELECT COUNT(*) FROM DBA_OB_SERVERS;"
```

Ждём: sys LS = `LEADER`, `paxos_replica_num=1`; `DBA_OB_SERVERS` отвечает = словарь ожил.

## 3. Теперь получить список остальных LS и оживить их

```bash
obclient -h127.0.0.1 -P2881 -uroot@sys -p'<пароль>' -e \
"SELECT tenant_id, ls_id, role FROM oceanbase.__all_virtual_log_stat
 WHERE svr_ip='192.168.55.205' AND NOT (tenant_id=1 AND ls_id=1) ORDER BY 1,2;"
```

Все LS без LEADER — в цикл (формат `tenant_id:ls_id`, включая META-тенанты):

```bash
for ls in 1001:1 1002:1 1002:1001 1003:1 1004:1 1004:1001; do
  ./ob_admin -h127.0.0.1 -p2882 force_set_ls_as_single_replica $ls
done
```

Критерий успеха — только фактическое состояние:

```bash
obclient -h127.0.0.1 -P2881 -uroot@sys -p'<пароль>' -e \
"SELECT tenant_id, ls_id, role, paxos_replica_num FROM oceanbase.__all_virtual_log_stat;"
# все LS: LEADER, paxos_replica_num=1 → кластер работает в одиночку, тенанты доступны
```

Прочие force-команды ob_admin 4.4.2 CE:

```
force_set_all_as_single_replica          # все LS узла одной командой, без аргументов
force_set_locality  <tenant_id> <locality>
force_set_server_list <replica_num> <ip:port> ...
```

**Риск force:** транзакции, закоммиченные мёртвым лидером и не долетевшие до выжившего, теряются.

## 4. Вернуть потерянный сервер в кластер

Locality и пулы НЕ трогать — схема по-прежнему требует реплики во всех зонах,
и DRM восстановит их автоматически, как только появится чистый узел.

Мёртвый узел включать «как есть» НЕЛЬЗЯ (старые member_list/эпоха Paxos). Порядок:

```bash
# на возвращаемом узле — observer НЕ должен стартовать со старыми данными
# (если стоит systemd-автостарт — остановить сразу после включения VM):
sudo systemctl stop observer

# зачистить ТОЛЬКО СОДЕРЖИМОЕ данных (пути = data_dir/redo_dir деплоя):
rm -rf /data/log1/clog/* /data/log1/slog/* /data/1/sstable/*
# каталоги, симлинки (clog/slog в /data/1, store), wallet/, etc/, bin/ — НЕ трогать!
# если снёс каталоги — восстановить структуру по образцу живого узла,
# иначе observer падает: -9100 OB_NO_SUCH_FILE_OR_DIRECTORY ... log_pool

sudo systemctl start observer
```

Контроль авто-восстановления (с любого клиента):

```sql
SELECT SVR_IP, STATUS FROM DBA_OB_SERVERS;              -- узел ACTIVE (~15 сек)
SELECT * FROM DBA_OB_LS_REPLICA_TASKS;                  -- ADD REPLICA задачи DRM
SELECT tenant_id, ls_id, svr_ip, role, paxos_replica_num, in_sync
  FROM GV$OB_LOG_STAT ORDER BY 1,2,3;
```

Фазы на каждый LS: реплика-learner (replica_num пока 1, узел виден в `learner_list`)
→ catch-up → membership change → `paxos_replica_num=2`, `in_sync=YES`.
User-тенанты восстанавливаются первыми, sys — последним. 
Финал: каждый LS в двух экземплярах на обоих узлах, все in_sync — кластер вернулся
в исходную конфигурацию без пересоздания.

Хроника решений DRM для разбора: 

```sql
SELECT TIMESTAMP, EVENT, NAME1, VALUE1, NAME2, VALUE2 FROM DBA_OB_ROOTSERVICE_EVENT_HISTORY
WHERE MODULE='disaster_recovery' ORDER BY TIMESTAMP DESC LIMIT 20;
```

## Тайминги (лаба, пустые тенанты)

| Этап | Время |
|---|---|
| force sys → живой словарь | секунды |
| force остальных 6 LS | ~1 мин |
| старт зачищенного узла → ACTIVE | ~15 сек |
| DRM: создание всех реплик | ~20 сек |
| learner → полный member 2/2 | ~3–5 мин |
