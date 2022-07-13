### Установка и настройка PostgreSQL в контейнере Docker ###

*Предварительно созданы 2 виртуальные машины с Ubuntu 20.04 в облаке Яндекса*

1. Устанавливаем и настраиваем докер Docker, для этого: 

    Выполняем команду скачивания "оболочки" для установки докера:

        curl -fsSL https://get.docker.com -o get-docker.sh

    Затем выполняем команду установки самого докера:

        sudo sh get-docker.sh

2. Докер установлен, теперь добавляем нашего пользователя ВМ в группу докера: 

        sudo usermod -aG docker $USER

    *Это делается для того, чтобы у пользователя было право на запуск самого докера*

3. Создаем докер-сеть, для этого выполняем команду: 

        sudo docker network create pg-net
    
    *Докер-сеть это виртуальная сеть созданная специально для докер-контейнера*

4. Добавляем/разворачиваем в этой сети контейнер с сервером Постгреса: 

        sudo docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
    

5. Создаем отдельный контейнер для доступа к нашему серверу и из него подключаемся к серверу: 

        sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
        
        Password for user postgres:
        psql (14.4 (Debian 14.4-1.pgdg110+1))
        Type "help" for help.

        postgres=#  

    Создаем   базу данных: 

        postgres=# CREATE DATABASE test;
        CREATE DATABASE
    
    Создаем таблицу, наполяем её данными (сделал такую же как в ДЗ_1):
        
        test=# create table persons(id serial, first_name text, second_name text);
        CREATE TABLE
        test=# insert into persons(first_name, second_name) values('ivan', 'ivanov');
        INSERT 0 1
        test=# select * from persons;
            id | first_name | second_name
        ----+------------+-------------
            1 | ivan       | ivanov
        (1 row)

6. Выйдем из контейнера с клиентом \q и проверим что контейнес с сервером остался запущенным: 

            sudo docker ps -a
    
    Видим: 

        CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
        8dd1eea47dca   postgres:14   "docker-entrypoint.s…"   13 minutes ago   Up 13 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker

    *Обращаем внимание что контейнер всего один, это произошло потому, что при создании контейнера с клиентом использовался ключ --rm который говорит о том, что при выходе из контейнера его надо удалять*

7. Подключаемся по локалхосту к нашему контейнеру с постгресом:

        denis@denis-db-pg-vm-1:~$ psql -h localhost -U postgres -d postgres
        Password for user postgres:
        psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
        WARNING: psql major version 12, server major version 14.
            Some psql features might not work.
        Type "help" for help.

        postgres=#

    *Мы смогли подключиться, так как у нас открыт доступ по порту 5432 - 0.0.0.0:5432->5432/tcp, :::5432->5432/tcp*

    Видим что созданная даза test присутствует (я создавал еще test2): 

        postgres=# \l
                                        List of databases
        Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
        -----------+----------+----------+------------+------------+-----------------------
        postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
        template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                    |          |          |            |            | postgres=CTc/postgres
        template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                    |          |          |            |            | postgres=CTc/postgres
        test      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
        test2     | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
        (5 rows)

    Заходим в test и видим что таблица так же на месте: 

        postgres=# \c test
        psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
        WARNING: psql major version 12, server major version 14.
                Some psql features might not work.
        You are now connected to database "test" as user "postgres".
        test=# select * from persons;
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        (1 row)

8. Подключаемся к контейнеру с сервером извне:

        denis_2@denis-db-pg-vm-2:~$ psql -p 5432 -U postgres -h 51.250.109.244 -d postgres -W
        Password:
        psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
        WARNING: psql major version 12, server major version 14.
                Some psql features might not work.
        Type "help" for help.

        postgres=# 

    *Я подключился со второй ВМ*

        
9. Удаляем контейнер с сервером:

    Выполняем команду:
    
        denis@denis-db-pg-vm-1:~$ sudo docker container rm 8dd1eea47dca

    И видим, что получили ошибку:

        Error response from daemon: You cannot remove a running container 8dd1eea47dca13563ebb3e91081e87b1abee7ad4e3dfa938cb4781de0ce4be57. Stop the container before attempting removal or force remove 

    Это произошло потому, что контейнер, запущен, а перед удалением его необходимо остановить: 

        denis@denis-db-pg-vm-1:~$ sudo docker container stop 8dd1eea47dca
        8dd1eea47dca

    Проверяем что контейнер остановлен: 

        denis@denis-db-pg-vm-1:~$ sudo docker ps -a
        CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS                      PORTS     NAMES
        8dd1eea47dca   postgres:14   "docker-entrypoint.s…"   About an hour ago   Exited (0) 50 seconds ago             pg-docker

    *Видим что статус стал Exited (0), порты отсутствуют*

    Повторяем удаление: 

        denis@denis-db-pg-vm-1:~$ sudo docker container rm 8dd1eea47dca
        8dd1eea47dca
    
    Проверяем что контейнера нет:

        denis@denis-db-pg-vm-1:~$ sudo docker ps -a
        CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
        denis@denis-db-pg-vm-1:~$

10. Заново создаем контейнер с сервером и подключаемся к нему: 

    Выполняем команду:

        sudo docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
        
    Проверяем что контейнер есть: 

        sudo docker ps -a

        CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
        5318bfcacce1   postgres:14   "docker-entrypoint.s…"   3 seconds ago   Up 3 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker

    Подключаемся к нему из контейнера с клиентом: 

        sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
        
        Password for user postgres:
        psql (14.4 (Debian 14.4-1.pgdg110+1))
        Type "help" for help.

        postgres=#

    Проверяем что созданные нами базы остались на месте: 

        postgres=# \l
                                        List of databases
        Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
        -----------+----------+----------+------------+------------+-----------------------
        postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
        template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                |          |          |            |            | postgres=CTc/postgres
        template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                |          |          |            |            | postgres=CTc/postgres
        test      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
        test2     | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
        (5 rows)

    Проверяем что созданная таблица в базе test так же осталась: 

        postgres=# \c test
        You are now connected to database "test" as user "postgres".
        test=# select * From persons;
        id | first_name | second_name
        ----+------------+-------------
        1 | ivan       | ivanov
        (1 row)


### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####
