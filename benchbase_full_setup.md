# 📘 Настройка конфигурации BenchBase

## Создание каталогов для конфигураций

``` bash
mkdir -p ~/benchbase-configs/postgres
mkdir -p ~/benchbase-configs/mysql
mkdir -p ~/benchbase-configs/oceanbase
```

------------------------------------------------------------------------

## 🐘 PostgreSQL

### Копируем шаблон и создаём конфиг:

``` bash
cd ~/benchbase-configs/postgres
cp ~/benchbase-postgres/config/postgres/sample_tpcc_config.xml pg_tpcc_10w.xml
nano pg_tpcc_10w.xml
```

### Пример настроек подключения:

``` xml
<url>jdbc:postgresql://192.168.55.211:5432/testdb?sslmode=disable&amp;ApplicationName=chbenchmark&amp;reWriteBatchedInserts=true</url>
<username>testdbuser</username>
<password>qaz123</password>
```

### Создание базы и загрузка данных:

``` bash
cd ~/benchbase-postgres
java -jar benchbase.jar -b tpcc -c ~/benchbase-configs/postgres/pg_tpcc_10w.xml --create=true --load=true
```

------------------------------------------------------------------------

## 🧪 CH-Benchmark для PostgreSQL

``` bash
cp ~/benchbase-postgres/config/postgres/sample_chbenchmark_config.xml ~/benchbase-configs/postgres/pg_ch_10w.xml
nano ~/benchbase-configs/postgres/pg_ch_10w.xml
```

Создание базы:

``` bash
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/postgres/pg_ch_10w.xml --create=true --load=true
```

Запуск теста:

``` bash
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/postgres/pg_ch_10w.xml --execute=true
```

------------------------------------------------------------------------

## 🐬 OceanBase (через MySQL-драйвер)

### Конфигурация для TPC-C

``` bash
cd ~/benchbase-configs/oceanbase
cp ~/benchbase-mysql/config/mysql/sample_tpcc_config.xml ob_tpcc_100w.xml
nano ob_tpcc_100w.xml
```

**Настройки подключения:**

``` xml
<url>jdbc:mysql://192.168.55.205:2881/tpcc_test?useSSL=false&amp;allowPublicKeyRetrieval=true&amp;serverTimezone=UTC&amp;socketTimeout=1800000&amp;connectTimeout=30000</url>
<username>root@sys</username>
<password>qaz123</password>
```

**Создание и загрузка базы:**

``` bash
cd ~/benchbase-mysql
java -jar benchbase.jar -b tpcc -c ~/benchbase-configs/oceanbase/ob_tpcc_100w.xml --create=true --load=true
```

------------------------------------------------------------------------

## 🧩 CH-Benchmark для OceanBase

``` bash
cp ~/benchbase-mysql/config/mysql/sample_chbenchmark_config.xml ~/benchbase-configs/oceanbase/ob_ch_100w.xml
nano ~/benchbase-configs/oceanbase/ob_ch_100w.xml
```

**Создание базы:**

``` bash
cd ~/benchbase-mysql
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --create=true --load=true
```

**Запуск теста:**

``` bash
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --execute=true
```

------------------------------------------------------------------------

## 📂 Структура после установки

    ~/benchbase-postgres-src/      ← исходники (можно удалить)
    ~/benchbase-mysql-src/         ← исходники (можно удалить)
    ~/benchbase-postgres/          ← готовая сборка PostgreSQL
    ~/benchbase-mysql/             ← готовая сборка MySQL/OceanBase
    ~/benchbase-configs/           ← твои XML-конфиги

------------------------------------------------------------------------

## 💡 Примечания


-   Для CHBenchmark создаются **дополнительные аналитические таблицы**:

    -   `customer_address`
    -   `item_market`
    -   `order_line_details`
