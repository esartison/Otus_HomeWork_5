# Домашнее задание Сартисона Евгения №5 #

Цель:
настроить надёжное резервное копирование и восстановление базы данных


Описание/Пошаговая инструкция выполнения домашнего задания:
Однажды Алексей узнал тревожную новость: у прямого конкурента "TropicalFruits" произошёл масштабный сбой, и их база данных лояльных клиентов была полностью потеряна. Это привело к хаосу: клиенты не могли воспользоваться своими бонусами, заказы обрабатывались с ошибками, а репутация компании пострадала. Некоторые клиенты даже начали переходить к "BananaFlow", но Алексей понимал, что его компания не застрахована от подобных проблем.

Он решил, что нужно срочно настроить современные инструменты для бэкапов — WAL-G или pg_probackup. Однако на пути к успеху его ждали новые вызовы:

***Как настроить бэкапы так, чтобы они не влияли на производительность системы?***
Желательно настроить бэкап на реплике, чтобы не было влияния на работу мастер база

***Как восстановить данные на другом кластере, если что-то пойдёт не так?***
Нужно восстановить pg_datа

***Как проверить, что бэкапы действительно работают?***
Единственный способ проверить что бэкап дейстивительно работает - это восстановить бэкап на другом сервере

Задание:

## Подготовка к заданию ##

