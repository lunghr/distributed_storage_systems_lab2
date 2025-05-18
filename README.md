# Distributed Storage Systems Course | Lab 2 | Tupichenko Mila | P3309

## Задание

Цель работы - на выделенном узле создать и сконфигурировать новый кластер БД Postgres, саму БД, табличные пространства и
новую роль, а также произвести наполнение базы в соответствии с заданием. Отчёт по работе должен содержать все команды
по настройке, скрипты, а также измененные строки конфигурационных файлов.

Способ подключения к узлу из сети Интернет через helios: ```ssh -J sXXXXXX@helios.cs.ifmo.ru:2222 postgresY@pgZZZ```
Способ подключения к узлу из сети факультета: ```ssh postgresY@pgZZZ```
Номер выделенного узла ```pgZZZ```, а также логин и пароль для подключения Вам выдаст преподаватель.

---

### Этап 1. Инициализация кластера БД

- Директория кластера: ```$HOME/tah70```
- Кодировка: ```ISO_8859_5```
- Локаль: английская
- Параметры инициализации задать через переменные окружения

---

### Этап 2. Конфигурация и запуск сервера БД

- Способы подключения: 1) Unix-domain сокет в режиме peer; 2) сокет TCP/IP, только localhost
- Номер порта: 9136
- Способ аутентификации TCP/IP клиентов: по паролю MD5
- Остальные способы подключений запретить.
- Настроить следующие параметры сервера БД:
    - ```max_connections```
    - ```shared_buffers```
    - ```temp_buffers```
    - ```work_mem```
    - ```checkpoint_timeout```
    - ```effective_cache_size```
    - ```fsync```
    - ```commit_delay```

Параметры должны быть подобраны в соответствии со сценарием OLAP:

- 6 одновременных пользователей, пакетная запись/чтение данных по 256МБ.
- Директория WAL файлов: ```$PGDATA/pg_wal```
- Формат лог-файлов: ```.log```
- Уровень сообщений лога: ```WARNING```
- Дополнительно логировать: контрольные точки и попытки подключения

---

### Этап 3. Дополнительные табличные пространства и наполнение базы

- Создать новое табличное пространство для индексов: ```$HOME/vwp40```
- На основе template1 создать новую базу: ```fargoldcity```
- Создать новую роль, предоставить необходимые права, разрешить подключение к базе.
- От имени новой роли (не администратора) произвести наполнение ВСЕХ созданных баз тестовыми наборами данных. ВСЕ
  табличные пространства должны использоваться по назначению.
- Вывести список всех табличных пространств кластера и содержащиеся в них объекты.

---

## Выполнение

### Этап 1. Инициализация кластера БД

```
mkdir $HOME/tah70
export PGDATA=$HOME/tah70
export LANG=C
export LC_ALL=C

initdb --encoding=ISO_8859_5 --username=postgres0 --auth=md5 --pwprompt
```

---

### Этап 2. Конфигурация и запуск сервера БД

Оказалось, что у меня в доступе вот столько памяти:

```
[postgres0@pg155 ~]$ top
last pid: 58873;  load averages:  0,96,  1,22,  1,24                                           up 27+00:35:35  01:28:16
31 processes:  1 running, 30 sleeping
CPU:  1,7% user,  0,0% nice,  5,6% system,  0,0% interrupt, 92,7% idle
Mem: 1694M Active, 10G Inact, 35G Wired, 78G Free
ARC: 16G Total, 7848M MFU, 5396M MRU, 16M Anon, 229M Header, 2874M Other
     11G Compressed, 32G Uncompressed, 3,01:1 Ratio
Swap: 64G Total, 64G Free
```

Но всё брать не буду и допустим у нас ```8GB``` свободной памяти. В таком случае характеристики для OLAP будем считать
следующим образом:

1. ```shared_buffers ≈ 25% RAM = 2GB```

   Это рекомендуемый максимум для PostgreSQL.

2. ```work_mem = (RAM - shared_buffers) / (2 × active_sessions) = (8GB - 2GB) / (2 × 6) = 512MB, но 128MB```

   На каждый JOIN, ORDER BY, GROUP BY, UNION — создаётся отдельная область work_mem. Поэтому мы делим не просто на число
   пользователей, а на удвоенное их количество: чтобы избежать переполнения памяти. Но по тз 6 пользователей, а значит
   мы случайно можем забить всю память (6 по 2 запроса по 512MB), так что уменьшим до 128MB.

