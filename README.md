# Цель домашнего задания
снимать резервные копии;
восстанавливать базу после сбоя;
настраивать master-slave репликацию.


# Решение:

Данный Vagrantfile развернёт 2 виртаульные машины ОС centos/7.

## Сервер master:
### Ставим percona-server-57

[root@master ~]# sudo yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y
[root@master ~]# yum install Percona-Server-server-57 -y

### Копируем конфиги из /vagrant/conf.d в /etc/my.cnf.d/

[root@master ~]# cp /vagrant/conf/conf.d/* /etc/my.cnf.d/

### После этого можно запустить службу:

[root@master ~]# systemctl start mysql

### При установке Percona автоматически генерирует пароль длā пользователя root и кладет его в файл /var/log/mysqld.log:
[root@master ~]# cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
0x+wt8Qn7Tgg

### Подключаемся к mysql и меняем пароль для доступа к полному функционалу:
[root@master ~]# mysql -uroot -p'0x+wt8Qn7Tgg'

mysql > ALTER USER USER() IDENTIFIED BY 'YourStrongPassword';

### Репликацию будем настраивать с использованием GTID.

### Следует обратить внимание,
что атрибут server-id на мастер-сервере должен обязательно
отличаться от server-id слейв-сервера. Проверить какая переменная установлена в текущий
момент можно следующим образом:
<pre>
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0,00 sec)
</pre>

### Убеждаемся что GTID включен:
<pre>
mysql> SHOW VARIABLES LIKE 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0,00 sec)

</pre>

###  Создадим тестовую базу bet и загрузим в нее дамп и проверим:
<pre>
mysql> CREATE DATABASE bet;
Query OK, 1 row affected (0.00 sec)

[root@master ~]# mysql -uroot -p -D bet < /vagrant/bet.dmp

mysql> USE bet;
Database changed
mysql> SHOW TABLES;
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
7 rows in set (0,00 sec)

</pre>


### Создадим пользователя для репликации и даем ему права на эту самую репликацию:
<pre>
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
mysql> SELECT user,host FROM mysql.user where user='repl';
+------+-----------+
| user | host |
+------+-----------+
| repl | % |
+------+-----------+
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
</pre>

### Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию:
<pre>
[root@master ~]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
</pre>

### На этом настройка Master-а завершена. Файл дампа нужно залить на слейв.

## Сервер slave:

### Также устанавливаем percona-server-57 и копируем файлы из /vagrant/conf.d в /etc/my.cnf.d/

### Правим в /etc/my.cnf.d/01-basics.cnf директиву server-id = 2
<pre>
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
1 row in set (0.00 sec)

</pre>

### Раскомментируем в /etc/my.cnf.d/05-binlog.cnf строки:
<pre>
#replicate-ignore-table=bet.events_on_demand
#replicate-ignore-table=bet.v_same_event
</pre>
Таким образом указываем таблицы которые будут игнорироваться при репликации

### Заливаем дамп мастера и убеждаемся что база есть и она без лишних таблиц:
<pre>
mysql> SOURCE /vagrant/master.sql
mysql> SHOW DATABASES LIKE 'bet';
+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
1 row in set (0.00 sec)

mysql> USE bet;
Database changed
mysql> SHOW TABLES;
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0.00 sec)

</pre>
### видим что таблиц v_same_event и events_on_demand нет

### Ну и собственно подключаем и запускаем слейв:
<pre>
mysql> CHANGE MASTER TO MASTER_HOST = "192.168.11.150", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2018", MASTER_AUTO_POSITION = 1;
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.11.150
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 451
               Relay_Log_File: slave-relay-bin.000004
                Relay_Log_Pos: 664
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 



</pre>

### Проверим репликацию в действии. На мастере:
<pre>
mysql> USE bet;
Database changed
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(31,'testrow');
Query OK, 1 row affected (0,01 sec)

mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
| 67 | 4xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
| 31 | testrow        |
|  3 | unibet         |
+----+----------------+
6 rows in set (0,00 sec)

mysql> 
</pre>


### На слейве:
<pre>
mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
| 67 | 4xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
| 31 | testrow        |
|  3 | unibet         |
+----+----------------+
6 rows in set (0.00 sec)
</pre>
### В binlog-ах на cлейве также видно последнее изменение, туда же он пишет информацию о GTID:
<pre>
#240429 14:31:51 server id 1  end_log_pos 292 CRC32 0xab18d783 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1714401111/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 292
#240429 14:31:51 server id 1  end_log_pos 420 CRC32 0x5ab70959 	Query	thread_id=3	exec_time=0	error_code=0
use `bet`/*!*/;
SET TIMESTAMP=1714401111/*!*/;
INSERT INTO bookmaker (id,bookmaker_name) VALUES(67,'4xbet')
/*!*/;
# at 420
#240429 14:31:51 server id 1  end_log_pos 451 CRC32 0x8482d601 	Xid = 35
COMMIT/*!*/;
# at 451
#240429 15:10:05 server id 1  end_log_pos 516 CRC32 0xa8935c8c 	GTID	last_committed=1	sequence_number=2	rbr_only=no
SET @@SESSION.GTID_NEXT= 'b02dcbb9-060f-11ef-b9ec-5254004d77d3:2'/*!*/;
# at 516
#240429 15:10:05 server id 1  end_log_pos 589 CRC32 0x1ac3d0e1 	Query	thread_id=7	exec_time=0	error_code=0
SET TIMESTAMP=1714403405/*!*/;
BEGIN
/*!*/;
# at 589
#240429 15:10:05 server id 1  end_log_pos 719 CRC32 0x21800820 	Query	thread_id=7	exec_time=0	error_code=0
SET TIMESTAMP=1714403405/*!*/;
INSERT INTO bookmaker (id,bookmaker_name) VALUES(31,'testrow')
/*!*/;
# at 719
#240429 15:10:05 server id 1  end_log_pos 750 CRC32 0xa989cbe3 	Xid = 44
COMMIT/*!*/;
SET @@SESSION

</pre>













