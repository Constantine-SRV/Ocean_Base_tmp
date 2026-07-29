# Диагностика производительности OceanBase CE 4.4.2.1

Практическое руководство. Все примеры вывода — реальные, с двух кластеров:
`obc442` (OceanBase_CE_CL 4.4.2.1, два узла, две зоны) и производственного
стенда на OceanBase CE 4.5.0.0 (три узла, три зоны, 800 конкурентных сессий).
Ничего не взято из документации без проверки. Версионные различия 4.4 / 4.5
отмечены по тексту.

---

# Часть 1. Модель

## 1.1 Что вообще есть в CE

| Инструмент Oracle | Аналог в OB CE 4.4.2 | Состояние |
|---|---|---|
| ASH | `V$ACTIVE_SESSION_HISTORY` / `GV$…` | есть, полный |
| `dba_hist_active_sess_history` | `DBA_WR_ACTIVE_SESSION_HISTORY` | есть, 7 суток, прорежен |
| AWR-репозиторий | WR: 24 вьюхи `DBA_WR_*` / `CDB_WR_*` | есть |
| `ashrpt.sql` | `DBMS_WORKLOAD_REPOSITORY.ASH_REPORT()` | есть |
| AWR-отчёт | **отдельного генератора нет** | `ash_report` сам читает WR |
| `v$sql_plan_monitor` | `GV$SQL_PLAN_MONITOR` | есть |
| — | `GV$OB_SQL_AUDIT` | аналога в Oracle нет |
| ADDM, SQL Tuning Advisor | — | нет |

Пакет `DBMS_WORKLOAD_REPOSITORY` в CE содержит ровно пять процедур:

```sql
SELECT p.package_name, r.routine_name
FROM oceanbase.__all_virtual_package p
JOIN oceanbase.__all_virtual_routine r
  ON r.tenant_id = p.tenant_id AND r.package_id = p.package_id
WHERE p.package_name LIKE '%WORKLOAD%';
```
```
CREATE_SNAPSHOT | DROP_SNAPSHOT_RANGE | MODIFY_SNAPSHOT_SETTINGS
ASH_REPORT | ASH_REPORT_TEXT
```

Никаких `WR_REPORT_HTML` / `AWR_REPORT_HTML` не существует. Утверждения о том,
что пакет недоступен в CE или требует Oracle-режима, — неверны.

Настройки WR кластерные, задаются из sys, действуют на все тенанты:

```
DBA_WR_CONTROL:  SNAP_INTERVAL +0 01:00:00 | RETENTION +7 00:00:00 | TOPNSQL 100
```

## 1.2 Главная метрика — AAS

**Average Active Sessions** = среднее число сессий, одновременно выполняющих
работу. Считается как «сэмплов / длительность окна в секундах», потому что ASH
сэмплирует раз в секунду.

Порог — число ядер тенанта (`max_cpu`). Но одного AAS мало: решает связка
**AAS + доля ON CPU**.

| AAS | % ON CPU | Диагноз |
|---|---|---|
| < max_cpu | любой | база справляется |
| > max_cpu | высокий | не хватает CPU: планы, full scan, мало ядер |
| > max_cpu | **низкий** | ресурсы простаивают, всё стоит в ожидании — идти в D4 |

Второй случай — самый частый и самый непонятный без ASH.

## 1.3 Три горизонта, и это главное ограничение

| Источник | Глубина | Тексты SQL | IP клиента |
|---|---|---|---|
| `GV$OB_SQL_AUDIT` | **минуты** | да, полные | да |
| живой ASH | ~10–30 мин | через plan cache | нет |
| WR ASH | 7 суток, прорежен | нет | нет |
| `__all_virtual_lock_wait_stat` | текущая секунда | нет | нет |
| `GV$OB_PROCESSLIST` | текущая секунда | да | адрес OBProxy |

**Буфер ASH ограничен количеством записей — ровно 17 712 на узел.**
Подтверждено многократно: значение совпадает до единицы на обоих узлах,
а `GV$` (два узла) всегда даёт 35 424.

Горизонт вычисляется так:

```
глубина (сек) ≈ 17712 / (число сэмплов в секунду на узле)
```

Реальный замер с одного и того же момента:

```
| svr_ip         | depth_sec | samples |
| 192.168.55.202 |       601 |   17712 |   <- под нагрузкой: 10 минут
| 192.168.55.205 |      2271 |   17712 |   <- почти простаивал: 38 минут
```

Один кластер, одна секунда, разница в четыре раза. Чем хуже ситуация,
тем быстрее исчезают данные о ней.

**Демонстрация того же для sql_audit.** Нагрузка остановилась в 22:57:26.
Запрос к аудиту в 23:03 за окно 22:45–23:00:

```
Empty set (0.04 sec)
```

Через шесть минут после остановки от нагрузки не осталось ни строки, при том
что ASH хранил её ещё пятнадцать минут. Отсюда единственно верный порядок
действий при инциденте: **сначала выгрузить sql_audit, потом разбираться**.

---

# Часть 2. Запросы с разбором

## 2.0 Окно анализа

```sql
SET SESSION group_concat_max_len = 65535;
SET @t_to   := NOW();
SET @t_from := @t_to - INTERVAL 15 MINUTE;
SET @win_sec := TIMESTAMPDIFF(SECOND, @t_from, @t_to);
SET @tenant := 1004;
```

Пользовательские переменные в OB работают. Агент подставляет параметрами.

## 2.1 D1 — сколько истории доступно

Всегда первый запрос: он говорит, есть ли ещё данные за интересующее окно.

```sql
SELECT svr_ip,
       MIN(sample_time) AS oldest,
       MAX(sample_time) AS newest,
       TIMESTAMPDIFF(SECOND, MIN(sample_time), NOW()) AS depth_sec,
       COUNT(*) AS samples
FROM oceanbase.GV$ACTIVE_SESSION_HISTORY
GROUP BY 1;
```

Если `@t_from` старше `oldest` — живой ASH не покрывает окно, идти в WR (D12).

## 2.2 D2 — сводка

```sql
SELECT con_id AS tenant_id,
       COUNT(*) AS samples,
       ROUND(COUNT(*)/@win_sec, 2) AS aas,
       ROUND(100*SUM(session_state='ON CPU')/COUNT(*), 1) AS pct_cpu,
       ROUND(100*SUM(COALESCE(blocking_session_id,0)>0)/COUNT(*), 1) AS pct_blocked,
       COUNT(DISTINCT session_id) AS sessions,
       COUNT(DISTINCT sql_id) AS uniq_sql,
       COUNT(DISTINCT NULLIF(blocking_session_id,0)) AS blockers
FROM oceanbase.V$ACTIVE_SESSION_HISTORY
WHERE sample_time BETWEEN @t_from AND @t_to
  AND session_type = 'FOREGROUND'
  AND COALESCE(wait_class,'') <> 'IDLE'
GROUP BY 1;
```

