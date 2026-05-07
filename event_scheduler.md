# event_scheduler 

```sql


USE admintools;

DROP TABLE IF EXISTS log_disk_usage_history;

CREATE TABLE log_disk_usage_history (
    id                BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    tenant_id         INT             NOT NULL,
    svr_ip            VARCHAR(46)     NOT NULL,
    allocated_GB      DECIMAL(10,1),
    active_GB         DECIMAL(10,1),
    recyclable_GB     DECIMAL(10,1),
    effective_free_GB DECIMAL(10,1),
    kill_limit_GB     DECIMAL(10,1),
    headroom_GB       DECIMAL(10,1),
    pct_of_limit      DECIMAL(10,1),
    checked_at        DATETIME        NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id)
) COMMENT = 'Snapshot log disk usage per tenant, collected every minute';




DROP PROCEDURE IF EXISTS collect_log_disk_usage;

DELIMITER //

CREATE PROCEDURE collect_log_disk_usage()
BEGIN
    INSERT INTO admintools.log_disk_usage_history
        (tenant_id, svr_ip, allocated_GB, active_GB, recyclable_GB,
         effective_free_GB, kill_limit_GB, headroom_GB, pct_of_limit, checked_at)
    SELECT
        u.tenant_id,
        u.svr_ip,
        ROUND(u.log_disk_size / 1024 / 1024 / 1024, 1),
        ROUND(SUM(ls.END_LSN  - ls.BASE_LSN)  / 1024 / 1024 / 1024, 1),
        ROUND(SUM(ls.BASE_LSN - ls.BEGIN_LSN) / 1024 / 1024 / 1024, 1),
        ROUND((u.log_disk_size
               - SUM(ls.END_LSN - ls.BASE_LSN)) / 1024 / 1024 / 1024, 1),
        ROUND(u.log_disk_size * 0.95 / 1024 / 1024 / 1024, 1),
        ROUND((u.log_disk_size * 0.95
               - SUM(ls.END_LSN - ls.BASE_LSN)) / 1024 / 1024 / 1024, 1),
        ROUND(SUM(ls.END_LSN  - ls.BASE_LSN)
              / (u.log_disk_size * 0.95) * 100, 1),
        NOW()
    FROM oceanbase.__all_virtual_unit u
    JOIN oceanbase.GV$OB_LOG_STAT ls
        ON  u.tenant_id = ls.TENANT_ID
        AND u.svr_ip    = ls.SVR_IP
    WHERE u.svr_ip = (SELECT SVR_IP FROM oceanbase.GV$OB_LOG_STAT LIMIT 1)
    GROUP BY u.tenant_id, u.svr_ip, u.log_disk_size;

    DELETE FROM admintools.log_disk_usage_history
    WHERE id <= (
        SELECT cutoff FROM (
            SELECT MAX(id) - 10000 AS cutoff
            FROM admintools.log_disk_usage_history
        ) t
    );
END//

DELIMITER ;




[rlm_sys|admintools] 17:11:18> SELECT * FROM admintools.log_disk_usage_history ORDER BY id DESC LIMIT 10;
+----+-----------+----------------+--------------+-----------+---------------+-------------------+---------------+-------------+--------------+---------------------+
| id | tenant_id | svr_ip         | allocated_GB | active_GB | recyclable_GB | effective_free_GB | kill_limit_GB | headroom_GB | pct_of_limit | checked_at          |
+----+-----------+----------------+--------------+-----------+---------------+-------------------+---------------+-------------+--------------+---------------------+
|  3 |      1002 | 192.168.5.205 |         18.0 |       0.2 |          14.0 |              17.8 |          17.1 |        16.9 |          1.2 | 2026-05-06 20:11:13 |
|  2 |      1001 | 192.168.5.205 |          2.0 |       0.2 |           1.4 |               1.8 |           1.9 |         1.7 |          8.4 | 2026-05-06 20:11:13 |
|  1 |         1 | 192.168.5.205 |          2.0 |       0.3 |           1.2 |               1.7 |           1.9 |         1.6 |         17.4 | 2026-05-06 20:11:13 |
+----+-----------+----------------+--------------+-----------+---------------+-------------------+---------------+-------------+--------------+---------------------+
3 rows in set (0.00 sec)


SET GLOBAL event_scheduler = ON;

DROP EVENT IF EXISTS evt_collect_log_disk_usage;

CREATE EVENT evt_collect_log_disk_usage
    ON SCHEDULE EVERY 1 MINUTE
    STARTS NOW()
    ON COMPLETION PRESERVE
    ENABLE
    DO
        CALL admintools.collect_log_disk_usage();



[rlm_sys|admintools] 17:11:09> SELECT event_name, status, last_executed, interval_value, interval_field, event_definition FROM information_schema.events;
+----------------------------+---------+---------------------+----------------+----------------+-------------------------------------------+
| event_name                 | status  | last_executed       | interval_value | interval_field | event_definition                          |
+----------------------------+---------+---------------------+----------------+----------------+-------------------------------------------+
| evt_collect_log_disk_usage | ENABLED | 2026-05-06 20:11:13 | 1              | MINUTE         | CALL admintools.collect_log_disk_usage(); |
+----------------------------+---------+---------------------+----------------+----------------+-------------------------------------------+
1 row in set (0.00 sec)





```
