# Тестирование через BenchBase

тестирование через benchbase
(Run the TPC-C benchmark test in OceanBase Database)
https://en.oceanbase.com/docs/community-observer-en-10000000000601813

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

```bash
# Распаковка готовой сборки в домашний каталог
tar -xzf target/benchbase-mysql.tgz -C ~/
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
```

### Собрать резуьтаты обоих тестов в один zip 2025-11-11_19-25-41 будет виден в выводе тестов
```bash
 zip -r ~/benchbase_results.zip ~/benchbase-mysql/results/chbenchmark_2025-11-11_19-25-41.{summary.json,results.csv,params.json} ~/benchbase-postgres/results/chbenchmark_2025-11-11_19-32-47.{summary.json,results.csv,params.json}
```
## Запросы по размерам таблиц
### 100 складов ПГ
```

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
```
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
