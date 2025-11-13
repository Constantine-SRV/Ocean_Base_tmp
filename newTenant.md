
### СОЗДАНИЕ ТЕНАНТА НА OCEANBASE STANDALONE (10GB RAM)

```
-- Шаг 1: Увеличиваем memory_limit с 8GB до 9GB (оставляем 1GB для ОС)
ALTER SYSTEM SET memory_limit = '9G';

-- Шаг 2: Устанавливаем минимальный размер памяти для resource pool в 1.5GB
ALTER SYSTEM SET __min_full_resource_pool_memory = '1610612736';  -- 1.5GB в байтах

-- Шаг 3: Создаём минимальный unit для sys tenant (1.5GB)
CREATE RESOURCE UNIT sys_unit_small
  MAX_CPU = 1,
  MIN_CPU = 1,
  MEMORY_SIZE = '1536M',  -- 1.5GB
  LOG_DISK_SIZE = '2G';

-- Шаг 4: Переключаем sys tenant на минимальный unit
ALTER RESOURCE POOL sys_pool UNIT = 'sys_unit_small';

-- Шаг 5: Проверяем освобожденную память
SELECT SVR_IP, 
       ROUND(MEM_CAPACITY/1024/1024/1024, 2) as MEM_CAPACITY_GB,
       ROUND(MEM_ASSIGNED/1024/1024/1024, 2) as ASSIGNED_GB,
       ROUND((MEM_CAPACITY - MEM_ASSIGNED)/1024/1024/1024, 2) as FREE_GB
FROM oceanbase.GV$OB_SERVERS;


-- Шаг 6: Удаляем ненужные unit configs (опционально, для чистоты)
DROP RESOURCE UNIT IF EXISTS sys_unit_config;
DROP RESOURCE UNIT IF EXISTS sys_unit_mini;
DROP RESOURCE UNIT IF EXISTS app_unit;

-- Шаг 7: Создаём unit для app_tenant (6.5GB - максимум что доступно)
CREATE RESOURCE UNIT app_unit
  MAX_CPU = 4,
  MIN_CPU = 3,
  MEMORY_SIZE = '6656M',  -- 6.5GB = 6656MB
  LOG_DISK_SIZE = '3G';

-- Шаг 8: Создаём resource pool
CREATE RESOURCE POOL app_pool
  UNIT = 'app_unit',
  UNIT_NUM = 1,
  ZONE_LIST = ('zone1');

-- Шаг 9: Создаём tenant
CREATE TENANT app_tenant
  RESOURCE_POOL_LIST = ('app_pool'),
  CHARSET = 'utf8mb4',
  PRIMARY_ZONE = 'zone1',
  LOCALITY = 'F@zone1';

-- Шаг 10: Проверяем созданные тенанты
SELECT TENANT_NAME, TENANT_ID, TENANT_TYPE, COMPATIBILITY_MODE
FROM oceanbase.DBA_OB_TENANTS;

-- Шаг 11: Проверяем распределение ресурсов
SELECT t.TENANT_NAME, 
       uc.NAME as UNIT_CONFIG, 
       ROUND(uc.MEMORY_SIZE/1024/1024/1024, 2) as MEMORY_GB,
       uc.MAX_CPU, 
       uc.MIN_CPU
FROM oceanbase.DBA_OB_TENANTS t
JOIN oceanbase.DBA_OB_RESOURCE_POOLS rp ON t.TENANT_ID = rp.TENANT_ID
JOIN oceanbase.DBA_OB_UNIT_CONFIGS uc ON rp.UNIT_CONFIG_ID = uc.UNIT_CONFIG_ID;

SELECT t.TENANT_NAME, rp.NAME as POOL_NAME, uc.NAME as UNIT_NAME,
ROUND(uc.MEMORY_SIZE/1024/1024/1024, 2) as MEMORY_GB
FROM oceanbase.DBA_OB_TENANTS t JOIN oceanbase.DBA_OB_RESOURCE_POOLS rp ON t.TENANT_ID = rp.TENANT_ID
JOIN oceanbase.DBA_OB_UNIT_CONFIGS uc ON rp.UNIT_CONFIG_ID = uc.UNIT_CONFIG_ID ;
+-------------+-----------+----------------+-----------+
| TENANT_NAME | POOL_NAME | UNIT_NAME      | MEMORY_GB |
+-------------+-----------+----------------+-----------+
| sys         | sys_pool  | sys_unit_small |      1.50 |
| app_tenant  | app_pool  | app_unit       |      6.50 |
+-------------+-----------+----------------+-----------+
2 rows in set (0.01 sec)

[oceanbase] 15:04:13>


```