```
| tenant_id | samples | aas   | pct_cpu | pct_blocked | sessions | uniq_sql | blockers |
|      1004 |   12541 | 13.93 |    10.7 |         0.0 |       51 |       11 |        0 |
```

**Разбор.** `max_cpu` тенанта = 2, а AAS 13.93 — семикратная перегрузка.
При этом `pct_cpu` всего 10.7: процессор свободен, четырнадцать сессий
чего-то ждут. `sessions 51` в точности совпадает с `concurrent = 50`
в конфиге теста плюс наша диагностическая сессия — значит потери
соединений нет, все потоки живы и стоят.

`uniq_sql 11` — нагрузка однородная, агрегация по `sql_id` будет осмысленной.
Сравните с DDL-тестом, где на 2830 сэмплов приходилось 2498 уникальных
`sql_id`: там группировка по `sql_id` не агрегирует ничего.

## 2.3 D3 — таймлайн

```sql
SELECT FLOOR(UNIX_TIMESTAMP(sample_time)/10)*10 AS slot_epoch,
       MIN(sample_time)                         AS slot_time,
       con_id                                   AS tenant_id,
       CASE WHEN session_state='ON CPU' THEN 'ON CPU'
            ELSE COALESCE(NULLIF(wait_class,''),'OTHER') END AS wclass,
       COUNT(*)                                 AS samples,
       ROUND(COUNT(*)/10, 2)                    AS aas
FROM oceanbase.V$ACTIVE_SESSION_HISTORY
WHERE sample_time BETWEEN @t_from AND @t_to
  AND session_type = 'FOREGROUND'
  AND COALESCE(wait_class,'') <> 'IDLE'
GROUP BY 1,3,4
ORDER BY 1,5 DESC;
```

```
| slot_time                  | wclass        | samples | aas   |
| 2026-07-28 22:50:24.449127 | COMMIT        |     191 | 19.10 |
| 2026-07-28 22:50:30.450133 | COMMIT        |     285 | 28.50 |
| 2026-07-28 22:50:40.452110 | COMMIT        |     432 | 43.20 |   <- пик
| 2026-07-28 22:51:00.454686 | COMMIT        |     426 | 42.60 |
...
| 2026-07-28 22:57:10.516996 | COMMIT        |     287 | 28.70 |
| 2026-07-28 22:57:20.519437 | COMMIT        |      53 |  5.30 |   <- нагрузка снята
```

**Разбор.** Данные для стекового графика: X — `slot_time`, Y — `aas`,
цвет — `wclass`, горизонтальная линия на уровне `max_cpu`. Начало и конец
инцидента датируются с точностью до десяти секунд.

Здесь же виден постоянный фон `CONFIGURATION` в 3–7 сэмплов каждый слот —
это `wait in request queue`, то есть примерно полсессии непрерывно стоят
в очереди за рабочим потоком. Само по себе некритично, но при росте
до значений порядка `max_cpu` означает нехватку потоков.

## 2.4 D4 — куда уходит время

```sql
SELECT con_id AS tenant_id,
       COALESCE(NULLIF(event,''),'ON CPU')      AS event,
       COALESCE(NULLIF(wait_class,''),'-')      AS wait_class,
       session_state,
       COUNT(*)                                 AS samples,
       ROUND(100*COUNT(*)/SUM(COUNT(*)) OVER (), 2) AS pct,
       ROUND(SUM(time_waited)/1000000, 2)       AS waited_sec,
       ROUND(AVG(NULLIF(time_waited,0))/1000, 2) AS avg_ms
FROM oceanbase.V$ACTIVE_SESSION_HISTORY
WHERE sample_time BETWEEN @t_from AND @t_to
  AND session_type = 'FOREGROUND'
  AND COALESCE(wait_class,'') <> 'IDLE'
GROUP BY 1,2,3,4
ORDER BY 5 DESC
LIMIT 20;
```

```
| event                 | wait_class    | samples | pct   | waited_sec | avg_ms |
| tx commiting wait     | COMMIT        |   10598 | 87.67 |     519.98 |  49.17 |
| ON CPU                | OTHER         |    1294 | 10.70 |       0.00 |   NULL |
| wait in request queue | CONFIGURATION |     194 |  1.60 |       0.00 |   NULL |
| sync rpc              | NETWORK       |       1 |  0.01 |       0.36 | 360.33 |
```

**Разбор.** 87.67% всего времени БД — ожидание подтверждения транзакции,
средняя стоимость одного ожидания 49 мс. Это и есть ответ на вопрос
«почему AAS 14 при свободном процессоре».

`avg_ms` информативнее, чем `samples`: он даёт стоимость единичного ожидания
и позволяет раскладывать составное ожидание на части (см. D10).

**Ловушка, которая стоила отдельного разбора.** Без фильтров
`session_type='FOREGROUND'` и `wait_class <> 'IDLE'` тот же запрос выдаёт вот это:

```
| con_id | type       | event                       | samples | pct   |
|      1 | BACKGROUND | ON CPU                      |    3743 | 12.24 |
|      1 | BACKGROUND | sleep wait            IDLE  |    3603 | 11.79 |
|   1003 | BACKGROUND | remote log writer cond wait |    3600 | 11.78 |
|      1 | BACKGROUND | remote log writer cond wait |    3600 | 11.78 |
|   1004 | BACKGROUND | remote log writer cond wait |    3599 | 11.77 |
```

Первые пять строк — 59% сэмплов — чистый шум. Обратите внимание на числа:
`3600` при окне в 1800 секунд = 2 узла × 1800 секунд. Это потоки, которые
ждут **всегда**, каждую секунду, независимо от нагрузки. Полезная строка
(`sync rpc`, 7%) оказывается шестой.

## 2.5 D5 — топ SQL с текстом

Исправленная версия: `stmt_type` берётся агрегатом, иначе один `sql_id`
расщепляется на несколько строк.

```sql
SELECT a.con_id                                 AS tenant_id,
       a.sql_id,
       MAX(a.stmt_type)                         AS stmt_type,
       COUNT(*)                                 AS samples,
       ROUND(COUNT(*)/@win_sec, 2)              AS aas,
       SUM(a.session_state='ON CPU')            AS on_cpu,
       SUM(a.session_state='WAITING')           AS waiting,
       SUM(COALESCE(a.blocking_session_id,0)>0) AS blocked,
       COUNT(DISTINCT a.session_id)             AS sessions,
       SUBSTRING_INDEX(GROUP_CONCAT(DISTINCT NULLIF(a.event,'')), ',', 3) AS top_events,
       ROUND(SUM(a.delta_read_io_bytes)/1048576, 1)  AS read_mb,
       MAX(pc.executions)                       AS plan_execs,
       LEFT(MAX(pc.query_sql), 300)             AS sql_text
FROM oceanbase.V$ACTIVE_SESSION_HISTORY a
LEFT JOIN (SELECT tenant_id, sql_id,
                  SUM(executions) AS executions,
                  MIN(query_sql)  AS query_sql
             FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT
            GROUP BY 1,2) pc
       ON pc.tenant_id = a.con_id AND pc.sql_id = a.sql_id
WHERE a.sample_time BETWEEN @t_from AND @t_to
  AND a.session_type = 'FOREGROUND'
  AND COALESCE(a.wait_class,'') <> 'IDLE'
  AND a.sql_id <> ''
GROUP BY 1,2
ORDER BY 4 DESC
LIMIT 20;
```

