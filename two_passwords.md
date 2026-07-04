

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
