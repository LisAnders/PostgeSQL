### Основная часть ДЗ ###

1. Создана виртуальная машина в Яндекс Облако

2. Устанавливаем PostgreSQL 14

        sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

3. Проверяем что кластер запущен: 

        denis@denis-vm-1:~$ sudo -u postgres pg_lsclusters
        Ver Cluster Port Status Owner    Data directory              Log file
        14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

4. Заходим из под пользователя postgres в pslq и создаем таблицу с данными: 

    Зашли из под пользователя postgres:

        denis@denis-vm-1:~$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=#

    Создали таблицу:

        postgres=# create table test(c1 text);
        CREATE TABLE

    Наполнили данными: 

        postgres=# insert into test values('1');
        INSERT 0 1
    
    Вышли: 

        postgres=# \q
        denis@denis-vm-1:~$

5. Останавливаем кластер: 

        denis@denis-vm-1:~$ sudo -u postgres pg_ctlcluster 14 main stop

6. Создаем новый диск в Яндекс Облако, для этого: 
    
    1. Переходим в раздел "Диски"
    2. Нажимаем кнопку "Создать Диск"
    3. Вводим имя диска, в моем случае - ddisk, зона доступности без изменений (та же что у ВМ - ru-central1-b), тип - HDD, размер блока - 4, размер диска - 10Гб, Наполнение - Пустой

7. Добавляем новый диск к созданной ВМ: 

    1. Заходим в "Обзор" диска в Яндекс Облаке
    2. В верхнем правом углу нажимаем на три точки -> "присоеденить"
    3. Выбираем нашу созданную ВМ, нажимаем подключить

8. Проинициализируем диск, для этого: 

    1. Установим утилиту parted

            denis@denis-vm-1:/dev$ sudo apt update
            denis@denis-vm-1:/dev$ sudo apt install parted

    2. Проверяем какой из дисков у нас не инициализированный (наш созданный диск): 

            denis@denis-vm-1:/dev$ sudo parted -l | grep Error
            Error: /dev/vdb: unrecognised disk label

        Или так же можем посмотреть какие имеются диски через lsblk: 

            denis@denis-vm-1:/dev$ lsblk
            NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
            vda    252:0    0  15G  0 disk
            ├─vda1 252:1    0   1M  0 part
            └─vda2 252:2    0  15G  0 part /
            vdb    252:16   0  10G  0 disk

    3. Выбираем стандарт разметки диска (согласно инструкции есть 2 варианта GPT и MBR, написано что GPT для "стандартного облочного сервера" более предпочтительный вариант, поэтому выбираем его): 

            denis@denis-vm-1:/dev$ sudo parted /dev/vdb mklabel gpt
        

    4. Создаем новое пространство которое будем в дальнейшем использовать: 

            denis@denis-vm-1:/dev$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%

        Видим что пространство появилось: 

            denis@denis-vm-1:/dev$ lsblk
            NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
            vda    252:0    0  15G  0 disk
            ├─vda1 252:1    0   1M  0 part
            └─vda2 252:2    0  15G  0 part /
            vdb    252:16   0  10G  0 disk
            └─vdb1 252:17   0  10G  0 part

    5. Создаем файловую систему нового пространства диска: 

            denis@denis-vm-1:/dev$ sudo mkfs.ext4 -L datapartition /dev/vdb1
            mke2fs 1.45.5 (07-Jan-2020)
            Creating filesystem with 2620928 4k blocks and 655360 inodes
            Filesystem UUID: 3c63544a-69b2-4029-bbf3-2dab4478d67f
            Superblock backups stored on blocks:
                    32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

            Allocating group tables: done
            Writing inode tables: done
            Creating journal (16384 blocks): done
            Writing superblocks and filesystem accounting information: done

        Видим что файловая система создалась:

            denis@denis-vm-1:/dev$ lsblk -fs
            NAME  FSTYPE LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINT
            vda1
            └─vda
            vda2  ext4                 82afb880-9c95-44d6-8df9-84129f3f2cd1   11.4G    18% /
            └─vda
            vdb1  ext4   datapartition 3c63544a-69b2-4029-bbf3-2dab4478d67f
            └─vdb

    6. Монтируем новый диск в /mnt/data 

        Создаем новую папку: 
        
            denis@denis-vm-1:/dev$ sudo mkdir -p /mnt/data

        Монтируем туда наш новый диск: 

            denis@denis-vm-1:/dev$ sudo mount -o defaults /dev/vdb1 /mnt/data/

        Проверяем что успешно примонтировалось:

            denis@denis-vm-1:/dev$ df -h -x tmpfs
            Filesystem      Size  Used Avail Use% Mounted on
            udev            1.9G     0  1.9G   0% /dev
            /dev/vda2        15G  2.7G   12G  19% /
            /dev/vdb1       9.8G   24K  9.3G   1% /mnt/data

