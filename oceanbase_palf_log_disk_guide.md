# OceanBase PALF Log Disk — Диагностика, Мониторинг и Управление

## Архитектура (что важно понимать)

PALF (Paxos-based Append-only Log File) — движок redo-логов в OceanBase.  
Он работает принципиально иначе чем в MSSQL/PostgreSQL:

| MSSQL | OceanBase PALF |
|-------|----------------|
| FULL recovery → лог не чистится без backup | ARCHIVELOG → лог не чистится без архивации |
| `DBCC SHRINKFILE` | нет прямого аналога |
| Log backup освобождает место | `MINOR FREEZE` продвигает checkpoint |
| VLF (Virtual Log Files) | clog-файлы по 64MB в `log_pool` |

**Ключевые принципы:**
- OceanBase создаёт **log_pool** — преаллоцированный пул clog-файлов (по 64MB каждый)
- `log_disk_in_use` — реальный объём файлов выделенных тенанту из pool'а
- PALF **не отдаёт** файлы обратно в pool пока `log_disk_in_use` не достигнет `log_disk_utilization_threshold` (по умолчанию 80%)
- **80% — это нормально**, не "почти полно" — это штатное рабочее состояние
- Опасный порог — `log_disk_utilization_limit_threshold` (по умолчанию 95%) — при нём транзакции убиваются (error 6002)

---

## Что держит логи (аналогия с MSSQL)

| Причина в MSSQL | Причина в OceanBase |
|-----------------|---------------------|
| Active transaction | Открытая долгая транзакция |
| Log backup not taken | Checkpoint SCN не продвинулся (memtable не сброшен) |
| Replication / CDC | Physical standby tenant |
| Database mirroring | Paxos follower отстал (IN_SYNC=NO) |
| — | ARCHIVELOG включён, но архивация не настроена/прервана |

---

## Мониторинг

### 1. Обзор по серверам (верхний уровень)

```sql
SELECT zone,
       CONCAT(SVR_IP, ':', SVR_PORT) AS observer,
       ROUND(log_disk_capacity/1024/1024/1024, 2)              AS log_total_GB,
       ROUND(log_disk_assigned/1024/1024/1024, 2)              AS log_assigned_GB,
       ROUND((log_disk_capacity - log_disk_assigned)/1024/1024/1024, 2) AS log_unassigned_GB,
       ROUND(data_disk_capacity/1024/1024/1024, 2)             AS data_total_GB,
       ROUND(data_disk_in_use/1024/1024/1024, 2)               AS data_used_GB,
       ROUND((data_disk_capacity - data_disk_in_use)/1024/1024/1024, 2) AS data_free_GB
FROM gv$ob_servers;
```

> **Важно:** `log_disk_capacity - log_disk_assigned = 0` — это нормально если все GB уже распределены по тенантам. Это не значит что диск полон.

---

### 2. Использование по тенантам (основной запрос мониторинга)

```sql
SELECT a.tenant_name,
       u.svr_ip,
       CAST(u.log_disk_size/1024/1024/1024    AS DECIMAL(10,2)) AS allocated_GB,
       CAST(u.log_disk_in_use/1024/1024/1024  AS DECIMAL(10,2)) AS used_GB,
       ROUND(u.log_disk_in_use/u.log_disk_size * 100, 1)        AS used_pct
FROM __all_virtual_unit u
JOIN dba_ob_tenants a ON u.tenant_id = a.tenant_id
ORDER BY used_pct DESC;
```

**Пороги:**
- `used_pct < 80%` — норма
- `used_pct ≈ 80%` — штатное рабочее состояние PALF (ожидаемо)
- `used_pct > 80%` — активная нагрузка выше обычной, наблюдать
- `used_pct > 95%` — **КРИТИЧНО**: транзакции будут убиваться (error 6002 "Transaction is killed")

---

### 3. Детализация по log streams (активное vs recyclable)

```sql
SELECT TENANT_ID, LS_ID, SVR_IP,
       ROUND((BASE_LSN - BEGIN_LSN)/1024/1024/1024, 2) AS recyclable_GB,
       ROUND((END_LSN  - BASE_LSN)/1024/1024/1024, 2)  AS active_GB
FROM oceanbase.GV$OB_LOG_STAT
WHERE TENANT_ID = 1002         -- поменяй на нужный тенант
  AND SVR_IP = '10.21.26.202'; -- один сервер, иначе суммируются все зоны
```

- `recyclable_GB` — уже за checkpoint, PALF может перезаписать
- `active_GB` — реально "живые" логи которые нельзя удалить

> Если `active_GB` мал (< 1GB) а `used_pct` высокий — это не проблема, просто PALF держит преаллоцированные файлы.

---

