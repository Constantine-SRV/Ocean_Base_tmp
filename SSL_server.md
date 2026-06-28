# SSL сертификат для OceanBase: установка на сервер

## Шаг 1: Создание конфигурации OpenSSL (на сервере OceanBase)

```bash
cd ~/cert

cat > server-ssl.cnf << EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
CN = obcluster200
O = lab
OU = Database
C = SU

[req_ext]
subjectAltName = @alt_names
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth


[alt_names]
DNS.1 = OBM55200
IP.1 = 192.168.55.200
DNS.2 = obsrv201
IP.2 = 192.168.55.201
DNS.3 = obsrv202
IP.3 = 192.168.55.202
DNS.4 = obsrv203
IP.4 = 192.168.55.203
DNS.5 = OB55205
IP.5 = 192.168.55.205

EOF


```

> `extendedKeyUsage = serverAuth, clientAuth` обязателен: `clientAuth` нужен
> для mTLS (Шаг 11), где этот же серт предъявляется как клиентский.

## Шаг 2: Генерация приватного ключа и CSR

```bash
cd ~/cert

# Приватный ключ (RSA 2048)
openssl genrsa -out server-key.pem 2048

# Запрос на сертификат (CSR)
openssl req -new -key server-key.pem -out server.csr -config server-ssl.cnf

# Проверить CSR (убедись что CN и SAN правильные)
openssl req -in server.csr -text -noout | grep -A1 "Subject:"
openssl req -in server.csr -text -noout | grep -A2 "Subject Alternative Name"
openssl req -in server.csr -text -noout | grep -A2 "Extended Key Usage"
cat server.csr
```

## Шаг 3: Подписание на Microsoft AD CA

### Вариант A: PowerShell

```powershell
# На Windows с доступом к CA
certreq -submit -attrib "CertificateTemplate:WebServer" server.csr server-cert.pem
```

### Вариант B: Веб-интерфейс CA

1. Открой `https://dc/certsrv/`
2. "Request a certificate" → "Advanced certificate request"
3. Вставь содержимое `server.csr`
4. Выбери шаблон с поддержкой SAN **и обоими EKU** (serverAuth + clientAuth).
   Проверить EKU шаблона:
   `certutil -dstemplate "1 Web Server" | Select-String "pKIExtendedKeyUsage"`
   Должны быть оба OID: `1.3.6.1.5.5.7.3.1` (serverAuth) и `1.3.6.1.5.5.7.3.2` (clientAuth)
5. Скачай сертификат в формате **Base 64 encoded** → получится `server-cert.cer`

## Шаг 4: Получение CA сертификата

### PowerShell

```powershell
# Экспорт корневого CA в PEM
certutil -ca.cert ca.cer
certutil -encode ca.cer ca.pem
```

### Или через веб-интерфейс

Скачай с `https://dc/certsrv/` → "Download a CA certificate" в формате Base 64 → `ca.cer`

## Шаг 5: Конвертация .cer → .pem и проверка сертификата

Файлы `.cer` из веб-интерфейса AD CS в формате **Base 64** — это **уже PEM**
(внутри `-----BEGIN CERTIFICATE-----`). Отдельной конвертации формата не нужно,
но нужен round-trip через `openssl x509`: он убирает Windows-окончания строк
(CRLF), отрезает лишнее и валидирует серт. Имена меняем на те, что ждёт OceanBase.

```bash
cd ~/cert

# server-cert.cer -> server-cert.pem (fallback на DER, если скачал в DER)
openssl x509 -in server-cert.cer -out server-cert.pem 2>/dev/null \
  || openssl x509 -inform DER -in server-cert.cer -out server-cert.pem

# ca.cer -> ca.pem
openssl x509 -in ca.cer -out ca.pem 2>/dev/null \
  || openssl x509 -inform DER -in ca.cer -out ca.pem
```

### Обязательные проверки перед установкой

```bash
# 1) Ключ соответствует сертификату — обе md5 ДОЛЖНЫ совпасть.
#    Если нет — AD CS подписал не тот CSR, observer не стартанёт.
openssl x509 -in server-cert.pem -noout -modulus | openssl md5
openssl rsa  -in server-key.pem  -noout -modulus | openssl md5

# 2) Цепочка валидна (серт подписан этим CA). Ожидается: server-cert.pem: OK
openssl verify -CAfile ca.pem server-cert.pem

# 3) SAN выжил в выписанном серте.
#    Шаблон WebServer в AD CS по умолчанию SAN из запроса НЕ пробрасывает —
#    нужен кастомный шаблон ("supply in request") или флаг CA
#    EDITF_ATTRIBUTESUBJECTALTNAME2. Если вывод пустой — перевыписывай серт.
openssl x509 -in server-cert.pem -noout -ext subjectAltName

# 4) EKU: для mTLS нужны ОБА — serverAuth и clientAuth
openssl x509 -in server-cert.pem -noout -ext extendedKeyUsage

# 5) Срок действия и Subject
openssl x509 -in server-cert.pem -noout -subject -dates
```

