# mysql-breaking-change

docker-compose up -d

## mysql5.7側の操作
mysql -h 127.0.0.1 --port 3320 -u root -proot

## mysql8.0側の操作
mysql -h 127.0.0.1 --port 3321 -u root -proot

## 以下共通
CREATE DATABASE restriction_test;
CREATE USER 'restrict'@'%' IDENTIFIED BY 'restrict';
CREATE USER 'admin'@'%' IDENTIFIED BY 'admin';
GRANT SELECT,UPDATE,INSERT,DELETE on `restriction_test`.* to 'restrict'@'%';
GRANT ALL PRIVILEGES on `restriction_test`.* to 'admin'@'%';

## 双方に流すSQL

### テーブル作成準備
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

### レコード挿入
USE restriction_test;
INSERT INTO users(id, name) values ("test", "testUser");
INSERT INTO threads(user_id, title) values ("test", "thread1");

### mysql5.7側での確認
#### adminアカウント
mysql -h 127.0.0.1 --port 3320 -u admin -padmin
use restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

DELETE FROM users where id = "test";
> ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

#### restrictアカウント
use restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

DELETE FROM users where id = "test";
> ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

### mysql8.0側での確認
#### adminアカウント
USE restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

DELETE FROM users where id = "test";
> ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`restriction_test`.`threads`, CONSTRAINT `fk_threads_user_id_users_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

#### restrictアカウント
use restriction_test;
INSERT INTO threads(user_id, title) values("notFoundUser", "testTitle");

> ERROR 1216 (23000): Cannot add or update a child row: a foreign key constraint fails

DELETE FROM users where id = "test";
> ERROR 1217 (23000): Cannot delete or update a parent row: a foreign key constraint fails
