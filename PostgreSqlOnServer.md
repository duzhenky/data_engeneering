# PostgreSQL на VDS-сервере и удалённый доступ

### Установка

1) Подключаемся к серверу через SSH
    > ssh root@111.11.111.111
2) Обноваляем список пакетов и сами пакеты
    > apt update && apt upgrade
3) Устанавливаем PostgreSQL 
    > apt install postgresql -y

### Создание пользователя

1) Подключаемся под пользователем postgres 
    > su postgres
3) Далее находясь под суперпользователем postgres, создаём нового пользователя
    > createuser --interactive
4) Определяем имя пользователя и основные права
    > Enter name of role to add: anlt
    
    > Shall the new role be a superuser? (y/n) n
    
    > Shall the new role be allowed to create databases? (y/n) n
    
    > Shall the new role be allowed to create more new roles? (y/n) n
5) Заходим в PostgreSQL
    > psql
6) Установим пароль пользователя
    > alter user anlt with encrypted password 'anlt1732';

### Создание базы данных для нового пользователя

1) Создадим базу данных
    > create database tg;
2) Теперь для нового пользователя нужно назначить права для данной БД
    > grant all privileges on database tg to anlt;
3) Выходим из psql и postgres
    > \q

    > exit

### Дополнительные настройки для удалённого доступа к базе данных

1) Проверяем какой порт прослушивает сервер
    > lsof -i -P -n | grep LISTEN

    В нашем случае это
    > postgres 25548 postgres    5u  IPv4 127871609      0t0  TCP 127.0.0.1:5432 (LISTEN)

    Мы видим, что порт 5432, но он на localhost 127.0.0.1. Это нужно исправить. Для этого
    нужно подкорректировать файл конфигурации postgres
    > vim /etc/postgresql/11/main/postgresql.conf
    
    В данном файле находим в разделе CONNERCTIONS AND AUTHENTIFICATION закомментированную строку 

    > #listen_addresses = 'localhost'
    
    Раскомментируем эту строку и поставим *, чтобы можно было подключаться с любых ip-адресов

    > listen_addresses = '*'

2) Перезапустим сервер

   > systemctl restart postgresql

3) Посмотрим статус, не упал ли наш сервер
   > systemctl status postgresql

4) Проверяем ещё раз какой порт прослушивает сервер

   > lsof -i -P -n | grep LISTEN

   И теперь видим, что он доступен с люых ip-адресов

   > postgres 27418 postgres    5u  IPv6 127888142      0t0  TCP *:5432 (LISTEN)

5) Этого ещё недостаточно для удалённого доступа к базе данных. В другом файле конфигурации нужно 
указать какой пользователь к каким базам данных и с каких ip-адресов имеет доступ. 
Для этого заходим в файл конфигурации

    > vim /etc/postgresql/11/main/pg_hba.conf

   В этом конфигурационном файле как раз прописываются условия подключения к базам данных.
   Ищем строку
   > #IPv4 local connections

   Ниже в соединениях добавляем следующую строку аналогично предыдущей
   
   > host tg anlt 0.0.0.0/0 md5

   где: 
   - tg - это база данных
   - anlt - пользователь
   - 0.0.0.0/0 - любой ip-адрес
   - md5 - тип шифрования 
   
   После этого сохраняем файл

7) Опять перезапускаем сервер postgres
   > systemctl restart postgresql
8) Проверяем активен ли наш сервер
   > systemctl status postgresql


После этого можно считать настройку PostgreSQL на VDS-сервере завершённой.

----

Данные для подключения у нас следующие:

- Хост - 111.11.111.111 (ip-адрес сервера)
- Порт - 5432
- База данных - tg
- Пользователь - anlt
- Пароль - anlt1732

С этими данными можно подключиться к серверу, например, через DBeaver
