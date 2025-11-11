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
mkdir -p ~/benchbase-configs/oceanbase
cd ~/benchbase-configs/oceanbase
cp ~/benchbase-mysql/config/mysql/sample_tpcc_config.xml ~/benchbase-configs/oceanbase/ob_tpcc_100w.xml
nano ~/benchbase-configs/oceanbase/ob_tpcc_100w.xml
```
конфиг ОБ
```xml
    <url>jdbc:mysql://192.168.55.205:2881/tpcc_test?useSSL=false&amp;allowPublicKeyRetrieval=true&amp;serverTimezone=UTC&amp;socketTimeout=1800000&amp;connectTimeout=30000</url>
    <url>jdbc:mysql://192.168.55.205:2881/tpcc_test?useSSL=false&amp;allowPublicKeyRetrieval=true&amp;serverTimezone=UTC</url>
    <username>root@sys</username>
    <password>!QAZ2wsx3edc</password>
```

```bash
ls -lh ~/benchbase-configs/oceanbase/
cd ~/benchbase-mysql
java -jar benchbase.jar -b tpcc -c ~/benchbase-configs/oceanbase/ob_tpcc_100w.xml --create=true --load=true
```


## Тест CH-Benchmark
```bash
cp ~/benchbase-mysql/config/mysql/sample_chbenchmark_config.xml ~/benchbase-configs/oceanbase/ob_ch_100w.xml
nano ~/benchbase-configs/oceanbase/ob_ch_100w.xml
cd ~/benchbase-mysql
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --execute=true
```

```bash
cd ~/benchbase-mysql
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --create=true --load=true
```

```bash
java -jar benchbase.jar -b chbenchmark -c ~/benchbase-configs/oceanbase/ob_ch_100w.xml --execute=true
```