После этого в `~/cert` лежат три готовых файла: `ca.pem`, `server-cert.pem`,
`server-key.pem`.

## Шаг 6: Копирование файлов на OceanBase сервер

Если серты выписаны на самом сервере OceanBase — этот шаг пропускаем, файлы уже на месте.
Если на другой машине — копируем:

```bash
scp ca.pem admin@192.168.55.205:/tmp/
scp server-cert.pem admin@192.168.55.205:/tmp/
scp server-key.pem admin@192.168.55.205:/tmp/
```

## Шаг 7: Установка в wallet (на сервере OceanBase)

```bash
# Путь к wallet (замени на свой). Найти home observer'а:
#   PID=$(pgrep -x observer | head -1); readlink -f /proc/$PID/cwd
WALLET_DIR=/data/obc1/oceanbase/wallet

# Копируй сертификаты (из ~/cert или /tmp)
sudo mkdir -p $WALLET_DIR
sudo cp ~/cert/ca.pem $WALLET_DIR/
sudo cp ~/cert/server-cert.pem $WALLET_DIR/
sudo cp ~/cert/server-key.pem $WALLET_DIR/

# Права доступа
sudo chown admin:admin $WALLET_DIR/*.pem
sudo chmod 644 $WALLET_DIR/ca.pem
sudo chmod 600 $WALLET_DIR/server-cert.pem
sudo chmod 600 $WALLET_DIR/server-key.pem

# Проверка
ls -la $WALLET_DIR/
```

## Шаг 8: Включение SSL в OceanBase

```sql
-- Подключись к sys тенанту
ALTER SYSTEM SET ssl_client_authentication = 'True';
ALTER SYSTEM SET sql_protocol_min_tls_version = 'TLSv1.2';
```

`ssl_client_authentication=True` — observer запрашивает и проверяет клиентский
серт (нужно для mTLS-пользователей из Шага 11). Параметр динамический.

> **Важно после замены файлов в wallet:** observer читает CA/cert/key в память
> один раз (при старте или по reload). После копирования новых сертов выполни
> горячую перезагрузку, иначе сменится только серверная сторона, а клиентский
> CA останется старым:
> ```sql
> ALTER SYSTEM RELOAD ssl;
> ```

## Шаг 9: Перезапуск OBServer (если RELOAD ssl недостаточно)

В большинстве случаев хватает `ALTER SYSTEM RELOAD ssl` из Шага 8 — рестарт не нужен.
Полный рестарт только если меняли параметры с `need_reboot` или reload не подхватил серты.

```bash
# Найди PID
PID=$(pgrep observer)

# Рабочая директория
WORK_DIR=$(readlink -f /proc/$PID/cwd)

# Остановка
kill -15 $PID
while pgrep observer; do sleep 2; echo "waiting..."; done

# Запуск
cd $WORK_DIR
nohup ./bin/observer -p 2881 &
```

Или через OBD (с клиента):

```bash
obd cluster restart <cluster_name>
```

## Шаг 10: Проверка

```bash
# С клиента (используй OpenSSL-клиент: mysql 8.0 или obclient, НЕ MariaDB)
mysql -h192.168.55.205 -P2881 -uroot@sys -p -e "\s" | grep SSL
```

Ожидаемый результат (cipher может быть любой из набора TLS 1.3):

```
SSL:                    Cipher in use is TLS_AES_256_GCM_SHA384
```

Проверка статуса SSL на сервере:

```sql
SELECT svr_ip, SSL_CERT_EXPIRED_TIME
FROM oceanbase.GV$OB_SERVERS;
```

Если `SSL_CERT_EXPIRED_TIME` показывает дату — сертификаты загружены.

## Шаг 11: Беспарольный пользователь по клиентскому сертификату (mTLS)

Пользователь авторизуется не паролем, а предъявлением валидного клиентского
серта с нужным subject. `server-cert.pem` выступает и серверным, и клиентским
сертом («shared fate»: если серт невалиден — observer и сам бы не поднялся).

> Требует `ssl_client_authentication=True` (Шаг 8) и `RELOAD ssl` после замены
> файлов в wallet — иначе observer не запрашивает клиентский серт и любой
> `REQUIRE X509`/`SUBJECT` отбивается с `Access denied (using password: NO)`.

### 11.1: Узнать точную строку subject