создал 2 виртуальные машины в YC 
![image](https://github.com/user-attachments/assets/26e5d891-af2e-48ac-8084-7692d460eed1)


pgtest01: установка Postgres
```
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo sh -c 'echo "deb [arch=amd64 signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt noble-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt install postgresql-17 postgresql-client-17
```
![image](https://github.com/user-attachments/assets/184e3fcc-9a71-49d4-a251-cfaffc5ca2a3)

pgtest01: примонтировать shared backup директорию
> mkdir /backup && mount -t virtiofs pgbackup /backup  && chown postgres:postgres /backup
![image](https://github.com/user-attachments/assets/4fed5c2b-b7a1-4081-8c92-d282de803150)

Выставить параметры и перезапустить кластер
```
postgres@pgtest01:~$ diff  /etc/postgresql/17/main/postgresql.conf /etc/postgresql/17/main/postgresql.conf_bkp4Jub2025
60c60
< listen_addresses = '*'                # what IP address(es) to listen on;
---
> #listen_addresses = 'localhost'               # what IP address(es) to listen on;
268c268
< archive_mode = on             # enables archiving; off, on, or always
---
> #archive_mode = off           # enables archiving; off, on, or always
274,275c274
<                               # placeholders: %p = path of file to archi
< archive_command = 'test ! -f /backup/archives/%f && cp %p /backup/archives/%f'
---
>                               # placeholders: %p = path of file to archive
```

## (1) Настройте бэкапы PostgreSQL с использованием WAL-G, pg_probackup или любого другого аналогичного ПО для базы данных "Лояльность оптовиков".

Делаем инкрементальный бэкап средствами pg_basebackup

pgtest01: проверка директории summaries - пустая ожидаемо
> ls /var/lib/postgresql/17/main/pg_wal/summaries
![image](https://github.com/user-attachments/assets/b1df11ca-c1c1-40f1-8ca0-58b93ab2c547)

pgtest01: выставить summarize_wal = on
![image](https://github.com/user-attachments/assets/57f1cf64-0fbc-41bf-b1d6-ab053bfdf4f7)


pgtest01: проверить что файлы стали появляться в директории summaries
> ls /var/lib/postgresql/17/main/pg_wal/summaries

pgtest01: создание бэкап директорию
>mkdir -p /backup/full
>chown -R postgres:postgres /backup
>chmod -R 755 /backup


pgtest01: выполнить полный бэкап
> postgres@pgtest01:   pg_basebackup -v -D /backup/full
```
postgres@pgtest01:~/17/main/pg_wal/summaries$ pg_basebackup -v -D /backup/full
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/5000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_4791"
pg_basebackup: write-ahead log end point: 0/5000158
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed

postgres@pgtest01:~/17/main/pg_wal/summaries$ du -sh  /backup/full
39M     /backup/full
```


pgtest01: создание тестовой таблицы
>create table demo01 (c1 text);
>insert into demo01 values ('Проверка инкрементального резервного копирования');


pgtest01:	создание папки для инкрементального резервного копирования
```
mkdir -p /backup/{incr_monday,incr_tuesday}
chown -R postgres:postgres /backup
chmod -R 755 /backup
```

pgtest01: выполнение инкрементального резервного копирования
```
sudo -u postgres pg_basebackup --incremental=/backup/full/backup_manifest -D /backup/incr_monday/

insert into demo01 values ('Проверка инкрементального резервного копирования 2');

sudo -u postgres pg_basebackup --incremental=/backup/incr_monday/backup_manifest -D /backup/incr_tuesday/

root@pgtest01:/backup# du -sh /backup/incr*
20M     /backup/incr_monday
20M     /backup/incr_tuesday
```


pgtest01:Создать комбинированную копию из full и инкрементальных бэкапов
```
root@pgtest01:~# mkdir -p /backup/comb
root@pgtest01:~# chown -R postgres:postgres /backup/comb
root@pgtest01:~# chmod -R 755 /backup/comb
root@pgtest01:~#
root@pgtest01:~#
root@pgtest01:~# sudo -u postgres /usr/lib/postgresql/17/bin/pg_combinebackup -o /backup/comb /backup/full /backup/incr_monday /backup/incr_tuesday
root@pgtest01:~# du -sh /backup/comb
39M     /backup/comb
```




## (2) Восстановите данные на другом кластере, чтобы убедиться, что бэкапы работают.
Восстанавливать данные буду с сервера pgtest01 на сервер pgtest02(обе в YC)


pgtest02: установка Postgres
```
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo sh -c 'echo "deb [arch=amd64 signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt noble-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt install postgresql-17 postgresql-client-17
```
![image](https://github.com/user-attachments/assets/76b00b3b-95a3-467b-970e-d4b735003f85)


pgtest02: примонтировать shared backup директорию
> mkdir /backup && mount -t virtiofs pgbackup /backup  && chown postgres:postgres /backup

>mkdir /backup/archives_repl/


pgtest02: Остановить кластер и очистить pg_data
sudo systemctl stop postgresql@17-main
rm -rf /var/lib/postgresql/17/main/


pgtest02: восстановить данные из бэкапа и запустить кластер
cp -R /backup/comb /var/lib/postgresql/17/
mv /var/lib/postgresql/17/comb /var/lib/postgresql/17/main
chmod -R 750 /var/lib/postgresql/17/main/
sudo systemctl start postgresql

pgtest02: данные из лога
![image](https://github.com/user-attachments/assets/b936d384-74e4-4a14-bd78-cc50b1c5b038)



## (3) Проверьте, что данные восстановлены корректно.
Данные были восстановлены успешно и есть информация после обоих инкрементальных бэкапов
![image](https://github.com/user-attachments/assets/9f34ddb8-c6d6-4c37-91c1-e05a628fb756)
Все прошло успешно.



## (4) Дополнительно: Снимите бэкап под нагрузкой с реплики.
pgnode1: создание тестовой таблицы
>create table test_replica_backup (c1 text);
>insert into demo02 values ('Проверка backup-а с реплики');
![image](https://github.com/user-attachments/assets/edbfadf3-6db3-4e8c-9f41-a835eee494b9)

запустить pgbench read-only, чтобы дать нагрузку
pgnode1:pgbench -U postgres -i -s 950 postgres
![image](https://github.com/user-attachments/assets/d61d0fb1-25d3-446f-bd5a-534c892484e3)

pgnode2:pgbench -U postgres -c 50 -j 2 -P 60 -T 600 -S postgres
![image](https://github.com/user-attachments/assets/ae1279cc-3880-4d79-9bf6-638805598726)



pgnode2: во время выполнения Pgbench сделать бэкап в другой сессии
> pg_basebackup -v -D /backup/full_replica

во время выполнения бэкапа с реплики, нужно сделать ручной checkpoint
![image](https://github.com/user-attachments/assets/d94563e7-349b-4664-b97f-743184eda811)

Бэкап завершился успешно
![image](https://github.com/user-attachments/assets/f5f038b0-b98c-4dd6-b0d9-66f83bd698ef)

во время бэкапа нагрузка на сервере реплики была значительной
![image](https://github.com/user-attachments/assets/981f4cce-077b-4170-acd3-7148d656e0d6)

