Запуск vagrantfile + playbook

```bash
vagrant up --no-provision
vagrant provision
```

ДЗ:<br>
1) Настройка hot_standby репликации с использованием слотов<br>
2) Настройка правильного резервного копирования<br>
<br>
<br>
<h2>Настройка hot_standby репликации с использованием слотов</h2>
Перед настройкой репликации необходимо установить postgres-server на хосты node1 и node2:<br><br>
1) Добавим postgres репозиторий:<br>

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm<br>
```
2) Исключаем старый postgresql модуль:<br>

```bash
yum -qy module disable postgresql
```

3) Устанавливаем postgresql-server 15:<br>

```bash
yum install -y postgresql15-server
```

4) Выполняем инициализацию кластера:<br>

```bash
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
```

5) Запускаем postgresql-server:<br>

```bash
systemctl start postgresql-15
```

6) Добавляем postgresql-server в автозагрузку:<br>

```bash
systemctl enable postgresql-15
```

<br><br>

Далее приступаем к настройке репликации:<br><br>
На хосте node1:<br>
1) Заходим в psql:<br>

```bash
[vagrant@node1 ~]$ sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.5)
Type "help" for help.

postgres=# 
```

2) В psql создаём пользователя replicator c правами репликации и паролем «Otus2022!»<br>

```sql
CREATE USER replicator WITH REPLICATION Encrypted PASSWORD 'Otus2022!';
```

3) В файле /var/lib/pgsql/15/data/postgresql.conf указываем следующие параметры:<br>

```bash
#Указываем ip-адреса, на которых postgres будет слушать трафик на порту 5432 (параметр port)
listen_addresses = 'localhost, 192.168.56.11'
#Указываем порт порт postgres
port = 5432 
#Устанавливаем максимально 100 одновременных подключений
max_connections = 100
log_directory = 'log' 
log_filename = 'postgresql-%a.log' 
log_rotation_age = 1d 
log_rotation_size = 0 
log_truncate_on_rotation = on 
max_wal_size = 1GB
min_wal_size = 80MB
log_line_prefix = '%m [%p] ' 
#Указываем часовой пояс для Москвы
log_timezone = 'UTC+3'
timezone = 'UTC+3'
datestyle = 'iso, mdy'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8' 
lc_numeric = 'en_US.UTF-8' 
lc_time = 'en_US.UTF-8' 
default_text_search_config = 'pg_catalog.english'
#можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления; 
hot_standby = on
#Включаем репликацию
wal_level = replica
#Количество планируемых слейвов
max_wal_senders = 3
#Максимальное количество слотов репликации
max_replication_slots = 3
#будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.
hot_standby_feedback = on
#Включаем использование зашифрованных паролей
password_encryption = scram-sha-256
```


4) Настраиваем параметры подключения в файле /var/lib/pgsql/15/data/pg_hba.conf:<br>

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all                  all                                                peer
# IPv4 local connections:
host    all                  all             127.0.0.1/32              scram-sha-256
# IPv6 local connections:
host    all                  all             ::1/128                       scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication      all                                                peer
host    replication     all             127.0.0.1/32               scram-sha-256
host    replication     all             ::1/128                        scram-sha-256
host    replication replication    192.168.56.11/32        scram-sha-256
host    replication replication    192.168.56.12/32        scram-sha-256
```

Две последние строки в файле разрешают репликацию пользователю replication.<br><br>

5) Перезапускаем postgresql-server:<br>

```bash
systemctl restart postgresql-15.service
```

<br>
<br>
На хосте node2:<br>
1) Останавливаем postgresql-server:<br>

```bash
systemctl stop postgresql-15.service
```

2) С помощью утилиты pg_basebackup копируем данные с node1:<br>

```bash
pg_basebackup -h 192.168.56.11 -U  replication -p 5432 -D /var/lib/pgsql/15/data/ -R -P
```

3) В файле  var/lib/pgsql/15/data/postgresql.conf меняем параметр:<br>

```bash
listen_addresses = 'localhost, 192.168.56.12'
```

4) Запускаем службу postgresql-server:<br>

```bash
systemctl start postgresql-15.service
```

Проверка репликации:<br>
На хосте node1 в psql создадим базу otus_test и выведем список БД:<br>

```sql
postgres=# CREATE DATABASE otus_test;
CREATE DATABASE
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus_test | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres  
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres    
(4 rows)
postgres=# 
```

На хосте node2 также в psql также проверим список БД (команда \l), в списке БД должна появится БД otus_test.<br>
<br>
Также можно проверить репликацию другим способом:<br>
На хосте node1 в psql вводим команду:<br>

```sql
select * from pg_stat_replication;
```

На хосте node2 в psql вводим команду:<br>

