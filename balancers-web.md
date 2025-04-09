# Настройка веб-серверов и балансировщиков нагрузки

## Описание

Настраиваются веб-серверы и балансировка нагрузки на них.

## Требования

* [Настройка резервирования балансировщиков нагрузки](balancers-redundancy.md)

## Выполнение

1. На хостах "Balancer1" и "Balancer2" установим nginx, запустим службу и включим её автозапуск:

    ```sh
    sudo apt update
    sudo apt install -y nginx
    sudo systemctl start nginx
    sudo systemctl enable nginx
    ```

2. На хостах "Balancer1" и "Balancer2" создадим и откроем файл дополнительной конфигурации nginx:

    ```sh
    sudo vim /etc/nginx/nginx.conf
    ```

    В редакторе обеспечим следующие содержимое файла:

    ```config
    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;
    include /etc/nginx/modules-enabled/*.conf;

    events {
        worker_connections 1024;
    }

    http {
        upstream backend {
            ip_hash;
            server 192.168.3.5:80 max_fails=1 fail_timeout=3s;
            server 192.168.3.6:80 max_fails=1 fail_timeout=3s;
        }

        server {
            listen 80;
            server_name _;

            location / {
                proxy_pass http://backend;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
    }
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

    Проверим конфигурацию nginx:

    ```sh
    sudo nginx -t
    ```

    Перезапустим nginx:

    ```sh
    sudo systemctl restart nginx
    ```

    Добавим правило межсетевого экрана для nginx:

    ```sh
    sudo ufw allow http
    ```

3. На хостах "Web1" и "Web2" установим nginx, запустим службу и включим её автозапуск, добавим правило межсетевого экрана:

    ```sh
    sudo apt update
    sudo apt install -y nginx
    sudo systemctl start nginx
    sudo systemctl enable nginx
    sudo ufw allow http
    ```

4. На роутерах "Router1" и "Router2" откроем их веб-интерфейсы на порту 80, в настройках "Webfig" - "IP" - "Services" переназначим порт для "www" с 80 на 81.

5. На роутерах "Router1" и "Router2" настроим перенаправление портов для подключения к веб-серверам через балансировщики по виртуальному адресу из внешней сети:

    ```mikrotik
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=80 action=dst-nat to-addresses=192.168.3.14 to-ports=80
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=8081 action=dst-nat to-addresses=192.168.3.3 to-ports=80
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=8082 action=dst-nat to-addresses=192.168.3.4 to-ports=80
    ```

6. Проверим подключение из внешней сети, в том числе при неработоспособности одного балансировщика и/или одного веб-сервера.
