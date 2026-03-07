### Попытки войти с сервера 192.168.73.31 напрямую на сервер ОБ 192.168.73.205 и через прокси 192.168.73.200 успешно и не успешно
```bash
ubuntu@ubunru31:~$ MYSQL_PS1="[\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 -uta@app_tenant -p'qaz123' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"

[oceanbase] 08:56:25> exit
Bye

ubuntu@ubunru31:~$ MYSQL_PS1="[\d] \R:\m:\s> " mysql -h192.168.73.200 -P2883 -uta@app_tenant -p'qaz124' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (42000): Access denied for user 'ta'@'xxx.xxx.xxx.xxx' (using password: YES)
ubuntu@ubunru31:~$


ubuntu@ubunru31:~$ MYSQL_PS1="[\d] \R:\m:\s> " mysql -h192.168.73.205 -P2881 -uta@app_tenant -p'qaz123' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"

[oceanbase] 08:57:02> exit
Bye

ubuntu@ubunru31:~$ MYSQL_PS1="[\d] \R:\m:\s> " mysql -h192.168.73.205 -P2881 -uta@app_tenant -p'qaz12' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (42000): Access denied for user 'ta'@'xxx.xxx.xxx.xxx' (using password: YES)
ubuntu@ubunru31:~$

```


## логи с сервера - логины успешные и нет
```bash

#!/bin/bash
# ob_audit_login.sh [minutes]

MINUTES=${1:-60}
LOG_DIR="/data/obc1/oceanbase/log"
SINCE=$(date -d "-${MINUTES} minutes" "+%Y-%m-%d %H:%M")
EXCLUDE_USERS="ocp_monitor|proxy_ro"

echo "=== Logins since ${SINCE} ==="
echo "TIMESTAMP|CLIENT_IP|TENANT|USER|SESSION_ID|SSL|CLIENT|RESULT"

grep "MySQL LOGIN" ${LOG_DIR}/observer.log* 2>/dev/null | \
  grep -Ev "user_name=(${EXCLUDE_USERS})," | \
  grep -Ev "client_ip=127.0.0.1.*user_name=root" | \
  awk -v since="${SINCE}" '
  {
    match($0, /\[([0-9-]+ [0-9:.]+)\]/, ts); timestamp = ts[1]
    if (timestamp < since) next
    match($0, /client_ip=([^,]+)/, a); client_ip = a[1]
    match($0, /tenant_name=([^,]+)/, a); tenant = a[1]
    match($0, /user_name=([^,]+)/, a); user = a[1]
    match($0, /sessid=([^,]+)/, a); sessid = a[1]
    match($0, /use_ssl=([^,]+)/, a); ssl = (a[1] == "true") ? "Y" : "N"
    match($0, /proc_ret=([^,]+)/, a); result = (a[1] == "0") ? "OK" : "FAIL"
    match($0, /client_type_=([0-9]+)/, a); ctype = a[1]
    if (ctype == "1") client = "OBCLIENT"
    else if (ctype == "2") client = "JDBC"
    else if (ctype == "3") client = "MYSQL_CLI"
    else client = "TYPE_" ctype
    printf "%s|%s|%s|%s|%s|%s|%s|%s\n", timestamp, client_ip, tenant, user, sessid, ssl, client, result
  }' | sort -t'|' -k1
```
рез
```
=== Logins since 2026-03-07 11:01 ===
TIMESTAMP|CLIENT_IP|TENANT|USER|SESSION_ID|SSL|CLIENT|RESULT
2026-03-07 11:56:25.191480|"192.168.73.200"|app_tenant|ta|3221553882|N|MYSQL_CLI|OK
2026-03-07 11:56:42.158420|"192.168.73.200"|app_tenant|ta|3221580239|N|MYSQL_CLI|FAIL
2026-03-07 11:57:02.973528|"192.168.73.31"|app_tenant|ta|3221616163|Y|MYSQL_CLI|OK
2026-03-07 11:57:16.866830|"192.168.73.31"|app_tenant|ta|3221638170|Y|MYSQL_CLI|FAIL

```

## события логофф в отдельном логе
их очень много т.к. отфильтровать служебные нельхзя сопоставляются с первым выводом по SESSION_ID 
```bash
 #!/bin/bash
# ob_audit_logoff.sh [minutes]

MINUTES=${1:-10}
LOG_DIR="/data/obc1/oceanbase/log"
SINCE=$(date -d "-${MINUTES} minutes" "+%Y-%m-%d %H:%M")

echo "=== Logoffs since ${SINCE} ==="
echo "TIMESTAMP|TENANT_ID|SESSION_ID"

grep "connection close" ${LOG_DIR}/observer.log* 2>/dev/null | \
  awk -v since="${SINCE}" '
  {
    match($0, /\[([0-9-]+ [0-9:.]+)\]/, ts); timestamp = ts[1]
    if (timestamp < since) next
    match($0, /sessid=([^,]+)/, a); sessid = a[1]
    match($0, /tenant_id=([^,]+)/, a); tenant_id = a[1]
    printf "%s|%s|%s\n", timestamp, tenant_id, sessid
  }' | sort -t'|' -k1
```