```
| sql_id       | samples | aas  | on_cpu | waiting | sessions | top_events        | sql_text
| F763CB60…    |    5262 | 5.85 |     66 |    5196 |       50 | tx commiting wait | INSERT INTO test_tbl (id, val) VALUES (305199, …)
| 2F1D5953…    |    5259 | 5.84 |     51 |    5208 |       50 | tx commiting wait | DELETE FROM test_tbl WHERE id = 305231
| A757D02C…    |     577 | 0.64 |    577 |       0 |       50 | NULL              | SELECT task_type … __all_detect_lock_info_v2 …
| E1A0DACF…    |      98 | 0.11 |     98 |       0 |       42 | NULL              | SELECT * FROM test_tbl WHERE id = {i+4}
```

**Разбор.** Два оператора дают 10 521 сэмпл из 12 541 — 84% времени БД.
Соотношение `waiting` к `on_cpu` около 80:1 у обоих. Читается однозначно:
проблема не в самих запросах, а в том, что происходит после них — в коммите.

`sessions 50` при `samples 5262` означает, что нагрузка равномерна: все
пятьдесят потоков делают одно и то же, а не один поток завис.

**Две оговорки по `sql_text`, обе проверены.**

`query_sql` в plan cache — текст **того выполнения, которое скомпилировало
план**, а не каждого. OB параметризует литералы, один `sql_id` обслуживает
все значения. Доказательство: один и тот же `sql_id 74F1A4AE…` в трёх замерах
показал `id = 99999`, затем `id = 434833`, затем `id = 305219`. Не читайте
показанное число как реальный аргумент.

DDL в plan cache не попадает — у `CREATE`/`DROP` нет плана. Для DDL `sql_text`
будет NULL, текст берётся только из D11 и только пока цел аудит.

## 2.6 D6 — запрос × его ожидание

```sql
SELECT a.con_id AS tenant_id, a.sql_id,
       COALESCE(NULLIF(a.event,''),'ON CPU') AS event,
       COALESCE(NULLIF(a.wait_class,''),'-') AS wait_class,
       COUNT(*)                              AS samples,
       ROUND(SUM(a.time_waited)/1000000, 2)  AS waited_sec,
       LEFT(MAX(pc.query_sql), 200)          AS sql_text
FROM oceanbase.V$ACTIVE_SESSION_HISTORY a
LEFT JOIN (SELECT tenant_id, sql_id, MIN(query_sql) AS query_sql
             FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT GROUP BY 1,2) pc
       ON pc.tenant_id = a.con_id AND pc.sql_id = a.sql_id
WHERE a.sample_time BETWEEN @t_from AND @t_to
  AND a.session_type = 'FOREGROUND'
  AND COALESCE(a.wait_class,'') <> 'IDLE'
  AND a.sql_id <> ''
GROUP BY 1,2,3,4
ORDER BY 5 DESC
LIMIT 25;
```

```
| sql_id     | event             | wait_class | samples | waited_sec | sql_text
| 2F1D5953…  | tx commiting wait | COMMIT     |    5151 |     245.04 | DELETE FROM test_tbl WHERE id = 305231
| F763CB60…  | tx commiting wait | COMMIT     |    5139 |     260.42 | INSERT INTO test_tbl …
| A757D02C…  | ON CPU            | OTHER      |     575 |       0.00 | SELECT task_type … 
```

**Разбор.** Это самая точная формулировка проблемы: не «DELETE медленный»,
а «DELETE 5151 сэмпл ждёт коммита, суммарно 245 секунд». Разница принципиальна
для выбора действия: оптимизировать запрос бессмысленно, надо чинить коммит.

## 2.7 D7 — блокировки задним числом

**Единственный источник, который помнит блокировку после её снятия.**
Исправленная версия: добавлен фильтр `FOREGROUND` (иначе 90% результата —
внутренние сессии с идентификаторами вида 4611686018485700630) и убран
`GROUP_CONCAT`, дававший 64 предупреждения об усечении.

```sql
SELECT b.tenant_id, b.holder_sid,
       b.victims, b.blocked_samples, b.dur_sec, b.t_from, b.t_to,
       h.holder_event,
       LEFT(hp.query_sql, 150) AS holder_sql,
       LEFT(vp.query_sql, 150) AS victim_sql
FROM (
    SELECT con_id                     AS tenant_id,
           blocking_session_id        AS holder_sid,
           COUNT(*)                   AS blocked_samples,
           COUNT(DISTINCT session_id) AS victims,
           MIN(sample_time)           AS t_from,
           MAX(sample_time)           AS t_to,
           TIMESTAMPDIFF(SECOND, MIN(sample_time), MAX(sample_time)) AS dur_sec,
           MIN(sql_id)                AS victim_sql_id
    FROM oceanbase.V$ACTIVE_SESSION_HISTORY
    WHERE sample_time BETWEEN @t_from AND @t_to
      AND COALESCE(blocking_session_id,0) > 0
      AND session_type = 'FOREGROUND'
    GROUP BY 1,2
    HAVING COUNT(*) >= 3
) b
LEFT JOIN (
    SELECT session_id, con_id,
           MAX(sql_id)            AS holder_sql_id,
           MAX(NULLIF(event,''))  AS holder_event
    FROM oceanbase.V$ACTIVE_SESSION_HISTORY
    WHERE sample_time BETWEEN @t_from AND @t_to
    GROUP BY 1,2
) h ON h.session_id = b.holder_sid AND h.con_id = b.tenant_id
LEFT JOIN (SELECT tenant_id, sql_id, MIN(query_sql) AS query_sql
             FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT GROUP BY 1,2) hp
       ON hp.tenant_id = b.tenant_id AND hp.sql_id = h.holder_sql_id
LEFT JOIN (SELECT tenant_id, sql_id, MIN(query_sql) AS query_sql
             FROM oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT GROUP BY 1,2) vp
       ON vp.tenant_id = b.tenant_id AND vp.sql_id = b.victim_sql_id
ORDER BY b.blocked_samples DESC
LIMIT 15;
```

Реальный вывод в момент, когда блокировки были:

```
| holder_sid | victims | blocked_samples | t_from            | t_to              | holder_event      | holder_sql
| 3221742626 |      24 |              24 | 22:26:00.108155   | 22:26:00.108155   | tx commiting wait | INSERT INTO test_tbl (id,val) VALUES (99998, …)
| 3221742865 |      18 |              18 | 22:26:07.109436   | 22:26:07.109436   | tx commiting wait | INSERT INTO test_tbl (id,val) VALUES (99998, …)
```

