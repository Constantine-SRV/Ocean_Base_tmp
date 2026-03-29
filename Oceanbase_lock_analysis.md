# OceanBase CE — Анализ блокировок (Lock Tree)

**Версия:** OceanBase CE 4.4.1  
**Тестовый прокси:** Oceanbase lock analysis:2883  
**Тенант:** app_tenant  

---

## 1. Создание демостенда

### 1.1 Подключение и создание БД, таблицы, пользователей

```bash
MYSQL_PS1="[root][\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 -uroot@app_tenant -p'qaz123' \
  -A --init-command="SET SESSION ob_query_timeout=10000000000; SET SESSION ob_trx_timeout=10000000000"
```

```sql
CREATE DATABASE IF NOT EXISTS locktest;
USE locktest;

CREATE TABLE accounts (
    id       INT PRIMARY KEY,
    owner    VARCHAR(50),
    balance  DECIMAL(10,2)
);

INSERT INTO accounts VALUES
    (1, 'alice',  1000.00),
    (2, 'bob',    2000.00),
    (3, 'carol',  3000.00),
    (4, 'dave',   4000.00);

CREATE USER 'usr1'@'%' IDENTIFIED BY 'qaz123';
CREATE USER 'usr2'@'%' IDENTIFIED BY 'qaz123';
CREATE USER 'usr3'@'%' IDENTIFIED BY 'qaz123';
CREATE USER 'usr4'@'%' IDENTIFIED BY 'qaz123';

GRANT ALL PRIVILEGES ON locktest.* TO 'usr1'@'%';
GRANT ALL PRIVILEGES ON locktest.* TO 'usr2'@'%';
GRANT ALL PRIVILEGES ON locktest.* TO 'usr3'@'%';
GRANT ALL PRIVILEGES ON locktest.* TO 'usr4'@'%';

FLUSH PRIVILEGES;
SELECT * FROM accounts;
```

---

### 1.2 Строки подключения для 4 терминалов

> **Важно:** Оба таймаута обязательны. `ob_query_timeout` — максимальное время одного запроса,
> `ob_trx_timeout` — время жизни транзакции. Оба в микросекундах: `10000000000` = ~2.7 часа.

**Терминал 1 — usr1**
```bash
MYSQL_PS1="[T1-usr1][\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 \
  -uusr1@app_tenant -p'qaz123' -A -D locktest \
  --init-command="SET SESSION ob_query_timeout=10000000000; SET SESSION ob_trx_timeout=10000000000"
```

**Терминал 2 — usr2**
```bash
MYSQL_PS1="[T2-usr2][\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 \
  -uusr2@app_tenant -p'qaz123' -A -D locktest \
  --init-command="SET SESSION ob_query_timeout=10000000000; SET SESSION ob_trx_timeout=10000000000"
```

**Терминал 3 — usr3**
```bash
MYSQL_PS1="[T3-usr3][\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 \
  -uusr3@app_tenant -p'qaz123' -A -D locktest \
  --init-command="SET SESSION ob_query_timeout=10000000000; SET SESSION ob_trx_timeout=10000000000"
```

**Терминал 4 — usr4**
```bash
MYSQL_PS1="[T4-usr4][\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 \
  -uusr4@app_tenant -p'qaz123' -A -D locktest \
  --init-command="SET SESSION ob_query_timeout=10000000000; SET SESSION ob_trx_timeout=10000000000"
```

**Терминал 5 — root@sys (наблюдатель)**
```bash
MYSQL_PS1="[T5-root][\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 \
  -uroot@sys -p'qaz123' -A -D oceanbase \
  --init-command="SET SESSION ob_query_timeout=10000000000"
```

---

### 1.3 Создание цепочки блокировок

Выполнять **строго по порядку**, каждый блок в своём терминале.
Не завершать предыдущие транзакции до окончания теста.

**Терминал 1** — usr1 захватывает строку id=1 и удерживает её:
```sql
SET autocommit = 0;
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SELECT SLEEP(3600);   -- держим блокировку
```

