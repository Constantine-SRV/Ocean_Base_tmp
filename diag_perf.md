## диагностика.

```bash
  -- 1. Активные сессии и что они делают
SELECT * FROM oceanbase.GV$OB_PROCESSLIST WHERE db = 'benchbasedb';

-- 2. Блокировки
SELECT * FROM oceanbase.GV$OB_LOCKS;

-- 3. Активные транзакции
SELECT * FROM oceanbase.GV$OB_TRANSACTION_PARTICIPANTS;

-- 4. Ожидания (что тормозит)
SELECT * FROM oceanbase.GV$SESSION_WAIT;

-- 5. Статус мемтейблов (могут быть переполнены)
SELECT * FROM oceanbase.GV$OB_MEMSTORE;

-- 6. SQL в процессе выполнения
SELECT svr_ip, sql_id, query_sql, elapsed_time/1000000 as elapsed_sec 
FROM oceanbase.GV$OB_SQL_AUDIT 
WHERE tenant_id = 1 
  AND request_time > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
ORDER BY request_time DESC 
LIMIT 20;

```

```
ALTER SYSTEM SET writing_throttling_trigger_percentage = 100;
```

```
-- Это OceanBase-специфичная команда, которая сбросит данные 
ALTER SYSTEM MAJOR FREEZE;
```
### процедура обновляет все таблицы в базе очень удобно
выполнить до и после запрос для понимания разницы
```
SELECT     t.table_name,     COALESCE(s.row_count, 0) AS row_count,     CONCAT(ROUND(s.data_size / 1024 / 1024, 2), ' MB') AS total_size FROM information_schema.tables AS t LEFT JOIN (     SELECT         table_name,         SUM(data_length + index_length) AS data_size,         SUM(table_rows) AS row_count     FROM information_schema.tables     WHERE table_schema = DATABASE()     GROUP
BY table_name ) AS s ON s.table_name = t.table_name WHERE t.table_schema = DATABASE() ORDER BY t.table_name;

```

```
DELIMITER $$

CREATE PROCEDURE analyze_all_tables()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE tbl_name VARCHAR(255);
    DECLARE cur CURSOR FOR 
        SELECT table_name 
        FROM information_schema.tables 
        WHERE table_schema = DATABASE();
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur;
    read_loop: LOOP
        FETCH cur INTO tbl_name;
        IF done THEN
            LEAVE read_loop;
        END IF;
        SET @sql = CONCAT('ANALYZE TABLE ', tbl_name);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END LOOP;
    CLOSE cur;
END$$

DELIMITER ;

-- Выполнить
CALL analyze_all_tables();

```

