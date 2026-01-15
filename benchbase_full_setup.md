# Тестирование через BenchBase

## тестирование через benchbase
У ОБ есть свой тест (Run the TPC-C benchmark test in OceanBase Database)
https://en.oceanbase.com/docs/community-observer-en-10000000000601813
на сколько сильно модифицирован код по сравнению с оригиналом не понятно, видно, что есть партиционирование.

## установка необходима  <java.version>23</java.version>
```bash
https://adoptium.net/temurin/releases/
скачиваем  OpenJDK23U-jdk_x64_linux_hotspot_23.0.2_7.tar.gz в домашний каталог
можно так если файл есть
cd ~
wget https://github.com/adoptium/temurin23-binaries/releases/download/jdk-23.0.2%2B7/OpenJDK23U-jdk_x64_linux_hotspot_23.0.2_7.tar.gz
# Создаем директорию для JVM
sudo mkdir -p /usr/lib/jvm
```

```bash
# Распаковываем Java 23
sudo tar xzf ~/OpenJDK23U-jdk_x64_linux_hotspot_23.0.2_7.tar.gz -C /usr/lib/jvm/
```

```bash
# Проверяем имя папки
ls /usr/lib/jvm/
```

```bash
# Настраиваем alternatives для java
sudo alternatives --install /usr/bin/java java /usr/lib/jvm/jdk-23.0.2+7/bin/java 2
```

```bash
# Настраиваем alternatives для javac
sudo alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk-23.0.2+7/bin/javac 2
```

```bash
# Переключаемся на Java 23
sudo alternatives --config java
# Выбери номер с /usr/lib/jvm/jdk-23.0.2+7
```

```bash
sudo alternatives --config javac
# Выбери номер с /usr/lib/jvm/jdk-23.0.2+7
```

```bash
# Проверяем версии
java -version
javac -version
```



```bash
установка
sudo dnf install -y git
```

```bash
# =======================================
# 🧹 1. Очистка старых установок
# =======================================
cd ~
rm -rf benchbase benchbase-* benchbase-pg benchbase-mysql \
       benchbase-postgres benchbase-postgres-src \
       benchbase-mysql benchbase-mysql-src
```

```bash
# =======================================
# 🐘 2. Установка BenchBase для PostgreSQL
# =======================================
git clone https://github.com/cmu-db/benchbase.git benchbase-postgres-src
cd benchbase-postgres-src
./mvnw clean package -P postgres -DskipTests
```

```bash
# Распаковка готовой сборки в домашний каталог
tar -xzf target/benchbase-postgres.tgz -C ~/
```

```bash
# =======================================
# 🐬 3. Установка BenchBase для MySQL
# =======================================
cd ~
git clone https://github.com/cmu-db/benchbase.git benchbase-mysql-src
cd benchbase-mysql-src
./mvnw clean package -P mysql -DskipTests
```
из своего форка
```
rm -rf ~/benchbase-mysql-src
rm -rf ~/benchbase-oceanbase-src
rm -rf ~/benchbase-mysql

cd ~
git clone https://github.com/Constantine-SRV/benchbase.git benchbase-mysql-src
cd benchbase-mysql-src

git remote -v

./mvnw clean package -P mysql -DskipTests

# Распаковка готовой сборки в домашний каталог
tar -xzf target/benchbase-mysql.tgz -C ~/
cd ~/benchbase-mysql/
```

```bash
# Распаковка готовой сборки в домашний каталог
tar -xzf target/benchbase-mysql.tgz -C ~/
```

### oracle
```bash
cd ~
rm -rf benchbase-oracle benchbase-oracle-src
git clone --depth 1 https://github.com/Constantine-SRV/benchbase.git benchbase-oracle-src
cd benchbase-oracle-src
git remote -v
./mvnw clean package -P oracle -DskipTests
tar -xzf target/benchbase-oracle.tgz -C ~/
```

### MSSQL
```bash
cd ~
rm -rf benchbase-sqlserver benchbase-mssql-src
git clone --depth 1 https://github.com/Constantine-SRV/benchbase.git benchbase-mssql-src
cd benchbase-mssql-src
git remote -v
./mvnw clean package -P sqlserver -DskipTests
tar -xzf target/benchbase-sqlserver.tgz -C ~/
cd ~/benchbase-sqlserver/
```
копирование конфигов по умолчанию
```bash
cp ~/benchbase-mysql/config/postgres/sample_tpcc_config.xml ~/benchbase-configs/postgres/pg_ch_10w.xml

cp ~/benchbase-mysql/config/mysql/sample_auctionmark_config.xml ~/benchbase-configs/oceanbase/
cp ~/benchbase-mysql/config/mysql/sample_auctionmark_config.xml ~/benchbase-configs/postgres/


```



После установки структура будет такой:
```bash
~/benchbase-postgres-src/   ← исходники (можно удалить)
~/benchbase-mysql-src/      ← исходники (можно удалить)
~/benchbase-postgres/       ← готовая сборка PostgreSQL
~/benchbase-mysql/          ← готовая сборка MySQL
```