9. Меняем владельца каталога /mnt/data на postgres

        denis@denis-vm-1:/dev$ sudo chown -R postgres:postgres /mnt/data/

        denis@denis-vm-1:/dev$ ls -l /mnt/
        total 4
        drwxr-xr-x 3 postgres postgres 4096 Jul 24 09:12 data

10. Переносим содержимое /var/lib/postgres/14 в /mnt/data

        denis@denis-vm-1:/dev$ sudo mv /var/lib/postgresql/14 /mnt/data/

11. Пробуем запустить кластер: 

        denis@denis-vm-1:/$ sudo -u postgres pg_ctlcluster 14 main start
    
    Возникла ошибка: 

        Error: /var/lib/postgresql/14/main is not accessible or does not exist

    Так как в конфигурационном файле по пути /etc/postgresql/14/main  -  postgresql.conf прописан путь по умолчанию = data_directory = '/var/lib/postgresql/14/main'

    Меняем путь на /mnt/data/14/main (то, куда перенесли все данные) и пробуем запустить кластер: 

        denis@denis-vm-1:/etc/postgresql/14/main$ sudo -u postgres pg_ctlcluster 14 main start
        
        denis@denis-vm-1:/etc/postgresql/14/main$ pg_lsclusters
        Ver Cluster Port Status Owner    Data directory    Log file
        14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

12. Заходим через psql и проверяем содержимое таблицы: 

        denis@denis-vm-1:/etc/postgresql/14/main$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# select * from test;
        c1
        ----
        1
        (1 row)
Итого, мы успешно перенесли кластер с данными на внешний диск и можем с ними работать.

***

### Задание со звездочкой * ###

1. Создана вторая ВМ в Яндекс Облако - denis-vm-2

2. На нее так же ставим PostgreSQL 14 

        denis@denis-vm-2:~$ pg_lsclusters
        Ver Cluster Port Status Owner    Data directory              Log file
        14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

3. Удаляем данные из /var/lib/postgresql

    1. Останавливаем кластер

            sudo -u postgres pg_ctlcluster 14 main stop

            pg_lsclusters
            Ver Cluster Port Status Owner    Data directory              Log file
            14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

    2. Удаляем данные 

            denis@denis-vm-2:/var/lib/postgresql$ sudo -u postgres rm -R ./*

            denis@denis-vm-2:/var/lib/postgresql$ ll
            total 8
            drwxr-xr-x  2 postgres postgres 4096 Jul 24 10:09 ./
            drwxr-xr-x 37 root     root     4096 Jul 24 10:03 ../

4. Перемонтируем жесткий диск который был ранее создан для первой ВМ:

    1. В консоли Яндекс Облака отсоединям диск от первой ВМ, присоединяем ко второй.
        
        Видим что теперь диск на второй ВМ:

            denis@denis-vm-2:/dev$ lsblk -fs
            NAME  FSTYPE LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINT
            vda1
            └─vda
            vda2  ext4                 82afb880-9c95-44d6-8df9-84129f3f2cd1   11.5G    18% /
            └─vda
            vdb1  ext4   datapartition 3c63544a-69b2-4029-bbf3-2dab4478d67f
            └─vdb

    2. монтируем его в каталок /mnt/data
    
      Создаем каталог:

            denis@denis-vm-2:/$ sudo mkdir -p /mnt/data
      
      Монтируем в него диск:
      
            denis@denis-vm-2:/$ sudo mount -o defaults /dev/vdb1 /mnt/data/
      
      Убеждаемся что все прошлоуспешно:
      
            denis@denis-vm-2:/$ df -h -x tmpfs
            Filesystem      Size  Used Avail Use% Mounted on
            udev            1.9G     0  1.9G   0% /dev
            /dev/vda2        15G  2.6G   12G  19% /
            /dev/vdb1       9.8G   42M  9.2G   1% /mnt/data

5. Изменим настройку в postgresql.conf на новый путь и попробуем запустить кластер: 

        denis@denis-vm-2:/$ sudo -u postgres pg_ctlcluster 14 main start

    Кластер поднялся: 

        denis@denis-vm-2:/$ pg_lsclusters
        Ver Cluster Port Status Owner    Data directory    Log file
        14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

6. Проверим что видим данные, созданные на первой ВМ: 

        denis@denis-vm-2:/$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# select * from test;
        c1
        ----
        1
        (1 row)

Итого, мы смогли перенести кластер с данными с одной ВМ на другую, и при этом работать с ними на внемшем устройстве.

### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####