**Терминал 2** — usr2 захватывает id=2, затем зависает на id=1:
```sql
SET autocommit = 0;
BEGIN;
UPDATE accounts SET balance = balance + 50 WHERE id = 2;   -- OK
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- ЗАВИСАЕТ (ждёт usr1)
```

**Терминал 3** — usr3 захватывает id=3, затем зависает на id=2:
```sql
SET autocommit = 0;
BEGIN;
UPDATE accounts SET balance = balance * 1.1 WHERE id = 3;  -- OK
UPDATE accounts SET balance = balance * 1.1 WHERE id = 2;  -- ЗАВИСАЕТ (ждёт usr2)
```

**Терминал 4** — usr4 захватывает id=4, затем зависает на id=3:
```sql
SET autocommit = 0;
BEGIN;
UPDATE accounts SET balance = 9999 WHERE id = 4;           -- OK
UPDATE accounts SET balance = 9999 WHERE id = 3;           -- ЗАВИСАЕТ (ждёт usr3)
```

Итоговая цепочка блокировок:
```
usr1 (держит id=1) → usr2 (ждёт id=1, держит id=2) → usr3 (ждёт id=2, держит id=3) → usr4 (ждёт id=3)
```

---

## 2. Анализ блокировок

Все запросы выполняются из **Терминала 5** (root@sys, база oceanbase).

### 2.1 Структура GV$OB_LOCKS — ключевые поля

OceanBase представляет блокировки тремя типами строк:

| TYPE | Смысл           | ID1                        | ID2                        | CTIME        |
|------|-----------------|----------------------------|----------------------------|--------------|
| `TX` | Транзакционная (строчная) блокировка | TRANS_ID своей транзакции  | SESSION_ID своей сессии    | Время удержания (мкс) |
| `TM` | Table intent lock | TRANS_ID своей транзакции  | SESSION_ID своей сессии    | Время удержания (мкс) |
| `TR` | Ожидание чужой блокировки | TRANS_ID **блокирующего** | SESSION_ID блокирующего | Время ожидания (мкс) |

> **Важно:** `CTIME` — микросекунды, делить на 1 000 000 для перевода в секунды.
> `BLOCK=1` — транзакция ждёт; `BLOCK=0` + `LMODE>0` — транзакция держит блокировку.

---

### 2.2 Быстрый обзор — все текущие блокировки

```sql
SELECT
    TRANS_ID,
    SVR_IP,
    TYPE,
    LMODE,
    REQUEST,
    ROUND(CTIME / 1000000, 1)  AS sec,
    BLOCK
FROM GV$OB_LOCKS
ORDER BY BLOCK DESC, CTIME DESC;
```

---

### 2.3 Кто кого блокирует — плоский список пар

```sql
SELECT
    lh.TRANS_ID                             AS blocking_trans,
    lw.TRANS_ID                             AS waiting_trans,
    lw.TYPE                                 AS lock_type,
    lh.LMODE                                AS held_mode,
    lw.REQUEST                              AS requested_mode,
    ROUND(lw.CTIME / 1000000, 1)           AS wait_sec,
    lw.ID2                                  AS blocker_session_id,
    p_block.USER                            AS blocker_user,
    p_block.INFO                            AS blocker_sql,
    tx_wait.ID2                             AS waiter_session_id,
    p_wait.USER                             AS waiter_user,
    p_wait.INFO                             AS waiter_sql
FROM GV$OB_LOCKS lw
JOIN GV$OB_LOCKS lh
    ON  lh.TRANS_ID = lw.ID1
    AND lh.TYPE     = 'TX'
LEFT JOIN GV$OB_LOCKS tx_wait
    ON  tx_wait.TRANS_ID = lw.TRANS_ID
    AND tx_wait.TYPE     = 'TX'
LEFT JOIN information_schema.PROCESSLIST p_block
    ON  p_block.ID = lw.ID2
LEFT JOIN information_schema.PROCESSLIST p_wait
    ON  p_wait.ID  = tx_wait.ID2
WHERE lw.BLOCK = 1
  AND lw.TYPE  = 'TR'
ORDER BY lw.CTIME DESC;
```

---

### 2.4 Дерево блокировок с рекурсией — основной запрос