**Разбор и главное доказательство ценности истории.** Мгновенный срез (D8)
в тот же момент вернул `Empty set` — блокировка уже снялась. А ASH через
десять минут после события восстановил полную картину: держатель, 24 жертвы,
тексты обоих запросов, точное время.

`holder_sql = NULL` — не ошибка запроса, а диагноз: держатель в момент
блокировки был неактивен, ASH его не сэмплировал. Это классическая забытая
открытая транзакция, и увидеть её можно только в D8, пока она жива.

На DML-нагрузке с исправленным генератором D7 дал развёрнутую картину:
держатель `INSERT` в событии `tx commiting wait` блокировал до 16 жертв
(`UPDATE`/`DELETE` по соседним строкам) с длительностями до 105 секунд.
Механика: строка остаётся залоченной всю фазу коммита, а коммит на этом
стенде стоит десятки миллисекунд — очередь за строкой выстраивается
из-за медленного коммита, а не из-за логики приложения.

**Порог чувствительности.** ASH сэмплирует раз в секунду, поэтому D7 видит
только блокировки длительностью порядка секунды и дольше. Проверено
кросс-тестом на кластере 4.5 со здоровыми дисками: коммит 7–9 мс, живые
блокировки жили 5–30 мс (`holder_idle_sec` 0.001–0.015), и D7 вернул
`Empty set` при том, что D8 в тот же период ловил блокировки. Это норма,
а не пропуск: субсекундная конкуренция короче шага сэмплирования, и видна
она только по последствиям — ретраям в аудите (D11) и уловам D8.
Пустой D7 при быстрых коммитах означает «блокировки есть, но мгновенные».

## 2.8 D8 — блокировки прямо сейчас

```sql
SELECT w.tenant_id,
       w.session_id                    AS waiter_sid,
       pw.user AS waiter_user, pw.host AS waiter_host,
       ROUND(w.time_after_recv/1000,1) AS waiting_ms,
       w.try_lock_times                AS retries,
       LEFT(pw.info,120)               AS waiter_sql,
       COALESCE(NULLIF(w.holder_session_id,0), w.block_session_id) AS holder_sid,
       ph.user AS holder_user, ph.host AS holder_host,
       ph.command AS holder_state, ph.time AS holder_idle_sec,
       LEFT(ph.info,120)               AS holder_sql,
       LEFT(w.rowkey,40)               AS rowkey
FROM oceanbase.__all_virtual_lock_wait_stat w
LEFT JOIN oceanbase.GV$OB_PROCESSLIST pw ON pw.id = w.session_id
LEFT JOIN oceanbase.GV$OB_PROCESSLIST ph
       ON ph.id = COALESCE(NULLIF(w.holder_session_id,0), w.block_session_id)
WHERE w.need_wait = 1
ORDER BY w.time_after_recv DESC
LIMIT 20;
```

Колонка называется `time_after_recv`, а не `wait_time` — последней в этой
вьюхе нет. `host` в processlist — адрес OBProxy, а не клиента: реальный IP
приложения есть только в `GV$OB_SQL_AUDIT.user_client_ip`.

`holder_state = 'Sleep'` при большом `holder_idle_sec` — готовый вердикт:
приложение открыло транзакцию и не закрыло.

Пример живого улова:

```
| waiter_sid | waiting_ms | holder_sid | holder_user | holder_host          | holder_sql                              | rowkey         |
|  134217728 |     4517.8 | 3221658884 | rust        | 192.168.55.200:34456 | update test_tbl set val='aaaa' WHERE …  | {"INT":746028} |
```

Нюанс: на 4.4.2 `waiter_sid` может быть 0 или 134217728 с
`waiter_user = NULL` — внутренний плейсхолдер исполнителя, ожидающая сторона
не всегда разрешается в processlist. Не ошибка: ключевая информация —
держатель, и он виден полностью, вплоть до конкретного `rowkey`.
На 4.5.0 ожидающая сторона разрешается корректно: реальный `session_id`,
пользователь и хост.

## 2.9 D9 — фазы выполнения

```sql
SELECT con_id AS tenant_id, COUNT(*) AS total,
       SUM(in_parse='Y')                AS ph_parse,
       SUM(in_sql_optimize='Y')         AS ph_optimize,
       SUM(in_sql_execution='Y')        AS ph_execute,
       SUM(in_committing='Y')           AS ph_commit,
       SUM(in_storage_read='Y')         AS ph_stor_read,
       SUM(in_storage_write='Y')        AS ph_stor_write,
       SUM(in_px_execution='Y')         AS ph_px,
       SUM(in_remote_das_execution='Y') AS ph_remote_das,
       SUM(in_connection_mgr='Y')       AS ph_conn_mgr,
       SUM(in_plan_cache='Y')           AS ph_plan_cache
FROM oceanbase.V$ACTIVE_SESSION_HISTORY
WHERE sample_time BETWEEN @t_from AND @t_to
  AND session_type = 'FOREGROUND'
GROUP BY 1;
```

```
| total | ph_parse | ph_optimize | ph_execute | ph_commit | ph_stor_read | ph_plan_cache |
| 11658 |       16 |           0 |        350 |     10226 |          104 |            61 |
```

**Разбор.** 10 226 из 11 658 — фаза коммита. Независимое подтверждение
диагноза из D4 другим механизмом: не по событиям ожидания, а по флагам фазы.
Когда два независимых среза дают одно и то же, диагноз можно считать твёрдым.

`ph_parse 16` и `ph_optimize 0` — планы переиспользуются, разбор SQL не является
проблемой. Обратная картина (сотни в parse/optimize) означала бы, что
параметризация не работает и каждый запрос компилируется заново.

Алиасы `optimize` и `execute` без префикса использовать нельзя —
зарезервированные слова, синтаксическая ошибка.

## 2.10 D10 — перекос по узлам и разложение коммита

```sql
SELECT svr_ip, session_type,
       COUNT(*) AS samples,
       SUM(session_state='ON CPU') AS on_cpu,
       ROUND(COUNT(*)/@win_sec, 2) AS aas
FROM oceanbase.GV$ACTIVE_SESSION_HISTORY
WHERE sample_time BETWEEN @t_from AND @t_to
GROUP BY 1,2 ORDER BY 3 DESC;
```

```
| svr_ip         | session_type | samples | on_cpu | aas   |
| 192.168.55.202 | FOREGROUND   |   11554 |   1272 | 12.84 |
| 192.168.55.205 | BACKGROUND   |    6959 |   1435 |  7.73 |
| 192.168.55.202 | BACKGROUND   |    4591 |    938 |  5.10 |
| 192.168.55.205 | FOREGROUND   |       1 |      1 |  0.00 |
```

**Разбор.** 11 554 клиентских сэмпла на .202 против **одного** на .205.
Фон при этом распределён нормально. Вся пользовательская нагрузка
обслуживается одним узлом: OBProxy маршрутизирует на лидера партиции,
а таблица `test_tbl` не партиционирована — значит у неё один лидер.