```sql
select * from pg_stat_wal_receiver;
```

Вывод обеих команд должен быть не пустым.<br>

На этом настройка репликации завершена.<br>

<br>
<br>
В случае выхода из строя master-хоста (node1), на slave-сервере (node2) в psql необхоимо выполнить команду:<br>

```sql
select pg_promote();
```

Также можно создать триггер-файл. Если в дальнейшем хост node1 заработает корректно, то для восстановления его работы (как master-сервера) необходимо:<br>
<ul>
<li>Настроить сервер node1 как slave-сервер</li>
<li>Также с помощью команды select pg_promote(); перевести режим его работы в master</li>
</ul>

<h2>Настройка резервного копирования (barman)</h2>

На хостах node1 и node2 необходимо установить утилиту barman-cli, для этого:<br>
1)	Устанавливаем epel-release:

```bash
dnf install epel-release -y 
```

2)	Устанавливаем barman-cli:

```bash
dnf install barman-cli
```
<br>
На хосте barman выполняем следующие настройки:<br>
1)	*предаварительно отключаем firewalld и SElinux<br>
2)	Устанавливаем epel-release:

```bash
dnf install epel-release -y
```

3)	Добавим postgres репозиторий:

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

4)	Исключаем старый postgresql модуль:

```bash
yum -qy module disable postgresql
```

5)	Устанавливаем пакеты barman и postgresql-client:

```bash
dnf install barman-cli barman postgresql15
```

6)	Переходим в пользователя barman и генерируем ssh-ключ:

```bash
su barman
cd 
ssh-keygen -t rsa -b 4096
```
<br>
На хосте node1:<br>
7)	Переходим в пользователя postgres и генерируем ssh-ключ:

```bash
su postgres
cd 
ssh-keygen -t rsa -b 4096
```

8)	После генерации ключа, выводим содержимое файла ~/.ssh/id_rsa.pub:

```bash
cat ~/.ssh/id_rsa.pub
```

Копируем содержимое файла на сервер barman в файл /var/lib/barman/.ssh/authorized_keys<br>

9)	В psql создаём пользователя barman c правами суперпользователя:

```sql
CREATE USER barman WITH REPLICATION Encrypted PASSWORD 'Otus2022!';
```

10)	В файл /var/lib/pgsql/15/data/pg_hba.conf добавляем разрешения для пользователя barman:<br>

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32                 scram-sha-256
host    replication     all             ::1/128                            scram-sha-256
host    replication replication   192.168.56.11/32        scram-sha-256
host    replication replication   192.168.56.12/32        scram-sha-256
host    all                 barman       192.168.56.13/32        scram-sha-256
host    replication   barman       192.168.56.13/32      scram-sha-256
```

11)	Перезапускаем службу postgresql-15:

```bash
systemctl restart postgresql-15
```

12)	В psql создадим тестовую базу otus:

```sql
CREATE DATABASE otus;
```

13)	В базе создаём таблицу test в базе otus:

```sql
\c otus; 
CREATE TABLE test (id int, name varchar(30));
INSERT INTO test VALUES (1, alex); 
```

<br>
<br>
На хосте barman:<br>
14)	После генерации ключа, выводим содержимое файла ~/.ssh/id_rsa.pub:

```bash
cat ~/.ssh/id_rsa.pub
```

Копируем содержимое файла на сервер postgres в файл /var/lib/pgsql/.ssh/authorized_keys<br>

15)	Находясь в пользователе barman создаём файл ~/.pgpass со следующим содержимым: 

```bash
192.168.56.11:5432:*:barman:Otus2022!
```
<br>
В данном файле указываются реквизиты доступа для postgres. Через знак двоеточия пишутся следующие параметры:<br>
<ul>
<li>ip-адрес</li>
<li>порт postgres</li>
<li>имя БД (* означает подключение к любой БД)</li>
<li>имя пользователя</li>
<li>пароль пользователя</li>
</ul>
             Файл должен быть с правами 600, владелец файла barman. <br>

16)	После создания postgres-пользователя barman необходимо проверить, что права для пользователя настроены корректно:<br><br>

Проверяем возможность подключения к postgres-серверу: <br>

```bash
bash-4.4$ psql -h 192.168.56.11 -U barman -d postgres 
psql (14.5)
Type "help" for help.

postgres=# \q
bash-4.4$ 
```

Проверяем репликацию: <br>

```bash
              bash-4.4$ psql -h 192.168.56.11 -U barman -c "IDENTIFY_SYSTEM" replication=1
                    systemid       | timeline |  xlogpos  | dbname 
---------------------+----------+-----------+--------
 7151863316617733050 |        1 | 0/4000E78 | 
(1 row)
              bash-4.4$
