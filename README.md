# Домашнее задание Сартисона Евгения №5 #

Цель:
настроить надёжное резервное копирование и восстановление базы данных


Описание/Пошаговая инструкция выполнения домашнего задания:
Однажды Алексей узнал тревожную новость: у прямого конкурента "TropicalFruits" произошёл масштабный сбой, и их база данных лояльных клиентов была полностью потеряна. Это привело к хаосу: клиенты не могли воспользоваться своими бонусами, заказы обрабатывались с ошибками, а репутация компании пострадала. Некоторые клиенты даже начали переходить к "BananaFlow", но Алексей понимал, что его компания не застрахована от подобных проблем.

Он решил, что нужно срочно настроить современные инструменты для бэкапов — WAL-G или pg_probackup. Однако на пути к успеху его ждали новые вызовы:

***Как настроить бэкапы так, чтобы они не влияли на производительность системы?***
***Как восстановить данные на другом кластере, если что-то пойдёт не так?***
***Как проверить, что бэкапы действительно работают?***

Задание:

Используем машины от ДЗ#4



Повторите шаги Алексея:

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

pgnode1: выполнить полный бэкап
> postgres@pgnode1:   pg_basebackup -v -D /backup/full
![image](https://github.com/user-attachments/assets/85a95435-70d3-48c2-92c2-73e7ee32dad8
![image](https://github.com/user-attachments/assets/a4177221-2112-4408-9079-1fa60268acd8)

pgnode1: создание тестовой таблицы
>create table demo000001 (c1 text);
>insert into demo00001 values ('Проверка инкрементального резервного копирования');

pgnode1: создать директории для инкрементальных бэкапов
sudo mkdir -p /backup/{incr_monday,incr_tuesday}
sudo chown -R postgres:postgres /backup
sudo chmod -R 755 /backup


pgnode1: сделать инкременатльный бэкап
>pg_basebackup --incremental=/backup/full/backup_manifest -D /backup/incr_monday/
![image](https://github.com/user-attachments/assets/e8afb94a-eb5e-4c7c-a638-49102aaa6464)


pgnode1: создание тестовой таблицы
>create table demo02 (c1 text);
>insert into demo02 values ('Проверка инкрементального резервного копирования');


pgnode1: сделать инкременатльный бэкап
>pg_basebackup --incremental=/backup/full/backup_manifest -D /backup/incr_tuesday/
![image](https://github.com/user-attachments/assets/e8afb94a-eb5e-4c7c-a638-49102aaa6464)



## (2) Восстановите данные на другом кластере, чтобы убедиться, что бэкапы работают.
Восстанавливать данные буду с Yandex Cloud в локальную виртуалку

(source)Где сделали бэкап в YC
![image](https://github.com/user-attachments/assets/8c617ba8-8f71-43e0-b8e1-7dff701ebd48)

(target)Целевой кластер для восстановления
>student:~$ sudo apt install postgresql-17
![image](https://github.com/user-attachments/assets/9f85d6cc-b22b-41b4-9e89-c3ea95aaac08)




## (3) Проверьте, что данные восстановлены корректно.


## (4) Дополнительно: Снимите бэкап под нагрузкой с реплики.
