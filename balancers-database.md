# Настройка балансировщиков нагрузки для серверов баз данных

## Описание

Настраивается балансировка нагрузки на серверы баз данных.

## Требования

* [Настройка серверов баз данных](database.md)

## Выполнение

1. На каждом из хостов "Balancer1" и "Balancer2" установим HAProxy:

    ```sh
    sudo apt update
    sudo apt install -y haproxy
    ```

2. На каждом из тех же хостов обеспечим следующие содержимое файла конфигурации HAProxy (`/etc/haproxy/haproxy.cfg`) (вместо `PASSWORD` зададим пароль для пользователя `admin` веб-интерфейса балансировщика):

    ```config
        global
            log /dev/log local0
            maxconn 2048
            daemon

        defaults
            log     global
            mode    tcp
            option  tcplog
            timeout connect 5s
            timeout client  30s
            timeout server  30s

        frontend postgres_clients
            bind *:5432
            default_backend postgres_servers

        backend postgres_servers
            mode tcp
            option httpchk GET /
            http-check send hdr Host localhost
            http-check expect status 200
            http-check expect string "\"role\": \"master\""

            server db1 192.168.3.7:5432 check port 8008 inter 2s fall 2 rise 3
            server db2 192.168.3.8:5432 check port 8008 inter 2s fall 2 rise 3
        
    listen stats
        bind *:7000
        mode http
        stats enable
        stats uri /stats
        stats refresh 5s
        stats auth admin:PASSWORD
    ```

3. На каждом из тех же хостов откроем порты для балансировки и веб-интерфейса:

    ```sh
    sudo ufw allow postgresql
    sudo ufw allow 5433/tcp
    sudo ufw allow 5434/tcp
    sudo ufw allow 7000/tcp
    ```

4. На каждом из тех же хостов перезапустим службу HAProxy и проверим её состояние:

    ```sh
    sudo systemctl restart haproxy
    sudo systemctl status haproxy
    ```

5. На каждом из тех же хостов проверим подключение к лидеру кластера:

    ```sh
    psql -h 127.0.0.1 -p 5433 -U postgres -c "SELECT pg_is_in_recovery();"
    ```

    При попадании на лидера должно вернуться `f`.

6. На каждом из тех же хостов проверим подключение к реплике кластера:

    ```sh
    psql -h 127.0.0.1 -p 5434 -U postgres -c "SELECT pg_is_in_recovery();"
    ```

    При попадании на лидера должно вернуться `t`.

7. На каждом из роутеров "Router1" и "Router2" выполним перенаправление портов для доступа к веб-интерфейсу HAProxy из внешней сети для проверки:

    ```mikrotik
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=7000 action=dst-nat to-addresses=192.168.3.14 to-ports=7000
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=7001 action=dst-nat to-addresses=192.168.3.3 to-ports=7000
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=7002 action=dst-nat to-addresses=192.168.3.4 to-ports=7000
    ```

    Проверим доступность веб-интерфейса и авторизации (каталог `/stats`).