Показывает полную цепочку `A → B → C → D` с пользователями, IP и SQL.
Для idle-сессий (корень цепочки) берёт последний SQL из `GV$OB_SQL_AUDIT`.

```sql
WITH RECURSIVE lock_chain AS (
    -- Корень: транзакции которые блокируют, сами не ожидают
    SELECT
        lh.TRANS_ID                         AS trans_id,
        lh.TRANS_ID                         AS root_trans_id,
        CAST(lh.TRANS_ID AS CHAR(500))      AS chain,
        lh.ID2                              AS session_id,
        0                                   AS depth
    FROM GV$OB_LOCKS lh
    WHERE lh.TYPE  = 'TX'
      AND lh.BLOCK = 0
      AND lh.TRANS_ID NOT IN (
          SELECT TRANS_ID FROM GV$OB_LOCKS WHERE BLOCK = 1 AND TYPE = 'TR'
      )

    UNION ALL

    -- Рекурсия: добавляем ожидающих
    SELECT
        lw.TRANS_ID,
        lc.root_trans_id,
        CONCAT(lc.chain, ' -> ', lw.TRANS_ID),
        tx_w.ID2,
        lc.depth + 1
    FROM GV$OB_LOCKS lw
    JOIN lock_chain lc
        ON lc.trans_id = lw.ID1
    LEFT JOIN GV$OB_LOCKS tx_w
        ON  tx_w.TRANS_ID = lw.TRANS_ID
        AND tx_w.TYPE     = 'TX'
    WHERE lw.BLOCK = 1
      AND lw.TYPE  = 'TR'
      AND lc.depth < 10
)
SELECT
    lc.depth,
    lc.chain,
    lc.session_id,
    p.USER                                  AS db_user,
    p.HOST                                  AS client_ip,
    ROUND(locks.CTIME / 1000000, 1)         AS hold_sec,
    ROUND(tr.CTIME   / 1000000, 1)         AS wait_sec,
    COALESCE(p.INFO, ls.query_sql)          AS sql_text,
    CASE WHEN p.INFO IS NULL THEN 'last'
         ELSE 'current'
    END                                     AS sql_source
FROM lock_chain lc
LEFT JOIN information_schema.PROCESSLIST p
    ON  p.ID = lc.session_id
LEFT JOIN GV$OB_LOCKS locks
    ON  locks.TRANS_ID = lc.trans_id
    AND locks.TYPE     = 'TX'
LEFT JOIN GV$OB_LOCKS tr
    ON  tr.TRANS_ID = lc.trans_id
    AND tr.TYPE     = 'TR'
    AND tr.BLOCK    = 1
LEFT JOIN (
    SELECT SID, query_sql
    FROM (
        SELECT
            SID,
            query_sql,
            ROW_NUMBER() OVER (
                PARTITION BY SID
                ORDER BY request_time DESC
            ) AS rn
        FROM oceanbase.GV$OB_SQL_AUDIT
    ) t
    WHERE rn = 1
) ls ON ls.SID = lc.session_id
WHERE p.USER IS NOT NULL
   OR lc.depth > 0
ORDER BY lc.root_trans_id, lc.depth;
```

**Пример вывода:**

```
+-------+----------------------------------------------+------------+---------+---------------------+----------+----------+----------------------------------------------------------+------------+
| depth | chain                                        | session_id | db_user | client_ip           | hold_sec | wait_sec | sql_text                                                 | sql_source |
+-------+----------------------------------------------+------------+---------+---------------------+----------+----------+----------------------------------------------------------+------------+
|     0 | 40895864                                     | 3221502420 | usr1    | 192.168.73.31:51544 |    371.4 |     NULL | UPDATE accounts SET balance = balance - 100 WHERE id = 1 | last       |
|     1 | 40895864 -> 40895925                         | 3221503628 | usr2    | 192.168.73.31:43000 |    354.5 |    353.6 | UPDATE accounts SET balance = balance + 50 WHERE id = 1  | current    |
|     2 | 40895864 -> 40895925 -> 40895991             | 3221504757 | usr3    | 192.168.73.31:41472 |    336.6 |    336.0 | UPDATE accounts SET balance = balance * 1.1 WHERE id = 2 | current    |
|     3 | 40895864 -> 40895925 -> 40895991 -> 40896051 | 3221506054 | usr4    | 192.168.73.31:49306 |    319.1 |    317.9 | UPDATE accounts SET balance = 9999 WHERE id = 3          | current    |
+-------+----------------------------------------------+------------+---------+---------------------+----------+----------+----------------------------------------------------------+------------+
```

