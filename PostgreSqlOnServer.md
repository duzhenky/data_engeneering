# PostgreSQL на сервере и удалённый доступ к нему

### 1. Установка

1.1 Подключаемся к серверу через SSH
```console
ssh root@111.11.111.111
```

1.2 Обноваляем список пакетов и сами пакеты
```console
apt update && apt upgrade
```

1.3 Устанавливаем PostgreSQL 
```console
apt install postgresql -y
```

### 2. Создание пользователя

2.1 Подключаемся под пользователем postgres
```console
su postgres
```

2.2 Далее находясь под суперпользователем postgres, создаём нового пользователя
```console
createuser --interactive
```

2.3 Определяем имя пользователя и основные права
```console
Enter name of role to add: username
Shall the new role be a superuser? (y/n): n
Shall the new role be allowed to create databases? (y/n): n
Shall the new role be allowed to create more new roles? (y/n): n
```
    
2.4 Заходим в PostgreSQL
```console
psql
```

2.5 Установим пароль пользователя
```console
alter user user_name with encrypted password 'user_password';
```

### 3. Создание базы данных для нового пользователя

3.1 Создадим базу данных
```console
create database user_db;
```

3.2 Теперь для нового пользователя нужно назначить права для данной БД
```console
grant all privileges on database user_db to user_name;
```

3.3 Выходим из psql и postgres
```console
\q
exit
```

### 4. Дополнительные настройки для удалённого доступа к базе данных

4.1 Проверяем какой порт прослушивает сервер
```console
lsof -i -P -n | grep LISTEN
```
В нашем случае это
```console
postgres 25548 postgres    5u  IPv4 127871609      0t0  TCP 127.0.0.1:5432 (LISTEN)
```
Мы видим, что порт 5432, но он на localhost 127.0.0.1. Это нужно исправить. Для этого нужно подкорректировать файл конфигурации postgres
```console
vim /etc/postgresql/11/main/postgresql.conf
```
В данном файле находим в разделе CONNERCTIONS AND AUTHENTIFICATION закомментированную строку 
```console
#listen_addresses = 'localhost'
```
Раскомментируем эту строку и поставим \*, чтобы можно было подключаться с любых ip-адресов
```console
listen_addresses = '*'
```

4.2 Перезапустим PostgreSQL
```console
systemctl restart postgresql
```

4.3 Посмотрим статус, не упал ли наш сервер
```console
systemctl status postgresql
```

4.4 Проверяем ещё раз какой порт прослушивает сервер
```console
lsof -i -P -n | grep LISTEN
```
И теперь видим, что он доступен с люых ip-адресов
```console
postgres 27418 postgres    5u  IPv6 127888142      0t0  TCP *:5432 (LISTEN)
```

4.5 Этого ещё недостаточно для удалённого доступа к базе данных. В другом файле конфигурации нужно 
указать какой пользователь к каким базам данных и с каких ip-адресов имеет доступ. 
Для этого заходим в файл конфигурации
```console
vim /etc/postgresql/11/main/pg_hba.conf
```
В этом конфигурационном файле как раз прописываются условия подключения к базам данных. Ищем строку
```console
#IPv4 local connections
```
Ниже в соединениях добавляем следующую строку аналогично предыдущей
```console
host user_db user_name 0.0.0.0/0 md5
```
где: 
- user_db - это база данных
- user_name - пользователь
- 0.0.0.0/0 - любой ip-адрес
- md5 - тип шифрования 
   
4.6 После этого сохраняем файл

4.7 Опять перезапускаем сервер PostgreSQL
```console
systemctl restart postgresql
```

4.8 Проверяем активен ли наш сервер
```console
systemctl status postgresql
```

----

После этого можно считать настройку PostgreSQL на VDS-сервере завершённой.

Данные для подключения у нас следующие:

- Хост - 111.11.111.111 (ip-адрес сервера)
- Порт - 5432
- База данных - user_db
- Пользователь - user_name
- Пароль - user_password

С этими данными можно подключиться к серверу, например, через DBeaver