Для однотабличного теста это ожидаемо. В продуктиве такой перекос означает
либо непартиционированные горячие таблицы, либо сосредоточение лидеров
на одном узле, и второй узел работает только как реплика.

```sql
SELECT svr_ip, event, COUNT(*) AS samples,
       ROUND(AVG(time_waited)/1000, 2) AS avg_ms
FROM oceanbase.GV$ACTIVE_SESSION_HISTORY
WHERE sample_time BETWEEN @t_from AND @t_to
  AND event IN ('palf write','tx commiting wait','sync tx commiting wait',
                'db file data read','db file compact read','db file compact write')
GROUP BY 1,2 ORDER BY 1,3 DESC;
```

```
| svr_ip         | event                  | samples | avg_ms |
| 192.168.55.202 | tx commiting wait      |   10052 |  45.53 |
| 192.168.55.202 | palf write             |     701 |  36.38 |
| 192.168.55.202 | sync tx commiting wait |     215 |  89.40 |
| 192.168.55.202 | db file compact read   |     180 |  23.38 |
| 192.168.55.202 | db file compact write  |      18 |  78.98 |
| 192.168.55.205 | palf write             |    1122 |  29.80 |
| 192.168.55.205 | db file compact read   |     278 |  27.94 |
| 192.168.55.205 | sync tx commiting wait |     181 |  61.88 |
| 192.168.55.205 | db file data read      |       2 | 466.84 |
```

**Разбор — итоговый диагноз стенда.** Коммит стоит 45.5 мс, из которых
30–36 мс — `palf write`, то есть запись журнала транзакций на диск.
Сетевая составляющая (`sync tx commiting wait`) встречается 215 раз против
10 052 и в общий счёт почти не входит.

Вывод: **узкое место — дисковая подсистема под clog**, а не сеть, не CPU
и не блокировки. Подтверждается и соседними строками: `db file compact read`
23–28 мс, `db file compact write` 79 мс, единичное `db file data read` 467 мс.
Все дисковые операции на порядки медленнее нормы.

Ориентиры для `palf write`:

| avg_ms | Оценка |
|---|---|
| < 1 | NVMe, норма |
| 1–5 | приемлемо |
| 5–20 | требует внимания |
| **> 20** | диск непригоден для clog |

Практическое следствие: наращивание параллелизма нагрузки не улучшит
пропускную способность, пока не решён вопрос с диском. На стенде это
виртуальные диски Hyper-V без write-back кэша.

Отдельно стоит учитывать конфигурацию из двух зон: кворум составляет
2 из 2, то есть каждый коммит ждёт подтверждения обеих реплик, и при этом
отказоустойчивости нет. Три зоны дают кворум 2 из 3 — коммит подтверждает
быстрейшая реплика, а падение узла переживается.

## 2.11 D11 — sql_audit

Единственный источник полного текста, точных таймингов, IP клиента и DDL.
Окно — минуты.

```sql
SELECT tenant_id, sql_id, stmt_type,
       COUNT(*)                            AS execs,
       ROUND(AVG(elapsed_time)/1000, 2)    AS avg_ms,
       ROUND(MAX(elapsed_time)/1000, 2)    AS max_ms,
       ROUND(SUM(elapsed_time)/1000000, 2) AS total_sec,
       ROUND(AVG(queue_time)/1000, 2)      AS queue_ms,
       ROUND(AVG(execute_time)/1000, 2)    AS exec_ms,
       SUM(table_scan) AS full_scans,
       SUM(retry_cnt)  AS retries,
       SUM(ret_code <> 0) AS errors,
       GROUP_CONCAT(DISTINCT user_name)      AS users,
       GROUP_CONCAT(DISTINCT user_client_ip) AS client_ips,
       LEFT(MAX(query_sql), 300) AS sql_text
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE tenant_id = @tenant
  AND is_inner_sql = 0
  AND request_time BETWEEN TIME_TO_USEC(@t_from) AND TIME_TO_USEC(@t_to)
GROUP BY 1,2,3
ORDER BY 7 DESC
LIMIT 20;
```

Колонки `user_name` и `user_client_ip` подтверждены, причём `user_client_ip`
содержит **реальный адрес клиента** (192.168.55.190 — хост генератора),
а не адрес OBProxy — проброс IP через прокси работает.

Пример с DML-нагрузки:

```
| sql_id     | stmt_type | execs | avg_ms | max_ms  | queue_ms | exec_ms | retries | client_ips     | sql_text
| C51772FC…  | UPDATE    |   471 | 225.93 | 7804.04 |    75.07 |   48.33 |    2000 | 192.168.55.190 | update test_tbl set val='aaaa' WHERE id < … and id < …
| 74F1A4AE…  | SELECT    |  1369 |  24.39 |   95.69 |    15.33 |    8.96 |       0 | 192.168.55.190 | SELECT * FROM test_tbl WHERE id = …
| ACC8AC62…  | UPDATE    |   907 |  28.86 |   90.34 |    15.93 |    1.34 |      32 | 192.168.55.190 | update test_tbl set val='bbbb' WHERE id = …
```

Range-update виден сразу: avg 226 мс против 29 мс у точечного, max 7.8 секунды,
2000 ретраев на 471 выполнение. Диапазонное условие захватывает больше строк —
больше конфликтов — больше перезапусков.

Пример с DDL-теста:

```
| sql_id      | execs | avg_ms   | queue_ms | exec_ms | retries | sample_sql
| 207E6D7A…   |     1 | 52630.52 |   112.21 |  861.93 |     483 | CREATE TABLE test_tbl_4389 …
| 3D735C2C…   |     1 | 14037.25 |    16.56 |  760.41 |     124 | DROP TABLE test_tbl_4339
| 1F25B0AF…   |     1 | 12304.37 |    12.90 |  974.89 |     106 | DROP TABLE test_tbl_4215
```

**Диагностическое правило:**

```
elapsed_time >> queue_time + execute_time   =>   смотреть retry_cnt
```

Один `CREATE TABLE` прожил 52.6 секунды, выполнялся 0.86 секунды, простоял
в очереди 0.11 секунды и был перезапущен 483 раза. Рекорд в другом прогоне —
93.5 секунды при 861 ретрае. Разрыв между `elapsed` и `execute` — это ретраи,
и ничто другое их не покажет.

`request_time` хранится в микросекундах: нужны `TIME_TO_USEC()` для фильтра
и `USEC_TO_TIME()` для отображения. `svr_port` в этой вьюхе — RPC-порт 2882,
а не 2881.

## 2.12 D12 — ретроспектива из WR (7 суток)

Три правила, каждое найдено на стенде:

**Только `CDB_WR_*`, не `DBA_WR_*`.** Из sys `DBA_`-вьюха показывает лишь
сам sys-тенант (767 строк против 3994 в `CDB_` за одно окно).