> **sql_source:**
> - `current` — сессия прямо сейчас выполняет этот запрос (висит на блокировке)
> - `last` — сессия idle, показан последний запрос из `GV$OB_SQL_AUDIT`

---

### 2.5 Держатели блокировок с именем таблицы

```sql
SELECT
    a.tenant_id,
    a.svr_ip,
    a.session_id,
    c.table_name,
    a.tablet_id,
    a.ctx_create_time
FROM __all_virtual_trans_lock_stat a
LEFT JOIN __all_virtual_tablet_to_ls b ON b.tablet_id = a.tablet_id
LEFT JOIN __all_virtual_table c        ON b.table_id  = c.table_id
WHERE c.table_name = 'accounts';    -- подставить нужное имя таблицы
```

---

### 2.6 Какие конкретные строки заблокированы (rowkey)

Показывает rowkey заблокированной строки, а также прямую связку ожидающий ↔ держатель.

> Схема полей актуальна для OceanBase CE 4.4.1.
> В версиях до 4.2 поля `holder_trans_id`, `holder_session_id` отсутствуют.

```sql
SELECT
    lw.svr_ip,
    lw.tablet_id,
    t.table_name,
    lw.rowkey,                                              -- конкретная строка!
    lw.session_id                                           AS waiter_session,
    lw.holder_session_id                                    AS holder_session,
    lw.trans_id                                             AS waiter_trans,
    lw.holder_trans_id                                      AS holder_trans,
    p_wait.USER                                             AS waiter_user,
    p_hold.USER                                             AS holder_user,
    lw.try_lock_times,
    lw.lock_mode,
    ROUND(lw.time_after_recv / 1000000, 1)                 AS wait_sec,
    ROUND((lw.abs_timeout - lw.lock_ts) / 1000000, 0)     AS timeout_sec
FROM __all_virtual_lock_wait_stat lw
LEFT JOIN __all_virtual_tablet_to_ls tl ON tl.tablet_id = lw.tablet_id
LEFT JOIN __all_virtual_table t         ON t.table_id   = tl.table_id
LEFT JOIN information_schema.PROCESSLIST p_wait ON p_wait.ID = lw.session_id
LEFT JOIN information_schema.PROCESSLIST p_hold ON p_hold.ID = lw.holder_session_id
WHERE lw.need_wait = 1
ORDER BY wait_sec DESC;
```

---

### 2.7 Исторические жертвы блокировок (из sql_audit)

Запросы у которых `elapsed_time >> execute_time` и много ретраев — признак ожидания блокировки.

```sql
SELECT
    usec_to_time(request_time)                              AS req_time,
    SID,
    user_name,
    query_sql,
    retry_cnt,
    ROUND(elapsed_time / 1000000, 2)                       AS elapsed_sec,
    ROUND(execute_time / 1000000, 2)                       AS execute_sec,
    ROUND((elapsed_time - execute_time) / 1000000, 2)     AS lock_wait_sec
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE retry_cnt > 3
  AND elapsed_time > execute_time * 5
ORDER BY retry_cnt DESC, elapsed_time DESC
LIMIT 20;
```

---

## 3. Устранение блокировок

### 3.1 KILL сессии

```sql
-- Найти session_id из запроса 2.4 (колонка session_id, depth=0)
-- KILL принимает SESSION_ID (не TRANS_ID!)

KILL 3221503628;   -- убивает сессию, транзакция откатывается автоматически
```

> **Внимание:** `KILL <TRANS_ID>` вернёт ошибку `Unknown thread id` — нужен именно `session_id`.

### 3.2 Последовательность при цепочке

При цепочке `A → B → C → D` достаточно убить **корень (depth=0)** — цепочка рассыплется.
Если нужно убить конкретное звено — остальные получат блокировку и продолжат работу.

