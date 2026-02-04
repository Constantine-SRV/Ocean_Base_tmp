# SSL сертификат для OceanBase: установка на сервер

## Шаг 1: Создание конфигурации OpenSSL (на сервере OceanBase)

```bash
# Замени значения на свои
SERVER_FQDN="ob55206.domain.local"   # Реальное имя сервера для CN
SERVER_IP="192.168.55.206"            # IP адрес сервера

cat > /tmp/server-ssl.cnf << EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
CN = ${SERVER_FQDN}
O = YourOrganization
OU = Database
C = RU

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = ${SERVER_FQDN}
IP.1 = ${SERVER_IP}
EOF
```

## Шаг 2: Генерация приватного ключа и CSR

```bash
cd /tmp

# Приватный ключ (RSA 2048)
openssl genrsa -out server-key.pem 2048

# Запрос на сертификат (CSR)
openssl req -new -key server-key.pem -out server.csr -config server-ssl.cnf

# Проверить CSR (убедись что CN и SAN правильные)
openssl req -in server.csr -text -noout | grep -A1 "Subject:"
openssl req -in server.csr -text -noout | grep -A2 "Subject Alternative Name"
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

## Шаг 5: Копирование файлов на OceanBase сервер

```bash
# С машины где файлы, на сервер OceanBase
scp /tmp/ca.pem admin@192.168.55.206:/tmp/
scp /tmp/server-cert.pem admin@192.168.55.206:/tmp/
scp /tmp/server-key.pem admin@192.168.55.206:/tmp/
```

## Шаг 6: Установка в wallet (на сервере OceanBase)

```bash
# Путь к wallet (замени на свой)
WALLET_DIR=/data/obc1/oceanbase/wallet

# Копируй сертификаты
sudo cp /tmp/ca.pem $WALLET_DIR/
sudo cp /tmp/server-cert.pem $WALLET_DIR/
sudo cp /tmp/server-key.pem $WALLET_DIR/

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