**Джойн к `DBA_WR_EVENT_NAME` использовать нельзя.** `EVENT_ID` справочника
и `EVENT_NO` в ASH — разные пространства номеров. Проверка на стенде:
`event_no=81` (`tx commiting wait` по живому ASH, 788 строк в WR)
в справочнике по id 81 отсутствует вовсе, а id 26 справочник называет
`latch: default drw lock wait`, тогда как в живом ASH `event_no=26` — это
`wait in request queue`. Джойн выдаёт правдоподобные, но неверные имена:
коммит маскируется под латчи. Диагноз по такому выводу будет ложным.

**ON CPU в WR — это `session_state='ON CPU'`, а не событие.** У таких строк
`event_no=0`, и наивный джойн превращает их в `latch: latch wait queue lock wait`
(event_id 0 справочника).

Правильный маппинг — словарь `event → event_no`, снятый из живого ASH:

```
0  = ON CPU (session_state)      81  = tx commiting wait
16 = palf write                  82  = sync tx commiting wait
26 = wait in request queue       89  = sleep: lock for read need wait...
39 = row lock wait               141 = remote log writer cond wait
44 = sleep wait                  175 = exec inner sql wait
```

Агент строит его динамически и кэширует:

```sql
SELECT DISTINCT event, event_no
FROM oceanbase.V$ACTIVE_SESSION_HISTORY
WHERE event_no > 0 AND event <> '';
```

Итоговый запрос:

```sql
SELECT FLOOR(UNIX_TIMESTAMP(a.sample_time)/60)*60 AS slot_epoch,
       MIN(a.sample_time) AS slot_time,
       a.tenant_id, a.svr_ip,
       CASE WHEN a.session_state='ON CPU' THEN 'ON CPU'
            ELSE COALESCE(m.event_name, CONCAT('event_no=',a.event_no)) END AS event,
       COUNT(*)                AS raw_samples,
       ROUND(SUM(a.weight),1)  AS weighted
FROM oceanbase.CDB_WR_ACTIVE_SESSION_HISTORY a
LEFT JOIN (
  SELECT 16 AS event_no,'palf write' AS event_name
  UNION ALL SELECT 26,'wait in request queue'
  UNION ALL SELECT 39,'row lock wait'
  UNION ALL SELECT 44,'sleep wait'
  UNION ALL SELECT 81,'tx commiting wait'
  UNION ALL SELECT 82,'sync tx commiting wait'
  UNION ALL SELECT 89,'lock for read wait'
  UNION ALL SELECT 141,'remote log writer cond wait'
  UNION ALL SELECT 175,'exec inner sql wait'
) m ON m.event_no = a.event_no
WHERE a.sample_time BETWEEN @t_from AND @t_to
GROUP BY 1,3,4,5
ORDER BY 1,6 DESC;
```

С правильным маппингом WR показывает тот же профиль, что и живой ASH
(81 + 89 + 16 доминируют), то есть **смещения выборки нет** — WR честно
хранит структуру нагрузки за 7 суток.

**`event_no` сдвигаются между версиями.** Сравнение 4.4.2 и 4.5.0:

| Событие | 4.4.2 | 4.5.0 |
|---|---|---|
| tx commiting wait | 81 | 79 |
| sync tx commiting wait | 82 | 80 |
| row lock wait | 39 | 36 |
| sleep wait | 44 | 42 |
| lock for read need wait... | 89 | 87 |
| exec inner sql wait | 175 | 122 |
| palf write | 16 | 16 |
| wait in request queue | 26 | 26 |

Часть кодов совпадает, часть съехала. Захардкоженный словарь на другой
версии даёт правдоподобные, но ложные имена — та же болезнь, что и джойн
к `DBA_WR_EVENT_NAME`, только межверсионная. Поэтому словарь в запросе ниже —
иллюстрация для 4.4.2, а агент обязан снимать его динамически при старте
и после реконнекта:

```sql
SELECT DISTINCT event, event_no
FROM oceanbase.V$ACTIVE_SESSION_HISTORY
WHERE event_no > 0 AND event <> '';
```

**Предупреждение про двойника.** Справочник по id 26 выдаёт
`latch: default drw lock wait`, тогда как в живом ASH `event_no=26` — это
`wait in request queue`. Если в WR-выводе аномально много «латчей» —
проверьте, не event_no=26 ли это: на самом деле речь об очереди
за рабочими потоками, а не о внутренней конкуренции.

**Про `weight` — только 4.4.x.** Он отличается от единицы в обе стороны:
0.1 — прореживание 10:1 при сбросе, но встречается и ~3 (event_no=26:
60 raw → 187.3 weighted, коэффициент одинаков по узлам). Точная формула
ядра не установлена; на 4.4 использовать `SUM(weight)` как оценку числа
активных сессий.

**В 4.5.0 колонка `WEIGHT` удалена** (66 колонок против 67 в 4.4.2,
исчез и флаг взвешивания) — запрос с `SUM(a.weight)` падает с
`Unknown column`. На 4.5 считать `COUNT(*)`; агент определяет вариант
по наличию колонки при старте. Взамен в WR-схеме 4.5 появились
`BLOCKING_SESSION_ID`, `TX_ID` и `TM_DELTA_CPU_TIME` / `TM_DELTA_DB_TIME` —
ретроспектива блокировок и разложение CPU/DB-времени доступны из
семисуточного архива, не только из живого буфера.

Степень прореживания зависит от профиля и окна: на стенде отношение живого
ASH к WR составляло от ~34:1 до ~87:1 (`tx commiting wait`: 6867 сэмплов
live против 788 raw в WR). Считать WR порядковой оценкой, а не точным счётом.

Ограничение: `CDB_WR_SQLTEXT` пуст даже после ручного снапшота — за 7 суток
доступна структура нагрузки (события, sql_id, тенанты, узлы), но не тексты.

---

# Часть 3. Два разобранных инцидента

## 3.1 DDL-шторм

**Симптомы.** AAS 23.41 при `max_cpu = 2`. On CPU — 18 сэмплов из 23 137,
то есть 0.08%. Топовое событие `sync rpc` (NETWORK), 1881 секунда ожидания.
Дальше `sleep: wait refresh schema` и `wait in request queue`.

**Что делала нагрузка.** `CREATE TABLE test_tbl_N` / `DROP TABLE test_tbl_N`
в 50 потоков, около 340 объектов за пять минут.

**Диагноз.** DDL в OceanBase — распределённая операция: согласование по RPC
между всеми observer и обновление версии схемы у всех тенантов. Параллельный
поток DDL создаёт непрерывный конфликт версий: каждая операция ждёт
`refresh schema`, получает отказ и ретраится сотни раз.

**Особенность для диагностики.** DDL не параметризуется: на 2830 сэмплов
пришлось 2498 уникальных `sql_id`. Группировка по `sql_id` в такой ситуации
не агрегирует ничего — нужна группировка по `stmt_type`.

**Рекомендация.** Миграции схемы выполнять последовательно, не в несколько
потоков. DDL в OB дорог.

## 3.2 Упор в коммит