Где лежат шаблоны конфигураций:
~/benchbase-postgres/config/postgres/
~/benchbase-mysql/config/mysql/


```bash
mkdir -p ~/benchbase-configs/postgres
mkdir -p ~/benchbase-configs/mysql
```



вначале создать конфиги
параметры постгреса
```xml
    <url>jdbc:postgresql://192.168.55.211:5432/testdb?sslmode=disable&amp;ApplicationName=chbenchmark&amp;reWriteBatchedInserts=true</url>
    <username>testdbuser</username>
    <password>qaz123</password>
```

```bash
все делаем в
cd ~
cd ~/benchbase-configs/postgres
cp ~/benchbase-postgres/config/postgres/sample_tpcc_config.xml pg_tpcc_10w.xml
nano pg_tpcc_10w.xml
```

создать базу
```bash
cd ~/benchbase-postgres
java -jar benchbase.jar -b tpcc -c ~/benchbase-configs/postgres/pg_tpcc_10w.xml --create=true --load=true
```

Создание конфига для CH-Benchmark необходимо заранее создать tpcc
```bash
mkdir -p ~/benchbase-configs/postgres
cd ~/benchbase-configs/postgres
cp ~/benchbase-mysql/config/postgres/sample_tpcc_config.xml ~/benchbase-configs/postgres/pg_ch_10w.xml
nano ~/benchbase-configs/postgres/pg_ch_10w.xml
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/postgres/pg_ch_10w.xml --create=true --load=true
```
запуск теста 
```bash
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/postgres/pg_ch_10w.xml  --execute=true
```
или если не хватает памяти
```
java -Xms2G -Xmx4G -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/postgres/pg_ch_100w.xml --execute=true
```

## Ocean Base
конфиг ОБ
```xml
    <url>jdbc:mysql://192.168.55.205:2881/benchbasedb?   <url>jdbc:mysql://192.168.55.205:2881/benchbasedb?useSSL=false&amp;allowPublicKeyRetrieval=true&amp;serverTimezone=UTC&amp;socketTimeout=1800000&amp;connectTimeout=30000&amp;rewriteBatchedStatements=true&amp;useServerPrepStmts=false&amp;cachePrepStmts=false</url>
    <username>root@app_tenant</username>
    <password>qaz123</password>
    <reconnectOnConnectionFailure>true</reconnectOnConnectionFailure>
    <isolation>TRANSACTION_READ_COMMITTED</isolation>
    <batchsize>1000</batchsize>
```

```bash
ls -lh ~/benchbase-configs/oceanbase/
cd ~/benchbase-mysql
cp ~/benchbase-mysql/config/mysql/sample_tpcc_config.xml ~/benchbase-configs/oceanbase/ob_tpcc_10w.xml
nano ~/benchbase-configs/oceanbase/ob_tpcc_10w.xml
java -jar benchbase.jar -b tpcc -c ~/benchbase-configs/oceanbase/ob_tpcc_10w.xml --create=true --load=true
```
не забть индексы
```
USE benchbasedb;
-- tpcc
CREATE INDEX idx_customer_name ON customer (c_w_id, c_d_id, c_last, c_first);

-- CH-Benchmark
CREATE INDEX supplier_nation_idx ON supplier (su_nationkey) LOCAL;

```
процедура обновления статистики всех таблиц
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
задать параметры под тест 
https://en.oceanbase.com/docs/common-oceanbase-database-10000000001970925#complex_oltp

### Тест CH-Benchmark
```bash
cp ~/benchbase-mysql/config/mysql/sample_chbenchmark_config.xml ~/benchbase-configs/oceanbase/ob_ch_10w.xml
nano ~/benchbase-configs/oceanbase/ob_ch_10w.xml
```

```bash
cd ~/benchbase-mysql
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_10w.xml --create=true --load=true
```

```bash
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_10w.xml --execute=true
java -Xms2G -Xmx4G -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --execute=true
```

