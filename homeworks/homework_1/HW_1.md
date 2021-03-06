### Предварительные настройки и подготовка к выполнению ДЗ: ###
#### Установка Ubuntu на Windows: ####

1. Установка WSL на ПК - через команду wsl --install в командной сторке Windows (Важно! Запуск от имени администратора, иначе получим ошибку)
2. По завершению установки - перезагрузка (именно перезагрузка, выключить/включить не считается) ПК
3. Скачивание приложения Ubuntu из магазина Windows
4. Запуск приложения - начнется установка Ubuntu, попросит ввести логин и пароль (Запомнить/записать пароль)
5. Готово! - теперь в консоль линукса можно попадать через терминал винды по команде bash

*Так же для удобства работы был установлен Windows Terminal из того же магазина Windows*

#### Создание виртуальной машины на Яндекс Облаке: ####

1. Заходим на само облако - https://console.cloud.yandex.ru регистрируемся, создаем платежный аккаунт
2. В списке сервисов выбираем Computer Cloud -> Создать ВМ
3. Даем имя ВМ, в моем случае это denis-db-pg-vm-1
4. Зона доступности - без изменений, в моем случае это ru-central1-b
5. ОС - Ubuntu 20.04
6. Диск - без изменений, 15 Гб должно хватить на наши нужды
7. Вычислительный ресурс - Ice Lake, vCPU - 2, Доля CPU - 100%, RAM - 4Гб, Прерываемая - НЕ СТАВИМ!!, судя по описанию её яндекс может тушить в любой момент как пожелает
8. Сетевые настройки - Подсеть - создать сеть, каталог default, имя сети - придумываем (в моем случае denis-vm-db-pg-net-1), ОБЯЗАТЕЛЬНО ставим "Создать подсети"
9. Доступ - задаем логин (в моем случае denis), и теперь самое интересное, генерация SSH Ключа: 
     1) в терминале винды, заходим в терминал линукса командой bash (возможно есть более простые методы генерации ключа, но пока я разобрался только с этим...)
     2) Выполняем команду ssh-keygen -t rsa -b 2048
     3) Попросит ввести файл в который сохранить ключ, тобиш имя файла, в моем случае dk_key, нажимаем энтер, вводим пароль (запомнить/записать пароль)
     4) Ключ создан, теперь надо изменить (ограничить) ему права, для этого выполняем команду chmod 600 ~/*каталог в котором ключ*/*заданное имя*.pub (важно, в винде по умолчанию ключ ложится в каталог с пользователем винды, поэтому путь будет mnt/c/Users и далее уже ваш пользователь и имя ключа)
     5) Посмотреть права ключа можно командой ll -lh (или -ls), увидим что права с -rw-r--r-- поменялись на -rw-------
     6) просматриваем публичную часть ключа командой cat *путь к ключу*/*имя ключа*.pub (в моем случае это cat ./dk_key.pub т.к. я уже находился в каталоге с ключем)
    10) Весь выведеный командой cat ключ копируем и вставляем в поле SSH-ключ, ставим галочку "разрешить доступ к серийной консоли", нажимаем создать
    11) Готово, у меня цена пользования такой ВМ рассчитало 3,52р в час (2535 в месяц) на 09.07.2022
 
#### Подключение к виртуальной машине: ####
 
1. В облаке яндекса находим свою ВМ, убеждаемся что она запущена (статус Running), проваливаемся в нее, смотрим в разделе Сеть публичный айпи и копируем его
2. Выполняем команду ssh -i /*путь к ключу*/*имя ключа* *имя пользователя*@*публичный айпи ВМ*, в моем случае это ssh -i ./dk_key denis@84.252.137.237 
3. Будет вопрос мол вы уверены что вам туды надо, говорим yes, первый раз меня отбило с комментарием что с той стороны разорвало соединение, со второй попытки попросил ввести пароль на ssh ключ и впустил
4. Готово, мы в консоли удаленной ВМ 

#### Установка, настройка, обновление PostgerSQL: ####

