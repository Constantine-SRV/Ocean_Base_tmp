# OceanBase Мониторинг и Анализ Размеров

Полное руководство по мониторингу дискового пространства и анализу размеров таблиц/индексов в OceanBase CE.

---

## 📊 1. Мониторинг Дискового Пространства

### 1.1 Общее использование диска OceanBase

```sql
-- Показывает сколько места выделено OceanBase и сколько используется
SELECT 
    svr_ip,
    ROUND(DATA_DISK_CAPACITY / 1024 / 1024 / 1024, 2) as capacity_gb,
    ROUND(DATA_DISK_IN_USE / 1024 / 1024 / 1024, 2) as in_use_gb,
    ROUND((DATA_DISK_CAPACITY - DATA_DISK_IN_USE) / 1024 / 1024 / 1024, 2) as free_gb,
    ROUND(DATA_DISK_IN_USE / DATA_DISK_CAPACITY * 100, 2) as used_pct
FROM oceanbase.GV$OB_SERVERS;
```

**Результат:**
- `capacity_gb` - Максимум выделенный OceanBase для данных
- `in_use_gb` - Реально используется прямо сейчас
- `free_gb` - Свободно для роста данных
- `used_pct` - Процент заполнения (⚠️ > 80% = мало места)

### 1.2 Размер логов (clog)

```sql
-- Проверка использования места под логи транзакций
SELECT 
    svr_ip,
    ROUND(LOG_DISK_CAPACITY / 1024 / 1024 / 1024, 2) as log_capacity_gb,
    ROUND(LOG_DISK_IN_USE / 1024 / 1024 / 1024, 2) as log_used_gb,
    ROUND((LOG_DISK_CAPACITY - LOG_DISK_IN_USE) / 1024 / 1024 / 1024, 2) as log_free_gb,
    ROUND(LOG_DISK_IN_USE / LOG_DISK_CAPACITY * 100, 2) as log_used_pct
FROM oceanbase.GV$OB_SERVERS;
```

### 1.3 Настройки дискового пространства

```sql
-- Посмотреть текущие лимиты
SHOW PARAMETERS LIKE 'datafile_size';  -- Максимум для данных
SHOW PARAMETERS LIKE 'log_disk_size';  -- Максимум для логов
```

**Как увеличить:**
```sql
-- Увеличить место под данные
ALTER SYSTEM SET datafile_size='40G';

-- Увеличить место под логи
ALTER SYSTEM SET log_disk_size='30G';
```

### 1.4 Размер по базам данных

```sql
-- Сколько места занимает каждая база
SELECT 
    d.database_name,
    COUNT(DISTINCT t.table_id) as tables_count,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024 / 1024, 2) as data_gb,
    ROUND(SUM(r.REQUIRED_SIZE) / 1024 / 1024 / 1024, 2) as required_gb
FROM oceanbase.__all_database d
JOIN oceanbase.__all_table t ON d.database_id = t.database_id
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
GROUP BY d.database_name
ORDER BY data_gb DESC;
```

---

## 📈 2. Анализ Размеров Таблиц

### 2.1 Размер конкретной таблицы (РЕКОМЕНДУЕТСЯ)

```sql
-- ✅ САМЫЙ ТОЧНЫЙ МЕТОД - реальный размер на диске
SELECT
    t.table_name,
    CASE t.table_type
        WHEN 3 THEN 'TABLE'
        WHEN 5 THEN 'INDEX'
    END as object_type,
    COUNT(DISTINCT r.TABLET_ID) as tablet_count,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024, 2) as data_mb,
    ROUND(SUM(r.REQUIRED_SIZE) / 1024 / 1024, 2) as required_mb
FROM oceanbase.__all_table t
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
WHERE t.database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'benchbasedb'
)
GROUP BY t.table_name, t.table_type
ORDER BY data_mb DESC;
```

**Почему этот метод лучший:**
- ✅ Всегда актуальные данные
- ✅ Учитывает реальное сжатие zstd/lz4
- ✅ Учитывает удаленные строки после major freeze
- ✅ Показывает фактический размер на диске

### 2.2 Размер через статистику (оценка)