### Собрать резуьтаты обоих тестов в один zip 2025-11-11_19-25-41 будет виден в выводе тестов
```bash
 zip -r ~/benchbase_results.zip ~/benchbase-mysql/results/chbenchmark_2025-11-11_19-25-41.{summary.json,results.csv,params.json} ~/benchbase-postgres/results/chbenchmark_2025-11-11_19-32-47.{summary.json,results.csv,params.json}
```
## Запросы по размерам таблиц
### 100 складов ПГ
```sql
benchbasedb=# VACUUM (FREEZE, ANALYZE);
VACUUM
benchbasedb=# SELECT
    table_name,
    (xpath('/row/count/text()', xml_count))[1]::text::bigint AS row_count,
    pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) AS total_size
FROM (
    SELECT
        table_name,
        query_to_xml(format('SELECT count(*) AS count FROM %I', table_name), false, true, '') AS xml_count
    FROM information_schema.tables
    WHERE table_schema = 'public'
) AS counts
ORDER BY table_name;
 table_name | row_count | total_size
------------+-----------+------------
 customer   |   3000000 | 2074 MB
 district   |      1000 | 200 kB
 history    |   3000000 | 247 MB
 item       |    100000 | 12 MB
 nation     |        62 | 64 kB
 new_order  |    900000 | 65 MB
 oorder     |   3000000 | 402 MB
 order_line |  30001892 | 3798 MB
 region     |         5 | 56 kB
 stock      |  10000000 | 3606 MB
 supplier   |     10000 | 2496 kB
 warehouse  |       100 | 64 kB
(12 rows)

benchbasedb=#  SELECT
    t.table_name,
    COALESCE(c.reltuples::bigint, 0) AS row_estimate,
    pg_size_pretty(pg_total_relation_size(quote_ident(t.table_name))) AS total_size
FROM information_schema.tables t
LEFT JOIN pg_class c
    ON c.relname = t.table_name
WHERE t.table_schema = 'public'
ORDER BY t.table_name;
 table_name | row_estimate | total_size
------------+--------------+------------
 customer   |      2999656 | 2074 MB
 district   |         1000 | 200 kB
 history    |      2999999 | 247 MB
 item       |       100000 | 12 MB
 nation     |           62 | 64 kB
 new_order  |       900000 | 65 MB
 oorder     |      3000000 | 402 MB
 order_line |     30001914 | 3798 MB
 region     |            5 | 56 kB
 stock      |     10000276 | 3606 MB
 supplier   |        10000 | 2496 kB
 warehouse  |          100 | 64 kB
(12 rows)

benchbasedb=#

```
### 100 складов ОБ
```sql
[benchbasedb] 09:09:28> CALL analyze_all_tables();
Query OK, 0 rows affected (43.27 sec)

[benchbasedb] 09:16:52> SELECT     t.table_name,     COALESCE(s.row_count, 0) AS row_count,     CONCAT(ROUND(s.data_size / 1024 / 1024, 2), ' MB') AS total_size FROM information_schema.tables AS t LEFT JOIN (     SELECT         table_name,         SUM(  data_length + index_length) AS data_size,         SUM(table_rows) AS row_count     FROM information_schema.tables     WHERE table_schema = DATABASE()     GROUP BY table_name ) AS s ON s.table_name = t.table_name WHERE t.table_schema = DATABASE() ORDER
BY t.table_name;
+------------+-----------+------------+
| table_name | row_count | total_size |
+------------+-----------+------------+
| customer   |   3000000 | 1054.00 MB |
| district   |      1000 | 2.00 MB    |
| history    |   3000000 | 56.00 MB   |
| item       |    100000 | 6.00 MB    |
| nation     |        62 | 0.00 MB    |
| new_order  |    900000 | 2.00 MB    |
| oorder     |   3000000 | 46.00 MB   |
| order_line |  30001892 | 592.00 MB  |
| region     |         5 | 0.00 MB    |
| stock      |  10000000 | 1724.00 MB |
| supplier   |     10000 | 0.00 MB    |
| warehouse  |       100 | 2.00 MB    |
+------------+-----------+------------+
12 rows in set (0.03 sec)


[benchbasedb] 19:00:11> SELECT
    d.database_name,
    t.table_name,
    CASE t.table_type
        WHEN 3 THEN 'TABLE'
        WHEN 5 THEN 'INDEX'
        ELSE CONCAT('TYPE_', t.table_type)
    END AS object_type,
    ts.row_cnt AS total_rows,
    ROUND(ts.avg_row_len, 1) AS avg_row_len,
    ROUND(ts.row_cnt * ts.avg_row_len / 1024 / 1024, 2) AS approx_data_mb,
    ROUND(SUM(r.data_size) / 1024 / 1024, 2) AS real_data_mb,
    ROUND(SUM(r.required_size) / 1024 / 1024, 2) AS required_mb,
    MAX(ts.last_analyzed) AS last_analyzed
FROM oceanbase.__all_table t
JOIN oceanbase.__all_database d
    ON t.database_id = d.database_id
LEFT JOIN oceanbase.__all_table_stat ts
    ON ts.table_id = t.table_id
JOIN oceanbase.__all_tablet_to_ls ttl
    ON t.table_id = ttl.table_id
JOIN oceanbase.dba_ob_tablet_replicas r
    ON ttl.tablet_id = r.tablet_id
WHERE d.database_name ='benchbasedb'
-- NOT IN ('oceanbase','mysql','information_schema','sys','ocs','sys_external_tbs')
GROUP BY d.database_name, t.table_name, t.table_type, ts.row_cnt, ts.avg_row_len, ts.last_analyzed
ORDER BY  1,2; 

+---------------+--------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
| database_name | table_name                     | object_type | total_rows | avg_row_len | approx_data_mb | real_data_mb | required_mb | last_analyzed              |
+---------------+--------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
| benchbasedb   | customer                       | TABLE       |    3000000 |       819.0 |        2343.18 |       945.48 |      945.48 | 2025-11-16 14:16:32.926813 |
| benchbasedb   | district                       | TABLE       |       1000 |       228.0 |           0.22 |         0.05 |        0.05 | 2025-11-16 14:16:33.015227 |
| benchbasedb   | history                        | TABLE       |    3000000 |       164.0 |         469.21 |        45.80 |       45.80 | 2025-11-16 14:16:19.160688 |
| benchbasedb   | item                           | TABLE       |     100000 |       135.0 |          12.87 |         4.53 |        4.53 | 2025-11-16 14:16:52.536466 |
| benchbasedb   | nation                         | TABLE       |         62 |       134.0 |           0.01 |         0.00 |        0.00 | 2025-11-16 14:16:09.449831 |
| benchbasedb   | new_order                      | TABLE       |     900000 |        60.0 |          51.50 |         0.24 |        0.25 | 2025-11-16 14:16:18.012746 |
| benchbasedb   | oorder                         | TABLE       |    3000000 |       154.0 |         440.60 |        10.46 |       10.46 | 2025-11-16 14:16:18.475621 |
| benchbasedb   | order_line                     | TABLE       |   30001892 |       205.0 |        5865.47 |       567.74 |      567.74 | 2025-11-16 14:16:17.864870 |
| benchbasedb   | region                         | TABLE       |          5 |       174.0 |           0.00 |         0.00 |        0.00 | 2025-11-16 14:16:09.484616 |
| benchbasedb   | stock                          | TABLE       |   10000000 |       515.0 |        4911.42 |      1668.13 |     1668.13 | 2025-11-16 14:16:52.430570 |
| benchbasedb   | supplier                       | TABLE       |      10000 |       254.0 |           2.42 |         0.00 |        0.00 | 2025-11-16 14:16:09.405067 |
| benchbasedb   | warehouse                      | TABLE       |        100 |       188.0 |           0.02 |         0.01 |        0.01 | 2025-11-16 14:16:52.584649 |
| benchbasedb   | __idx_500286_idx_customer_name | INDEX       |    2640000 |       107.0 |         269.39 |        45.33 |       45.33 | 2025-11-16 05:59:45.310423 |
| benchbasedb   | __idx_500291_o_w_id            | INDEX       |    2640000 |        80.0 |         201.42 |        10.82 |       10.82 | 2025-11-16 06:00:03.238869 |
+---------------+--------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
14 rows in set (0.02 sec)
```

