# OceanBase CE: оживление одного выжившего сервера после потери кворума

Проверено: CE 4.4.2.1, two-zone, второй узел безвозвратно потерян. Выживший: 192.168.55.205.

## 1. Получить ob_admin (в системе его нет — только в utils-RPM)

Версия RPM должна **точно совпадать** с установленным oceanbase-ce (версия и release-суффикс):

```bash
# на машине с интернетом (release-суффикс подогнать под свой observer):
curl -LO https://mirrors.aliyun.com/oceanbase/community/stable/el/8/x86_64/oceanbase-ce-utils-4.4.2.1-101000022026050611.el8.x86_64.rpm

# распаковать без установки:
rpm2cpio oceanbase-ce-utils-4.4.2.1-*.rpm | cpio -idmv ./usr/bin/ob_admin

# доставить на выживший узел:
scp ./usr/bin/ob_admin admin@192.168.55.205:/home/admin/
ssh admin@192.168.55.205 'chmod +x /home/admin/ob_admin'
```

## 2. Узнать список LS, которые надо оживлять

На выжившем узле observer должен работать. Подключиться напрямую (не через прокси), таймаут короткий:

```bash
obclient -h127.0.0.1 -P2881 -uroot@sys -p'<пароль>' -Doceanbase \
  --init-command="SET SESSION ob_query_timeout=5000000" -e \
"SELECT tenant_id, ls_id, role, paxos_replica_num FROM __all_virtual_log_stat WHERE svr_ip='192.168.55.205';"
```

Пример вывода (все FOLLOWER = кворума нет, оживляем все):

```
+-----------+-------+----------+-------------------+
| tenant_id | ls_id | role     | paxos_replica_num |
|         1 |     1 | FOLLOWER |                 2 |
|      1001 |     1 | FOLLOWER |                 2 |
|      1002 |     1 | FOLLOWER |                 2 |
|      1002 |  1001 | FOLLOWER |                 2 |
|      1003 |     1 | FOLLOWER |                 2 |
|      1004 |     1 | FOLLOWER |                 2 |
|      1004 |  1001 | FOLLOWER |                 2 |
+-----------+-------+----------+-------------------+
```

## 3. Force single replica — оживление

На выжившем узле, **loopback обязателен** (127.0.0.1 обходит ussl-аутентификацию):

```bash
cd /home/admin
export OB_ADMIN_LOG_DIR=~/.ob_admin_log

# СНАЧАЛА sys (1:1) — оживляет словарь и rootservice:
./ob_admin -h127.0.0.1 -p2882 force_set_ls_as_single_replica 1:1

# проверить, что sys ожил, ПЕРЕД остальными:
obclient -h127.0.0.1 -P2881 -uroot@sys -p'<пароль>' -e \
"SELECT tenant_id, ls_id, role, paxos_replica_num FROM oceanbase.__all_virtual_log_stat WHERE tenant_id=1;"
# ждём: LEADER, paxos_replica_num=1

# затем цикл по всем остальным LS из шага 2 (формат tenant_id:ls_id):
for ls in 1001:1 1002:1 1002:1001 1003:1 1004:1 1004:1001; do
  ./ob_admin -h127.0.0.1 -p2882 force_set_ls_as_single_replica $ls
done
```

Вывод ob_admin шумный, в конце `Session has timed out` — **это не ошибка**, команда доходит.
Критерий успеха только по факту:

```bash
obclient -h127.0.0.1 -P2881 -uroot@sys -p'<пароль>' -e \
"SELECT tenant_id, ls_id, role, paxos_replica_num FROM oceanbase.__all_virtual_log_stat;
SELECT COUNT(*) FROM oceanbase.DBA_OB_SERVERS;"
```

Все LS: `LEADER`, `paxos_replica_num=1`. `DBA_OB_SERVERS` отвечает = словарь жив, сервер работает в одиночку. Тенанты доступны на чтение и запись.

Прочие force-команды ob_admin в 4.4.2 CE (на всякий случай):

```
force_set_all_as_single_replica          # все LS узла одной командой, без аргументов
force_set_locality  <tenant_id> <locality>
force_set_server_list <replica_num> <ip:port> ...
```

## 4. Риски

- Транзакции, закоммиченные мёртвым лидером и не долетевшие до выжившего — **теряются**.
- Мёртвый узел включать «как есть» НЕЛЬЗЯ (старые member_list/эпоха). Только зачистка + rebuild:

```bash
# на возвращаемом узле (observer остановлен):
rm -rf /data/log1/clog/* /data/log1/slog/* /data/1/sstable/*
# только СОДЕРЖИМОЕ; каталоги, симлинки, wallet, etc/, bin/ не трогать
systemctl start observer
```

Дальше DRM сам восстановит реплики по locality (контроль: `DBA_OB_LS_REPLICA_TASKS`,
`GV$OB_LOG_STAT` до paxos_replica_num=2 и in_sync=YES у всех LS; sys восстанавливается последним).
