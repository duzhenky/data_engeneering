# Airflow на сервере и удалённый доступ к нему

### 1. Подготовка к установке
1.1 Создаём папку для Airflow
```console
mkdir airflow
```

1.2 Выдаём права на всех пользователей для этой папки (на проде так не делать - только для конкретных) 
```console
chmod 777 airflow
```

1.3 В папке airflow нужно создать две папки: с дагами и плагинами
```console
mkdir dags
mkdir plugins
```

1.4 Затем выдать на них права
```console
chmod 777 -R dags
chmod 777 -R plugins
```
В будущем, когда мы будем создавать даги, мы будем складывать их в папку dags. Можно ещё также создать папку scripts и выдать права

1.5 Создаём переменную AIRFLOW_HOME (после выхода с сервера данная переменная удалится, далее по инструкции будет создан демон, чтобы этого не происходило)
```console
export AIRFLOW_HOME=/путь/к/папке/airflow
```

1.6 Проверяем что переменная создалась
```console
printenv
```

1.7 Устанавливаем pip3, если он не установлен
```console
apt install python3-pip
```

### 2. Установка Airflow и зависимых библиотек
2.1 Устанавливаем Airflow
```console
pip3 install apache-airflow==2.6.1
```

2.2 Также устанавливаем Flask именно этой версии
```console
pip3 install Flask-Session==0.5.0
```

2.3 Устанавливаем psycopg2
```console
apt install python-psycopg2
apt install libpq-dev
pip3 install psycopg2-binary
```

2.4 Заходим в директорию airflow
```console
cd /путь/к/папке/airflow
```

2.5 открываем все конфиги, чтобы в директории с Airflow создался файл с конфигами airflow.cfg
```console
airflow config list
```

### 3. Пользователь PostgreSQL для Airflow
3.1 В PostgreSQL ([поднимаем PostgreSQL на тестовом сервере](./PostgreSqlOnServer.md)) нужно создать БД и пользователя для Airflow, а также выдать ему гранты
```console
su postgres
psql
create database airflow_metadata;
create user airflow with password 'admin';
grant all privileges on database airflow_metadata to airflow;
```

3.2 Выходим из psql
```console 
\q
exit
```

### Редактирование airflow.cfg
4.1 Открывваем airflow.cfg
```console 
nano /путь/к/папке/airflow/airflow.cfg
```

4.2 Меняем строчки с переменными
```console 
executor = LocalExecutor 
sql_alchemy_conn = postgresql+psycopg2://airflow:admin@localhost:5432/airflow_metadata
```
*LocalExecutor может запускать более одной таски одновременно, а тот что по умолчанию - только одну*

### 4. База данных для Airflow

4.1 Инициализация БД
```console 
airflow db init
```

4.2 Создаём в PostgreSQL пользователя Airflow
```console
airflow users create --username AirflowAdmin --firstname YourFirstName --lastname YourLastName --role Admin --email your@email.com
```
Затем будет предложено задать пароль и повторить его

----
### 5. Запуск
**На данном этапе Airflow уже может работать**

5.1 Для этого в одном окне терминала запустим scheduler
```console
airflow scheduler
```

5.2 В другом окне терминала webserver
```console
airflow webserver
```

5.3 После этого можно зайти в веб-интерфейс по ip-адресу сервера, указав порт  8080
http://111.11.111.111:8080

5.4 Вводим логин и пароль, после чего Airflow откроется

----


### 6. Создание демонов
Чтобы каждый раз не открывать два окна: scheduler и webserver, можно создать системных демонов

6.1 Переходим в etc и создаём папку для конфигов
```console
cd /etc
mkdir sysconfig
```

6.2 Выдаём права
```console
chmod 777 -R sysconfig
```

6.3 В данной папке создаём файл airflow
```console
nano airflow
```

6.4 Вставляем в редакторе следующий код, указываем корректные пути к конфигу и папке с ним из п. 2.5 и 1.1 (последние две строчки)
```console
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

# This file is the environment file for Airflow. Put this file in /etc/sysconfig/airflow per default
# configuration of the systemd unit files.
#
AIRFLOW_CONFIG=/путь/к/папке/airflow/airflow.cfg
AIRFLOW_HOME=/путь/к/папке/airflow
```

6.5 Переходим в следующую директорию
```console
cd /usr/lib/systemd/system
```

6.6 Находясь в ней создаём файл демона для scheduler
```console
nano airflow-scheduler.service
```

6.7 Вставляем в редакторе код ниже и сохраняем
```console
[Unit]
Description=Airflow scheduler daemon

After=network.target postgresql.service 
Wants=postgresql.service

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/airflow scheduler
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```


6.8 Оставаясь в этой же лиректории создаём файл демона для webserver
```console
nano airflow-webserver.service
```

6.9 Вставляем в редакторе код ниже и сохраняем
```console
[Unit]
Description=Airflow webserver daemon
After=network.target postgresql.service
Wants=postgresql.service

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/airflow webserver --pid /airflow/airflow-webserver.pid
Restart=on-failure
RestartSec=5s
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

----

### 7. Запуск

7.1 Запускаем поочерёдно следующие команды для запуска сервиса
```console
systemctl daemon-reload

systemctl enable airflow-scheduler
systemctl restart airflow-scheduler
systemctl status airflow-scheduler

systemctl enable airflow-webserver
systemctl restart airflow-webserver
systemctl status airflow-webserver
```

7.2 После этого можно зайти в веб-интерфейс по ip-адресу сервера, указав порт  8080
http://111.11.111.111:8080

7.3 Вводим логин и пароль, после чего Airflow откроется

----
### 8. Для отключения Airflow
```console
systemctl disable airflow-scheduler
systemctl disable airflow-webserver
```






















