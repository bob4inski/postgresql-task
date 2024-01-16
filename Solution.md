# postgresql-task
task from mario

## Задания
## Установка:
1. Установить пакеты Postgresql
    ```bash
    sudo yum install postgresql14-contrib postgresql14 postgresql14-libs postgresql14-server
    ```
2. Работа с разделами.
Примонтировать Disk1 в директорию /pgsql/pg_data. 
Владелец директории postgres.postgres. Права 700
```bash
sudo mkdir -p  /psql/pg_data/14
sudo chown -R postgres:postgres /psql
```

3. Выполните инициализацию экземпляра с помощью утилиты initdb в раздел /pgsql/pg_data/14. [копипаста](https://dba.stackexchange.com/questions/292431/postgresql-setup-initdb-with-custom-data-directory)
```bash
sudo nano /lib/systemd/system/postgresql-14.service
# Environment=PGDATA= поменять на свое
sudo systemctl edit postgresql-14.service;
#записать туда
[Service]
Environment=PGDATA=/pgdata/14/data


sudo  systemctl daemon-reload
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb

sudo systemctl enable postgresql-14
sudo systemctl start postgresql-14
```
## Сами задания
### 1. Роли и группы:
- Создайте новую базу данных testdb
- зайдите в созданную базу данных под пользователем postgres и создайте новую схему testnm
- создайте новую таблицу t1 с одной колонкой c1 типа integer
- вставьте строку со значением c1=1
```sql
create database testdb;
\c testdb
create schema testnm;

Create table t1
(c1 int);

insert into t1 values (1);
```

- создайте новую роль readonly
- дайте новой роли право на подключение к базе данных testdb
- дайте новой роли право на использование схемы testnm
- дайте новой роли право на select для всех таблиц схемы testnm
```sql
create role readonly;
alter role readonly login;
grant connect on database testdb to readonly;
grant select, usage  on schema testnm to readonly;
grant select on all tables in schema testnm to readonly;
```
- создайте пользователя testread с паролем test123
- дайте роль readonly пользователю testread
- зайдите под пользователем testread в базу данных testdb
```sql
create role testread with password 'test123';
grant readonly to testread;
\c testdb testread
```




### 2. Изменение конфигурации Postgres

    необходимо изменить параметры
    ```
    listen_addresses = '*'
    max_connections = 150         
    ```

 Произведите расчет параметров shared_buffers, work_mem, maintenance_work_mem, effective_cache_size, random_page_cost, effective_io_concurrency
 и измените их в конфиг-файле postgresql.conf
 Произведите перезагрузку экземпляра и убедитесь, что параметры вступили в силу.
 В целом с этим помогает этот [сайт](https://pgconfigurator.cybertec.at/)

### 3.	Установка дополнительных расширений
Установите в пользовательской БД testdb расширения pg_stat_statesments, postgres_fdw, pg_profile.
Для активации расширения пропишите в конфиг-файле  postgresql.conf следующие параметры:
```
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

```sql
create extension pg_stat_statements;
create extension pg_profile;
```

#### pg_stat_statements 
Создайте базу данных bench. Запустите нагрузочное тестирование, используя pgbench. Если pgbench не установлен, установите его.
Запустите нагрузочное тестирование
Установка - *dnf -y install postgresql-contrib*
```bash
pgbench -i bench
pgbench -c10 -t300 bench
```


После завершения нагрузочного тестирования, получите список самых ресурсоемких по IO time write и самых длительных запросов (execution time) из представления pg_stat_statements 

```sql
select query, total_exec_time from pg_stat_statements
order by 2 DESC
limit 5;
                                                    query                                                    |  total_exec_time
-------------------------------------------------------------------------------------------------------------+--------------------
 insert into test_partitioning_table                                                                        +|       63648.565547
 (region, sale_date)                                                                                        +|
 select region, sale_date from                                                                              +|
 generate_series($1,$2) as region(text), generate_series(current_date, current_date + interval $3, interval +|
 $4)                                                                                                        +|
 as sale_date(date)                                                                                          |
 UPDATE pgbench_branches SET bbalance = bbalance + $1 WHERE bid = $2                                         | 17283.668098999995
 UPDATE pgbench_tellers SET tbalance = tbalance + $1 WHERE tid = $2                                          |  14549.27690499996
 select extract($1 from sale_date) as boba from test_partitioning_table group by boba                        |         4752.27858
 select count(id) from test_partitioning_table                                                               | 1736.8796610000002
```

#### postgres_fdw

Создайте базу данных fdw. Установите в ней расширение postgres_fdw. Импортируйте в базу данных fdw таблицу testnm.t1 из базы данных testdb. 
Попробуйте выполнить вставку новых значений в импортированную внешнюю таблицу testnm.t1
1. необходимо включить расширение
```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER foreign_server
        FOREIGN DATA WRAPPER postgres_fdw
        OPTIONS (host 'localhost', port '5432', dbname 'test2');

CREATE USER MAPPING FOR postgres
        SERVER foreign_server
        OPTIONS (user 'postgres', password 'postgres');

CREATE FOREIGN TABLE t1 (
        c1 int
)
        SERVER foreign_server
        OPTIONS (schema_name 'test_schema', table_name 't1');
```
2. Пробуем вставить значения в таблицу 
```sql
fdw=# insert into t1 values (1);
INSERT 0 1
fdw=# select * from t1;
 c1
----
  1
  1
(2 rows)

fdw=# \c test2
test2=# select * from test_schema.t1 ;
 c1
----
  1
  1
(2 rows)
```

### 4. Секционирование
Создайте таблицу. С помощью generate_series сгенериуйте десять миллионов записей и вставьте их в таблицу.
Секционируйте эту таблицу по годам по полю sale_date
```sql 
CREATE TABLE IF NOT EXISTS test_partitioning_table (
                               id serial,
                               region text,
                               sale_date date not null
                )
            PARTITION BY RANGE (sale_date);

-- таблицу создали и потом необходимо создать партиции для таблицы

create table test_partitioning_table_2020 partition of test_partitioning_table
for VALUES from ('2020-01-01') to ('2021-01-01');

-- и потом уже делать insert в таблицу
insert into test_partitioning_table (region, sale_date)

select region, sale_date from
generate_series(1,100) as region(text), generate_series('2020-01-01','2020-12-31' , interval '1 day') as sale_date(date);

-- сюда большой insert не ставил, тк сильно грузит бдт

```


 
### 5.	[Анализ блокировок](https://www.youtube.com/watch?v=8FIUWUrq474&t=1303) 
1. Настройте логирование сервера так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.
```sql
-- Для проверки текущих значений логгироваия и блокировок 
postgres=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)

alter system set deadlock_timeout = 200;
alter system set log_lock_waits = on;

```
также указал, что надо логгировать всё:
```
log_min_duration_statement = 0
log_statement = 'all'
```
```bash
#необходимо перезапустить сервер
systemctl restart postgresql-14.service
```
2. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения:
3. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие
блокировки в представлении pg_locks и убедитесь, что все они понятны. Выпишите список блокировок и объясните, что значит
каждая. 
```sql
create database locks;  

create table test_locks 
(
    id int, 
    "name" text
); 

insert into test_locks (id, "name") values                                    
(1, 'robert'),                                                                
(2, 'mario'),                                                                 
(3,'boba');
```
открываю сразу несколько терминалов, чтобы посмотреть что будет с блокировками

```
   locktype    |  pid  |  relation  | virtxid | xid  |       mode       | granted                                                                             
---------------+-------+------------+---------+------+------------------+---------                                                                           
 relation      | 38298 | test_locks |         |      | RowExclusiveLock | t                                                                                   
 transactionid | 38298 |            |         | 3929 | ExclusiveLock    | t                                                                                
 virtualxid    | 38298 |            | 6/82    |      | ExclusiveLock    | t
 -------------------------------------------------------------------------
 relation      | 38300 | test_locks |         |      | RowExclusiveLock | t                                                                                    
 transactionid | 38300 |            |         | 3930 | ExclusiveLock    | t                                                                                  
 transactionid | 38300 |            |         | 3929 | ShareLock        | f                                                                                  
 tuple         | 38300 | test_locks |         |      | ExclusiveLock    | t                                                                                   
 virtualxid    | 38300 |            | 5/10    |      | ExclusiveLock    | t         
--------------------------------------------------------------------------------                                                                         
 relation      | 38301 | test_locks |         |      | RowExclusiveLock | t                                                                                   
 transactionid | 38301 |            |         | 3931 | ExclusiveLock    | t                                                                                   
 tuple         | 38301 | test_locks |         |      | ExclusiveLock    | f                                                                                   
 virtualxid    | 38301 |            | 4/504   |      | ExclusiveLock    | t                                                                                   
(12 rows)
```
С помощью select pg_blocking_pids(); смотрю, какая транзакция какую блокирует
и они блокируются по очереди
300 ждет 298, а 301 ждет 300
В транзакции 300 показывается, что мы пытаемся получить данные из  share, но пока что не может тк предыдущая транзакция еще не завершилась
И при завершении первой транзакции подобная картина будет уже со следущими транзакциями
```
2024-01-12 14:11:47.395 MSK [38298] LOG:  statement: update test_locks set name = 'test' where id = 1;                                                       
2024-01-12 14:11:47.595 MSK [38298] LOG:  process 38298 still waiting for ShareLock on transaction 3930 after 200.161 ms                                    
2024-01-12 14:11:47.595 MSK [38298] DETAIL:  Process holding the lock: 38300. Wait queue: 38301, 38298.                                                    
2024-01-12 14:11:47.595 MSK [38298] CONTEXT:  while updating tuple (0,7) in relation "test_locks"                                                           
2024-01-12 14:11:47.595 MSK [38298] STATEMENT:  update test_locks set name = 'test' where id = 1;
```
4. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
смотрим лог и выполняем транзакции и что мы видим
Я сделал примерно также, как и [тут](https://github.com/radchenkoam/OTUS-postgres-2020-05/issues/7)
В целом понятно по логу, тк описан каждый шаг транзакции и потом показывается какая транзакция какую прерывает
```log
2024-01-12 14:24:34.024 MSK [38301] LOG:  process 38301 detected deadlock while waiting for ShareLock on transaction 3935 after 200.160 ms      
2024-01-12 14:24:34.024 MSK [38301] ERROR:  deadlock detected
2024-01-12 14:24:34.024 MSK [38301] DETAIL:  Process 38301 waits for ShareLock on transaction 3935; blocked by process 38298.            
        Process 38298 waits for ShareLock on transaction 3936; blocked by process 38300.                                                                 
        Process 38300 waits for ShareLock on transaction 3937; blocked by process 38301.
```
5. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
 
### 6. Резервное копирование и восстановление
#### pg_dump
Создайте копию (дамп) базы данных testdb утилитой pg_dump в формате directory. (а у меня ж есть таблица test_partitioning_table ее мы и задампим )
После создания дампа, зайдите в psql и удалите базу данных testdb.
Восстановите базу данных используя дамп. Убедитесь, что база данных восстановилась.
```bash
pg_dump -Fd testdb -f /tmp/testdb
[postgres@postgres-main ~]$ ls /tmp/testdb/
4282.dat.gz  toc.dat
```
```sql
drop database testdb ;
DROP DATABASE
```
Далее создаем целевую бд в psql и восстанавливаем таблицы через pg_restore
```bash
pg_restore  -d testdb -Fd /tmp/testdb/

testdb=# \dt
                             List of relations
 Schema |               Name                |       Type        |  Owner
--------+-----------------------------------+-------------------+----------
 public | test_partitioning_table           | partitioned table | postgres
 public | test_partitioning_table_2020_2039 | table             | postgres
(2 rows)
```



#### pg_basebackup
1. Создайте резервную копию экземпляра утилитой pg_basebackup в виде tar-файла.
Удалите/измените несколько таблиц и остановите экземпляр
```bash
pg_basebackup --format=tar -z -D /tmp/postgres_archive_2 -Ft -Xf
rm -rf /psql/pg_data/14
tar -xzf /tmp/postgres_archive_2/base.tar.gz  -C /psql/pg_data/14/
chmod -R 700  /psql/pg_data/14/
# далее просто запускаем postgres
```
2. Выполните несколько обновляющих транзакций, создайте точку восстановления и выполните еще несколько транзакций.
Остановите экземпляр и восстановите его с резервной копии по состоянию на точку восстановления.
[подсказка](https://www.scalingpostgres.com/tutorials/postgresql-backup-point-in-time-recovery/)

1. Для начала создадим папку в которой будем валы сохранять
```bash
mkdir /var/lib/postgresql/pg_log_archive
nano /psql/pg_data/15/postgresql.conf
```
2. Изменить `postgresql.conf`
```yml
wal_level = replica
archive_mode = on # (change requires restart)
archive_command = 'test ! -f /var/lib/postgresql/pg_log_archive/%f && cp %p /var/lib/postgresql/pg_log_archive/%f'
```
3. перзапустить постгрес и создать архив 
```bash
sudo systemctl restart postgresql-15.service

pg_basebackup --format=tar -z -D /tmp/postgres_archive_2 -Ft -Xf
```
4. Делаем определенные манипуляции с бд
```sql
create database test;
\c test
create table test_table (id int);
insert into test_table values (1),(2),(3);

--<time is ~2024-01-16 14:50:00>
--archive the logs
select pg_switch_wal();
--уже после точки восстановления
insert into test_table values (10),(22),(33);
```
5. Останавливает экземпляр, и восстанавливаем из бекапа
```bash
systemctl stop postgresql-15.service

mv /psql/pg_data/15/ /psql/pg_data/15-old
mkdir /psql/pg_data/15/
tar -xzf /tmp/postgres_archive_2/base.tar.gz  -C /psql/pg_data/15/
```
6. Определяем точку до которой нужно восстановиться
```bash
nano nano /psql/pg_data/15/postgresql.conf
        restore_command = 'cp /var/lib/postgresql/pg_log_archive/%f %p'
        recovery_target_time = '2024-01-16 14:50:00'

sudo systemctl restart postgresql-15.service
```
7. Проверка
```sql
test=# select * from test_table;
 id
----
  1
  2
  3
(3 rows)
```


### 7.	[Потоковая репликация](https://dbaclass.com/article/how-to-configure-streaming-replication-in-postgres-14/)
Создайте виртуальную машину PG2 с техническими характеристиками как у виртуальной машины PG1.

1. Выполните пункты 1-3 для ВМ PG2.
Настройте потоковую репликацию между узлами PG1 и PG2, где PG1 мастер, а PG2 - реплика. Настройте синхронную репликацию.
-  Для начала нужно поменять настройки pg_hba и разрешить подключение для репликации (также создать пользователя для репликации)
```
host    replication     repl_user       212.233.96.192/32       trust
host    replication     repl_user       10.0.3.0/24     trust
```
- заходим на второй сервер и прописываем команду чтобы сразу скопировать все файлы в нужную директорию с первого хоста
```bash
pg_basebackup -D /psql/pg_data/14 -X fetch -p 5432 -U repl_user -h  10.0.3.8 -R
```
- проверяем наличие файла `standby.signal` и содержимое `postgresql.auto.conf`
```bash
[redos@postgres-slave ~]$ sudo cat /psql/pg_data/14/postgresql.auto.conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=repl_user passfile=''/root/.pgpass'' channel_binding=prefer host=10.0.3.8 port=5432 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
```

2. Создайте на мастере PG1 в БД  testdb таблицу test_replication c произвольным набором колонок, заполнитне таблицу используя generate_series. Проверьте что данные появились на реплике и содержимое таблицы test_replication совпадают на мастере и на реплике.

3. Выполните переключение ([switchover](https://dbaclass.com/article/how-to-do-switchover-in-postgres/)) мастера таким образом, чтобы PG2 стал мастером, а PG1-репликой.

on master:
```bash
sudo -iu postgres
/usr/pgsql-14/bin/pg_ctl stop -D /psql/pg_data/14/
```
on slave
```bash
[postgres@postgres-slave ~]$ /usr/pgsql-14/bin/pg_ctl  promote -D /psql/pg_data/14
waiting for server to promote.... done
server promoted
```

on old-master 
```bash
touch /psql/pg_data/14/standby.signal
nano postgresql.auto.conf 
#insert smth like this 
#primary_conninfo = 'user=repl_user passfile=''/root/.pgpass'' channel_binding=prefer host=10.0.3.6 port=5432 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'

/usr/pgsql-14/bin/pg_ctl start -D /psql/pg_data/14/
```

### 8.	Обновление версии
Используя утилиту pg_upgrade попробуйте выполнить обновление с 14 на 15 версию PostgreSQL на всех узлах установленного в предыдущем пункте отказоустойчивого кластера. 
1. on master
```bash
mkdir /psql/pg_data/15
sudo yum install postgresql15-contrib postgresql15 postgresql15-libs postgresql15-server
```
2. Далее повторить пункты из установки
3. Остановить сервер и запустить обновление 
```bash
/usr/pgsql-15/bin/pg_ctl -D /psql/pg_data/14 stop

/usr/pgsql-15/bin/pg_upgrade -d /psql/pg_data/14 -D /psql/pg_data/15 -b /usr/pgsql-14/bin -B /usr/pgsql-15/bin
/usr/pgsql-15/bin/pg_ctl -D /psql/pg_data/15 start
```
on slave:
1. Установить новую версию
```bash
mkdir /psql/pg_data/15
sudo yum install postgresql15-contrib postgresql15 postgresql15-libs postgresql15-server
```
2. проинициализировать бд
3. rsync с мастера (но у меня не получилось, поэтому я сделаю чуть иначе)
```bash
rsync --archive --delete --hard-links --size-only --no-inc-recursive /psql/pg_data/14 /psql/pg_data/15 10.0.3.6:/psql/pg_data/15
```
Сделал в итоге так, с rsync надо проверять еще
```bash
pg_basebackup -D /psql/pg_data/14 -X fetch -p 5432 -U repl_user -h  10.0.3.8 -R
```
4. Запуск и проверка
```sql
postgres=# select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 43348
usesysid         | 24576
usename          | repl_user
application_name | walreceiver
client_addr      | 10.0.3.8
client_hostname  |
client_port      | 37238
backend_start    | 2024-01-15 17:54:31.209473+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/25000060
write_lsn        | 0/25000060
flush_lsn        | 0/25000060
replay_lsn       | 0/25000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-01-15 17:56:11.411474+03
```



## Дополнительные задачи со звездочкой
1)	**Установка дополнительного расширения pg_profile**
Установите расширение pg_profile в любой в базе данных bench.
Запустите нагрузочное тестирование используя pgbench. Пока идет тестирование подключитесь к БД bench и снимите отчет используя функцию profile.take_sample().
Сгенерируйте html-отчет на основе созданного сэмпла.
```bash
wget https://github.com/zubkov-andrei/pg_profile/releases/download/4.3/pg_profile--4.3.tar.gz
sudo tar -xzf pg_profile--4.3.tar.gz -C /usr/pgsql-14/share/extension/
```


```sql
CREATE SCHEMA profile;
CREATE EXTENSION pg_profile SCHEMA profile;
create extension pg_stat_statements;
```
```bash
psql -d "bench" -Aqtc "SELECT profile.get_report(tstzrange(now()-interval '1 day',now()))" -o report_0900_1000.html
```
2)	**Логическая репликация**
Разорвите потоковую репликацию, настроенную в предыдущей задаче. Выполните promote реплики, удалите слот репликации с мастера.
Переопределите значение параметра wal_level на logical на PG1 и cоздайте таблицу logrepl c полями (id serial, val text). Создайте публикацию данной таблицы на PG1 и подпишитесь на публикацию на PG2.
Наполните таблицу logrepl данными и убедитесь, что данные среплицировались на PG2.
Удалите логическую репликацию.
3)	**Восстановление с помощью pg_probackup к заданной точке во времени (Point in time recovery)**
Произведите установку pg_probackup на узле PG1.
Подключите к серверу PG1 новый диск для создания и хранения резервных копий.
Смонтируйте этот диск в раздел /pgsql/backup
Произведите инициализацию и настройку pg_probackup c хранением РК в каталоге /pgsql/backup.
Снимите полную резервную копию экземпляра. Настройте непрерывное архивирование WAL-журналов.
Зафиксируйте время, затем удалите БД testdb. Попробуйте выполнить PITR-восстановление. Удаленная БД восстановилась?
4)	**Развертывание типового решения master-slave**
Одним из типовых решений ЦК СУБД является отказоустойчивая конфигурация Patroni кластера (рисунок ниже), состоящая из:
1.	2 Сервера БД (PG1 и PG2)
2.	3 Сервера DCS Consul (DCS1, DCS2, DCS3).
3.	ПО VIP-manager для однозначного определения мастер-сервера.
Для создания отказоустойчивой конфигурации создайте соответствующие ВМ для DCS Consul (1 ядро, 512 МБ RAM)
1.	Установите на PG1 и PG2 оркестратор репликации Patroni и Postgres (если он еще не установлен).
2.	Установите на DCS1, DCS2, DCS3 ПО Consul и настройте в режиме работы server
3.	Установите на PG1 и PG2 ПО Consul и настройте в режиме работы client
4.	Настройте оркестратор репликации Patroni в режиме master-replica (Leader – Sync Standy).
5.	Установите на PG1 и PG2 и настройте ПО VIP-manager (предварительно выделите соседний IP из этой же подсети
 
Запустите службу Patroni, проверьте состояние кластера командой patronictl list
Попробуйте выполнить switchover мастера с узла PG1 на узел PG2, а затем failover c PG2 на PG1. Чем отличается failover от switchover в patroni cluster? Что означает TL в выводе команды patronictl list?
Измените параметр max_connections используя patronictl edit-config. Что поменялось в выводе команды patronictl list? 
Попробуйте выполнить switchover используя консольную утилиту curl и Patroni REST API.