### 4. Проверка репликации (не держит ли follower логи)

```sql
SELECT TENANT_ID, LS_ID, SVR_IP, ROLE, IN_SYNC
FROM oceanbase.GV$OB_LOG_STAT
WHERE TENANT_ID = 1002;
```

Если `IN_SYNC = NO` у фолловера — он отстал и GC не может освободить логи.

---

### 5. Проверка состояния архивации

```sql
SELECT TENANT_ID, STATUS, CHECKPOINT_SCN_DISPLAY, COMMENT
FROM oceanbase.CDB_OB_ARCHIVELOG;
```

Если `STATUS = INTERRUPTED` — архивация сломана и может держать логи.  
Если архивация не нужна:

```sql
ALTER SYSTEM NOARCHIVELOG;  -- выполнять внутри нужного тенанта
```

---

## Диагностика при проблеме (error 6002 или connection drop)

### Шаг 1 — Смотрим alert.log на сервере

```bash
grep "PALF\|log disk\|OB_LOG_DISK" /home/admin/oceanbase/log/alert.log | tail -50
```

Признаки проблемы:
```
OB_LOG_DISK_SPACE_ALMOST_FULL  used_percent=80%   ← предупреждение
OB_FAILURE_LOG_DISK_HUNG                          ← диск завис
```

### Шаг 2 — Определяем кто занимает место

```sql
-- Быстрая сводка
SELECT a.tenant_name, u.svr_ip,
       CAST(u.log_disk_in_use/1024/1024/1024  AS DECIMAL(10,2)) AS used_GB,
       CAST(u.log_disk_size/1024/1024/1024    AS DECIMAL(10,2)) AS allocated_GB,
       ROUND(u.log_disk_in_use/u.log_disk_size * 100, 1)        AS used_pct
FROM __all_virtual_unit u
JOIN dba_ob_tenants a ON u.tenant_id = a.tenant_id
ORDER BY used_pct DESC;
```

### Шаг 3 — Смотрим recyclable vs active

```sql
SELECT TENANT_ID, LS_ID, SVR_IP,
       ROUND((BASE_LSN - BEGIN_LSN)/1024/1024/1024, 2) AS recyclable_GB,
       ROUND((END_LSN  - BASE_LSN)/1024/1024/1024, 2)  AS active_GB
FROM oceanbase.GV$OB_LOG_STAT
WHERE SVR_IP = '10.21.26.202'  -- один сервер
ORDER BY TENANT_ID, LS_ID;
```

**Интерпретация:**
- `recyclable_GB` большой, `active_GB` маленький → PALF просто держит файлы, GC не сработал → снизить threshold
- `active_GB` большой → реальная нагрузка, checkpoint не продвигается → minor freeze + расширить диск

---

## Как чистить

### Метод 1 — Принудительный GC через снижение threshold (самый эффективный)

Работает когда `recyclable_GB` большой (checkpoint уже прошёл, файлы просто не возвращены в pool).

```sql
-- Применяется к тенанту в контексте которого выполняется
-- Или от sys с указанием тенанта:
ALTER SYSTEM SET log_disk_utilization_threshold = 10 TENANT = 'bench';

-- Подождать 30-60 секунд, проверить used_pct
-- Вернуть обратно:
ALTER SYSTEM SET log_disk_utilization_threshold = 80 TENANT = 'bench';
```

> Минимальное значение: **10** (значения < 10 не принимаются).

### Метод 2 — Minor Freeze (продвигает checkpoint)

Работает когда `active_GB` большой — сбрасывает memtable на диск, продвигает checkpoint SCN, после чего PALF может освободить старые сегменты.

```sql
-- Для конкретного тенанта
ALTER SYSTEM MINOR FREEZE TENANT = 'bench';

-- Для всех тенантов
ALTER SYSTEM MINOR FREEZE;
```

> **Важно:** minor freeze не освобождает место мгновенно. Он продвигает checkpoint → GC срабатывает → файлы возвращаются в pool. Видимый эффект через несколько минут.

### Метод 3 — Расширить лимит тенанта (экстренная мера)

Когда нет времени ждать GC, а тест/нагрузка продолжается:

```sql
-- Найти unit config тенанта
SELECT p.NAME AS pool_name, u.NAME AS unit_name,
       u.LOG_DISK_SIZE/1024/1024/1024 AS current_log_GB
FROM oceanbase.DBA_OB_RESOURCE_POOLS p
JOIN oceanbase.DBA_OB_UNIT_CONFIGS u ON p.UNIT_CONFIG_ID = u.UNIT_CONFIG_ID
WHERE p.TENANT_ID = 1002;

-- Увеличить
ALTER RESOURCE UNIT <unit_name> LOG_DISK_SIZE = '120G';
```