результат
```
=== Logoffs since 2026-03-07 11:53 ===
TIMESTAMP|TENANT_ID|SESSION_ID

2026-03-07 11:56:25.641478|1|3221556693
2026-03-07 11:56:25.641695|1|3221550234
2026-03-07 11:56:26.548863|1|3221547086
2026-03-07 11:56:29.777561|1002|3221553882
2026-03-07 11:56:40.527714|1|3221579370
2026-03-07 11:56:40.528901|1|3221579192
2026-03-07 11:56:40.528972|1|3221580291
2026-03-07 11:56:40.530218|1|3221580519
2026-03-07 11:56:40.530251|1|3221579559
2026-03-07 11:56:40.533150|1|3221576986
2026-03-07 11:56:40.538655|1|3221576570
2026-03-07 11:56:40.541671|1|3221580201
2026-03-07 11:56:40.541756|1|3221579747
2026-03-07 11:56:40.562502|1|3221574046
2026-03-07 11:56:40.790034|1|3221580626
2026-03-07 11:56:41.539729|1|3221580184
2026-03-07 11:56:41.637380|1|3221579663
2026-03-07 11:56:42.158549|1002|3221580239
2026-03-07 11:56:55.527737|1|3221601636
2026-03-07 11:56:55.529174|1|3221604253
2026-03-07 11:56:55.532300|1|3221601847
2026-03-07 11:56:55.533735|1|3221603835
2026-03-07 11:56:55.534171|1|3221602569
2026-03-07 11:56:55.535437|1|3221603180
2026-03-07 11:56:55.535481|1|3221602668
2026-03-07 11:56:55.535520|1|3221603288
2026-03-07 11:56:55.536827|1|3221603118
2026-03-07 11:56:55.536927|1|3221604167
2026-03-07 11:56:55.541179|1|3221603946
2026-03-07 11:56:55.546714|1|3221603356
2026-03-07 11:56:55.549556|1|3221603853
2026-03-07 11:56:55.563905|1|3221604268
2026-03-07 11:56:55.565682|1|3221602827
2026-03-07 11:56:55.829096|1|3221599679
2026-03-07 11:56:56.558227|1|3221603740
2026-03-07 11:57:04.171874|1|3221603766
2026-03-07 11:57:09.481230|1002|3221616163
2026-03-07 11:57:10.530723|1|3221626971
2026-03-07 11:57:10.530801|1|3221627284
2026-03-07 11:57:10.531350|1|3221626089
2026-03-07 11:57:10.533901|1|3221627170
2026-03-07 11:57:10.534067|1|3221623284
2026-03-07 11:57:10.534362|1|3221612011
2026-03-07 11:57:10.536172|1|3221625272
2026-03-07 11:57:10.546498|1|3221627273
2026-03-07 11:57:10.565122|1|3221626726
2026-03-07 11:57:10.565259|1|3221626462
2026-03-07 11:57:10.646862|1|3221626201
2026-03-07 11:57:10.647169|1|3221622122
2026-03-07 11:57:11.552387|1|3221626700
2026-03-07 11:57:16.866956|1002|3221638170
2026-03-07 11:57:25.528587|1|3221649530
2026-03-07 11:57:25.528714|1|3221647939
2026-03-07 11:57:25.528883|1|3221649736
2026-03-07 11:57:25.529015|1|3221650549
2026-03-07 11:57:25.529496|1|3221650586
```