---

## 4. Создание VIEW для быстрого мониторинга

Создаётся один раз от `root@sys`. После этого запрос из любого инструмента сводится к `SELECT * FROM oceanbase.v_lock_tree`.

```sql
CREATE OR REPLACE VIEW oceanbase.v_lock_tree AS
WITH RECURSIVE lock_chain AS (
    SELECT
        lh.TRANS_ID                         AS trans_id,
        lh.TRANS_ID                         AS root_trans_id,
        CAST(lh.TRANS_ID AS CHAR(500))      AS chain,
        lh.ID2                              AS session_id,
        0                                   AS depth
    FROM GV$OB_LOCKS lh
    WHERE lh.TYPE  = 'TX'
      AND lh.BLOCK = 0
      AND lh.TRANS_ID NOT IN (
          SELECT TRANS_ID FROM GV$OB_LOCKS WHERE BLOCK = 1 AND TYPE = 'TR'
      )
    UNION ALL
    SELECT
        lw.TRANS_ID,
        lc.root_trans_id,
        CONCAT(lc.chain, ' -> ', lw.TRANS_ID),
        tx_w.ID2,
        lc.depth + 1
    FROM GV$OB_LOCKS lw
    JOIN lock_chain lc   ON lc.trans_id = lw.ID1
    LEFT JOIN GV$OB_LOCKS tx_w
        ON  tx_w.TRANS_ID = lw.TRANS_ID
        AND tx_w.TYPE     = 'TX'
    WHERE lw.BLOCK = 1
      AND lw.TYPE  = 'TR'
      AND lc.depth < 10
)
SELECT
    lc.depth,
    lc.root_trans_id,
    lc.chain,
    lc.session_id,
    p.USER                                  AS db_user,
    p.HOST                                  AS client_ip,
    ROUND(locks.CTIME / 1000000, 1)         AS hold_sec,
    ROUND(tr.CTIME   / 1000000, 1)         AS wait_sec,
    COALESCE(p.INFO, ls.query_sql)          AS sql_text,
    CASE WHEN p.INFO IS NULL THEN 'last' ELSE 'current' END AS sql_source
FROM lock_chain lc
LEFT JOIN information_schema.PROCESSLIST p
    ON  p.ID = lc.session_id
LEFT JOIN GV$OB_LOCKS locks
    ON  locks.TRANS_ID = lc.trans_id AND locks.TYPE = 'TX'
LEFT JOIN GV$OB_LOCKS tr
    ON  tr.TRANS_ID = lc.trans_id AND tr.TYPE = 'TR' AND tr.BLOCK = 1
LEFT JOIN (
    SELECT SID, query_sql FROM (
        SELECT SID, query_sql,
            ROW_NUMBER() OVER (PARTITION BY SID ORDER BY request_time DESC) AS rn
        FROM oceanbase.GV$OB_SQL_AUDIT
    ) t WHERE rn = 1
) ls ON ls.SID = lc.session_id
WHERE p.USER IS NOT NULL OR lc.depth > 0;
```

Использование:
```sql
-- В терминале (вертикальный вывод)
SELECT * FROM oceanbase.v_lock_tree ORDER BY root_trans_id, depth\G

-- В DBeaver
SELECT * FROM oceanbase.v_lock_tree ORDER BY root_trans_id, depth;
```

---

## 5. Ограничения OceanBase CE

| Ограничение | Деталь |
|---|---|
| `INNODB_LOCK_WAITS` — заглушка | Всегда возвращает нули, реальных данных нет |
| `GV$OB_LOCKS` только из sys tenant | Из tenant-подключения вернёт ошибку `Table doesn't exist` |
| `CTIME` в микросекундах | Делить на 1 000 000 для секунд |
| `__all_virtual_lock_wait_stat` — поле `table_id` | В v4.4.1 переименовано в `tablet_id`, схема из старых статей устарела |
| Фантомные строки `depth=1, session_id=NULL` | Системные транзакции OB (`__all_weak_read_service` и др.) — фильтруются через `WHERE p.USER IS NOT NULL` |