### 4000 складов ОБ
```sql
 SELECT     t.table_name,     CASE t.table_type         WHEN 3 THEN 'TABLE'         WHEN 5 THEN 'INDEX'     END as object_type,     COUNT(DISTINCT r.TABLET_ID) as tablet_count,     ROUND(SUM(r.DATA_SIZE) / 1024 / 1024, 2) as data_mb,     ROUND(SUM(r.REQUIRED_SIZE) / 1024 / 1024, 2) as required_mb FROM oceanbase.__all_table t JOIN oceanbase.__all_tablet_to_ls ttl ON t.table_id = ttl.table_id JOIN oceanbase.DBA_OB_TABLET_REPLICAS r ON ttl.tablet_id = r.TABLET_ID WHERE t.database_id = (     SELECT database_id FROM oceanbase.__all_database WHERE database_name = 'benchbasedb' ) GROUP BY t.table_name, t.table_type ORDER BY data_mb DESC;
+--------------------------------+-------------+--------------+-----------+-------------+
| table_name                     | object_type | tablet_count | data_mb   | required_mb |
+--------------------------------+-------------+--------------+-----------+-------------+
| stock                          | TABLE       |            9 | 173797.14 |   173797.14 |
| customer                       | TABLE       |            9 |  89673.28 |    89673.28 |
| order_line                     | TABLE       |            9 |  55672.71 |    55672.71 |
| __idx_500463_idx_customer_name | INDEX       |            9 |   4924.95 |     4924.95 |
| history                        | TABLE       |            9 |   4195.40 |     4195.40 |
| oorder                         | TABLE       |            9 |    853.86 |      853.86 |
| __idx_500486_o_w_id            | INDEX       |            9 |    691.65 |      691.65 |
| new_order                      | TABLE       |            9 |      6.54 |        6.73 |
| warehouse                      | TABLE       |            9 |      0.00 |        0.00 |
| item                           | TABLE       |            1 |      0.00 |        0.00 |
| district                       | TABLE       |            9 |      0.00 |        0.00 |
+--------------------------------+-------------+--------------+-----------+-------------+
11 rows in set (0.02 sec)

SELECT     t.table_name,     COALESCE(s.row_count, 0) AS row_count,     CONCAT(ROUND(s.data_size / 1024 / 1024, 2), ' MB') AS total_size FROM information_schema.tables AS t LEFT JOIN (     SELECT         table_name,         SUM(  data_length + index_length) AS data_size,         SUM(table_rows) AS row_count     FROM information_schema.tables     WHERE table_schema = DATABASE()     GROUP BY table_name ) AS s ON s.table_name = t.table_name WHERE t.table_schema = DATABASE() ORDER BY t.table_name;
+------------+------------+-------------+
| table_name | row_count  | total_size  |
+------------+------------+-------------+
| customer   |  119970000 | 46368.00 MB |
| district   |      39990 | 36.00 MB    |
| history    |  119970000 | 2444.00 MB  |
| item       |     100000 | 8.00 MB     |
| new_order  |   35991000 | 256.00 MB   |
| oorder     |  119970000 | 1742.00 MB  |
| order_line | 1199678863 | 29312.00 MB |
| stock      |  399900000 | 74682.00 MB |
| warehouse  |       3999 | 36.00 MB    |
+------------+------------+-------------+
9 rows in set (0.12 sec)

SELECT
    d.database_name,
    t.table_name,
    CASE t.table_type
        WHEN 3 THEN 'TABLE'
        WHEN 5 THEN 'INDEX'
        ELSE CONCAT('TYPE_', t.table_type)
    END AS object_type,
    ts.row_cnt AS total_rows,
    ROUND(ts.avg_row_len, 1) AS avg_row_len,
    ROUND(ts.row_cnt * ts.avg_row_len / 1024 / 1024, 2) AS approx_data_mb,
    ROUND(SUM(r.data_size) / 1024 / 1024, 2) AS real_data_mb,
    ROUND(SUM(r.required_size) / 1024 / 1024, 2) AS required_mb,
    MAX(ts.last_analyzed) AS last_analyzed
FROM oceanbase.__all_table t
JOIN oceanbase.__all_database d
    ON t.database_id = d.database_id
LEFT JOIN oceanbase.__all_table_stat ts
    ON ts.table_id = t.table_id
JOIN oceanbase.__all_tablet_to_ls ttl
    ON t.table_id = ttl.table_id
JOIN oceanbase.dba_ob_tablet_replicas r
    ON ttl.tablet_id = r.tablet_id
WHERE d.database_name ='benchbasedb'
-- NOT IN ('oceanbase','mysql','information_schema','sys','ocs','sys_external_tbs')
GROUP BY d.database_name, t.table_name, t.table_type, ts.row_cnt, ts.avg_row_len, ts.last_analyzed
ORDER BY  1,2; 
+---------------+--------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
| database_name | table_name                     | object_type | total_rows | avg_row_len | approx_data_mb | real_data_mb | required_mb | last_analyzed              |
+---------------+--------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
| benchbasedb   | customer                       | TABLE       |   13350000 |       819.0 |       10427.14 |    269019.84 |   269019.84 | 2025-11-28 23:39:01.495245 |
| benchbasedb   | customer                       | TABLE       |   13320000 |       819.0 |       10403.71 |    538039.69 |   538039.69 | 2025-11-28 23:39:01.495245 |
| benchbasedb   | customer                       | TABLE       |  119970000 |       819.0 |       93703.68 |     89673.28 |    89673.28 | 2025-11-28 23:39:02.398769 |
| benchbasedb   | district                       | TABLE       |      39990 |       228.0 |           8.70 |         2.09 |        2.11 | 2025-11-29 00:27:38.469833 |
| benchbasedb   | district                       | TABLE       |       4440 |       228.0 |           0.97 |        12.56 |       12.66 | 2025-11-29 00:27:38.453240 |
| benchbasedb   | district                       | TABLE       |       4450 |       228.0 |           0.97 |         6.28 |        6.33 | 2025-11-29 00:27:38.453240 |
| benchbasedb   | history                        | TABLE       |   13350000 |       164.0 |        2087.97 |     12586.19 |    12586.19 | 2025-11-28 23:39:16.234651 |
| benchbasedb   | history                        | TABLE       |   13320000 |       164.0 |        2083.28 |     25172.38 |    25172.38 | 2025-11-28 23:39:16.234651 |
| benchbasedb   | history                        | TABLE       |  119970000 |       164.0 |       18763.62 |      4195.40 |     4195.40 | 2025-11-28 23:39:16.259100 |
| benchbasedb   | item                           | TABLE       |     100000 |       135.0 |          12.87 |         0.00 |        0.00 | 2025-11-28 23:30:21.428806 |
| benchbasedb   | new_order                      | TABLE       |    4005000 |        60.0 |         229.17 |        19.63 |       20.18 | 2025-11-28 23:39:45.368711 |
| benchbasedb   | new_order                      | TABLE       |   35991000 |        60.0 |        2059.42 |         6.54 |        6.73 | 2025-11-28 23:39:45.592697 |
| benchbasedb   | new_order                      | TABLE       |    3996000 |        60.0 |         228.65 |        39.26 |       40.36 | 2025-11-28 23:39:45.368711 |
| benchbasedb   | oorder                         | TABLE       |  119970000 |       154.0 |       17619.50 |       853.86 |      853.86 | 2025-11-28 23:39:39.394923 |
| benchbasedb   | oorder                         | TABLE       |   13320000 |       154.0 |        1956.25 |      5123.15 |     5123.15 | 2025-11-28 23:39:39.291606 |
| benchbasedb   | oorder                         | TABLE       |   13350000 |       154.0 |        1960.66 |      2561.57 |     2561.57 | 2025-11-28 23:39:39.291606 |
| benchbasedb   | order_line                     | TABLE       |  133500440 |       205.0 |       26099.77 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       |  133196290 |       205.0 |       26040.31 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       | 1199678863 |       205.0 |      234541.10 |     55672.71 |    55672.71 | 2025-11-28 23:44:50.510581 |
| benchbasedb   | order_line                     | TABLE       |  133495716 |       205.0 |       26098.84 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       |  133497638 |       205.0 |       26099.22 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       |  133199883 |       205.0 |       26041.01 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       |  133195668 |       205.0 |       26040.18 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       |  133199702 |       205.0 |       26040.97 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       |  133193370 |       205.0 |       26039.73 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | order_line                     | TABLE       |  133200156 |       205.0 |       26041.06 |     55672.71 |    55672.71 | 2025-11-28 23:44:49.977662 |
| benchbasedb   | stock                          | TABLE       |   44500000 |       515.0 |       21855.83 |    521391.43 |   521391.43 | 2025-11-28 23:36:01.818981 |
| benchbasedb   | stock                          | TABLE       |   44400000 |       515.0 |       21806.72 |   1042782.85 |  1042782.85 | 2025-11-28 23:36:01.818981 |
| benchbasedb   | stock                          | TABLE       |  399900000 |       515.0 |      196407.79 |    173797.14 |   173797.14 | 2025-11-28 23:36:02.800845 |
| benchbasedb   | warehouse                      | TABLE       |       3999 |       188.0 |           0.72 |         0.00 |        0.00 | 2025-11-29 00:27:38.228856 |
| benchbasedb   | warehouse                      | TABLE       |        445 |       188.0 |           0.08 |         0.00 |        0.00 | 2025-11-29 00:27:38.216388 |
| benchbasedb   | warehouse                      | TABLE       |        444 |       188.0 |           0.08 |         0.00 |        0.00 | 2025-11-29 00:27:38.216388 |
| benchbasedb   | __idx_500463_idx_customer_name | INDEX       |       NULL |        NULL |           NULL |      4924.95 |     4924.95 | NULL                       |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   12096589 |        80.0 |         922.90 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   14707897 |        80.0 |        1122.12 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   12216211 |        80.0 |         932.02 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   14908644 |        80.0 |        1137.44 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   15508918 |        80.0 |        1183.24 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   16981189 |        80.0 |        1295.56 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   16742421 |        80.0 |        1277.35 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   13627095 |        80.0 |        1039.66 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |   10469541 |        80.0 |         798.76 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
| benchbasedb   | __idx_500486_o_w_id            | INDEX       |  127258505 |        80.0 |        9709.05 |       691.65 |      691.65 | 2025-11-28 22:07:18.933909 |
+---------------+--------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
43 rows in set (0.14 sec)



 SELECT      svr_ip,     ROUND(DATA_DISK_CAPACITY / 1024 / 1024 / 1024, 2) as capacity_gb,     ROUND(DATA_DISK_IN_USE / 1024 / 1024 / 1024, 2) as in_use_gb,     ROUND((DATA_DISK_CAPACITY - DATA_DISK_IN_USE) / 1024 / 1024 / 1024, 2) as free_gb,     ROUND(DATA_DISK_IN_USE / DATA_DISK_CAPACITY * 100, 2) as used_pct FROM oceanbase.GV$OB_SERVERS;
+----------------+-------------+-----------+---------+----------+
| svr_ip         | capacity_gb | in_use_gb | free_gb | used_pct |
+----------------+-------------+-----------+---------+----------+
| 192.168.32.158 |      350.00 |    151.83 |  198.17 |    43.38 |
| 192.168.32.159 |      350.00 |    150.44 |  199.56 |    42.98 |
| 192.168.32.160 |      350.00 |    147.68 |  202.32 |    42.19 |
+----------------+-------------+-----------+---------+----------+
3 rows in set (0.00 sec)

```