# Прокси
## через проксивсе логины и успешные и нет
``` bash
MINUTES=60
LOG_DIR="/data/obc1/obproxy/log"
SINCE=$(date -d "-${MINUTES} minutes" "+%Y-%m-%d %H:%M")

echo "TIMESTAMP|CLIENT_IP|TENANT|USER|CLUSTER|CS_ID|PROXY_SESSID|SERVER_SESSID"

grep -E "server session born|succ to set proxy_sessid" ${LOG_DIR}/obproxy.log* 2>/dev/null | \
awk -v since="${SINCE}" '
/server session born/ {
  match($0, /\[([0-9-]+ [0-9:.]+)\]/, ts)
  if (ts[1] < since) next

  match($0, /cs_id=([0-9]+)/, a); cs_id = a[1]
  match($0, /cluster_name:"([^"]+)"/, a); cluster[cs_id] = a[1]
  match($0, /tenant_name:"([^"]+)"/, a); tenant[cs_id] = a[1]
  match($0, /user_name:"([^"]+)"/, a); user[cs_id] = a[1]
}

/succ to set proxy_sessid/ {
  match($0, /\[([0-9-]+ [0-9:.]+)\]/, ts); timestamp = ts[1]
  if (timestamp < since) next

  match($0, /cs_id=([0-9]+)/, a); cs_id = a[1]
  match($0, /proxy_sessid=([0-9]+)/, a); proxy_sessid = a[1]
  match($0, /server_sessid=([0-9]+)/, a); server_sessid = a[1]
  match($0, /client_addr="([^"]+)"/, a); client_addr = a[1]

  t = tenant[cs_id]; u = user[cs_id]; c = cluster[cs_id]
  if (u == "proxyro" || t == "") next

  printf "%s|%s|%s|%s|%s|%s|%s|%s\n", timestamp, client_addr, t, u, c, cs_id, proxy_sessid, server_sessid
}
' | sort -t'|' -k1
```
результат
```
TIMESTAMP|CLIENT_IP|TENANT|USER|CLUSTER|CS_ID|PROXY_SESSID|SERVER_SESSID
2026-03-07 11:56:25.185604|192.168.73.31:49494|app_tenant|ta|obc1|22|13882407183691481105|3221553882
2026-03-07 11:56:42.153339|192.168.73.31:45970|app_tenant|ta|obc1|23|13882407183691481106|3221580239
```
### не успешные логины в отдельном файле
```bash
MINUTES=60
LOG_DIR="/data/obc1/obproxy/log"
SINCE=$(date -d "-${MINUTES} minutes" "+%Y-%m-%d %H:%M")

echo "TIMESTAMP|CLIENT_IP|TENANT|USER|CLUSTER|ERROR_CODE|CS_ID|PROXY_SESSID|SERVER_SESSID"

grep -E "error_transfer.*OB_MYSQL_COM_LOGIN|client session do_io_close" ${LOG_DIR}/obproxy.log* 2>/dev/null | \
awk -v since="${SINCE}" '
/error_transfer.*OB_MYSQL_COM_LOGIN/ {
  match($0, /\[([0-9-]+ [0-9:.]+)\]/, ts); timestamp = ts[1]
  if (timestamp < since) next

  match($0, /sm_id=([0-9]+)/, a); sm_id = a[1]
  match($0, /client_ip=\{([^}]+)\}/, a); client_ip = a[1]

  fail_time[sm_id] = timestamp
  fail_ip[sm_id] = client_ip
}

/client session do_io_close/ {
  match($0, /\[([0-9-]+ [0-9:.]+)\]/, ts); timestamp = ts[1]
  if (timestamp < since) next

  match($0, /cs_id=([0-9]+)/, a); cs_id = a[1]

  if (!(cs_id in fail_time)) next

  match($0, /proxy_sessid=([0-9]+)/, a); proxy_sessid = a[1]
  match($0, /last_server_sessid=([0-9]+)/, a); server_sessid = a[1]
  match($0, /cluster=([^,]+)/, a); cluster = a[1]
  match($0, /tenant=([^,]+)/, a); tenant = a[1]
  match($0, /user=([^,]+)/, a); user = a[1]

  printf "%s|%s|%s|%s|%s|1045|%s|%s|%s\n", fail_time[cs_id], fail_ip[cs_id], tenant, user, cluster, cs_id, proxy_sessid, server_sessid
}
' | sort -t'|' -k1
```
результат
```bash
TIMESTAMP|CLIENT_IP|TENANT|USER|CLUSTER|ERROR_CODE|CS_ID|PROXY_SESSID|SERVER_SESSID
2026-03-07 11:56:42.162593|192.168.73.31:45970|app_tenant|ta|obc1|1045|23|13882407183691481106|3221580239
```

### события логофф
```bash
MINUTES=60
LOG_DIR="/data/obc1/obproxy/log"
SINCE=$(date -d "-${MINUTES} minutes" "+%Y-%m-%d %H:%M")

echo "TIMESTAMP|CLIENT_IP|TENANT|USER|CLUSTER|CS_ID|PROXY_SESSID"

grep "handle_server_connection_break" ${LOG_DIR}/obproxy.log* 2>/dev/null | \
grep "COM_QUIT" | \
awk -v since="${SINCE}" '
{
  match($0, /\[([0-9-]+ [0-9:.]+)\]/, ts); timestamp = ts[1]
  if (timestamp < since) next

  match($0, /client_ip=\{([^}]+)\}/, a); client_ip = a[1]
  match($0, /cs_id=([0-9]+)/, a); cs_id = a[1]
  match($0, /proxy_sessid=([0-9]+)/, a); proxy_sessid = a[1]

  # proxy_user_name=ta@app_tenant#obc1
  match($0, /proxy_user_name=([^,]+)/, a); proxy_user = a[1]

  # Разбираем user@tenant#cluster
  split(proxy_user, parts, "@")
  user = parts[1]
  split(parts[2], parts2, "#")
  tenant = parts2[1]
  cluster = parts2[2]

  if (user == "proxyro" || tenant == "") next

  printf "%s|%s|%s|%s|%s|%s|%s\n", timestamp, client_ip, tenant, user, cluster, cs_id, proxy_sessid
}' | sort -t'|' -k1
```

результат
```bash
TIMESTAMP|CLIENT_IP|TENANT|USER|CLUSTER|CS_ID|PROXY_SESSID
2026-03-07 11:56:29.777692|192.168.73.31:49494|app_tenant|ta|obc1|22|13882407183691481105
```