OB сравнивает `REQUIRE SUBJECT` побайтово с `X509_NAME_oneline`. AD CS может
переупорядочить RDN'ы (часто в порядок C→CN), поэтому строку НЕ угадываем, а
берём из реального серта:

```bash
WALLET_DIR=/data/obc1/oceanbase/wallet
sudo openssl x509 -in $WALLET_DIR/server-cert.pem -noout -subject -nameopt compat
# -> subject=/C=SU/O=lab/OU=Database/CN=obcluster200
```

Берём всё после `subject=`.

### 11.2: Создать пользователя

```sql
-- subject — РОВНО строка из вывода выше
CREATE USER 'test_mtls'@'%'
  REQUIRE SUBJECT '/C=SU/O=lab/OU=Database/CN=obcluster200';

GRANT SELECT ON oceanbase.* TO 'test_mtls'@'%';
```

> Хост: `@'%'` надёжнее `@'127.0.0.1'`. На части нод даже коннект на `127.0.0.1`
> observer видит с внешнего IP ноды (зависит от сетевой конфигурации), и
> `@'127.0.0.1'` не матчится по хосту.

### 11.3: Клиентская копия сертов (чтобы не ходить под sudo)

Серты в wallet принадлежат `admin:admin` с правами `600` — обычный пользователь
их не прочитает. Делаем отдельную клиентскую копию:

```bash
mkdir -p ~/cert-client
sudo cp $WALLET_DIR/{ca.pem,server-cert.pem,server-key.pem} ~/cert-client/
sudo chown $USER:$USER ~/cert-client/*.pem
chmod 600 ~/cert-client/server-key.pem
chmod 644 ~/cert-client/ca.pem ~/cert-client/server-cert.pem
```

### 11.4: Подключение (без пароля)

Только OpenSSL-клиент. **mysql 8.0** — с `--ssl-mode`:

```bash
CLI=~/cert-client
mysql -h127.0.0.1 -P2881 -utest_mtls \
  --ssl-mode=VERIFY_CA \
  --ssl-ca=$CLI/ca.pem \
  --ssl-cert=$CLI/server-cert.pem \
  --ssl-key=$CLI/server-key.pem \
  -e "SELECT current_user(), user();"
```

**obclient / MariaDB-клиент** — без `--ssl-mode` (этой опции у них нет):

```bash
obclient -h127.0.0.1 -P2881 -utest_mtls \
  --ssl-ca=$CLI/ca.pem \
  --ssl-cert=$CLI/server-cert.pem \
  --ssl-key=$CLI/server-key.pem \
  -e "SELECT current_user(), user();"
```

Пароль не спрашивается, `current_user()` показывает `test_mtls@%`.

> **Внимание:** клиент MariaDB (`mysql Ver 15.1 Distrib ...-MariaDB`) с TLS-стеком
> OceanBase несовместим — даёт `TLS/SSL error` мусорными байтами. Нужен
> OpenSSL-based клиент. Замена на одной машине:
> `sudo dnf install -y mysql --allowerasing` (снесёт mariadb, поставит mysql 8.0).

### Диагностика mTLS

| Симптом | Причина |
|---------|---------|
| `TLS/SSL error: ...` мусор | Клиент MariaDB вместо OpenSSL (mysql 8.0 / obclient) |
| `Access denied ... (using password: NO)`, host ≠ ожидаемый | Хост не совпал — используй `@'%'` |
| `Access denied ... (using password: NO)`, host ок | subject ≠ вывод `-nameopt compat`, либо observer не перечитал wallet (`RELOAD ssl`) |
| `unknown variable 'ssl-mode'` | Это MariaDB/obclient — убери `--ssl-mode` |

Зонд «запрашивает ли observer клиентский серт вообще» (TLS до авторизации):

```bash
openssl s_client -connect 127.0.0.1:2881 -starttls mysql \
  -cert $CLI/server-cert.pem -key $CLI/server-key.pem -CAfile $CLI/ca.pem \
  </dev/null 2>/dev/null | grep -iE "Acceptable client certificate|certificate CA names"
```

`Acceptable client certificate CA names` есть → серт запрашивается (копай subject/CA).
Пусто → observer не запрашивает серт (проверь `ssl_client_authentication` и `RELOAD ssl`).

## Шаг 12: Установка SSL на OBProxy

OBProxy шифрует **две независимые ноги**, каждая своим переключателем:

| Параметр | Что шифрует |
|----------|-------------|
| `enable_client_ssl` | клиент → obproxy |
| `enable_server_ssl` | obproxy → observer |

Для сквозного TLS (client→proxy→server) включают **оба**. Серты — тот же комплект.

