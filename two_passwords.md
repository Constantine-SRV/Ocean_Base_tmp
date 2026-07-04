

## commands
```
CREATE USER 'dp_test'@'%' IDENTIFIED BY 'qaz123';
```
```
ALTER USER 'dp_test'@'%' IDENTIFIED BY 'qaz123new' RETAIN CURRENT PASSWORD;

SELECT * FROM __all_virtual_user WHERE user_name='dp_test'\G
```
```
ALTER USER 'dp_test'@'%' DISCARD OLD PASSWORD;
```
## full output

```
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -uroot@app_tenant#obc442 -p'qaz123' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 21
Server version: 5.6.25 OceanBase_CE_CL 4.4.2.1 (r1-9bbb1020419c85aeace403c593859a165aa72470) (Built Jul  4 2026 14:15:08)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

[root|oceanbase] 19:28:38> CREATE USER 'dp_test'@'%' IDENTIFIED BY 'qaz123';
Query OK, 0 rows affected (0.10 sec)

[root|oceanbase] 19:29:18> exit
Bye
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -udp_test@app_tenant#obc442 -p'qaz123'  --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 22
Server version: 5.6.25 OceanBase_CE_CL 4.4.2.1 (r1-9bbb1020419c85aeace403c593859a165aa72470) (Built Jul  4 2026 14:15:08)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

[dp_test|(none)] 19:29:48> exit
Bye
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -uroot@app_tenant#obc442 -p'qaz123' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 23
Server version: 5.6.25 OceanBase_CE_CL 4.4.2.1 (r1-9bbb1020419c85aeace403c593859a165aa72470) (Built Jul  4 2026 14:15:08)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

[root|oceanbase] 19:29:58> ALTER USER 'dp_test'@'%' IDENTIFIED BY 'qaz123new' RETAIN CURRENT PASSWORD;
Query OK, 0 rows affected (0.30 sec)

[root|oceanbase] 19:30:04>
[root|oceanbase] 19:30:04> SELECT * FROM __all_virtual_user WHERE user_name='dp_test'\G
*************************** 1. row ***************************
                tenant_id: 1004
                  user_id: 501026
               gmt_create: 2026-07-05 00:29:18.025034
             gmt_modified: 2026-07-05 00:30:04.205191
                user_name: dp_test
                     host: %
                   passwd: *6eb748de45f519642828f4dd2f78eab2137c0069
                     info:
               priv_alter: 0
              priv_create: 0
              priv_delete: 0
                priv_drop: 0
        priv_grant_option: 0
              priv_insert: 0
              priv_update: 0
              priv_select: 0
               priv_index: 0
         priv_create_view: 0
           priv_show_view: 0
             priv_show_db: 0
         priv_create_user: 0
               priv_super: 0
                is_locked: 0
             priv_process: 0
      priv_create_synonym: 0
                 ssl_type: 0
               ssl_cipher:
              x509_issuer:
             x509_subject:
                     type: 0
               profile_id: -1
    password_last_changed: 2026-07-05 00:30:04.204762
                priv_file: 0
        priv_alter_tenant: 0
        priv_alter_system: 0
priv_create_resource_pool: 0
priv_create_resource_unit: 0
          max_connections: 0
     max_user_connections: 0
          priv_repl_slave: 0
         priv_repl_client: 0
  priv_drop_database_link: 0
priv_create_database_link: 0
              priv_others: 0
                    flags: 0
                   plugin: mysql_native_password
             old_password: *e927669432d80af9548191114cb0d181ada2d0d3
  old_password_start_time: 1783182604204758
1 row in set (0.03 sec)

[root|oceanbase] 19:30:06> exit
Bye
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -udp_test@app_tenant#obc442 -p'qaz123'  --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 25
Server version: 5.6.25 OceanBase_CE_CL 4.4.2.1 (r1-9bbb1020419c85aeace403c593859a165aa72470) (Built Jul  4 2026 14:15:08)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

[dp_test|(none)] 19:30:16> exit
Bye
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -udp_test@app_tenant#obc442 -p'qaz123new'  --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 26
Server version: 5.6.25 OceanBase_CE_CL 4.4.2.1 (r1-9bbb1020419c85aeace403c593859a165aa72470) (Built Jul  4 2026 14:15:08)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

[dp_test|(none)] 19:30:27> exit
Bye
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -uroot@app_tenant#obc442 -p'qaz123' -A -D oceanbase --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 27
Server version: 5.6.25 OceanBase_CE_CL 4.4.2.1 (r1-9bbb1020419c85aeace403c593859a165aa72470) (Built Jul  4 2026 14:15:08)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

[root|oceanbase] 19:30:34> ALTER USER 'dp_test'@'%' DISCARD OLD PASSWORD;
Query OK, 0 rows affected (0.04 sec)

[root|oceanbase] 19:30:41> exit
Bye
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -udp_test@app_tenant#obc442 -p'qaz123'  --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (42000): Access denied for user 'dp_test'@'xxx.xxx.xxx.xxx' (using password: YES)
[costa@OBM55200 ~]$ MYSQL_PS1="[\u|\d] \R:\m:\s> " mysql -h192.168.55.200 -P2883 -udp_test@app_tenant#obc442 -p'qaz123new'  --init-command="SET SESSION ob_query_timeout = 10000000000"
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 29
Server version: 5.6.25 OceanBase_CE_CL 4.4.2.1 (r1-9bbb1020419c85aeace403c593859a165aa72470) (Built Jul  4 2026 14:15:08)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

[dp_test|(none)] 19:30:55> exit
Bye
[costa@OBM55200 ~]$

```