```

17)	Создаём файл /etc/barman.conf со следующим содержимым 
 Владельцем файла должен быть пользователь barman.

```bash
[barman]
#Указываем каталог, в котором будут храниться бекапы
barman_home = /var/lib/barman
#Указываем каталог, в котором будут храниться файлы конфигурации бекапов
configuration_files_directory = /etc/barman.d
#пользователь, от которого будет запускаться barman
barman_user = barman
#расположение файла с логами
log_file = /var/log/barman/barman.log
#Используемый тип сжатия
compression = gzip
#Используемый метод бекапа
backup_method = rsync
archiver = on
retention_policy = REDUNDANCY 3
immediate_checkpoint = true
#Глубина архива
last_backup_maximum_age = 4 DAYS
minimum_redundancy = 1
```

18)	 Создаём файл /etc/barman.d/node1.conf со следующим содержимым 
 Владельцем файла должен быть пользователь barman.

 ```bash
[node1]
#Описание задания
description = "backup node1"
#Команда подключения к хосту node1
ssh_command = ssh postgres@192.168.56.11 
#Команда для подключения к postgres-серверу
conninfo = host=192.168.56.11 user=barman port=5432 dbname=postgres
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 7 days
wal_retention_policy = main
streaming_archiver=on
#Указание префикса, который будет использоваться как $PATH на хосте node1
path_prefix = /usr/pgsql-15/bin/
#настройки слота
create_slot = auto
slot_name = node1
#Команда для потоковой передачи от postgres-сервера
streaming_conninfo = host=192.168.56.11 user=barman 
#Тип выполняемого бекапа
backup_method = postgres
archiver = off
```

19)	На этом настройка бекапа завершена. Теперь проверим работу barman:<br>

```bash
bash-4.4$ barman switch-wal node1
The WAL file 000000010000000000000005 has been closed on server 'node1'
bash-4.4$ barman cron 
Starting WAL archiving for server node1
bash-4.4$ barman check node1
Server node1:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
	backup minimum size: OK (0 B)
	wal maximum age: OK (no last_wal_maximum_age provided)
	wal size: OK (0 B)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK (no system Id stored on disk)
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archiver errors: OK
bash-4.4$ 
```

Если во всех пунктах, кроме выделенных будет OK, значит бекап отработает корректно. Если в остальных пунктах вы видите FAILED, то бекап у вас не запустится. Требуется посмотреть в логах, в чём может быть проблема...<br>

20)	 После этого запускаем резервную копию: <br>

```bash
bash-4.4$ barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20221008T010731
Backup start at LSN: 0/6000148 (000000010000000000000006, 00000148)
Starting backup copy via pg_basebackup for 20221008T010731
Copy done (time: 1 second)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
	000000010000000000000004 from server node1 has been removed
	000000010000000000000005 from server node1 has been removed
Backup size: 41.8 MiB
Backup end at LSN: 0/8000000 (000000010000000000000007, 00000000)
Backup completed (start time: 2022-10-08 01:07:31.939958, elapsed time: 1 second)
Processing xlog segments from streaming for node1
	000000010000000000000006
	000000010000000000000007
bash-4.4$ 
```

На этом процесс настройки бекапа закончен, для удобства команду barman backup node1 требуется добавить в crontab. <br>

Проверка восстановления из бекапов:<br>

На хосте node1 в psql удаляем базы Otus: <br>

```bash
bash-4.4$ psql
psql (14.5)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 otus_test | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(5 rows)

postgres=# DROP DATABASE otus;
DROP DATABASE
postgres=# 
postgres=# DROP DATABASE otus_test; 
DROP DATABASE
postgres=# \l 
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)

postgres=# 
```

Далее на хосте barman запустим восстановление:<br> 

```bash
bash-4.4$ barman list-backup node1
node1 20221008T010731 - Fri Oct  7 22:07:50 2022 - Size: 41.8 MiB - WAL Size: 0 B
bash-4.4$ 
bash-4.4$ barman recover node1 20221008T010731 /var/lib/pgsql/15/data/ --remote-ssh-comman "ssh postgres@192.168.56.11"
The authenticity of host '192.168.56.11 (192.168.56.11)' can't be established.
ECDSA key fingerprint is SHA256:NDadubkUsCyw+X3o+WPVePaWJ+5Bl99wfYw5/JdrNYs.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20221008T010731
Destination directory: /var/lib/pgsql/15/data/
Remote command: ssh postgres@192.168.56.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2022-10-08 01:20:24.864427+00:00, elapsed time: 4 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
bash-4.4$ 
```

Далее на хосте node1 потребуется перезапустить postgresql-сервер и снова проверить список БД. Базы otus должны вернуться обратно… 