**Симптомы.** AAS 13.93 при `max_cpu = 2`, ON CPU 10.7%.
`tx commiting wait` — 87.67% времени, 49 мс на ожидание.
Фаза `IN_COMMITTING` — 10 226 сэмплов из 11 658.

**Что делала нагрузка.** `INSERT` / `SELECT` / `UPDATE` / `DELETE`
по одной строке в 50 потоков.

**Диагноз.** `palf write` 30–36 мс. Две трети времени коммита — запись
журнала на диск. Сеть почти не участвует. Узкое место — дисковая подсистема.

**Рекомендация.** Разбираться с диском под clog. Увеличение параллелизма
или оптимизация запросов не дадут ничего.

## 3.3 Нехватка рабочих потоков (кластер 4.5, три зоны)

**Симптомы.** AAS 58.17 при 800 сессиях, ON CPU 0.6%.
`wait in request queue` (CONFIGURATION) — абсолютный лидер, ~30 000 сэмплов.
В аудите: `queue_ms` 58–133 при `exec_ms` 0.98–4.7.

**Диагноз.** Запрос выполняется 1–5 мс, а в очереди за рабочим потоком стоит
58–133 мс — 95% времени отклика съедает очередь. Диски здоровы (`palf write`
1.5–2.2 мс), коммит быстрый (7–9 мс), блокировок нет. Классический thread
starvation: 800 конкурентных сессий против маленького `max_cpu` тенанта.

**Рекомендация.** Лечится только ресурсами тенанта (`max_cpu` юнита,
при необходимости `cpu_quota_concurrency`) — оптимизация запросов и дисков
ничего не даст.

**Две побочные находки.** Генератор слал `SET SESSION ob_query_timeout`
перед каждой пачкой — 54 529 выполнений по 133 мс средних, каждый SET
проходит ту же очередь: заметный самоналог теста. И перекос по узлам
воспроизвёлся на трёх зонах: 51 299 FG-сэмплов на одном узле против 184
на другом — лидер непартиционированной таблицы один, остальной кластер
работает репликами.

Итого один и тот же набор запросов различил три разных диагноза:
медленный диск под clog (3.2), DDL-шторм (3.1) и нехватку потоков (3.3) —
при внешне одинаковом симптоме «AAS много выше max_cpu, CPU свободен».

## 3.4 Побочная находка: ODBC-escape в грамматике

В конфиге теста были шаблоны `{i+5}`, `{i+4}`, `{i-2}`, которые генератор
не подставляет (документирован только `{i}`) и которые уходят в базу буквально.
Синтаксической ошибки при этом не возникает:

```sql
SELECT {i+5} AS a, {i+4} AS b, {i-2} AS c;
```
```
| a | b | c  |
| 5 | 4 | -2 |
```

MySQL-грамматика содержит правило ODBC-escape вида `'{' ident expr '}'`,
которое отбрасывает идентификатор и возвращает выражение. OceanBase его
унаследовал. В результате четыре запроса из семи били не по своей строке,
а по трём фиксированным — что и создало искусственную конкуренцию за строки,
наблюдавшуюся в 22:25.

Практический вывод шире теста: **опечатка в шаблонизаторе может молча
превратиться в валидный SQL** и исказить картину нагрузки, не дав ни одной
ошибки в `ret_code`.

---

# Часть 4. Ограничения и особенности, найденные на стенде

## 4.1 Штатный ash_report ненадёжен под нагрузкой

| Файл | Размер | Секций | Завершён |
|---|---|---|---|
| холостой ход | 134 КБ | 19 | да |
| под нагрузкой | 64 КБ | **4** | **нет** |

Отчёт под нагрузкой обрывается посреди секции Top Groups, без закрывающего
`</html>`. Из девятнадцати секций сгенерированы четыре: до Top Sessions,
Top SQL и Top Blocking Sessions дело не дошло. Код возврата при этом нулевой —
сбой молчаливый.

Вывод: `ash_report` даёт полную картину на спокойной системе и разваливается
ровно тогда, когда нужен. Основным инструментом инцидент-разбора он быть
не может.

## 4.2 Ошибка 1222 при пересечении границы источника

```
ERROR 1222 (21000): The used SELECT statements have a different number of columns
```

Возникает, когда окно отчёта частично лежит в живом буфере, частично в WR.
Отчёт делает между ними UNION, а схемы источников различаются (72 колонки
против 67). Воспроизводится тем вернее, чем выше нагрузка: буфер сжимается,
и заданное окно перестаёт в него помещаться.

Шапка отчёта показывает, какой источник выбран:

```
Ash Data Source: oceanbase.GV$ACTIVE_SESSION_HISTORY
Wr  Data Source: oceanbase.DBA_WR_ACTIVE_SESSION_HISTORY
```

Обходной путь: перед запуском смотреть D1 и не задавать окно, пересекающее
нижнюю границу живого буфера.

## 4.3 Ошибка 4013 при нехватке памяти тенанта

```
ERROR 4013 (HY001): No memory or reach tenant memory limit
```

Отчёт целиком собирается в памяти PL. При `sys_unit_config` в 1 ГБ
и текущем потреблении 617–682 МБ его не хватает. Признаки того, что дело
именно в памяти, а не в объёме данных: размер окна не влияет (15, 5 и 3 минуты
падают одинаково), `'text'` и `'html'` падают одинаково, фильтр по тенанту
не помогает — при том что полный скан всей ASH-таблицы на 533 868 строк
в той же сессии отрабатывает за 2 секунды.

Лечится онлайн:

```sql
ALTER RESOURCE UNIT sys_unit_config MEMORY_SIZE = '2G';
```

Нижняя граница юнита — 1 ГБ (`__min_full_resource_pool_memory`), уменьшить
чужой юнит ниже неё нельзя. При освобождении памяти удалением тенанта
обязательна последовательность: `DROP TENANT … FORCE` → `DROP RESOURCE POOL`
→ `DROP RESOURCE UNIT`. Без `FORCE` тенант уходит в отложенное удаление
и память не освобождается; без удаления пула и юнита `mem_assigned`
не изменится.

## 4.4 Тексты SQL в WR отсутствуют

`CDB_WR_SQLTEXT` пуст даже после ручного `CREATE_SNAPSHOT()`. За семь суток
доступна структура нагрузки, но не тексты запросов — только `sql_id`.

## 4.5 audit_log_* в CE нефункциональны

Параметры видны, но реализация закрыта в EE. Единственный работающий путь
аудита в CE — `GV$OB_SQL_AUDIT` плюс разбор логов.

---

# Часть 5. Правила и грабли

