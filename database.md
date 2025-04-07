# Настройка серверов баз данных

## Описание

Настраиваются сервера баз данных: два - основные (с репликацией), третий - свидетель для кворума. Используются etcd и Patroni.

## Требования

* [Настройка MediaWiki](mediawiki.md)

## Выполнение

### Настройка

1. На каждом из хостов "DB1", "DB2" и "Backup" установим Patroni, etcd, и PostgreSQL:

    ```sh
    sudo apt update
    sudo apt install -y postgresql patroni etcd-client etcd-server
    ```

2. На каждом из тех же хостов откроем конфигурационный файл etcd `/etc/default/etcd` со следующим содержанием, указав соответственно следующие параметры:

    * `IP`: `192.168.3.7`, `192.168.3.8`, `192.168.3.9`;
    * `NAME`: `db1`, `db2`, `backup`;
    * `C_STATE`: `new`, `existing`, `existing`;

    ```sh
    ETCD_NAME="NAME"
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://IP:2380"
    ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
    ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
    ETCD_ADVERTISE_CLIENT_URLS="http://IP2379"
    ETCD_INITIAL_CLUSTER="db1=http://192.168.3.7:2380,db2=http://192.168.3.8:2380,backup=http://192.168.3.9:2380"
    ETCD_INITIAL_CLUSTER_STATE="C_STATE"
    ETCD_INITIAL_CLUSTER_TOKEN="patroni-cluster"
    ```

    Перезапустим etcd:

    ```sh
    sudo systemctl restart etcd
    ```

3. На каждом из тех же хостов в файле основной конфигурации PostgreSQL (`/etc/postgresql/16/main/postgresql.base.conf` для "DB1" и "DB2", `/etc/postgresql/16/main/postgresql.base.conf` для "Bakckup") разрешим доступ с адресов всех этих хостов:

    ```config
    listen_addresses = '192.168.3.7,192.168.3.8,192.168.3.9'
    ```

4. На каждом из тех же хостов в файл `/etc/postgresql/16/main/pg_hba.conf` добавим следующие строки:

    ```config
    host    all             all             192.168.3.0/28          scram-sha-256
    host    replication     replicator      192.168.3.0/28          scram-sha-256
    ```

    Перезапустим службу PostgreSQL:

    ```sh
    sudo systemctl restart postgresql
    ```

5. На каждом из тех же хостов выполним вход в консоль сервера PostgreSQL:

    ```sh
    sudo -u postgres psql
    ```

    Установим пароль пользователя `postgres` (вместо `PGPASS` укажем задаваемый пароль), создадим пользователя для репликации и зададим ему пароль (вместо `REPLPASS` укажем задаваемый пароль):

    ```sql
    ALTER USER postgres WITH PASSWORD 'PGPASS';
    CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'REPLPASS';
    EXIT;
    ```

    На хосте "DB1" в консоли сервера PostgreSQL создадим слоты репликации:

    ```sql
    SELECT pg_create_physical_replication_slot('db2');
    SELECT pg_create_physical_replication_slot('backup');
    ```

6. На каждом из тех же хостов создадим конфигурационный файл patroni `/etc/patroni.yml` со следующим содержанием, указав соответственно следующие параметры:

    * `IP`: `192.168.3.7`, `192.168.3.8`, `192.168.3.9`
    * `NAME`: `db1`, `db2`, `backup`;
    * `ADMINPASS`: пароль администратора (на всех хостах один и тот же);
    * `REPLPASS`: пароль пользователя для репликации, установленный ранее (на всех хостах один и тот же);
    * `PGPASS`: пароль основного пользователя для PostgreSQL, установленный ранее (на всех хостах один и тот же).
    * раздел `bootstrap` укажем только для "DB1"; для "DB2" и "Backup" его не будет.

    ```yaml
    scope: wiki-cluster
    namespace: /wiki/
    name: NAME

    restapi:
      listen: IP:8008
      connect_address: IP:8008

    etcd3:
      hosts:
      - 192.168.3.7:2379
      - 192.168.3.8:2379
      - 192.168.3.9:2379

    bootstrap:
      dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
        use_pg_rewind: true
        use_slots: true
      initdb:
      - encoding: UTF8
      - data-checksums
      users:
        admin:
        password: ADMINPASS
        options:
            - createrole
            - createdb

    postgresql:
      listen: IP:5432
      connect_address: IP:5432
      data_dir: /var/lib/postgresql/16/main
      config_dir: /etc/postgresql/16/main
      bin_dir: /usr/lib/postgresql/16/bin
      authentication:
        replication:
        username: replicator
        password: REPLPASS
        superuser:
        username: postgres
        password: PGPASS
      parameters:
        max_connections: 100
        shared_buffers: 256MB
        wal_level: replica
        hot_standby: "on"

    tags:
      nofailover: false
      noloadbalance: false
      clonefrom: false
      nosync: false
    ```

