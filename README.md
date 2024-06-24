# MariaDB TDE test

## 前提

- docker がインストールされていること
- mysql-client がインストールされていること
- その他 openssl などのコマンドが実行可能であること

## 準備

```shell
mkdir mariadb-tde-test
cd mariadb-tde-test
```

データが暗号化されているかを検証するため、データ保管領域を用意しておく。非暗号化用：`data` 暗号化用：`data-enc`

```shell
mkdir data data-enc
```

MariaDB起動確認

```shell
docker pull mariadb:lts
docker run -d --rm --name mariadb \
  -e MARIADB_ROOT_PASSWORD=my-secret-pw \
  -v ./data:/var/lib/mysql \
  -p 3306:3306 \
  mariadb:lts
mysql -h 127.0.0.1 -P 3306 -u root -p
```

バージョンと暗号化設定確認

```sql
mysql> select version();
+------------------------+
| version()              |
+------------------------+
| 11.4.2-MariaDB-ubu2404 |
+------------------------+
1 row in set (0.00 sec)

mysql> SHOW GLOBAL VARIABLES like 'innodb%encrypt%';
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| innodb_default_encryption_key_id | 1     |
| innodb_encrypt_log               | OFF   |
| innodb_encrypt_tables            | OFF   |
| innodb_encrypt_temporary_tables  | OFF   |
| innodb_encryption_rotate_key_age | 1     |
| innodb_encryption_rotation_iops  | 100   |
| innodb_encryption_threads        | 0     |
+----------------------------------+-------+
7 rows in set (0.00 sec)
```

一旦停止

```shell
docker stop mariadb
```

## 設定ファイル作成

```shell
vi ./my.cnf
```

```ini
[mysqld]
plugin_load_add = file_key_management
file_key_management_filename = /etc/mysql/keys/innodb_key
innodb_encrypt_tables = ON
innodb_encrypt_log = ON
```

## キーファイル作成

```shell
mkdir keys
touch keys/innodb_key
(echo -n "1;" ; openssl rand -hex 32 ) | tee -a keys/innodb_key
```

## 起動

```shell
docker run -d --rm --name mariadb \
  -e MARIADB_ROOT_PASSWORD=my-secret-pw \
  -v ./data-enc:/var/lib/mysql \
  -v ./my.cnf:/etc/mysql/my.cnf \
  -v ./keys:/etc/mysql/keys \
  -p 3306:3306 \
  mariadb:lts
```

```sql
mysql> SHOW GLOBAL VARIABLES like 'innodb%encrypt%';
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| innodb_default_encryption_key_id | 1     |
| innodb_encrypt_log               | ON    |
| innodb_encrypt_tables            | ON    |
| innodb_encrypt_temporary_tables  | OFF   |
| innodb_encryption_rotate_key_age | 1     |
| innodb_encryption_rotation_iops  | 100   |
| innodb_encryption_threads        | 0     |
+----------------------------------+-------+
7 rows in set (0.00 sec)
```

適当にテーブルを作ってデータをインサート。

```sql
mysql> CREATE DATABASE test;
Query OK, 1 row affected (0.01 sec)

mysql> use test
Database changed

mysql> CREATE TABLE test_table (
    id INT PRIMARY KEY,
    data VARCHAR(100)
) ENGINE=InnoDB;
Query OK, 0 rows affected (0.02 sec)

mysql> INSERT INTO test_table (id, data) VALUES (1, 'Hello, World!');
Query OK, 1 row affected (0.01 sec)

mysql> SELECT * FROM test_table;
+----+---------------+
| id | data          |
+----+---------------+
|  1 | Hello, World! |
+----+---------------+
1 row in set (0.00 sec)
```

fgrep で見てみる。
データファイルはヒットしない。

```shell
fgrep -r Hello data-enc 
Binary file data-enc/mysql/help_topic.MAD matches
```

暗号化設定をせずに起動して同様にデータをインサートした場合は、以下のようにデータファイルとREDOログファイルがヒットした。

```shell
fgrep -r Hello data    
Binary file data/test/test_table.ibd matches
Binary file data/ib_logfile0 matches
Binary file data/mysql/help_topic.MAD matches
```

## TODO

- innodb_encrypt_temporary_tables も ON にしたほうがいいのではないか
- キーファイルの暗号化をしてよりセキュアにできるが、今回それをしていない
- innodb_encrypt_tables は ON だとテーブルの暗号化を有効にするが、暗号化されていないテーブルの作成も許される。`FORCE` にすることで暗号化されていないテーブルの作成を許可しないようにできる。
- パフォーマンスがどの程度下がるか未検証


## see

- https://hub.docker.com/_/mariadb
- https://mariadb.com/kb/en/innodb-enabling-encryption/