## twitter 80K
ОБ
```sql

SELECT      svr_ip,     ROUND(DATA_DISK_CAPACITY / 1024 / 1024 / 1024, 2) as capacity_gb,     ROUND(DATA_DISK_IN_USE / 1024 / 1024 / 1024, 2) as in_use_gb,     ROUND((DATA_DISK_CAPACITY - DATA_DISK_IN_USE) / 1024 / 1024 / 1024, 2) as free_gb,     ROUND(DATA_DISK_IN_USE / DATA_DISK_CAPACITY * 100, 2) as used_pct FROM oceanbase.GV$OB_SERVERS;
+----------------+-------------+-----------+---------+----------+
| svr_ip         | capacity_gb | in_use_gb | free_gb | used_pct |
+----------------+-------------+-----------+---------+----------+
| 192.168.32.158 |      350.00 |    116.29 |  233.71 |    33.23 |
| 192.168.32.159 |      350.00 |    116.46 |  233.54 |    33.28 |
| 192.168.32.160 |      350.00 |    116.47 |  233.53 |    33.28 |
+----------------+-------------+-----------+---------+----------+
3 rows in set (0.00 sec)

[oceanbase] 08:02:45>


SELECT     t.table_name,     COALESCE(s.row_count, 0) AS row_count,     CONCAT(ROUND(s.data_size / 1024 / 1024, 2), ' MB') AS total_size FROM information_schema.tables AS t LEFT JOIN (     SELECT         table_name,         SUM(  data_length + index_length) AS data_size,         SUM(table_rows) AS row_count     FROM information_schema.tables     WHERE table_schema = DATABASE()     GROUP BY table_name ) AS s ON s.table_name = t.table_name WHERE t.table_schema = DATABASE() ORDER
    -> BY t.table_name;
+---------------+------------+--------------+
| table_name    | row_count  | total_size   |
+---------------+------------+--------------+
| added_tweets  |          0 | 0.00 MB      |
| followers     |  122605619 | 498.00 MB    |
| follows       |  122605619 | 324.00 MB    |
| tweets        | 1600000000 | 113588.00 MB |
| user_profiles |   40000000 | 1386.00 MB   |
+---------------+------------+--------------+
5 rows in set (0.01 sec)

[twitterdb] 08:02:16>




 SELECT
    ->     d.database_name,
    ->     t.table_name,
    ->     CASE t.table_type
    ->         WHEN 3 THEN 'TABLE'
    ->         WHEN 5 THEN 'INDEX'
    ->         ELSE CONCAT('TYPE_', t.table_type)
    ->     END AS object_type,
    ->     ts.row_cnt AS total_rows,
    ->     ROUND(ts.avg_row_len, 1) AS avg_row_len,
    ->     ROUND(ts.row_cnt * ts.avg_row_len / 1024 / 1024, 2) AS approx_data_mb,
    ->     ROUND(SUM(r.data_size) / 1024 / 1024, 2) AS real_data_mb,
    ->     ROUND(SUM(r.required_size) / 1024 / 1024, 2) AS required_mb,
    ->     MAX(ts.last_analyzed) AS last_analyzed
    -> FROM oceanbase.__all_table t
    -> JOIN oceanbase.__all_database d
    ->     ON t.database_id = d.database_id
    -> LEFT JOIN oceanbase.__all_table_stat ts
    ->     ON ts.table_id = t.table_id
    -> JOIN oceanbase.__all_tablet_to_ls ttl
    ->     ON t.table_id = ttl.table_id
    -> JOIN oceanbase.dba_ob_tablet_replicas r
    ->     ON ttl.tablet_id = r.tablet_id
    -> WHERE d.database_name ='twitterdb'
    -> -- NOT IN ('oceanbase','mysql','information_schema','sys','ocs','sys_external_tbs')
    -> GROUP BY d.database_name, t.table_name, t.table_type, ts.row_cnt, ts.avg_row_len, ts.last_analyzed
    -> ORDER BY  1,2;
+---------------+-----------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
| database_name | table_name                        | object_type | total_rows | avg_row_len | approx_data_mb | real_data_mb | required_mb | last_analyzed              |
+---------------+-----------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
| twitterdb     | added_tweets                      | TABLE       |          0 |         0.0 |           0.00 |         0.00 |        0.00 | 2025-12-02 12:50:06.257790 |
| twitterdb     | added_tweets                      | TABLE       |          0 |         0.0 |           0.00 |         0.00 |        0.00 | 2025-12-02 12:50:06.274475 |
| twitterdb     | followers                         | TABLE       |   12077835 |        40.0 |         460.73 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |  122605619 |        40.0 |        4677.03 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.861560 |
| twitterdb     | followers                         | TABLE       |    7493713 |        40.0 |         285.86 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |   32962379 |        40.0 |        1257.41 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |   18790363 |        40.0 |         716.80 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |   15072355 |        40.0 |         574.96 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |   10400605 |        40.0 |         396.75 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |    9313968 |        40.0 |         355.30 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |    8541792 |        40.0 |         325.84 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | followers                         | TABLE       |    7952609 |        40.0 |         303.37 |      1436.58 |     1436.58 | 2025-12-02 12:45:23.713088 |
| twitterdb     | follows                           | TABLE       |   13655183 |        40.0 |         520.90 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |  122605619 |        40.0 |        4677.03 |       923.88 |      923.88 | 2025-12-02 12:45:26.764578 |
| twitterdb     | follows                           | TABLE       |   13708338 |        40.0 |         522.93 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |   13568100 |        40.0 |         517.58 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |   13597631 |        40.0 |         518.71 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |   13664833 |        40.0 |         521.27 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |   13619717 |        40.0 |         519.55 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |   13662285 |        40.0 |         521.17 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |   13593986 |        40.0 |         518.57 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | follows                           | TABLE       |   13535546 |        40.0 |         516.34 |       923.88 |      923.88 | 2025-12-02 12:45:26.262883 |
| twitterdb     | tweets                            | TABLE       | 1600000000 |       122.0 |      186157.23 |    299037.28 |   299037.28 | 2025-12-02 12:50:06.157124 |
| twitterdb     | tweets                            | TABLE       |  154248857 |       122.0 |       17946.59 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  227293066 |       122.0 |       26445.15 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  152733503 |       122.0 |       17770.28 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  150919000 |       122.0 |       17559.16 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  249236383 |       122.0 |       28998.22 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  153190503 |       122.0 |       17823.45 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  154689718 |       122.0 |       17997.88 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  204770627 |       122.0 |       23824.71 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | tweets                            | TABLE       |  152918343 |       122.0 |       17791.78 |    299037.28 |   299037.28 | 2025-12-02 12:50:04.957877 |
| twitterdb     | user_profiles                     | TABLE       |    4444444 |        78.0 |         330.61 |      8942.60 |     8942.60 | 2025-12-02 12:45:18.390601 |
| twitterdb     | user_profiles                     | TABLE       |   40000000 |        78.0 |        2975.46 |      1788.52 |     1788.52 | 2025-12-02 12:45:18.434093 |
| twitterdb     | user_profiles                     | TABLE       |    4444445 |        78.0 |         330.61 |      7154.08 |     7154.08 | 2025-12-02 12:45:18.390601 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |   37733270 |        20.0 |         719.71 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    4444444 |        20.0 |          84.77 |       508.38 |      508.38 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    4606825 |        20.0 |          87.87 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    3279392 |        20.0 |          62.55 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    4444445 |        20.0 |          84.77 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    4680287 |        20.0 |          89.27 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    4718354 |        20.0 |          90.00 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    3558572 |        20.0 |          67.87 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_followers   | INDEX       |    3556507 |        20.0 |          67.83 |       254.19 |      254.19 | 2025-12-02 03:44:59.819387 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    3558572 |        20.0 |          67.87 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    4444444 |        20.0 |          84.77 |       508.38 |      508.38 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    4606825 |        20.0 |          87.87 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    3279392 |        20.0 |          62.55 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    4444445 |        20.0 |          84.77 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    4680287 |        20.0 |          89.27 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    4718354 |        20.0 |          90.00 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |    3556507 |        20.0 |          67.83 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501181_idx_user_partition   | INDEX       |   37733270 |        20.0 |         719.71 |       254.19 |      254.19 | 2025-12-02 03:44:59.903186 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |   72536903 |        40.0 |        2767.06 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |   61675411 |        40.0 |        2352.73 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |   91920999 |        40.0 |        3506.51 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |   74915684 |        40.0 |        2857.81 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |   55862730 |        40.0 |        2130.99 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |  101319598 |        40.0 |        3865.04 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |   69345875 |        40.0 |        2645.34 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |  103567988 |        40.0 |        3950.81 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |   57738802 |        40.0 |        2202.56 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501235_idx_tweets_uid       | INDEX       |  688883990 |        40.0 |       26278.84 |     39145.50 |    39145.50 | 2025-12-02 04:30:19.348702 |
| twitterdb     | __idx_501256_idx_added_tweets_uid | INDEX       |       NULL |        NULL |           NULL |         0.00 |        0.00 | NULL                       |
+---------------+-----------------------------------+-------------+------------+-------------+----------------+--------------+-------------+----------------------------+
64 rows in set (0.11 sec)

[twitterdb] 08:00:45>





[twitterdb] 07:50:06> SELECT     t.table_name,     COALESCE(s.row_count, 0) AS row_count,     CONCAT(ROUND(s.data_size / 1024 / 1024, 2), ' MB') AS total_size FROM information_schema.tables AS t LEFT JOIN (     SELECT         table_name,         SUM(  data_length + index_length) AS data_size,         SUM(table_rows) AS row_count     FROM information_schema.tables     WHERE table_schema = DATABASE()     GROUP BY table_name ) AS s ON s.table_name = t.table_name WHERE t.table_schema = DATABASE() ORDER
    -> BY t.table_name;
+---------------+------------+--------------+
| table_name    | row_count  | total_size   |
+---------------+------------+--------------+
| added_tweets  |          0 | 0.00 MB      |
| followers     |  122605619 | 498.00 MB    |
| follows       |  122605619 | 324.00 MB    |
| tweets        | 1600000000 | 113588.00 MB |
| user_profiles |   40000000 | 1386.00 MB   |
+---------------+------------+--------------+
5 rows in set (0.13 sec)
```

PG
```sql


twitterdb=# SELECT
    table_name,
    (xpath('/row/count/text()', xml_count))[1]::text::bigint AS row_count,
    pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) AS total_size
FROM (
    SELECT
        table_name,
        query_to_xml(format('SELECT count(*) AS count FROM %I', table_name), false, true, '') AS xml_count
    FROM information_schema.tables
    WHERE table_schema = 'public'
) AS counts
ORDER BY table_name;
  table_name   | row_count  | total_size
---------------+------------+------------
 added_tweets  |          0 | 16 kB
 followers     |  122828101 | 7266 MB
 follows       |  122828101 | 7563 MB
 tweets        | 1600000000 | 356 GB
 user_profiles |   40000000 | 4966 MB
(5 rows)

twitterdb=# VACUUM (FREEZE, ANALYZE);

```
