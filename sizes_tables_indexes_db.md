
```sql
[tpchdb] 10:20:35> SELECT
    CASE
        WHEN TABLE_NAME LIKE '__idx_%' THEN 'INDEX'
        ELSE 'TABLE'
    END AS object_type,
    TABLE_NAME,
    ROUND(OCCUPY_SIZE / 1024 / 1024 / 1024, 3) AS disk_gb,
    ROUND(REQUIRED_SIZE / 1024 / 1024 / 1024, 3) AS allocated_gb
FROM oceanbase.DBA_OB_TABLE_SPACE_USAGE
WHERE DATABASE_NAME = DATABASE()
ORDER BY OCCUPY_SIZE DESC;
+-------------+-----------------------+---------+--------------+
| object_type | TABLE_NAME            | disk_gb | allocated_gb |
+-------------+-----------------------+---------+--------------+
| TABLE       | lineitem              |   8.989 |        9.592 |
| TABLE       | orders                |   2.274 |        2.356 |
| INDEX       | __idx_501523_l_pk_sk  |   2.198 |        2.250 |
| INDEX       | __idx_501523_l_sk_pk  |   2.035 |        2.074 |
| TABLE       | partsupp              |   1.639 |        1.688 |
| INDEX       | __idx_501523_l_pk     |   1.522 |        1.557 |
| INDEX       | __idx_501523_l_sk     |   1.184 |        1.231 |
| INDEX       | __idx_501523_l_rd     |   1.117 |        1.160 |
| INDEX       | __idx_501523_l_sd     |   1.117 |        1.160 |
| INDEX       | __idx_501523_l_cd     |   1.117 |        1.160 |
| INDEX       | __idx_501523_l_ok     |   0.576 |        0.598 |
| TABLE       | customer              |   0.448 |        0.457 |
| INDEX       | __idx_501504_o_ck     |   0.371 |        0.387 |
| TABLE       | part                  |   0.296 |        0.309 |
| INDEX       | __idx_501504_o_od     |   0.257 |        0.281 |
| INDEX       | __idx_501504_o_ok     |   0.161 |        0.176 |
| INDEX       | __idx_501466_ps_sk_pk |   0.147 |        0.176 |
| INDEX       | __idx_501466_ps_pk_sk |   0.147 |        0.176 |
| INDEX       | __idx_501466_ps_sk    |   0.146 |        0.176 |
| INDEX       | __idx_501466_ps_pk    |   0.146 |        0.176 |
| TABLE       | supplier              |   0.028 |        0.029 |
| INDEX       | __idx_501485_c_nk     |   0.020 |        0.035 |
| INDEX       | __idx_501464_p_pk     |   0.016 |        0.018 |
| INDEX       | __idx_501485_c_ck     |   0.015 |        0.016 |
| INDEX       | __idx_501465_s_nk     |   0.001 |        0.002 |
| INDEX       | __idx_501465_s_sk     |   0.001 |        0.001 |
| TABLE       | nation                |   0.000 |        0.000 |
| TABLE       | region                |   0.000 |        0.000 |
| INDEX       | __idx_501463_n_rk     |   0.000 |        0.000 |
| INDEX       | __idx_501463_n_nk     |   0.000 |        0.000 |
| INDEX       | __idx_501462_r_rk     |   0.000 |        0.000 |
+-------------+-----------------------+---------+--------------+
31 rows in set (0.03 sec)

[tpchdb] 10:28:24> SELECT
    CASE
        WHEN su.TABLE_NAME LIKE '__idx_%' THEN 'INDEX'
        ELSE 'TABLE'
    END AS object_type,
    su.TABLE_NAME,
    t.TABLE_ROWS AS row_count,
    ROUND(su.OCCUPY_SIZE / 1024 / 1024 / 1024, 3) AS disk_gb,
    ROUND(su.REQUIRED_SIZE / 1024 / 1024 / 1024, 3) AS allocated_gb
FROM oceanbase.DBA_OB_TABLE_SPACE_USAGE su
LEFT JOIN information_schema.tables t
    ON t.TABLE_SCHEMA = su.DATABASE_NAME
    AND t.TABLE_NAME = su.TABLE_NAME
WHERE su.DATABASE_NAME = DATABASE()
ORDER BY su.OCCUPY_SIZE DESC;
+-------------+-----------------------+-----------+---------+--------------+
| object_type | TABLE_NAME            | row_count | disk_gb | allocated_gb |
+-------------+-----------------------+-----------+---------+--------------+
| TABLE       | lineitem              | 300005811 |   8.989 |        9.592 |
| TABLE       | orders                |  75000000 |   2.274 |        2.356 |
| INDEX       | __idx_501523_l_pk_sk  |      NULL |   2.198 |        2.250 |
| INDEX       | __idx_501523_l_sk_pk  |      NULL |   2.035 |        2.074 |
| TABLE       | partsupp              |  40000000 |   1.639 |        1.688 |
| INDEX       | __idx_501523_l_pk     |      NULL |   1.522 |        1.557 |
| INDEX       | __idx_501523_l_sk     |      NULL |   1.184 |        1.231 |
| INDEX       | __idx_501523_l_rd     |      NULL |   1.117 |        1.160 |
| INDEX       | __idx_501523_l_sd     |      NULL |   1.117 |        1.160 |
| INDEX       | __idx_501523_l_cd     |      NULL |   1.117 |        1.160 |
| INDEX       | __idx_501523_l_ok     |      NULL |   0.576 |        0.598 |
| TABLE       | customer              |   7500000 |   0.448 |        0.457 |
| INDEX       | __idx_501504_o_ck     |      NULL |   0.371 |        0.387 |
| TABLE       | part                  |  10000000 |   0.296 |        0.309 |
| INDEX       | __idx_501504_o_od     |      NULL |   0.257 |        0.281 |
| INDEX       | __idx_501504_o_ok     |      NULL |   0.161 |        0.176 |
| INDEX       | __idx_501466_ps_sk_pk |      NULL |   0.147 |        0.176 |
| INDEX       | __idx_501466_ps_pk_sk |      NULL |   0.147 |        0.176 |
| INDEX       | __idx_501466_ps_sk    |      NULL |   0.146 |        0.176 |
| INDEX       | __idx_501466_ps_pk    |      NULL |   0.146 |        0.176 |
| TABLE       | supplier              |    500000 |   0.028 |        0.029 |
| INDEX       | __idx_501485_c_nk     |      NULL |   0.020 |        0.035 |
| INDEX       | __idx_501464_p_pk     |      NULL |   0.016 |        0.018 |
| INDEX       | __idx_501485_c_ck     |      NULL |   0.015 |        0.016 |
| INDEX       | __idx_501465_s_nk     |      NULL |   0.001 |        0.002 |
| INDEX       | __idx_501465_s_sk     |      NULL |   0.001 |        0.001 |
| TABLE       | nation                |        25 |   0.000 |        0.000 |
| TABLE       | region                |         5 |   0.000 |        0.000 |
| INDEX       | __idx_501463_n_rk     |      NULL |   0.000 |        0.000 |
| INDEX       | __idx_501463_n_nk     |      NULL |   0.000 |        0.000 |
| INDEX       | __idx_501462_r_rk     |      NULL |   0.000 |        0.000 |
+-------------+-----------------------+-----------+---------+--------------+
31 rows in set (0.06 sec)

[tpchdb] 10:28:34> SELECT
    su.TABLE_NAME,
    CASE
        WHEN su.TABLE_NAME LIKE '__idx_%' THEN base_tbl.TABLE_NAME
        ELSE su.TABLE_NAME
    END AS base_table,
    CASE
        WHEN su.TABLE_NAME LIKE '__idx_%' THEN 'INDEX'
        ELSE 'TABLE'
    END AS object_type,
    ROUND(su.OCCUPY_SIZE / 1024 / 1024 / 1024, 3) AS disk_gb
FROM oceanbase.DBA_OB_TABLE_SPACE_USAGE su
LEFT JOIN oceanbase.DBA_OB_TABLE_SPACE_USAGE base_tbl
    ON base_tbl.DATABASE_NAME = su.DATABASE_NAME
    AND base_tbl.TABLE_NAME NOT LIKE '__idx_%'
    AND su.TABLE_NAME LIKE CONCAT('__idx_', base_tbl.TABLE_ID, '_%')
WHERE su.DATABASE_NAME = DATABASE()
ORDER BY base_table, object_type, su.OCCUPY_SIZE DESC;
+-----------------------+------------+-------------+---------+
| TABLE_NAME            | base_table | object_type | disk_gb |
+-----------------------+------------+-------------+---------+
| __idx_501485_c_nk     | customer   | INDEX       |   0.020 |
| __idx_501485_c_ck     | customer   | INDEX       |   0.015 |
| customer              | customer   | TABLE       |   0.448 |
| __idx_501523_l_pk_sk  | lineitem   | INDEX       |   2.198 |
| __idx_501523_l_sk_pk  | lineitem   | INDEX       |   2.035 |
| __idx_501523_l_pk     | lineitem   | INDEX       |   1.522 |
| __idx_501523_l_sk     | lineitem   | INDEX       |   1.184 |
| __idx_501523_l_rd     | lineitem   | INDEX       |   1.117 |
| __idx_501523_l_sd     | lineitem   | INDEX       |   1.117 |
| __idx_501523_l_cd     | lineitem   | INDEX       |   1.117 |
| __idx_501523_l_ok     | lineitem   | INDEX       |   0.576 |
| lineitem              | lineitem   | TABLE       |   8.989 |
| __idx_501463_n_rk     | nation     | INDEX       |   0.000 |
| __idx_501463_n_nk     | nation     | INDEX       |   0.000 |
| nation                | nation     | TABLE       |   0.000 |
| __idx_501504_o_ck     | orders     | INDEX       |   0.371 |
| __idx_501504_o_od     | orders     | INDEX       |   0.257 |
| __idx_501504_o_ok     | orders     | INDEX       |   0.161 |
| orders                | orders     | TABLE       |   2.274 |
| __idx_501464_p_pk     | part       | INDEX       |   0.016 |
| part                  | part       | TABLE       |   0.296 |
| __idx_501466_ps_sk_pk | partsupp   | INDEX       |   0.147 |
| __idx_501466_ps_pk_sk | partsupp   | INDEX       |   0.147 |
| __idx_501466_ps_sk    | partsupp   | INDEX       |   0.146 |
| __idx_501466_ps_pk    | partsupp   | INDEX       |   0.146 |
| partsupp              | partsupp   | TABLE       |   1.639 |
| __idx_501462_r_rk     | region     | INDEX       |   0.000 |
| region                | region     | TABLE       |   0.000 |
| __idx_501465_s_nk     | supplier   | INDEX       |   0.001 |
| __idx_501465_s_sk     | supplier   | INDEX       |   0.001 |
| supplier              | supplier   | TABLE       |   0.028 |
+-----------------------+------------+-------------+---------+
31 rows in set (0.06 sec)



[tpchdb] 10:28:50> SELECT
    COALESCE(base_tbl.TABLE_NAME, su.TABLE_NAME) AS base_table,
    ROUND(SUM(CASE WHEN su.TABLE_NAME NOT LIKE '__idx_%' THEN su.OCCUPY_SIZE ELSE 0 END) / 1024/1024/1024, 3) AS table_gb,
    ROUND(SUM(CASE WHEN su.TABLE_NAME LIKE '__idx_%' THEN su.OCCUPY_SIZE ELSE 0 END) / 1024/1024/1024, 3) AS indexes_gb,
    ROUND(SUM(su.OCCUPY_SIZE) / 1024/1024/1024, 3) AS total_gb
FROM oceanbase.DBA_OB_TABLE_SPACE_USAGE su
    AND base_tbl.TABLE_NAME NOT LIKE '__idx_%'
LEFT JOIN oceanbase.DBA_OB_TABLE_SPACE_USAGE base_tbl
    ON base_tbl.DATABASE_NAME = su.DATABASE_NAME
    AND base_tbl.TABLE_NAME NOT LIKE '__idx_%'
    AND su.TABLE_NAME LIKE CONCAT('__idx_', base_tbl.TABLE_ID, '_%')
WHERE su.DATABASE_NAME = DATABASE()
GROUP BY COALESCE(base_tbl.TABLE_NAME, su.TABLE_NAME)
ORDER BY total_gb DESC;
+------------+----------+------------+----------+
| base_table | table_gb | indexes_gb | total_gb |
+------------+----------+------------+----------+
| lineitem   |    8.989 |     10.867 |   19.855 |
| orders     |    2.274 |      0.789 |    3.063 |
| partsupp   |    1.639 |      0.586 |    2.225 |
| customer   |    0.448 |      0.035 |    0.483 |
| part       |    0.296 |      0.016 |    0.312 |
| supplier   |    0.028 |      0.002 |    0.030 |
| region     |    0.000 |      0.000 |    0.000 |
| nation     |    0.000 |      0.000 |    0.000 |
+------------+----------+------------+----------+
8 rows in set (0.07 sec)


[tpchdb] 10:29:16> SELECT
    COALESCE(base_tbl.TABLE_NAME, su.TABLE_NAME) AS base_table,
    t.TABLE_ROWS AS row_count,
    ROUND(SUM(CASE WHEN su.TABLE_NAME NOT LIKE '__idx_%' THEN su.OCCUPY_SIZE ELSE 0 END) / 1024/1024/1024, 3) AS table_gb,
    ROUND(SUM(CASE WHEN su.TABLE_NAME LIKE '__idx_%' THEN su.OCCUPY_SIZE ELSE 0 END) / 1024/1024/1024, 3) AS indexes_gb,
    ROUND(SUM(su.OCCUPY_SIZE) / 1024/1024/1024, 3) AS total_gb
FROM oceanbase.DBA_OB_TABLE_SPACE_USAGE su
LEFT JOIN oceanbase.DBA_OB_TABLE_SPACE_USAGE base_tbl
    ON base_tbl.DATABASE_NAME = su.DATABASE_NAME
    AND base_tbl.TABLE_NAME NOT LIKE '__idx_%'
    AND su.TABLE_NAME LIKE CONCAT('__idx_', base_tbl.TABLE_ID, '_%')
LEFT JOIN information_schema.tables t
    ON t.TABLE_SCHEMA = su.DATABASE_NAME
    AND t.TABLE_NAME = COALESCE(base_tbl.TABLE_NAME, su.TABLE_NAME)
WHERE su.DATABASE_NAME = DATABASE()
GROUP BY COALESCE(base_tbl.TABLE_NAME, su.TABLE_NAME), t.TABLE_ROWS
ORDER BY total_gb DESC;
+------------+-----------+----------+------------+----------+
| base_table | row_count | table_gb | indexes_gb | total_gb |
+------------+-----------+----------+------------+----------+
| lineitem   | 300005811 |    8.989 |     10.867 |   19.855 |
| orders     |  75000000 |    2.274 |      0.789 |    3.063 |
| partsupp   |  40000000 |    1.639 |      0.586 |    2.225 |
| customer   |   7500000 |    0.448 |      0.035 |    0.483 |
| part       |  10000000 |    0.296 |      0.016 |    0.312 |
| supplier   |    500000 |    0.028 |      0.002 |    0.030 |
| region     |         5 |    0.000 |      0.000 |    0.000 |
| nation     |        25 |    0.000 |      0.000 |    0.000 |
+------------+-----------+----------+------------+----------+
8 rows in set (0.15 sec)

[tpchdb] 10:30:19>



[tpchdb] 10:23:35> SELECT
    ROUND(SUM(CASE WHEN TABLE_NAME NOT LIKE '__idx_%' THEN OCCUPY_SIZE ELSE 0 END) / 1024/1024/1024, 2) AS tables_gb,
    ROUND(SUM(CASE WHEN TABLE_NAME LIKE '__idx_%' THEN OCCUPY_SIZE ELSE 0 END) / 1024/1024/1024, 2) AS indexes_gb,
    ROUND(SUM(OCCUPY_SIZE) / 1024/1024/1024, 2) AS total_gb
FROM oceanbase.DBA_OB_TABLE_SPACE_USAGE
WHERE DATABASE_NAME = DATABASE();
+-----------+------------+----------+
| tables_gb | indexes_gb | total_gb |
+-----------+------------+----------+
|     13.67 |      12.29 |    25.97 |
+-----------+------------+----------+
1 row in set (0.03 sec)

[tpchdb] 10:24:01>
```

```sql

в системном тенанте проверка

[oceanbase] 12:41:20> SELECT      svr_ip,     ROUND(DATA_DISK_CAPACITY / 1024 / 1024 / 1024, 2) as capacity_gb,     ROUND(DATA_DISK_IN_USE / 1024 / 1024 / 1024, 2) as in_use_gb,     ROUND((DATA_DISK_CAPACITY - DATA_DISK_IN_USE) / 1024 / 1024 / 1024, 2) as free_gb,     ROUND(DATA_DISK_IN_USE / DATA_DISK_CAPACITY * 100, 2) as used_pct FROM oceanbase.GV$OB_SERVERS;
+----------------+-------------+-----------+---------+----------+
| svr_ip         | capacity_gb | in_use_gb | free_gb | used_pct |
+----------------+-------------+-----------+---------+----------+
| 192.168.32.154 |      350.00 |     28.09 |  321.91 |     8.03 |
| 192.168.32.152 |      350.00 |     28.08 |  321.92 |     8.02 |
| 192.168.32.153 |      350.00 |     28.09 |  321.91 |     8.03 |
+----------------+-------------+-----------+---------+----------+
3 rows in set (0.03 sec)

[oceanbase] 12:41:27>
```
