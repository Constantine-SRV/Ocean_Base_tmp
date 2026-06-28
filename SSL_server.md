# SSL сертификат для OceanBase: установка на сервер

## Шаг 1: Создание конфигурации OpenSSL (на сервере OceanBase)

```bash
cd cert

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

## Шаг 2: Генерация приватного ключа и CSR

```bash
cd /cert

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

1. Открой `https://ca-server/certsrv`
2. "Request a certificate" → "Advanced certificate request"
3. Вставь содержимое `server.csr`
4. Выбери шаблон (WebServer или аналогичный с поддержкой SAN)
5. Скачай сертификат в формате Base64 (.cer/.pem)

## Шаг 4: Получение CA сертификата

### PowerShell

```powershell
# Экспорт корневого CA в PEM
certutil -ca.cert ca.cer
certutil -encode ca.cer ca.pem
```

### Или через веб-интерфейс

Скачай с `https://ca-server/certsrv` → "Download a CA certificate"

## Шаг 5: Конвертация .cer → .pem и проверка (на сервере OceanBase)

Из веб-интерфейса AD CS (`https://доменконтроллер/certsrv/`) при выборе
**Base 64 encoded** скачиваются файлы `.cer`, которые **уже являются PEM**
(внутри `-----BEGIN CERTIFICATE-----`). Нужен только round-trip через
`openssl x509`: он убирает Windows-окончания строк (CRLF), отрезает лишнее
и валидирует сертификат. Имена меняем на те, что ждёт OceanBase.

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

# 2) Цепочка валидна (серт подписан этим CA)
openssl verify -CAfile ca.pem server-cert.pem
# Ожидается: server-cert.pem: OK

# 3) SAN выжил в выписанном серте.
#    Шаблон WebServer в AD CS по умолчанию SAN из запроса НЕ пробрасывает —
#    нужен кастомный шаблон ("supply in request") или флаг CA
#    EDITF_ATTRIBUTESUBJECTALTNAME2. Если вывод пустой — перевыписывай серт.
openssl x509 -in server-cert.pem -noout -ext subjectAltName

# 4) Срок действия и Subject
openssl x509 -in server-cert.pem -noout -subject -dates
```
еще проверка
```bash
[admin@obsrv202 cert]$ openssl x509 -in server-cert.pem -noout -ext extendedKeyUsage
X509v3 Extended Key Usage:
    TLS Web Client Authentication, TLS Web Server Authentication
[admin@obsrv202 cert]$ openssl x509 -in server-cert.pem -noout -ext keyUsage
X509v3 Key Usage: critical
    Digital Signature, Key Encipherment
[admin@obsrv202 cert]$
```

После этого в `~/cert` лежат три готовых файла: `ca.pem`,
`server-cert.pem`, `server-key.pem` — переходи к копированию в wallet.

## Шаг 6: Установка в wallet (на сервере OceanBase)

```bash
# Путь к wallet (замени на свой)
pgrep -a observer

WALLET_DIR=/data/obc1/oceanbase/wallet

# Копируй сертификаты
sudo cp ./ca.pem $WALLET_DIR/
sudo cp ./server-cert.pem $WALLET_DIR/
sudo cp ./server-key.pem $WALLET_DIR/

# Права доступа
sudo chown admin:admin $WALLET_DIR/*.pem
sudo chmod 644 $WALLET_DIR/ca.pem
sudo chmod 600 $WALLET_DIR/server-cert.pem
sudo chmod 600 $WALLET_DIR/server-key.pem

# Проверка
ls -la $WALLET_DIR/
```

## Шаг 7: Включение SSL в OceanBase

```sql
-- Подключись к sys тенанту
ALTER SYSTEM SET ssl_client_authentication = 'TRUE';
ALTER SYSTEM SET sql_protocol_min_tls_version = 'TLSv1.2';
```

## Шаг 8: Перезапуск OBServer

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

## Шаг 9: Проверка

```bash
# С клиента
mysql -h192.168.55.206 -P2881 -uroot@sys -p -e "\s" | grep SSL
```

Ожидаемый результат:

```
SSL:                    Cipher in use is TLS_AES_256_GCM_SHA384
```

## Проверка статуса SSL на сервере

```sql
SELECT svr_ip, SSL_CERT_EXPIRED_TIME 
FROM oceanbase.GV$OB_SERVERS;
```

Если `SSL_CERT_EXPIRED_TIME` показывает дату — сертификаты загружены.

---

## Важно

**Имена файлов в wallet (обязательные):**

| Файл | Описание |
|------|----------|
| `ca.pem` | Сертификат CA |
| `server-cert.pem` | Сертификат сервера |
| `server-key.pem` | Приватный ключ |

**Права доступа:**

| Файл | Права |
|------|-------|
| `ca.pem` | 644 |
| `server-cert.pem` | 600 |
| `server-key.pem` | 600 |

**Владелец:** `admin:admin` (пользователь под которым работает observer)
