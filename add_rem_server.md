# Удаление и добавление серверов OceanBase (OBD)



## Часть 1: Удаление серверов

### 1.1. Изменить locality тенанта (убрать удаляемые зоны)

Подключиться к sys tenant:

Изменить locality, убрав зоны которые будут удалены:

```sql
-- Пример: оставляем только zone1-zone4, удаляем zone5-zone6
ALTER TENANT bench LOCALITY = 'FULL{1}@zone1,FULL{1}@zone2,FULL{1}@zone3,COLUMNSTORE{1}@zone4';
```

### 1.2. Сузить resource pool тенанта

```sql
ALTER RESOURCE POOL bench_pool ZONE_LIST = ('zone1','zone2','zone3','zone4');
```

### 1.3. Удалить серверы из кластера

```sql
ALTER SYSTEM DELETE SERVER '192.168.38.57:2882' ZONE='zone5';
ALTER SYSTEM DELETE SERVER '192.168.38.219:2882' ZONE='zone6';
```

### 1.4. Удалить зоны

```sql
ALTER SYSTEM DELETE ZONE 'zone5';
ALTER SYSTEM DELETE ZONE 'zone6';
```

### 1.5. Редактировать конфиги OBD

На управляющем сервере:

```bash
# Сделать бэкапы
cp ~/.obd/cluster/obcl01/config.yaml ~/.obd/cluster/obcl01/config.yaml.bak
cp ~/.obd/cluster/obcl01/inner_config.yaml ~/.obd/cluster/obcl01/inner_config.yaml.bak

# Удалить записи о серверах из обоих файлов
vi ~/.obd/cluster/obcl01/config.yaml
vi ~/.obd/cluster/obcl01/inner_config.yaml
```

> **Важно:** Удалить записи о серверах нужно из ОБОИХ файлов!

### 1.6. Остановить процессы на удаляемых серверах

На каждом удаляемом сервере:

```bash
pkill -9 observer
pkill -9 obagent
pkill -9 obshell
```

### 1.7. Очистить каталоги данных

> **Критично:** Использовать `{*,.*}` для удаления скрытых файлов!

```bash
sudo rm -rf /obdata/1/{*,.*} 2>/dev/null
sudo rm -rf /obredo/log1/{*,.*} 2>/dev/null
sudo rm -rf /ob/obcl01/oceanbase/store/{*,.*} 2>/dev/null
sudo rm -rf /ob/obcl01/oceanbase/etc/{*,.*} 2>/dev/null
sudo rm -rf /ob/obcl01/oceanbase/log/{*,.*} 2>/dev/null
sudo rm -rf /ob/obcl01/oceanbase/run/{*,.*} 2>/dev/null
```

Проверить что директории пусты:

```bash
ls -la /obdata/1/
ls -la /obredo/log1/
```

---

## Часть 2: Добавление серверов

### 2.1. Создать YAML-файл для scale_out

Создать файл `add_servers.yaml`:

```yaml
oceanbase-ce:
  192.168.38.57:
    zone: zone5
  192.168.38.219:
    zone: zone6
  servers:
  - 192.168.38.57
  - 192.168.38.219
obagent:
  servers:
  - 192.168.38.57
  - 192.168.38.219
```

### 2.2. Запустить scale_out

```bash
obd cluster scale_out obcl01 -c add_servers.yaml
```

### 2.3. Расширить resource pool тенанта

Подключиться к sys tenant:

```sql
ALTER RESOURCE POOL bench_pool ZONE_LIST = ('zone1','zone2','zone3','zone4','zone5','zone6');
```

### 2.4. Настроить locality тенанта

```sql
ALTER TENANT bench LOCALITY = 'FULL{1}@zone1,FULL{1}@zone2,FULL{1}@zone3,COLUMNSTORE{1}@zone4,COLUMNSTORE{1}@zone5,COLUMNSTORE{1}@zone6';
```

### 2.5. Дождаться репликации данных

Проверить прогресс:

```sql
SELECT svr_ip, ROUND(total_size/1024/1024/1024, 2) AS total_gb,
       ROUND(used_size/1024/1024/1024, 2) AS used_gb 
FROM oceanbase.__all_virtual_disk_stat;
```

---

## Часть 3: Полезные проверки

### Серверы в кластере

```sql
SELECT svr_ip, svr_port, zone, status, with_rootserver 
FROM oceanbase.DBA_OB_SERVERS;
```

### Зоны кластера

```sql
SELECT zone, status FROM oceanbase.DBA_OB_ZONES;
```

### Locality тенанта

```sql
SELECT tenant_name, locality 
FROM oceanbase.DBA_OB_TENANTS 
WHERE tenant_name = 'bench';
```

### Resource pool тенанта

```sql
SELECT t.tenant_name, p.name AS pool_name, p.zone_list, p.unit_count
FROM oceanbase.DBA_OB_TENANTS t
JOIN oceanbase.DBA_OB_RESOURCE_POOLS p ON t.tenant_id = p.tenant_id
WHERE t.tenant_name = 'bench';
```

### Использование диска по серверам

```sql
SELECT svr_ip, 
       ROUND(total_size/1024/1024/1024, 2) AS total_gb,
       ROUND(used_size/1024/1024/1024, 2) AS used_gb 
FROM oceanbase.__all_virtual_disk_stat;
```

---