---

## Параметры управления

```sql
-- Посмотреть все log-параметры
SHOW PARAMETERS LIKE '%log_disk%';
```

| Параметр | Дефолт | Описание |
|----------|--------|----------|
| `log_disk_utilization_threshold` | 80 | При каком % PALF начинает GC и возвращает файлы в pool. Диапазон: [10, 95) |
| `log_disk_utilization_limit_threshold` | 95 | При каком % транзакции начинают убиваться. Диапазон: [80, 100] |
| `freeze_trigger_percentage` | 20 | При каком % заполнения memtable триггерить minor freeze |
| `log_disk_size` | 0 (авто) | Общий размер лог-диска на сервере |

**Рекомендация по размеру:** `log_disk_size на сервере = memory_limit × 3~4`

---

## Быстрый чеклист при `Transaction is killed` (error 6002)

```
1. grep alert.log → ищем OB_LOG_DISK_SPACE_ALMOST_FULL
2. SELECT used_pct FROM __all_virtual_unit → кто > 80%
3. SELECT recyclable_GB, active_GB FROM GV$OB_LOG_STAT → что держит
4. Если recyclable большой → ALTER SYSTEM SET log_disk_utilization_threshold = 10 TENANT = '...'
5. Если active большой → ALTER SYSTEM MINOR FREEZE TENANT = '...'
6. Если не помогает → ALTER RESOURCE UNIT ... LOG_DISK_SIZE = '...'
7. Вернуть threshold на 80
```
## мониторинг 

```sql
SELECT
   a.tenant_name,
   u.svr_ip,
   ROUND(u.log_disk_size/1024/1024/1024, 1)                         AS allocated_GB,
   ROUND(SUM(ls.END_LSN - ls.BASE_LSN)/1024/1024/1024, 1)          AS active_GB,
   ROUND(SUM(ls.BASE_LSN - ls.BEGIN_LSN)/1024/1024/1024, 1)        AS recyclable_GB,
   ROUND((u.log_disk_size
          - SUM(ls.END_LSN - ls.BASE_LSN))/1024/1024/1024, 1)      AS effective_free_GB,
   ROUND(u.log_disk_size * 0.95/1024/1024/1024, 1)                 AS kill_limit_GB,
   ROUND((u.log_disk_size * 0.95
          - SUM(ls.END_LSN - ls.BASE_LSN))/1024/1024/1024, 1)      AS headroom_GB,
   ROUND(SUM(ls.END_LSN - ls.BASE_LSN)
         / (u.log_disk_size * 0.95) * 100, 1)                      AS pct_of_limit,
   NOW()                                                             AS checked_at
    FROM __all_virtual_unit u
    JOIN dba_ob_tenants a ON u.tenant_id = a.tenant_id
    JOIN oceanbase.GV$OB_LOG_STAT ls
    ON u.tenant_id = ls.TENANT_ID
   AND u.svr_ip    = ls.SVR_IP
    WHERE u.svr_ip = (SELECT SVR_IP FROM oceanbase.GV$OB_LOG_STAT LIMIT 1)
    GROUP BY a.tenant_name, u.svr_ip, u.log_disk_size
    ORDER BY pct_of_limit DESC;
+-------------+----------------+--------------+-----------+---------------+-------------------+---------------+-------------+--------------+---------------------+
| tenant_name | svr_ip         | allocated_GB | active_GB | recyclable_GB | effective_free_GB | kill_limit_GB | headroom_GB | pct_of_limit | checked_at          |
+-------------+----------------+--------------+-----------+---------------+-------------------+---------------+-------------+--------------+---------------------+
| sys         | 192.168.5.205 |          2.0 |       0.3 |           1.2 |               1.7 |           1.9 |         1.6 |         16.1 | 2026-05-06 18:37:25 |
| META$1002   | 192.168.5.205 |          2.0 |       0.1 |           1.4 |               1.9 |           1.9 |         1.8 |          5.9 | 2026-05-06 18:37:25 |
| app_tenant  | 192.168.5.205 |         18.0 |       0.2 |          14.0 |              17.8 |          17.1 |        16.9 |          1.1 | 2026-05-06 18:37:25 |
+-------------+----------------+--------------+-----------+---------------+-------------------+---------------+-------------+--------------+---------------------+
3 rows in set (0.02 sec)

[rlm_sys|oceanbase] 15:37:25>
```
active_GBаналог "Log Used" в MSSQL — реально нельзя удалить

recyclable_GBбудет перезаписан, считай свободным

effective_free_GB сколько реально осталось для роста

headroom_GB сколько осталось до Transaction is killed

pct_of_limitглавная метрика — при 90%+ начинай действовать