### 12.1: Положить wallet в home OBProxy

```bash
# рабочий каталог процесса obproxy
pgrep -a obproxy
PID=$(pgrep -x obproxy | head -1)
HOME_PROXY=$(readlink -f /proc/$PID/cwd)   # напр. /data/obc1/obproxy
echo "obproxy home: $HOME_PROXY"

sudo mkdir -p $HOME_PROXY/wallet
sudo cp ~/cert/{ca.pem,server-cert.pem,server-key.pem} $HOME_PROXY/wallet/

# владелец = OS-пользователь, под которым работает obproxy (здесь admin)
sudo chown admin:admin $HOME_PROXY/wallet/*.pem
sudo chmod 644 $HOME_PROXY/wallet/ca.pem
sudo chmod 600 $HOME_PROXY/wallet/server-cert.pem $HOME_PROXY/wallet/server-key.pem
ls -la $HOME_PROXY/wallet/
```

### 12.2: Включить SSL и прописать пути к сертам

Подключаемся к служебному порту прокси (proxysys):

```bash
obclient -h192.168.55.200 -P2883 -uroot@proxysys -p'qaz123' -Doceanbase -A
```

```sql
-- включить обе ноги (need_reboot=false → действует для НОВЫХ коннектов)
ALTER PROXYCONFIG SET enable_server_ssl = true;
ALTER PROXYCONFIG SET enable_client_ssl = true;

-- пути к сертам ОТНОСИТЕЛЬНО home OBProxy, для нужного кластера (APP_NAME)
UPDATE proxyconfig.security_config
   SET CONFIG_VAL = '{"sourceType":"FILE","CA":"wallet/ca.pem","publicKey":"wallet/server-cert.pem","privateKey":"wallet/server-key.pem"}'
   WHERE APP_NAME = 'obc442' AND VERSION = '1';
```

> Один OBProxy обслуживает несколько кластеров — у каждого своя строка в
> `security_config` (своё `APP_NAME`). Повтори UPDATE для каждого кластера,
> которому нужен TLS, иначе его коннекты пойдут без серта прокси.

### 12.3: Проверки

**Какие кластеры покрыты SSL и статус переключателей:**

```sql
SELECT APP_NAME, VERSION, CONFIG_VAL FROM proxyconfig.security_config;
SHOW PROXYCONFIG LIKE '%ssl%';
-- ждём enable_client_ssl=True, enable_server_ssl=True
```

**Нога клиент → proxy (новый коннект через порт прокси 2883):**

```bash
mysql -h192.168.55.200 -P2883 -uroot@sys#obc442 -p'qaz123' -e "\s" | grep SSL
# Ожидается: SSL: Cipher in use is TLS_AES_256_GCM_SHA384
```

**Нога proxy → observer (смотрим со стороны кластера):**

```sql
-- коннект прилетает с IP прокси (192.168.55.200); SSL_CIPHER заполнен = нога шифруется
SELECT SSL_CIPHER, user, host
FROM oceanbase.GV$OB_PROCESSLIST
WHERE state='active';
```

Если в `\s` есть cipher И в `GV$OB_PROCESSLIST` у коннекта с IP прокси заполнен
`SSL_CIPHER` — сквозной TLS клиент→прокси→сервер работает.

> Принудительный TLS (`ssl_attributes` с `using_ssl=ENABLE_FORCE`) для этой схемы
> не рекомендуется: создаёт проблемы с mTLS на ноге proxy→observer. Оставляй
> `ssl_attributes` пустым (best-effort) — оба `enable_*_ssl=True` поднимают TLS,
> когда обе стороны умеют.

---

## Важно

**Имена файлов в wallet (обязательные):**

| Файл | Описание |
|------|----------|
| `ca.pem` | Сертификат CA |
| `server-cert.pem` | Сертификат сервера (он же клиентский для mTLS) |
| `server-key.pem` | Приватный ключ |

**Права доступа:**

| Файл | Права |
|------|-------|
| `ca.pem` | 644 |
| `server-cert.pem` | 600 |
| `server-key.pem` | 600 |

**Владелец:** `admin:admin` (пользователь под которым работает observer/obproxy)

**Ключевые требования:**

- EKU серта: **serverAuth + clientAuth** (иначе mTLS не работает)
- `ssl_client_authentication=True` на observer — для клиентской аутентификации
- После замены файлов в wallet: **`ALTER SYSTEM RELOAD ssl`** (горячо, без рестарта)
- Клиент: только **OpenSSL-based** (mysql 8.0 / obclient), MariaDB-клиент несовместим
- `REQUIRE SUBJECT` — строка строго из `openssl x509 ... -nameopt compat`