```sql
-- ⚠️ ПРИБЛИЗИТЕЛЬНАЯ ОЦЕНКА через __all_table_stat
SELECT
    t.table_name,
    d.database_name,
    ts.table_id,
    SUM(ts.row_cnt) AS total_rows,
    ROUND(AVG(ts.avg_row_len), 1) AS avg_row_len,
    SUM(ts.macro_blk_cnt) AS macro_blocks,
    ROUND(SUM(ts.row_cnt * ts.avg_row_len) / 1024 / 1024, 2) AS approx_data_mb,
    ROUND(SUM(ts.macro_blk_cnt) * 2, 2) AS approx_disk_mb,
    MAX(ts.last_analyzed) AS last_analyzed
FROM oceanbase.__all_table_stat ts
JOIN oceanbase.__all_table t ON ts.table_id = t.table_id
JOIN oceanbase.__all_database d ON t.database_id = d.database_id
WHERE d.database_name = 'testdb'
  AND t.table_name = 'taxi_trips1'
GROUP BY t.table_name, d.database_name, ts.table_id;
```

**Почему этот метод неточный:**
- ❌ Данные собираются только при ANALYZE TABLE
- ❌ `macro_blk_cnt * 2 MB` - грубая оценка (блоки могут быть неполными)
- ❌ Не учитывает реальное сжатие
- ✅ Но полезен для оценки количества строк и планирования запросов

### 2.3 Сравнение методов

| Метод | Источник | Точность | Когда использовать |
|-------|----------|----------|-------------------|
| **DBA_OB_TABLET_REPLICAS** | Реальные данные | ✅ 100% | **Всегда для размеров!** |
| __all_table_stat | Статистика ANALYZE | ⚠️ 50-200% | Для оценки строк, планирования |
| information_schema.TABLES | Копия статистики | ⚠️ 50-200% | MySQL совместимость |

**Пример разницы на реальных данных:**
```
Таблица taxi_trips1 (607M строк):
- DBA_OB_TABLET_REPLICAS: 10,428 MB  ← Реальный размер ✅
- __all_table_stat:       16,434 MB  ← Оценка (на 58% больше!) ⚠️
```

---

## 🔍 3. Анализ Таблиц с Индексами

### 3.1 Размеры таблицы + все индексы (УНИВЕРСАЛЬНЫЙ)

```sql
-- ✅ Работает для партиционированных И непартиционированных таблиц
-- ✅ Показывает РЕАЛЬНЫЕ размеры на диске
SELECT 
    COALESCE(
        CASE t.table_type 
            WHEN 3 THEN 'TABLE (PRIMARY KEY)' 
            WHEN 5 THEN CONCAT('INDEX: ', SUBSTRING_INDEX(t.table_name, '_', -1))
        END,
        'TOTAL'
    ) as object_type,
    COUNT(DISTINCT r.TABLET_ID) as tablets,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024, 2) as real_disk_mb,
    ROUND(SUM(r.REQUIRED_SIZE) / 1024 / 1024, 2) as required_mb
FROM oceanbase.__all_table t
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
WHERE t.database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'testdb'
)
AND (t.table_name = 'taxi_trips1' OR t.data_table_id = (
    SELECT table_id FROM oceanbase.__all_table t2
    JOIN oceanbase.__all_database d ON t2.database_id = d.database_id
    WHERE d.database_name = 'testdb' AND t2.table_name = 'taxi_trips1'
))
GROUP BY t.table_type, t.table_name WITH ROLLUP;
```

**Пример результата:**
```
+---------------------+---------+--------------+-------------+
| object_type         | tablets | real_disk_mb | required_mb |
+---------------------+---------+--------------+-------------+
| TABLE (PRIMARY KEY) |       1 |     10428.83 |    10428.83 |
| INDEX: location     |       1 |      1388.41 |     1388.41 |
| INDEX: location     |       1 |      1417.33 |     1417.33 |
| INDEX: code         |       1 |      1012.48 |     1012.48 |
| TOTAL               |       4 |     14247.04 |    14247.04 |
+---------------------+---------+--------------+-------------+

Итого: Таблица 10.4 GB, Индексы 3.8 GB = 14.2 GB
```

### 3.2 Список всех индексов таблицы

```sql
-- Простой список индексов
SHOW INDEX FROM taxi_trips1;

-- Или через системные таблицы с table_id
SELECT 
    table_id,
    table_name,
    CASE table_type 
        WHEN 3 THEN 'TABLE' 
        WHEN 5 THEN 'INDEX' 
    END as type,
    data_table_id
FROM oceanbase.__all_table
WHERE database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'testdb'
)
ORDER BY table_type, table_name;
```

