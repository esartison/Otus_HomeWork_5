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
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo sh -c 'echo "deb [arch=amd64 signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt noble-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt install postgresql-17 postgresql-client-17
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

проверка что wal идут в /backup/archives/
>SELECT pg_switch_wal();
![image](https://github.com/user-attachments/assets/e0f07d9e-2bca-478e-9986-5b47e459964c)



## (1) Настройте бэкапы PostgreSQL с использованием WAL-G, pg_probackup или любого другого аналогичного ПО для базы данных "Лояльность оптовиков".

Делаем инкрементальный бэкап средствами pg_basebackup

pgnode1: проверка директории summaries - пустая ожидаемо
![image](https://github.com/user-attachments/assets/8ec5aea8-0219-4e54-b170-bbf5db216c00)

pgnode1: выставить summarize_wal = on
![image](https://github.com/user-attachments/assets/28ae7ad9-7bb2-46f6-912c-5b9db491a3f1)
![image](https://github.com/user-attachments/assets/1cfcea80-b243-47c8-9cc4-c6901465953b)

pgnode1: проверить что файлы стали появляться в директории summaries
![image](https://github.com/user-attachments/assets/85f5d733-02bc-499b-a0fc-3be8feac77e4)

pgnode1: создание бэкап директорию
>sudo mkdir -p /backup/full
>sudo chown -R postgres:postgres /backup
>sudo chmod -R 755 /backup

pgnode1: создание тестовой таблицы
>create table demo000001 (c1 text);
>insert into demo00001 values ('Проверка инкрементального резервного копирования');



pgnode1: выполнить полный бэкап
> postgres@pgnode1:   pg_basebackup -v -D /backup/full
![image](https://github.com/user-attachments/assets/85a95435-70d3-48c2-92c2-73e7ee32dad8
![image](https://github.com/user-attachments/assets/a4177221-2112-4408-9079-1fa60268acd8)



## (2) Восстановите данные на другом кластере, чтобы убедиться, что бэкапы работают.
Восстанавливать данные буду с Yandex Cloud в локальную виртуалку

(source)Где сделали бэкап в YC
![image](https://github.com/user-attachments/assets/8c617ba8-8f71-43e0-b8e1-7dff701ebd48)

(target)Целевой кластер для восстановления
>student:~$ sudo apt install postgresql-17
![image](https://github.com/user-attachments/assets/9f85d6cc-b22b-41b4-9e89-c3ea95aaac08)




## (3) Проверьте, что данные восстановлены корректно.


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