7. На каждом из тех же хостов создадим юнит службы systemd `/etc/systemd/system/patroni.service`:

    ```ini
    [Unit]
    Description=High availability PostgreSQL via Patroni
    After=network.target

    [Service]
    Type=simple
    User=postgres
    ExecStart=/usr/bin/patroni /etc/patroni.yml
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```

    На каждом из хостов активируем службу:

    ```sh
    sudo systemctl daemon-reexec
    sudo systemctl daemon-reload
    sudo systemctl enable patroni
    sudo systemctl start patroni
    ```

8. На каждом из тех же хостов откроем необходимые порты:

    ```sh
    sudo ufw allow postgresql
    sudo ufw allow 8008/tcp
    sudo ufw allow 2379/tcp
    sudo ufw allow 2380/tcp
    ```

### Средства диагностики и устранения неполадок

#### Сетевая доступность

* Проверка открытых портов:

    ```sh
    sudo ufw status
    ```

#### Состояние кластера Patroni

* Проверка с помощью `patronictl`:

    ```sh
    patronictl -c /etc/patroni.yml list
    ```

* Проверка состояния конкретного узла:

    ```sh
    curl http://localhost:8008
    ```

* Запуск Patroni с отладкой:

    ```sh
    patroni /etc/patroni.yml --debug
    ```

* Проверка конфигурации Patroni:

    ```sh
    patronictl -c /etc/patroni.yml show-config
    ```

### Состояние кластера postgreSQL

* Проверка статуса репликации на мастере:

    ```sql
    SELECT * FROM pg_stat_replication;
    ```

* Проверка статуса репликации на репликах:

    ```sql
    SELECT pg_is_in_recovery();
    ```

* Проверка статусов слотов для репликации:

    ```sql
    SELECT * FROM pg_replication_slots;
    ```

### Состояние кластера etcd

* Проверка состояния:

    ```sh
    etcdctl --endpoints=http://127.0.0.1:2379 endpoint health
    ```

* Проверка членства в кластере:

    ```sh
    etcdctl member list
    ```

* Просмотр ключей Patroni:

    ```sh
    etcdctl get / --prefix --keys-only
    ```

* Просмотр содержимого ключей:

    ```sh
    etcdctl get / --prefix
    ```

### Частые проблемы

| Проблема | Причина | Как решать |
| - | - | -|
| `system ID mismatch` | `patroni` не может синхронизироваться с мастером - данные на реплике невалидны | Очистить `data_dir`, инициализировать заново |
| `pg_basebackup: no pg_hba.conf entry` | Неправильная настройка `pg_hba.conf` на мастере | Добавить строк вида `host replication replicator подсеть scram-sha-256` |
| `replication slot does not exist` | Узел пытается использовать несуществующий слот | Убедиться, что слот создан на мастере, либо отключить `replication_slots` в Patroni |
| `timeline is not a child` | Реплика ушла в отставание и пытается получить несоответствующий WAL | Удалить `data_dir`, заново сделать bootstrap |
| `FATAL: could not start WAL streaming` | Реплика не может подключиться | Проверить логины, порты, `pg_hba.conf` |
| `Can not find suitable configuration of distributed configuration store` | etcd не работает или не настроен | Проверить etcd, `patroni.yml` |

### Логи

* Основной журнал системы: `/var/log/syslog`

* PostgreSQL: `/var/log/postgresql/postgresql-<версия>-main.log`

* Patroni: `sudo journalctl -u patroni` или лог-файл, если задан

* ETCD: `sudo journalctl -u etcd`

### Дополнительно

* Перед очисткой `data_dir` нужно останавливать службу PostgreSQL

* Если реплика отстаёт и не может синхронизироваться, чаще всего поможет переинициализация

* Нужно убедиться, что все узлы используют одинаковую конфигурацию Patroni (особенно имя кластера и настройки etcd)