### 3.3 Анализ эффективности индексов

```sql
-- Сколько байт на строку занимает каждый объект
SELECT 
    t.table_name,
    CASE t.table_type WHEN 3 THEN 'TABLE' WHEN 5 THEN 'INDEX' END as type,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024, 2) as disk_mb,
    ROUND(SUM(r.DATA_SIZE) / 607000000, 2) as bytes_per_row  -- Делим на кол-во строк
FROM oceanbase.__all_table t
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
WHERE t.database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'testdb'
)
AND (t.table_name = 'taxi_trips1' OR t.data_table_id = (
    SELECT table_id FROM oceanbase.__all_table t2
    JOIN oceanbase.__all_database d ON t2.database_id = d.database_id
    WHERE d.database_name = 'testdb' AND t2.table_name = 'taxi_trips1'
))
GROUP BY t.table_name, t.table_id, t.table_type
ORDER BY t.table_type;
```

**Как понимать bytes_per_row:**
- **12-18 байт** = Отличное сжатие для индекса ✅
- **> 20 байт** = Проверьте структуру индекса (большой PRIMARY KEY?)
- **< 10 байт** = Супер сжатие (маленькие значения + zstd) ⭐

---

## 🛠️ 4. Сбор Статистики и Оптимизация

### 4.1 Сбор статистики

```sql
-- Обязательно после создания индексов или больших изменений!
ANALYZE TABLE taxi_trips1;

-- Проверить когда последний раз собиралась
SELECT 
    table_name,
    last_analyzed
FROM oceanbase.__all_table_stat ts
JOIN oceanbase.__all_table t ON ts.table_id = t.table_id
JOIN oceanbase.__all_database d ON t.database_id = d.database_id
WHERE d.database_name = 'testdb'
  AND t.table_name = 'taxi_trips1'
LIMIT 1;
```

### 4.2 Major Freeze (очистка старых версий)

```sql
-- Запустить major freeze (сжатие и очистка)
ALTER SYSTEM MAJOR FREEZE;

-- Проверить статус
SELECT 
    TENANT_ID,
    STATUS,
    START_TIME,
    TIMEDIFF(NOW(), START_TIME) as duration,
    IS_ERROR
FROM oceanbase.CDB_OB_MAJOR_COMPACTION;

-- Статусы:
-- IDLE       - Завершен, можно работать ✅
-- COMPACTING - Идет процесс, подождите ⏳
```

### 4.3 Проверка настроек сжатия

```sql
-- Какое сжатие используется для таблицы и индексов
SELECT 
    table_name,
    CASE table_type WHEN 3 THEN 'TABLE' WHEN 5 THEN 'INDEX' END as type,
    compress_func_name,
    row_store_type
FROM oceanbase.__all_table
WHERE database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'testdb'
)
AND (table_name = 'taxi_trips1' OR data_table_id IN (
    SELECT table_id FROM oceanbase.__all_table 
    WHERE table_name = 'taxi_trips1'
))
ORDER BY table_type;
```

**Типы сжатия:**
- `zstd_1.3.8` - Лучшее сжатие (медленнее) ⭐
- `lz4_1.0` - Быстрое сжатие (больше места)
- `none` - Без сжатия (не используйте!)

---

## 📋 5. Быстрая Диагностика

### 5.1 Чек-лист проблем с местом

```sql
-- 1. Сколько свободно?
SELECT 
    ROUND(DATA_DISK_IN_USE / DATA_DISK_CAPACITY * 100, 2) as used_pct
FROM oceanbase.GV$OB_SERVERS;

-- Если > 90% → ПРОБЛЕМА! ⚠️

-- 2. Идет ли major freeze?
SELECT STATUS FROM oceanbase.CDB_OB_MAJOR_COMPACTION;

-- Если COMPACTING → подождите завершения

-- 3. Кто занимает место?
SELECT 
    d.database_name,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024 / 1024, 2) as data_gb
FROM oceanbase.__all_database d
JOIN oceanbase.__all_table t ON d.database_id = t.database_id
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
GROUP BY d.database_name
ORDER BY data_gb DESC;

-- 4. Топ больших таблиц
SELECT 
    t.table_name,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024 / 1024, 2) as data_gb
FROM oceanbase.__all_table t
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
WHERE t.database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'testdb'
)
GROUP BY t.table_name, t.table_id
ORDER BY data_gb DESC
LIMIT 10;
```