| Правило | Что будет без него |
|---|---|
| `session_type = 'FOREGROUND'` | 59% сэмплов — вечно ждущие фоновые потоки, полезное на шестом месте |
| `COALESCE(wait_class,'') <> 'IDLE'` | `wait_class` бывает NULL, `NULL <> 'IDLE'` = NULL, строки режутся молча |
| `COALESCE(blocking_session_id,0) > 0` | `SUM(bsid > 0)` возвращает NULL вместо числа |
| `NULLIF(event,'')` | у строк ON CPU событие — пустая строка, а не NULL |
| `MAX(pc.executions)`, не `SUM` | джойн размножает строки, SUM даёт миллиарды вместо тысяч |
| `MAX(stmt_type)`, не в `GROUP BY` | один `sql_id` расщепляется на несколько строк |
| `SET group_concat_max_len` | `GROUP_CONCAT` молча усекается, десятки предупреждений |
| фильтр `FOREGROUND` в D7 | результат забит внутренними сессиями с id > 4.6e18 |
| `V$`, а не `GV$` в агенте | `GV$` ходит по RPC и зависает при потере узла |
| `TIME_TO_USEC()` для `request_time` | микросекунды, фильтр по timestamp не сработает |
| `time_after_recv`, не `wait_time` | колонки `wait_time` в `lock_wait_stat` не существует |
| префиксы у алиасов `optimize` / `execute` | зарезервированные слова, синтаксическая ошибка |

Дополнительно: `V$ACTIVE_SESSION_HISTORY` из sys показывает **все тенанты**
своего узла — проверено сравнением с `GV$`. Отдельного подключения к каждому
тенанту не требуется.

---

# Приложение A. Словарь stmt_type

Снят эмпирически связкой ASH ↔ sql_audit по `sql_id`.

| Код в ASH | Значение |
|---|---|
| 1 | SELECT |
| 2 | INSERT |
| 4 | DELETE |
| 5 | UPDATE |
| 20 | CREATE TABLE |
| 21 | DROP TABLE |
| 66 | VARIABLE_SET |
| 70 | END_TRANS |
| 144 | CALL_PROCEDURE |

Проверен на DML-прогоне: 5=UPDATE (3539 совпадений), 2=INSERT, 1=SELECT,
4=DELETE — стабильно.

```sql
SELECT ash.stmt_type AS ash_code, aud.stmt_type AS name, COUNT(*) AS cnt
FROM oceanbase.V$ACTIVE_SESSION_HISTORY ash
JOIN (SELECT DISTINCT tenant_id, sql_id, stmt_type
        FROM oceanbase.GV$OB_SQL_AUDIT
       WHERE request_time > TIME_TO_USEC(NOW() - INTERVAL 5 MINUTE)) aud
  ON aud.tenant_id = ash.con_id AND aud.sql_id = ash.sql_id
WHERE ash.sample_time > NOW() - INTERVAL 5 MINUTE
GROUP BY 1,2 ORDER BY 3 DESC;
```

Единичные несовпадения (код 4 с именем SELECT) — переиспользование `sql_id`
между тенантами, на статистику не влияют.

---

# Приложение B. Справочник событий

| Событие | Класс | Значение | Что проверять |
|---|---|---|---|
| ON CPU | — | считает | планы, full scan, `max_cpu` |
| `tx commiting wait` | COMMIT | ждёт подтверждения транзакции | разложить через `palf write` |
| `sync tx commiting wait` | COMMIT | ждёт удалённую реплику | сеть между зонами |
| `palf write` | SYSTEM_IO | запись журнала на диск | **> 20 мс — диск непригоден** |
| `row lock wait` | APPLICATION | конкуренция за строку | D7, D8 |
| `sleep: lock for read need wait for concurrency control` | CONCURRENCY | читатель ждёт незакоммиченную версию строки | тот же упор в коммит, вид со стороны чтения |
| `sync rpc` | NETWORK | ждёт другой узел | распределённые операции, DDL |
| `sleep: wait refresh schema` | ADMINISTRATIVE | ждёт версию схемы | параллельный DDL |
| `wait in request queue` | CONFIGURATION | нет рабочего потока | число потоков, `max_cpu` |
| `db file compact read` | SYSTEM_IO | чтение с диска | индексы, кэш, диск |
| `db file data read` | USER_IO | чтение данных | индексы |
| `latch: *` | CONCURRENCY | внутренняя конкуренция | обычно вторичный симптом |
| `exec inner sql wait` | OTHER | ждёт внутренний SQL | DDL, работа со словарём |
| `remote log writer cond wait` | CONCURRENCY | **фоновый шум** | игнорировать |
| `sleep wait` | IDLE | **фоновый шум** | игнорировать |
| `mysql response wait client` | IDLE | ждёт клиента | не проблема БД |

---

# Приложение C. Раскладка по агенту RIF

| ID | Запрос | Область | Таймаут |
|---|---|---|---|
| `ash.depth` | D1 | local | 5 с |
| `ash.summary` | D2 | local | 5 с |
| `ash.timeline` | D3 | local | 10 с |
| `ash.top_events` | D4 | local | 10 с |
| `ash.top_sql` | D5 | local | 15 с |
| `ash.sql_events` | D6 | local | 10 с |
| `ash.blocking_history` | D7 | local | 15 с |
| `lock.waits_now` | D8 | local | 5 с |
| `ash.phases` | D9 | local | 5 с |
| `ash.node_skew` | D10 | сервер (слияние) | — |
| `audit.top_sql` | D11 | local | 20 с |
| `wr.timeline` | D12 | **leader** | 60 с |

Все локальные запросы — через `127.0.0.1:2881`, вьюхи `V$`. Ошибка и таймаут
возвращаются внутри конверта ответа, а не как отказ: неответивший узел
сам по себе является диагностическим сигналом.

**Порядок разбора инцидента:**

```
D1  есть ли данные за нужное окно
D2  насколько всё плохо, и упёрлись ли в CPU
D3  когда началось
D4  чего именно ждём
D5, D6  какие запросы в этом участвуют
D7, D8  если есть блокировки
D10 перекос по узлам, разложение коммита
D11 точные тайминги, полные тексты, IP — пока живы
```

---

# Приложение D. Закрытые вопросы (история проверок)

Все вопросы, открытые по ходу исследования, закрыты экспериментально:

**`EVENT_NO` vs `EVENT_ID`** — разные пространства номеров; джойн WR
к `DBA_WR_EVENT_NAME` даёт ложные имена событий. Решение в разделе 2.12.

**«Смещение» WR** — оказалось артефактом неверного джойна. С правильным
маппингом WR показывает тот же профиль событий, что и живой ASH.

**`DBA_` vs `CDB_`** — из sys `DBA_WR_*` видит только sys; для кластера
использовать `CDB_WR_*`.

**`user_client_ip` в sql_audit** — существует и содержит реальный IP клиента
за OBProxy.

**Семантика `weight`** — двунаправленная поправка (0.1 при прореживании,
>1 при компенсации коротких событий); использовать `SUM(weight)` на 4.4.x.

**Кросс-версионная проверка на 4.5.0.0** — пакет и все 12 запросов работают;
отличия: `event_no` частично сдвинуты (словарь снимать динамически),
колонка `WEIGHT` удалена из WR (использовать `COUNT(*)`), в WR добавлены
`BLOCKING_SESSION_ID` / `TX_ID` / `TM_DELTA_*`, ожидающая сторона в D8
разрешается корректно.