1. Для установки PostgreSQL используется команда sudo apt-get -y install postgresql где sudo - повышение прав до суперпользователя (до root), по умолчанию поставится постгрес версии 12
2. С помощью команды pg_lsclusters смотрим кластер постгреса: 
      
        12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log

*здесь видим версию pg, название кластера, порт, состояние (online - рабочий, down - потушен), владельца, путь к кластеру и путь к логам*

3. Обновляем PG командами: 
   
   Обновляем список пакетов:  sudo apt update && sudo apt upgrade -y -q 
   
   Обновляем ключ(?):  sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - 
   
   Повторно обновляем список пакетов:  sudo apt-get update
   
4. Ставим 14 версию PG, для этого выполняем команду: 
    
    sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
    
5. Смотрим версию кластера pg_lsclusters и видем что теперь их два: 

        12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
        14  main    5433 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
  
      *Обращаем внимание что оба кластера называются одинаково main, это нормальная ситуация, так как имена кластеров должны быть уникальны в разрезе версии PG*

6. Сравним методы шифрования паролей для версии 12 и 14, для этого: 
     1) выполняем команду sudo cat /etc/postgresql/12/main/pg_hba.conf где 12 это версия PG, видим, что метод шифрования - md5
     2) выполняем аналогичную команду заменив 12 на 14, и видим что для 14й версии метод шифрования - scram-sha-256
     
     *Таким образом надо быть аккуратнее при переходе с версии на версию, так как пароли могут "поломаться" из-за различия методов шифрования*
        
7. Переименуем кластер PG 12, для этого: 
      1) остановим кластер командой sudo pg_ctlcluster 12 main stop, видим что кластер стал down: 
      
              12  main    5432 down   postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
            
      2) командой sudo pg_renamecluster 12 main test переименовали кластер из main в test 
            
              12  test    5432 down   postgres /var/lib/postgresql/12/test /var/log/postgresql/postgresql-12-test.log
            
      3) командой sudo pg_ctlcluster 12 test start запустим переименованный класте, видим что он стал online:
            
              12  test    5432 online postgres /var/lib/postgresql/12/test /var/log/postgresql/postgresql-12-test.log
            
8. Удалим оба клсатера, для этого: 
      1) Останавливаем кластера той же командой sudo pg_ctlcluster 12 test stop для PG12 и sudo pg_ctlcluster 14 main stop для PG14
            
              12  test    5432 down   postgres /var/lib/postgresql/12/test /var/log/postgresql/postgresql-12-test.log
              14  main    5433 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
      
      2) Выполняем команду sudo pg_dropcluster 12 test и sudo pg_dropcluster 14 main соответственно, видим что команда pg_lsclusters возвращает нам только "шапку", без самих кластеров:
            
              Ver Cluster Port Status Owner Data directory Log file
 
9. Создадим новый кластер для 14й версии, для этого: 
      1) выполняем команду sudo -u postgres pg_createcluster 14 main, где  postgres - это пользователь кластера, 14 - версия PG, main - имя кластера
      2) видим что кластер создан, но не запущен: 
            
              14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
        
      3) запускаем кластер sudo pg_ctlcluster 14 main start
            
              14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
        
            *Обращаем внимание на то, что порт у кластера стал = 5432, хотя когда был кластер 12й версии, он был 5433*
 
10. Готово, СУБД подготовлена к дальнейшей работе

***

### "Основная" часть домашнего задания: ###

1. Запускаем psql под пользователем postgres, для этого выполняем команду sudo -u postgres psql (запуск под своим пользователем, из под пользователя постгрес)
          
      *Так же можно изменить пользователя со своего на postgres командой sudo su postgres и далее уже просто выполнить команду psql* 
          
      Видим что зашли в сам постгрес: 
   
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.
    
        postgres=#
    
