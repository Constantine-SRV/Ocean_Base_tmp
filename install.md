# Установка OceanBase CE (RHEL 9 / AlmaLinux 9)

## 1️⃣ Подготовка ОС (RHEL 9 / AlmaLinux 9)

Создаём пользователя admin без пароля (во всех примерах документации используется именно это имя):

```bash
sudo useradd -m admin
sudo passwd -d admin
sudo usermod -aG wheel admin
echo "admin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/admin
sudo chmod 440 /etc/sudoers.d/admin
```

---

## 2️⃣ Создание и распространение SSH-ключей

На управляющем сервере (192.168.55.200) под пользователем admin:

```bash
su - admin
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

tar czf /tmp/admin_ssh_keys.tar.gz -C ~/.ssh .
```

**Копируем ключи:**

> У admin нет пароля, поэтому первый раз копируем под пользователем, у которого есть пароль и SSH доступ:

```bash
scp /tmp/admin_ssh_keys.tar.gz root@192.168.55.205:/tmp/
scp /tmp/admin_ssh_keys.tar.gz root@192.168.55.206:/tmp/
```

---

## 3️⃣ Разворачивание ключей на нодах

На каждой новой ноде под root:

```bash
sudo mkdir -p /home/admin/.ssh
sudo tar xzf /tmp/admin_ssh_keys.tar.gz -C /home/admin/.ssh
sudo chown -R admin:admin /home/admin/.ssh
sudo chmod 700 /home/admin/.ssh
sudo chmod 600 /home/admin/.ssh/id_rsa
sudo chmod 644 /home/admin/.ssh/id_rsa.pub
sudo chmod 600 /home/admin/.ssh/authorized_keys
```

---

## 4️⃣ Проверка SSH-доступа

```bash
ssh admin@192.168.55.200 hostname
ssh admin@192.168.55.205 hostname
ssh admin@192.168.55.206 hostname
```

С серверов БД не нужен доступ до управляющего, достаточно прямого доступа от управляющего и между нодами.

---

## 5️⃣ Подготовка каталогов и прав

```bash
sudo mkdir -p /data/1 /data/log1
sudo chown -R admin:admin /data
```

Рекомендуется размещать `/data/1` и `/data/log1` на отдельных дисках.

---

## 6️⃣ Настройки ядра и лимитов

```bash
sudo tee -a /etc/sysctl.conf <<'EOF'
fs.aio-max-nr = 1048576
vm.max_map_count = 655360
EOF
sudo sysctl -p

sudo tee /etc/security/limits.d/oceanbase.conf <<'EOF'
* soft nofile 655350
* hard nofile 655350
* soft stack unlimited
* hard stack unlimited
EOF

ulimit -n
ulimit -s
```

---

## 7️⃣ Открытие портов

```bash
sudo firewall-cmd --permanent --add-port=2881/tcp  # OceanBase SQL
sudo firewall-cmd --permanent --add-port=2882/tcp  # OceanBase RPC
sudo firewall-cmd --permanent --add-port=2886/tcp  # OBServer shell API
sudo firewall-cmd --permanent --add-port=2888/tcp  # Internal heartbeat
sudo firewall-cmd --permanent --add-port=8680/tcp  # OBD Web UI
sudo firewall-cmd --permanent --add-port=8088/tcp  # OBAgent HTTP
sudo firewall-cmd --permanent --add-port=8089/tcp  # OBAgent Metrics
sudo firewall-cmd --permanent --add-port=8080/tcp  # OCP Web (optional)
sudo firewall-cmd --permanent --add-port=9090/tcp  # Prometheus (optional)
sudo firewall-cmd --permanent --add-port=3000/tcp  # Grafana (optional)
sudo firewall-cmd --reload
```

---

## 8️⃣ Установка Deployer

Когда пользователь admin создан, заходим под ним, скачиваем и устанавливаем на управляющий сервер дистрибутив OceanBase.

Ссылка на установку: https://en.oceanbase.com/docs/common-oceanbase-database-10000000001973974

Прямая ссылка на дистрибутив есть на GitHub OceanBase.

Далее в веб-интерфейсе всё понятно.

Управляющий сервер сам подключится к серверам баз данных с SSH-ключом, который был прописан, и установит всё необходимое.

---

## 🔟 Проверка установки

**Проверка OceanBase:**

```bash
curl http://192.168.55.205:2886/api/v1/info
```

**Проверка агента с паролем из конфига:**

```bash
curl -u admin:91nMuGN1n http://192.168.55.205:8088/metrics/stat
```

Отдаёт все метрики — можно настраивать в Victoria Metrics.

**Подключение к SQL:**

```bash
sudo dnf install -y mysql
MYSQL_PS1="[\d] \R:\m:\s> " mysql -h192.168.55.205 -P2881 -uroot@sys -p'qaz123' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"
```

Параметры:
- `MYSQL_PS1="[\d] \R:\m:\s> "` — задаёт формат командной строки со временем (очень удобно при долгих командах)
- `--init-command="..."` — устанавливает большой таймаут
- `-uroot@sys` — системный тенант
- `-A` — значительно ускоряет подключение (не читает список таблиц)
- `-D oceanbase` — основная база со статистикой

---

## Пример реального конфигурационного файла

Установка одного сервера в одну зону.

Документация: https://en.oceanbase.com/docs/community-obd-en-10000000001022728

`root_password: 'qaz123'` — задавал в web сам, остальные сгенерировал. Есть шифровальщик, который шифрует их по ключу с парольной фразой, лежащему на каждом сервере. Надо смотреть конфиг агента, чтобы увидеть, зашифрован ли пароль.

Документация по агенту: https://github.com/oceanbase/obagent/tree/master/docs

Документация управления настройками агента: https://en.oceanbase.com/docs/community-obd-en-10000000002136449

```yaml
user:
  username: admin
  port: 22
  password:
oceanbase-ce:
  version: 4.4.1.0
  release: 100000032025101610.el8
  package_hash: 1309bc20bff8d9e64d19e9cf7433798a7c696452
  192.168.55.205:
    zone: zone1
  servers:
  - 192.168.55.205
  global:
    appname: obc1
    root_password: 'qaz123'
    mysql_port: 2881
    rpc_port: 2882
    data_dir: /data/1
    redo_dir: /data/log1
    obshell_port: 2886
    home_path: /data/obc1/oceanbase
    scenario: htap
    cluster_id: 1762092279
    ocp_agent_monitor_password: y6i27N30AD
    enable_syslog_wf: false
    max_syslog_file_count: 16
    production_mode: false
    memory_limit: 8G
    datafile_size: 18G
    system_memory: 1G
    log_disk_size: 22G
    cpu_count: 8
    datafile_maxsize: 22G
    datafile_next: 2G
obagent:
  version: 4.2.2
  package_hash: bf152b880953c2043ddaf80d6180cf22bb8c8ac2
  release: 100000042024011120.el8
  servers:
  - 192.168.55.205
  global:
    monagent_http_port: 8088
    mgragent_http_port: 8089
    home_path: /data/obc1/obagent
    http_basic_auth_password: 91nMuGN1n
    ob_monitor_status: active
  depends:
  - oceanbase-ce
```

### задать часовой пояс
```sql
SET GLOBAL time_zone = '+03:00';
```
