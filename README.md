# postgresql-task
task from mario

## Задания
### Установка:
1. Установить пакеты Postgresql
    ```
    postgresql14-contrib postgresql14 postgresql14-libs postgresql14-server
    ```
2. Работа с разделами.
Примонтировать Disk1 в директорию /pgsql/pg_data. 
Владелец директории postgres.postgres. Права 700
3. Выполните инициализацию экземпляра с помощью утилиты initdb в раздел /pgsql/pg_data/14.

### Сами задания
1. Роли и группы:
- Создайте новую базу данных testdb
- зайдите в созданную базу данных под пользователем postgres и создайте новую схему testnm
- создайте новую таблицу t1 с одной колонкой c1 типа integer
- вставьте строку со значением c1=1
- создайте новую роль readonly
- дайте новой роли право на подключение к базе данных testdb
- дайте новой роли право на использование схемы testnm
- дайте новой роли право на select для всех таблиц схемы testnm
- создайте пользователя testread с паролем test123
- дайте роль readonly пользователю testread
- зайдите под пользователем testread в базу данных testdb




2. Изменение конфигурации Postgres

    необходимо изменить параметры
    ```
    listen_addresses = '*'
    max_connections = 150         
    ```

 Произведите расчет параметров shared_buffers, work_mem, maintenance_work_mem, effective_cache_size, random_page_cost, effective_io_concurrency
 и измените их в конфиг-файле postgresql.conf
 Произведите перезагрузку экземпляра и убедитесь, что параметры вступили в силу.

3.	Установка дополнительных расширений
Установите в пользовательской БД testdb расширения pg_stat_statesments, postgres_fdw, pg_profile.
    - pg_stat_statesments
Для активации расширения пропишите в конфиг-файле  postgresql.conf следующие параметры:
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
Создайте базу данных bench. 
Запустите нагрузочное тестирование, используя pgbench. Если pgbench не установлен, установите его.
Запустите нагрузочное тестирование
pgbench -i bench
pgbench -c10 -t300 bench
После завершения нагрузочного тестирования, получите список самых ресурсоемких по IO time write и самых длительных запросов (execution time) из представления pg_stat_statements 

    - postgres_fdw
Создайте базу данных fdw. Установите в ней расширение postgres_fdw. Импортируйте в базу данных fdw таблицу testnm.t1 из базы данных testdb. 
Попробуйте выполнить вставку новых значений в импортированную внешнюю таблицу testnm.t1


4. Секционирование
Создайте таблицу
                CREATE TABLE IF NOT EXISTS test_partitioning_table (
                               id serial,
                               region text,
                               sale_date date not null
                ); 
С помощью generate_series сгенериуйте десять миллионов записей и вставьте их в таблицу.
Секционируйте эту таблицу по годам по полю sale_date
 
5)	Анализ блокировок
Настройте логирование сервера так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.
Воспроизведите ситуацию, при которой в журнале появятся такие сообщения:
Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие
блокировки в представлении pg_locks и убедитесь, что все они понятны. Выпишите список блокировок и объясните, что значит
каждая. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
 
6)	Резервное копирование и восстановление
       pg_dump
                Создайте копию (дамп) базы данных testdb утилитой pg_dump в формате directory.
                После создания дампа, зайдите в psql и удалите базу данных testdb.
                Восстановите базу данных используя дамп. Убедитесь, что база данных восстановилась.
       pg_basebackup
                Создайте резервную копию экземпляра утилитой pg_basebackup в виде tar-файла. Выполните несколько обновляющих транзакций, создайте точку восстановления и выполните еще несколько транзакций.
                Остановите экземпляр и восстановите его с резервной копии по состоянию на точку восстановления.
7)	Потоковая репликация
Создайте виртуальную машину PG2 с техническими характеристиками как у виртуальной машины PG1. Выполните пункты 1-3 для ВМ PG2.
 Настройте потоковую репликацию между узлами PG1 и PG2, где PG1 мастер, а PG2 - реплика. Настройте синхронную репликацию.
Создайте на мастере PG1 в БД  testdb таблицу test_replication c произвольным набором колонок, заполнитне таблицу используя generate_series. Проверьте что данные появились на реплике и содержимое таблицы test_replication совпадают на мастере и на реплике.
Выполните переключение (switchover) мастера таким образом, чтобы PG2 стал мастером, а PG1-репликой.

8)	Обновление версии
Используя утилиту pg_upgrade попробуйте выполнить обновление с 14 на 15 версию PostgreSQL на всех узлах установленного в предыдущем пункте отказоустойчивого кластера. 
Дополнительные задачи со звездочкой
1)	**Установка дополнительного расширения pg_profile**
Установите расширение pg_profile в любой в базе данных bench.
Запустите нагрузочное тестирование используя pgbench. Пока идет тестирование подключитесь к БД bench и снимите отчет используя функцию profile.take_sample().
Сгенерируйте html-отчет на основе созданного сэмпла.

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