2. Создадим свою базу данных в которой будем работать, для этого: 

    1) выполняем команду CREATE DATABASE test; видим результат: 
                        
            CREATE DATABASE
        
     2) командой \l просматриваем список всех БД, видим что наша появилась: 

                                              List of databases
               Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
            -----------+----------+----------+-------------+-------------+-----------------------
            postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
            template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                       |          |          |             |             | postgres=CTc/postgres
            template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                       |          |          |             |             | postgres=CTc/postgres
             test      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
            (4 rows)
        
      3) заходим в нее командой \c test, видим результат:
  
              You are now connected to database "test" as user "postgres".
              test=#

3. Проверяем включен ли автокоммит и выключаем его, для этого: 
      1) выполняем команду  \echo :AUTOCOMMIT, видим результат = on, т.е автокоммит включен
      
      2) выключаем его коммандой \set AUTOCOMMIT OFF, проверяем, видим что значение = OFF
      
      3) Так же можно проверить командой SELECT txid_current(); что транзакция не меняется, выполнив команду несколько раз: 
            
              test=# SELECT txid_current();
              txid_current
              --------------
                      734
              (1 row)

              test=*# SELECT txid_current();
              txid_current
              --------------
                      734
              (1 row)

              test=*# SELECT txid_current();
              txid_current
              --------------
                      734
              (1 row)
            
            *Видим что значение статично, т.е. мы в одной и той же транзакции*

4. Создадим таблицу наполняем её данными: 
        
        create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
        insert into persons(first_name, second_name) values('petr', 'petrov'); 
        commit;
        
      Проверяем что данные внесены: 
        
        select * from persons;
        
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        2 | petr       | petrov
        (2 rows)
      
      В двух разных подключениях к БД видим что таблица выглядит идентично.
    
5. Проверим текущий уровень изоляции транзакций: 

        show transaction isolation level;
        
      Видим что уровень изоляции read committed: 
         
         transaction_isolation
        -----------------------
        read committed
        (1 row)

      На уровне изоляции read committed допускаются такие аномалии как: 
      
      **Неповторяющееся чтение** - т.е. если в первой транзации изменить или удалить данные, то вторая транзакция их увидет после того как первая транзакция завершит свои действия подтверждением внесения изменений (commit).
            
      **Фантомное чтение** - т.е если одна транзакция выбрала данные ко какому либо условию, затем вторая транзакция добавила новые строки по этому же условию и подтвердила действие, то повторный запрос в рамках первой транзакции выведет результат уже с новой строкой
           
      **Аномалия сериализации**  - когда результат фиксации группы транзакций оказывается несогласованным при любых вариантах проведения этих транзакций по очереди

6. Выполним действия по добавлению нового поля на уровне изоляции **read committed**:
        
      В первой транзакции добавим новую строку 
          
        insert into persons(first_name, second_name) values('sergey', 'sergeev');
      
      Видим что в первой транзакции строка имеется (id = 4 потому что я предварительно уже игрался с добавлением строки): 
        
        select * from persons;
        
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        2 | petr       | petrov
        4 | sergey     | sergeev
        (3 rows)

      Во второй транзакции та же выборка эту строку не выведет: 
        
        select * from persons;
        
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        2 | petr       | petrov
        (2 rows)
        
      Завершаем изменение в первой транзакции подтверждая изменения - commit;
        
      Видим что во втрой транзакции теперь так же 3 строки: 
         
        select * from persons;
         
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        2 | petr       | petrov
        4 | sergey     | sergeev
        (3 rows)

      Это происходит так как в данном случае срабатывает аномалия "Фантомное чтение", транзакция 2, которая читает таблицу в рамках своей сессии,  увидет изменения транзакции 1 после подтверждаения действий, при этом завершать транзакцию 2 не требуется.
      
      Так же можно провести "эксперимент" по выявлению аномалии "Неповторяющееся чтение", для этого: 
        
      В транзакции 2 выберем значение поля с id =2: 
        
        select * from persons where id = 2;
        
         id | first_name | second_name
        ----+------------+-------------
         2 | petr       | petrov
        (1 row)

      В транзакции 1 изменим значение поля first_name записи 2 с petr на pavel
        
        update persons set first_name = 'pavel' where id = 2;

      Проверяем в транзакции 2, видим что поле все еще равно petr
        
      Подтверждаем транзакцию 1 (commit)
      
      И теперь видим что выборка по этому же id вывело другое значение first_name в текущей транзакции 2
        
        select * from persons where id = 2;
        
        id | first_name | second_name
        ----+------------+-------------
        2 | pavel      | petrov
        (1 row)
        
      Т.е. каждый раз когда будет изменяться значение по опеределенному условию с подтверждением изменений, транзакция 2 будет видеть это новое значение по тому же условию, т.е. чтение будет неповторяющимся в рамках одной транзакции 2.

