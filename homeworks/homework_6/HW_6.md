
### Задание 1 ###

#### Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения. ####

   Настраиваем запись в журнал блокировок, удерживаемых более 200 миллисекунд, для этого: 

   В файле postgresql.conf установил значения:
        
        log_lock_waits = on
        deadlock_timeout = 200ms
   
   и перезапустил кластер.
    

   Создана тестовая база test и таблица test_tab и в нее внесены значения: 

        test=# select * From test_tab;
        i |   v
        ---+--------
        1 | один
        2 | два
        3 | три
        4 | четыре
        5 | пять
        (5 rows)

   Создано 3 пользователя user1, user2 и user3, создана роль test_role со всеми правами на test_tab

   Теперь в одном терминальном окне заходим в БД под пользователем user1 и в рамках одной транзакции пробуем изменить запись в таблице: 

        denis@denis-vm1:/etc/postgresql/14/main$ sudo -u postgres psql;
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# \c test user1
        Password for user user1:
        You are now connected to database "test" as user "user1".
        test=> begin;
        BEGIN
        test=*> SELECT pg_backend_pid();
        pg_backend_pid
        ----------------
                17141
        (1 row)

        test=*> update test_tab set v = 'семь' where i = 1;
        UPDATE 1
        test=*>

   Транзакцию не завершаем, и во втором терминальном окне заходим под пользователем user2 и делаем такие же действия: 

        denis@denis-vm1:~$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# \c test user2
        Password for user user2:
        You are now connected to database "test" as user "user2".
        test=> begin;
        BEGIN
        test=*> SELECT pg_backend_pid();
        pg_backend_pid
        ----------------
                17204
        (1 row)

        test=*> update test_tab set v = 'семь' where i = 1;


   Видим что для второго пользователя update не завершается, а ждет когда завершится транзакция первого пользователя.

   Проверяем журнал 

        denis@denis-vm1:/var/log/postgresql$ sudo tail ./postgresql-14-main.log

   Видим следующие записи: 

        2022-08-07 17:27:56.707 UTC [17204] user2@test LOG:  process 17204 still waiting for ShareLock on transaction 750 after 200.103 ms
        2022-08-07 17:27:56.707 UTC [17204] user2@test DETAIL:  Process holding the lock: 17141. Wait queue: 17204.
        2022-08-07 17:27:56.707 UTC [17204] user2@test CONTEXT:  while updating tuple (0,1) in relation "test_tab"
        2022-08-07 17:27:56.707 UTC [17204] user2@test STATEMENT:  update test_tab set v = 'семь' where i = 1;

   Т.е мы видим что в журнал по истечении 200 миллисекунд (after 200.103 ms) записалась информация о том, что транзакция 17204 (наш user2) ожидает завершения транзакции 17141 (нащ user1) чтобы провести изменение строки в таблице test_tab

***

### Задание 2 ###

#### Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая. ####

   Проводим те же манипуляции, но добавляем еще пользователя 3: 

        denis@denis-vm1:~$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# \c test user3
        Password for user user3:
        You are now connected to database "test" as user "user3".
        test=> begin;
        BEGIN
        test=*> SELECT pg_backend_pid();
        pg_backend_pid
        ----------------
                17328
        (1 row)

        test=*> update test_tab set v = 'семь' where i = 1;

   Теперь уже два пользователя ожидают доступ к строке таблицы test_tab, смотрим данные в pg_locks: 

   Для user1 (pid = 17141)

        test=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
        FROM pg_locks WHERE pid = 17141;
        locktype    | relation | virtxid | xid |       mode       | granted
        ---------------+----------+---------+-----+------------------+---------
        relation      | test_tab |         |     | RowExclusiveLock | t
        virtualxid    |          | 3/12    |     | ExclusiveLock    | t
        transactionid |          |         | 752 | ExclusiveLock    | t
        (3 rows)

   Для user2 (pid = 17204) 

        test=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
        FROM pg_locks WHERE pid = 17204;
        locktype    | relation | virtxid | xid |       mode       | granted
        ---------------+----------+---------+-----+------------------+---------
        relation      | test_tab |         |     | RowExclusiveLock | t
        virtualxid    |          | 5/9     |     | ExclusiveLock    | t
        transactionid |          |         | 753 | ExclusiveLock    | t
        transactionid |          |         | 752 | ShareLock        | f
        tuple         | test_tab |         |     | ExclusiveLock    | t
        (5 rows)
   Для user3 (pid = 17328)

        test=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
        FROM pg_locks WHERE pid = 17328;
        locktype    | relation | virtxid | xid |       mode       | granted
        ---------------+----------+---------+-----+------------------+---------
        relation      | test_tab |         |     | RowExclusiveLock | t
        virtualxid    |          | 7/3     |     | ExclusiveLock    | t
        transactionid |          |         | 754 | ExclusiveLock    | t
        tuple         | test_tab |         |     | ExclusiveLock    | f
        (4 rows)

   Так же можно посмотреть так: 

        test=# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'test_tab'::regclass;
        locktype |       mode       | granted |  pid  | wait_for
        ----------+------------------+---------+-------+----------
        relation | RowExclusiveLock | t       | 17328 | {17204}
        relation | RowExclusiveLock | t       | 17204 | {17141}
        relation | RowExclusiveLock | t       | 17141 | {}
        tuple    | ExclusiveLock    | f       | 17328 | {17204}
        tuple    | ExclusiveLock    | t       | 17204 | {17141}
        (5 rows)

   Т.е. таким образом мы видим что транзакция 17141 у нас никого не ждет, транзакция 17204 жден завершения действий транзакции 17141 (колонка wait_for), а транзакция 17328 ждет завершения транзакции 17204

   Теперь завершаем транзакцию первого пользователя и проверяем pg_locks: 

        test=# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'test_tab'::regclass;
        locktype |       mode       | granted |  pid  | wait_for
        ----------+------------------+---------+-------+----------
        relation | RowExclusiveLock | t       | 17328 | {17204}
        relation | RowExclusiveLock | t       | 17204 | {}
        tuple    | ExclusiveLock    | t       | 17328 | {17204}
        (3 rows)
    
   Видим что первый пользователь "ушел" из выборки, транзакция 17204 никого не ждет, транзакция 17328 ожидает завершения транзакции 17204 

   Завершаем транзакцию второго пользователя, и повторно смотрим: 

        test=# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'test_tab'::regclass;
        locktype |       mode       | granted |  pid  | wait_for
        ----------+------------------+---------+-------+----------
        relation | RowExclusiveLock | t       | 17328 | {}
        (1 row)

   Осталась только транзакция 17328, более блокировок нет. 

***

### Задание 3 ###

#### Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений? ####

   Простой способ воспроизведения взаимоблокировки сводится к следующим шагам: 

   Шаг 1. Пользователь 1 измеяет строку 1 : 

        test=> begin;
        BEGIN
        test=*> update test_tab set v = 'два' where i = 1;

   Шаг 2. Пользователь 2 изменяет строку 2: 

        test=> begin;
        BEGIN
        test=*> update test_tab set v = 'один' where i = 2;

   Шаг 3. Пользователь 1 все в той же транзакции изменяет строку 2: 

         test=*> update test_tab set v = 'три' where i = 2;

   При этом мы просто становимя в ожидание доступа к записи которую держит пользователь 2 в своей транзакции

   Шаг 4. Пользователь 2 в своей транзакции пытается изменить строку 1: 

        test=*> update test_tab set v = 'четыре' where i = 1;
        ERROR:  deadlock detected
        DETAIL:  Process 17204 waits for ShareLock on transaction 755; blocked by process 17141.
        Process 17141 waits for ShareLock on transaction 756; blocked by process 17204.
        HINT:  See server log for query details.
        CONTEXT:  while updating tuple (0,1) in relation "test_tab"
        test=!>

   В результате чего возникает взаимоблокировка - ERROR:  deadlock detected

   Теперь посмотрим в журнал, и видим что данная информация так же отразилась в журнале: 

        2022-08-07 18:26:02.617 UTC [17204] user2@test LOG:  process 17204 detected deadlock while waiting for ShareLock on transaction 755 after 200.085 ms
        2022-08-07 18:26:02.617 UTC [17204] user2@test DETAIL:  Process holding the lock: 17141. Wait queue: .
        2022-08-07 18:26:02.617 UTC [17204] user2@test CONTEXT:  while updating tuple (0,1) in relation "test_tab"
        2022-08-07 18:26:02.617 UTC [17204] user2@test STATEMENT:  update test_tab set v = 'четыре' where i = 1;
        2022-08-07 18:26:02.617 UTC [17204] user2@test ERROR:  deadlock detected
        2022-08-07 18:26:02.617 UTC [17204] user2@test DETAIL:  Process 17204 waits for ShareLock on transaction 755; blocked by process 17141.
                Process 17141 waits for ShareLock on transaction 756; blocked by process 17204.
                Process 17204: update test_tab set v = 'четыре' where i = 1;
                Process 17141: update test_tab set v = 'три' where i = 2;
        2022-08-07 18:26:02.617 UTC [17204] user2@test HINT:  See server log for query details.
        2022-08-07 18:26:02.617 UTC [17204] user2@test CONTEXT:  while updating tuple (0,1) in relation "test_tab"
        2022-08-07 18:26:02.617 UTC [17204] user2@test STATEMENT:  update test_tab set v = 'четыре' where i = 1;
        2022-08-07 18:26:02.617 UTC [17141] user1@test LOG:  process 17141 acquired ShareLock on transaction 756 after 14534.100 ms
        2022-08-07 18:26:02.617 UTC [17141] user1@test CONTEXT:  while updating tuple (0,2) in relation "test_tab"
        2022-08-07 18:26:02.617 UTC [17141] user1@test STATEMENT:  update test_tab set v = 'три' where i = 2;
        2022-08-07 18:26:02.617 UTC [17141] user1@test LOG:  duration: 14534.319 ms  statement: update test_tab set v = 'три' where i = 2;

***

### Задание 4 ###

#### Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга? ####

#### Попробуйте воспроизвести такую ситуацию. ####

   При выполнении команды update без условия where не возникнет взаимоблокировки, но возникнет блокировка RowExclusiveLock 

   Выполняем update под пользователем 1: 

        denis@denis-vm1:~$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# \c test user1
        Password for user user1:
        You are now connected to database "test" as user "user1".
        test=> begin;
        BEGIN
        test=*> SELECT pg_backend_pid();
        pg_backend_pid
        ----------------
                1246
        (1 row)

        test=*> update test_tab set v = '---';
        UPDATE 5
        test=*>    

   Теперь выполняем под пользователем 2: 


        denis@denis-vm1:~$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# \c test user2
        Password for user user2:
        You are now connected to database "test" as user "user2".
        test=> begin;
        BEGIN
        test=*> SELECT pg_backend_pid();
        pg_backend_pid
        ----------------
                1266
        (1 row)

        test=*> update test_tab set v = '+++';

   Смотрим в pg_locks таблицы test_tab

        test=# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'test_tab'::regclass;
        locktype |       mode       | granted | pid  | wait_for
        ----------+------------------+---------+------+----------
        relation | RowExclusiveLock | t       | 1266 | {1246}
        relation | RowExclusiveLock | t       | 1246 | {}
        tuple    | ExclusiveLock    | t       | 1266 | {1246}
        (3 rows)

   Видим что пользователь 1 (pid 1246) никого не ждет, а вот пользователь 2 (pid 1266) ждет завершения транзакции пользователя 1


### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####    
