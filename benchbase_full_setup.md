# Тестирование через BenchBase

тестирование через benchbase


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
## Ocean Base
конфиг ОБ
```xml
    <url>jdbc:mysql://192.168.55.205:2881/tpcc_test?useSSL=false&amp;allowPublicKeyRetrieval=true&amp;serverTimezone=UTC&amp;socketTimeout=1800000&amp;connectTimeout=30000</url>
    <username>root@sys</username>
    <password>qaz123</password>
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
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --create=true --load=true
```

```bash
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --execute=true
```

### Собрать резуьтаты обоих тестов в один zip 2025-11-11_19-25-41 будет виден в выводе тестов
```bash
 zip -r ~/benchbase_results.zip ~/benchbase-mysql/results/chbenchmark_2025-11-11_19-25-41.{summary.json,results.csv,params.json} ~/benchbase-postgres/results/chbenchmark_2025-11-11_19-32-47.{summary.json,results.csv,params.json}
```
10 складов ПГ
```
testdb=# SELECT
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
 customer   |       300000 | 208 MB
 district   |          100 | 64 kB
 history    |       300000 | 25 MB
 item       |       100000 | 12 MB
 nation     |           62 | 64 kB
 new_order  |        90000 | 6808 kB
 oorder     |       300000 | 40 MB
 order_line |      3000610 | 380 MB
 orders     |      3505504 | 472 MB
 region     |            5 | 56 kB
 stock      |      1000027 | 361 MB
 supplier   |        10000 | 2496 kB
 warehouse  |           10 | 56 kB
(13 rows)

testdb=# SELECT 'customer'   AS table_name, COUNT(*) AS row_count FROM customer
UNION ALL SELECT 'district',   COUNT(*) FROM district
UNION ALL SELECT 'warehouse',  COUNT(*) FROM warehouse
UNION ALL SELECT 'item',       COUNT(*) FROM item
UNION ALL SELECT 'stock',      COUNT(*) FROM stock
UNION ALL SELECT 'orders',     COUNT(*) FROM orders
UNION ALL SELECT 'order_line', COUNT(*) FROM order_line
UNION ALL SELECT 'new_order',  COUNT(*) FROM new_order
UNION ALL SELECT 'history',    COUNT(*) FROM history
UNION ALL SELECT 'nation',     COUNT(*) FROM nation
UNION ALL SELECT 'region',     COUNT(*) FROM region
UNION ALL SELECT 'supplier',   COUNT(*) FROM supplier
UNION ALL SELECT 'oorder',     COUNT(*) FROM oorder
ORDER BY 1;
 table_name | row_count
------------+-----------
 customer   |    300000
 district   |       100
 history    |    300000
 item       |    100000
 nation     |        62
 new_order  |     90000
 oorder     |    300000
 order_line |   3000644
 orders     |   3505504
 region     |         5
 stock      |   1000000
 supplier   |     10000
 warehouse  |        10
(13 rows)

testdb=#


testdb=# VACUUM (FREEZE, ANALYZE);
VACUUM
testdb=#


testdb=# SELECT
  table_name,
  pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) AS total_size
FROM information_schema.tables
WHERE table_schema='public'
ORDER BY 1;
 table_name | total_size
------------+------------
 customer   | 208 MB
 district   | 64 kB
 history    | 25 MB
 item       | 12 MB
 nation     | 64 kB
 new_order  | 6808 kB
 oorder     | 40 MB
 order_line | 380 MB
 orders     | 472 MB
 region     | 56 kB
 stock      | 361 MB
 supplier   | 2496 kB
 warehouse  | 56 kB
(13 rows)

testdb=#


```