7. Выполним действия по добавлению нового поля на уровне изоляции **repeatable read**:

      Для этого сменим уровень изоляции транзакций командой: 
        
        set transaction isolation level repeatable read;
        
        show transaction isolation level;
        
        transaction_isolation
        -----------------------
        repeatable read
        (1 row)

      *На уровне изоляции repeatable read в PostgreSQL допускается только Аномалия сериализации*

      В первой транзакции добавить новую запись 
        
        insert into persons(first_name, second_name) values('sveta', 'svetova');
        
        INSERT 0 1
        
      В первой транзакции видим что строка появилась:
        
        select * from persons;
        
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        4 | sergey     | sergeev
        2 | pavel      | petrov
        6 | sveta      | svetova
        (4 rows)

      Во второй транзакции строки нет:
        
        select * from persons;
        
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        4 | sergey     | sergeev
        2 | pavel      | petrov
        (3 rows)

      Завершаем первую транзакцию, но не завершаем вторую транзакцию , вибираем данные из таблице  во второй транзакции: 
        
        select * from persons;
        
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        4 | sergey     | sergeev
        2 | pavel      | petrov
        (3 rows)
      
      Видим что результат остался точно таким же как и был до завершения первой транзакции, так как в PG не допускается фантомное чтение записей на уровне изоляции repeatable read

      Теперь завершаем вторую транзакцию и повторно выбираем данные: 
        
        select * from persons;
        
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        4 | sergey     | sergeev
        2 | pavel      | petrov
        6 | sveta      | svetova
        (4 rows)

      Видим что строка стала отображаться только после того как мы попробовали выбрать её в новой транзакции, а не в той же, в которой выбирали до изменений.

      Аналогично себя будет вести и изменение строки, например: 
        
      *Для примера создал еще одну запись insert into persons(first_name, second_name) values('sveta2', 'svetova'); т.е. теперь у меня таблица с колонками*:
      
         id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        4 | sergey     | sergeev
        2 | pavel      | petrov
        6 | sveta      | svetova
        7 | sveta2     | svetova
        (5 rows)

      Открываем 2 транзакции с уровнем изоляции repeatable read и в первой транзакции изменяем sveta2 на olga: 
      
        update persons set first_name = 'olga' where id = 7;

      Смотрим первую транзакцию, данные поменялись:
        
        select * from persons where id = 7;
        
        id | first_name | second_name
        ----+------------+-------------
        7 | olga       | svetova
        (1 row)

      Смотрим вторую транзакцию, данные остались прежними: 
        
        select * from persons where id = 7;
        
        id | first_name | second_name
        ----+------------+-------------
        7 | sveta2     | svetova
        (1 row)

      Завершаем первую транзакцию, смотрим во вторую: 
        
        select * from persons where id = 7;
        
        id | first_name | second_name
        ----+------------+-------------
        7 | sveta2     | svetova
        (1 row)

      Данные остались теми же, теперь завершаем вторую транзакцию и выбираем таблицу: 
        
        select * from persons where id = 7;
        
        id | first_name | second_name
        ----+------------+-------------
        7 | olga       | svetova
        (1 row)

      т.е. новые значение мы увидим только в рамках новой транзакции, а не в текущей, как того допускает уровень изоляции read committed.


### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####