3. ```effective_cache_size ≈ 0.75 × RAM = 6GB```

   Параметр effective_cache_size не управляет памятью напрямую. Он сообщает планировщику PostgreSQL, сколько памяти,
   вероятно, доступно ОС для кэширования файлов. Это влияет на выбор стратегии выполнения запросов: при большом
   effective_cache_size PostgreSQL предпочитает использовать индексы (Index Scan), поскольку предполагает, что они уже
   загружены в память.

4. ```temp_buffers = 32–64MB```

   Когда PostgreSQL внутри запроса создаёт временные таблицы, он хранит их во временном буфере. Этот буфер — это как
   выделенное место в оперативной памяти на одну сессию. При аналитических (OLAP) запросах часто создаются промежуточные
   результаты, которые выгоднее хранить в памяти, а не на диске.
   Поэтому temp_buffers должен быть не минимальным — обычно 32–64MB, в зависимости от доступной RAM, а в нашем случае
   можно не экономить

5. ```checkpoint_timeout = 15min```

   PostgreSQL через каждые 15 минут заставляет себя сбросить буферы на диск, даже если никто не просит. При
   OLAP-нагрузке
   такие сбросы могут мешать аналитическим операциям (ибо в OLAP мы почти всегда читаем), так как создают дополнительную
   нагрузку на диск.
   Увеличение этого значения (например, до 15 минут) позволяет выполнять длительные запросы без прерываний на записи,
   что улучшает производительность анализа данных.

6. ```commit_delay = 1000 (1ms)```

   Параметр commit_delay задаёт небольшую задержку (в микросекундах) перед фактической записью данных на диск при
   выполнении команды COMMIT. Это позволяет сгруппировать несколько коммитов в одну операцию fsync, снижая нагрузку на
   диск.

```
# -----------------------------
# PostgreSQL configuration file
# -----------------------------

listen_addresses = 'localhost'          
port = 9136                             
max_connections = 10                    

work_mem = 128MB                        
shared_buffers = 2GB                 	
temp_buffers = 64MB                    

wal_level = replica                   
fsync = on                           
commit_delay = 1000                
checkpoint_timeout = 15min             
effective_cache_size = 6GB

logging_collector = on         
log_directory = 'log'                 
log_filename = 'postgresql.log'                                       
log_file_mode = 0640                                                        
log_rotation_age = 1d                                                         
log_rotation_size = 10MB                                                   
log_truncate_on_rotation = on       

...    

```

---

### Этап 3. Дополнительные табличные пространства и наполнение базы

Подключаемся к кластеру и создаём табличное пространство для индексов:

```
mkdir $HOME/vwp40
chmod 700 $HOME/vwp40

pg_ctl start -D $PGDATA 
tail -f $PGDATA/log/postgresql.log
psql -U postgres0 -d postgres -p 9136

CREATE TABLESPACE index_space LOCATION '/var/db/postgres0/vwp40';
```

Создаём базу данных и новую роль:

```
CREATE DATABASE fargoldcity WITH TEMPLATE template1 OWNER postgres0;
CREATE ROLE cityuser LOGIN PASSWORD '1234';
GRANT CONNECT ON DATABASE fargoldcity TO cityuser;
GRANT CREATE ON DATABASE fargoldcity TO cityuser;
```

Ещё нужно выдать роли права на использование схемы public и табличного пространства index_space:

```
psql -U postgres0 -d fargoldcity -p 9136
ALTER SCHEMA public OWNER TO postgres0;

GRANT USAGE ON SCHEMA public TO cityuser;
GRANT CREATE ON SCHEMA public TO cityuser;

GRANT CREATE ON TABLESPACE index_space TO cityuser;
```

Теперь можно подключиться к базе от имени cityuser и создать таблицы:

```
psql -U cityuser -d fargoldcity -h 127.0.0.1 -p 9136

CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    name TEXT
);

CREATE INDEX idx_name ON test_table(name) TABLESPACE index_space;
```

Заполняем таблицы тестовыми данными:

```
INSERT INTO test_table (name) VALUES
('Alice'),
('Bob'),
('Charlie'),
('David'),
('Eve'),
('Frank'),
('Grace');
```

Смотрим результаты

``` 
 SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace; 
 SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';
 SELECT indexname, tablespace FROM pg_indexes WHERE schemaname = 'public';
 SELECT * FROM test_table;
```

---

## Вывод

В ходе выполнения лабораторной работы был создан кластер PostgreSQL с заданными параметрами, настроены параметры
сервера, создана база данных и роль, а также заполнены таблицы тестовыми данными. Все табличные пространства были
использованы по назначению. В результате была получена работающая система, готовая к использованию. Таким образом,
данная лабораторная работа была полезна и позволила закрепить знания по настройке и администрированию PostgreSQL.



