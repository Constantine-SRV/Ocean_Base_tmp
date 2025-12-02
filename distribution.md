

```sql
SELECT 
    d.database_name,
    t.table_name,
    r.svr_ip,
    COUNT(DISTINCT r.tablet_id) as tablet_count,
    ROUND(SUM(r.data_size) / 1024 / 1024, 2) AS data_mb,
    ROUND(SUM(r.required_size) / 1024 / 1024, 2) AS required_mb
FROM oceanbase.__all_table t
JOIN oceanbase.__all_database d ON t.database_id = d.database_id
JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id
JOIN oceanbase.dba_ob_tablet_replicas r ON ttl.tablet_id = r.tablet_id
WHERE d.database_name = 'twitterdb'
  AND t.table_type = 3  -- только таблицы, без индексов
GROUP BY d.database_name, t.table_name, r.svr_ip
ORDER BY t.table_name, r.svr_ip;

+---------------+---------------+----------------+--------------+----------+-------------+
| database_name | table_name    | svr_ip         | tablet_count | data_mb  | required_mb |
+---------------+---------------+----------------+--------------+----------+-------------+
| twitterdb     | added_tweets  | 192.168.32.155 |            3 |    45.87 |       91.75 |
| twitterdb     | added_tweets  | 192.168.32.156 |            3 |    45.87 |       91.75 |
| twitterdb     | added_tweets  | 192.168.32.157 |            3 |    45.87 |       91.75 |
| twitterdb     | added_tweets  | 192.168.32.158 |            6 |    91.53 |      183.07 |
| twitterdb     | added_tweets  | 192.168.32.159 |            6 |    91.53 |      183.07 |
| twitterdb     | added_tweets  | 192.168.32.160 |            6 |    91.53 |      183.07 |
| twitterdb     | followers     | 192.168.32.155 |            3 |   132.08 |      264.17 |
| twitterdb     | followers     | 192.168.32.156 |            3 |   132.08 |      264.17 |
| twitterdb     | followers     | 192.168.32.157 |            3 |   132.08 |      264.17 |
| twitterdb     | followers     | 192.168.32.158 |            6 |   346.37 |      692.75 |
| twitterdb     | followers     | 192.168.32.159 |            6 |   346.37 |      692.75 |
| twitterdb     | followers     | 192.168.32.160 |            6 |   346.37 |      692.75 |
| twitterdb     | follows       | 192.168.32.155 |            3 |   102.69 |      205.37 |
| twitterdb     | follows       | 192.168.32.156 |            3 |   102.69 |      205.37 |
| twitterdb     | follows       | 192.168.32.157 |            3 |   102.69 |      205.37 |
| twitterdb     | follows       | 192.168.32.158 |            6 |   205.17 |      410.34 |
| twitterdb     | follows       | 192.168.32.159 |            6 |   205.17 |      410.34 |
| twitterdb     | follows       | 192.168.32.160 |            6 |   205.17 |      410.34 |
| twitterdb     | tweets        | 192.168.32.155 |            3 | 31939.73 |    63879.46 |
| twitterdb     | tweets        | 192.168.32.156 |            3 | 31939.73 |    63879.46 |
| twitterdb     | tweets        | 192.168.32.157 |            3 | 31939.73 |    63879.46 |
| twitterdb     | tweets        | 192.168.32.158 |            6 | 67740.00 |   135479.96 |
| twitterdb     | tweets        | 192.168.32.159 |            6 | 67740.00 |   135479.96 |
| twitterdb     | tweets        | 192.168.32.160 |            6 | 67740.00 |   135479.96 |
| twitterdb     | user_profiles | 192.168.32.155 |            3 |   198.79 |      397.58 |
| twitterdb     | user_profiles | 192.168.32.156 |            3 |   198.79 |      397.58 |
| twitterdb     | user_profiles | 192.168.32.157 |            3 |   198.79 |      397.58 |
| twitterdb     | user_profiles | 192.168.32.158 |            6 |   397.61 |      795.23 |
| twitterdb     | user_profiles | 192.168.32.159 |            6 |   397.61 |      795.23 |
| twitterdb     | user_profiles | 192.168.32.160 |            6 |   397.61 |      795.23 |
+---------------+---------------+----------------+--------------+----------+-------------+
30 rows in set (0.07 sec)

```
