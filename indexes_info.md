
```sql
[tpchdb] 13:04:21> SELECT
    INDEX_NAME,
    GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS columns,
    NON_UNIQUE,
    INDEX_TYPE
FROM information_schema.STATISTICS
GROUP BY INDEX_NAME, NON_UNIQUE, INDEX_TYPE;WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'lineitem'
GROUP BY INDEX_NAME, NON_UNIQUE, INDEX_TYPE;
+------------+-------------------------+------------+------------+
| INDEX_NAME | columns                 | NON_UNIQUE | INDEX_TYPE |
+------------+-------------------------+------------+------------+
| l_rd       | l_receiptdate           |          1 | BTREE      |
| l_sk_pk    | l_suppkey,l_partkey     |          1 | BTREE      |
| l_cd       | l_commitdate            |          1 | BTREE      |
| l_sd       | l_shipdate              |          1 | BTREE      |
| l_sk       | l_suppkey               |          1 | BTREE      |
| l_ok       | l_orderkey              |          1 | BTREE      |
| PRIMARY    | l_orderkey,l_linenumber |          0 | BTREE      |
| l_pk       | l_partkey               |          1 | BTREE      |
| l_pk_sk    | l_partkey,l_suppkey     |          1 | BTREE      |
+------------+-------------------------+------------+------------+
9 rows in set (0.03 sec)



[tpchdb] 13:04:47> SHOW INDEX FROM lineitem;
+----------+------------+----------+--------------+---------------+-----------+-------------+----------+--------+------+------------+-----------+---------------+---------+------------+
| Table    | Non_unique | Key_name | Seq_in_index | Column_name   | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment   | Index_comment | Visible | Expression |
+----------+------------+----------+--------------+---------------+-----------+-------------+----------+--------+------+------------+-----------+---------------+---------+------------+
| lineitem |          0 | PRIMARY  |            1 | l_orderkey    | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          0 | PRIMARY  |            2 | l_linenumber  | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_ok     |            1 | l_orderkey    | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_pk     |            1 | l_partkey     | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_sk     |            1 | l_suppkey     | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_sd     |            1 | l_shipdate    | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_cd     |            1 | l_commitdate  | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_rd     |            1 | l_receiptdate | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_pk_sk  |            1 | l_partkey     | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_pk_sk  |            2 | l_suppkey     | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_sk_pk  |            1 | l_suppkey     | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
| lineitem |          1 | l_sk_pk  |            2 | l_partkey     | A         |        NULL | NULL     | NULL   |      | BTREE      | available |               | YES     | NULL       |
+----------+------------+----------+--------------+---------------+-----------+-------------+----------+--------+------+------------+-----------+---------------+---------+------------+
12 rows in set (0.00 sec)

[tpchdb] 13:05:51>
```