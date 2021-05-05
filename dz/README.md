## Mysql.

Развернуть базу на мастере и настроить так, чтобы реплицировались таблицы: | bookmaker | | competition | | market | | odds | | outcome

Настроить GTID репликацию.

### Реализация.

Установим пакет Percona-Server-57:

`mysql-master`
```
[root@mysql-master ~]# yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
[root@mysql-master ~]# yum install -y Percona-Server-server-57
```

`mysql-slave`
```
[root@mysql-slave ~]# yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
[root@mysql-slave ~]# yum install -y Percona-Server-server-57
```

В директорию `/etc/my.cnf.d/` скопируем наши конфиги и запустим службу `mysqld`:

`mysql-master`
```
[root@mysql-master ~]# systemctl enable --now mysqld
```

`mysql-slave`
```
[root@mysql-slave ~]# systemctl enable --now mysqld
```

При установке Percona автоматически генерирует пароль для пользователя root, в файл `/var/log/mysqld.log`, для поиска сгенирированного пароля воспользуемся командой:

```
# cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'

temp passwd
```

Подключаемся к mysql и меняем пароль для доступа к полному функционалу:

```
# mysql -uroot -p'temp passwd'
mysql > ALTER USER USER() IDENTIFIED BY 'Pass121@@212';
```

Создадим конфигурации файл `.my.cnf` для MySQL, со следующим содержимым. Позволит подключаться к mysql без указания учетных данных пользователя root:

`mysql-master`
```
[root@mysql-master ~]# vi .my.cnf

[client]
user=root
password='Pass121@@212'
```

`mysql-slave`
```
[root@mysql-slave ~]# vi .my.cnf

[client]
user=root
password='Pass121@@212'
```
 
Репликацию будем настраивать с использованием `GTID`. Обратим внимание, что бы атрибут `server-id` на мастер-сервере должен обязательно отличаться от `server-id` слейв-сервера:

`mysql-master`
```
[root@mysql-master ~]# mysql -e 'SELECT @@server_id;'

+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
```

`mysql-slave`
```
[root@mysql-slave ~]# mysql -e 'SELECT @@server_id;'

+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
```

Убеждаемся что GTID включен:

`mysql-master`
```
[root@mysql-master ~]# mysql -e "SHOW VARIABLES LIKE 'gtid_mode';"

+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
```

`mysql-slave`
```
[root@mysql-slave ~]# mysql -e "SHOW VARIABLES LIKE 'gtid_mode';"

+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
```

Создадим тестовую базу `bet` и загрузим в нее дамп:

```
[root@mysql-master ~]# mysql -e "CREATE DATABASE bet;"
[root@mysql-master ~]# mysql -D bet < bet.dmp
```

Проверим:

```
[root@mysql-master ~]# mysql -e "USE bet; SHOW TABLES;"

+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
```

Создадим пользователя для репликации и даем ему права на эту самую репликацию:

```
[root@mysql-master ~]# mysql -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'Pass121##212';"
[root@mysql-master ~]# mysql -e "mysql -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY 'Pass121##212';"
```

Проверим, что пользователя имеет право на репликацию:

```
[root@mysql-master ~]# mysql -e "SELECT user,host,repl_slave_priv FROM mysql.user WHERE user='repl';"

+------+------+-----------------+
| user | host | repl_slave_priv |
+------+------+-----------------+
| repl | %    | Y               |
+------+------+-----------------+
```

Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию:

```
[root@mysql-master ~]# mysqldump --all-databases --triggers --routines --master-data
--ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```

На этом настройка Master-а завершена. Файл дампа нужно залить на слейв.

Настраивем слейв для репликации. Раскомментируем в `/etc/my.cnf.d/05-binlog.cnf` строки:

```
#replicate-ignore-table=bet.events_on_demand
#replicate-ignore-table=bet.v_same_event
```

Таким образом указываем таблицы которые будут игнорироваться при репликации.

```
[root@mysql-slave ~]# mysql -e "SOURCE master.sql;"
[root@mysql-slave ~]# mysql -e "SHOW DATABASES LIKE 'bet';"

+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
```
```
[root@mysql-slave ~]# mysql -e "USE bet; SHOW TABLES;"

+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
```

Ну и собственно подключаем и запускаем слейв:

```
[root@mysql-slave ~]# mysql -e "change master to master_host='192.168.11.10', master_port=3306, master_user='repl', master_password='Pass121##212', master_auto_position=1;"
[root@mysql-slave ~]# mysql -e "start slave;"
[root@mysql-slave ~]# mysql -e "show slave status\G;"
```

Проверим репликацию в действии. На мастере:

```
[root@mysql-master ~]# mysql -e "USE bet; SELECT * FROM bookmaker;"

+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
```
```
[root@mysql-master ~]# mysql -e "USE bet; INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');"
[root@mysql-master ~]# mysql -e "USE bet; SELECT * FROM bookmaker;"

+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
```

На слейве:

```
[root@mysql-slave ~]# mysql -e "USE bet; SELECT * FROM bookmaker;"

+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
```

Проверка задания
----------------

1. Выполнить `vagrant up`.

2. Проверим репликацию в действии.