### 5.2 Освобождение места

**Если нет места для индексов:**

1. **Дождитесь major freeze:**
```sql
ALTER SYSTEM MAJOR FREEZE;
-- Ждите пока STATUS != 'COMPACTING'
```

2. **Увеличьте datafile_size:**
```sql
ALTER SYSTEM SET datafile_size='50G';  -- Увеличьте под ваши нужды
```

3. **Удалите ненужные индексы:**
```sql
DROP INDEX idx_unused ON table_name;
```

4. **Расширьте физический диск сервера** (если нужно)

---

## 💡 6. Важные Замечания

### 6.1 Про partition_id = -1

**Старые запросы с фильтром `partition_id = -1` НЕ РАБОТАЮТ для непартиционированных таблиц!**

❌ **Неправильно (не работает для всех таблиц):**
```sql
WHERE ts.partition_id = -1  -- Есть только в партиционированных таблицах!
```

✅ **Правильно (универсально):**
```sql
-- Используйте GROUP BY без фильтра partition_id
GROUP BY t.table_name, d.database_name, ts.table_id;
```

### 6.2 Размер PRIMARY KEY влияет на индексы

**Каждый индекс хранит PRIMARY KEY!**

```
Если PRIMARY KEY = 16 байт (pickup_datetime + trip_id):
- Индекс на колонке 2 байта → фактически 18 байт на строку
- Индекс на колонке 1 байт → фактически 17 байт на строку

Оптимизация: уменьшите PRIMARY KEY до INT (4 байта) вместо BIGINT (8 байт)
→ Экономия ~50% на индексах!
```

### 6.3 Создание индексов требует места

**При создании индекса нужно ~2x места от финального размера!**

Пример:
```
Таблица: 10 GB
Индекс финальный: 4 GB
При создании временно нужно: 10 GB + 4 GB + 4 GB (temp) = 18 GB
```

**Решение:** Создавайте индексы по одному, а не все сразу.

---

## 🎯 7. Итоговая Шпаргалка

### Для мониторинга места:
```sql
-- Быстрая проверка
SELECT 
    ROUND(DATA_DISK_IN_USE / DATA_DISK_CAPACITY * 100, 2) as used_pct
FROM oceanbase.GV$OB_SERVERS;
```

### Для анализа размера таблицы:
```sql
-- Используйте ТОЛЬКО этот!
SELECT 
    t.table_name,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024, 2) as real_mb
FROM oceanbase.__all_table t
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
WHERE t.database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'YOUR_DB'
)
AND t.table_name = 'YOUR_TABLE'
GROUP BY t.table_name;
```

### Для анализа таблицы + индексов:
```sql
-- С итогами по типам
SELECT 
    COALESCE(
        CASE t.table_type WHEN 3 THEN 'TABLE' WHEN 5 THEN 'INDEX' END,
        'TOTAL'
    ) as type,
    ROUND(SUM(r.DATA_SIZE) / 1024 / 1024, 2) as mb
FROM oceanbase.__all_table t
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID
WHERE t.database_id = (
    SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'YOUR_DB'
)
AND (t.table_name = 'YOUR_TABLE' OR t.data_table_id = (
    SELECT table_id FROM oceanbase.__all_table t2
    JOIN oceanbase.__all_database d ON t2.database_id = d.database_id
    WHERE d.database_name = 'YOUR_DB' AND t2.table_name = 'YOUR_TABLE'
))
GROUP BY t.table_type WITH ROLLUP;
```

---

## 📚 Дополнительные Ресурсы

- Официальная документация: https://www.oceanbase.com/docs
- GitHub: https://github.com/oceanbase/oceanbase
- Community: https://ask.oceanbase.com

---

**Версия документа:** 1.0  
**Дата:** 2025-11-04  
**OceanBase версия:** 4.3.5.4 CE  
**Автор:** Based on production experience

---

## ✨ Заключение

**Ключевые правила:**

1. ✅ **ВСЕГДА** используйте `DBA_OB_TABLET_REPLICAS` для реальных размеров
2. ⚠️ `__all_table_stat` - только для оценок и планирования
3. 🔄 Запускайте `ANALYZE TABLE` после изменений
4. 📊 Мониторьте `used_pct < 80%` для комфортной работы
5. 🛠️ Используйте `ALTER SYSTEM MAJOR FREEZE` для очистки

**Удачи в работе с OceanBase!** 🚀
