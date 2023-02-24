# mysql-breaking-change
mysqlにログインして、以下のコマンドを実行

`docker-compose up -d`

### mysql5.7側の操作
```.sh
mysql -h 127.0.0.1 --port 3320 -u root -proot
```

### mysql8.0側の操作
```.sh
mysql -h 127.0.0.1 --port 3321 -u root -proot
```

### 以下mysql5.7,8.0共通のユーザー作成

restrictionユーザーはアプリ用のユーザーを想定し、最低限のDB操作のみ可能な権限を付与します。
adminユーザーは例えばmigration用だったり、本番環境でのオペレーションなどに使用する想定でGRANT ALL PRIVILEGESにより全権限を付与しています。

```.sql
CREATE DATABASE restriction_test;
CREATE USER 'restrict'@'%' IDENTIFIED BY 'restrict';
CREATE USER 'admin'@'%' IDENTIFIED BY 'admin';
GRANT SELECT,UPDATE,INSERT,DELETE on `restriction_test`.* to 'restrict'@'%';
GRANT ALL PRIVILEGES on `restriction_test`.* to 'admin'@'%';
```

### テーブル作成準備

```.sql
CREATE TABLE `restriction_test`.`users` (
  `id` VARCHAR(255) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`id`));

CREATE TABLE `restriction_test`.`threads` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `user_id` VARCHAR(255) NOT NULL,
  `title` TEXT NOT NULL,
  PRIMARY KEY (`id`));

ALTER TABLE `restriction_test`.`threads`
ADD INDEX `fk_threads_user_id_users_id_idx` (`user_id` ASC);

ALTER TABLE `restriction_test`.`threads`
ADD CONSTRAINT `fk_threads_user_id_users_id`
  FOREIGN KEY (`user_id`)
  REFERENCES `restriction_test`.`users` (`id`);
```

### レコード挿入

1人のユーザーが1つのスレッドを立てた状態のデータを作成します。

```.sql
USE restriction_test;
INSERT INTO users(id, name) values ("test", "testUser");
INSERT INTO threads(user_id, title) values ("test", "thread1");
```

### mysql5.7側での確認
#### adminアカウント
mysql5.7側では、3320ポートとマッピングしているので、3320にアクセスして操作を行います。

```
mysql -h 127.0.0.1 --port 3320 -u admin -padmin

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.7.41 MySQL Community Server (GPL)
```
```.sql
use restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

DELETE FROM users where id = "test";
> ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))
```

#### restrictアカウント
```
mysql -h 127.0.0.1 --port 3320 -u restrict -prestrict

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.7.41 MySQL Community Server (GPL)
```

```.sql
use restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

DELETE FROM users where id = "test";
> ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))
```

#### 比較
どちらも同じレスポンスを返しており、権限による影響は特にありません。

### mysql8.0側での確認
mysql8.0側では、3321ポートとマッピングしているので、3321にアクセスして操作を行います。

#### adminアカウント
```
mysql -h 127.0.0.1 --port 3321 -u admin -padmin

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 8.0.30 MySQL Community Server - GPL
```

```
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.7.41 MySQL Community Server (GPL)
```

```.sql
USE restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

DELETE FROM users where id = "test";
> ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))
```

#### restrictアカウント
```
mysql -h 127.0.0.1 --port 3321 -u restrict -prestrict

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 8.0.30 MySQL Community Server - GPL
```
```.sql
use restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1216 (23000): Cannot add or update a child row: a foreign key constraint fails

DELETE FROM users where id = "test";
> ERROR 1217 (23000): Cannot delete or update a parent row: a foreign key constraint fails
```

#### 比較
adminアカウントでは特に変化は起きませんでしたが、restrictアカウントで変化が起きました。
mysql5.7では、restrictアカウントでもテーブルの情報とどこがキー制約に掛かっているのかまで詳細に書かれていたところ、mysql8.0ではエラーコードが別のものになり、メッセージもテーブル情報等が省かれ簡素なものとなりました。
