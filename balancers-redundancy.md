# Настройка резервирования балансировщиков нагрузки

## Описание

Настраивается резервирование балансировщиков нагрузки с использованием протокола VRRP.

## Требования

* [Создание структуры сети](network-structure.md)

## Выполнение

1. На хостах "Balancer1" и "Balancer2" установим keepalived:

    ```sh
    sudo apt update
    sudo apt install -y keepalived
    ```

2. На хосте "Balancer1" создадим и откроем файл конфигурации keepalived:

    ```sh
    sudo vim /etc/keepalived/keepalived.conf
    ```

    В редакторе обеспечим следующее содержимое файла (вместо `SECRET` зададим пароль для синхронизации):

    ```config
    vrrp_instance VI_1 {
        state MASTER
        interface enp0s3
        virtual_router_id 1
        priority 200
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass SECRET
        }

        virtual_ipaddress {
            192.168.3.14/28
        }
    }
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

3. На хосте "Balancer1" создадим и откроем файл конфигурации keepalived:

    ```sh
    sudo vim /etc/keepalived/keepalived.conf
    ```

    В редакторе обеспечим следующее содержимое файла (вместо `SECRET` зададим пароль для синхронизации):

    ```config
    vrrp_instance VI_1 {
        state BACKUP
        interface enp0s3
        virtual_router_id 1
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass SECRET
        }

        virtual_ipaddress {
            192.168.3.14/28
        }
    }
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

    Перезапустим службу keepalived:

    ```sh
    sudo systemctl restart keepalived
    sudo systemctl enable keepalived
    ```

    Добавим правило межсетевого экрана, разрешающее получение пакетов VRRP от второго банансировщика:

    ```sh
    sudo ufw allow proto vrrp from 192.168.3.4
    ```

4. На хосте "Balancer2" создадим и откроем файл конфигурации keepalived:

    ```sh
    sudo vim /etc/keepalived/keepalived.conf
    ```

    В редакторе обеспечим следующее содержимое файла (вместо `SECRET` зададим пароль для синхронизации):

    ```config
    vrrp_instance VI_1 {
        state BACKUP
        interface enp0s3
        virtual_router_id 1
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass SECRET
        }

        virtual_ipaddress {
            192.168.3.14/28
        }
    }
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

    Перезапустим службу keepalived и включим её автозапуск:

    ```sh
    sudo systemctl restart keepalived
    sudo systemctl enable keepalived
    ```

    Добавим правило межсетевого экрана, разрешающее получение пакетов VRRP от первого банансировщика:

    ```sh
    sudo ufw allow proto vrrp from 192.168.3.3
    ```

5. Проверим назначение виртуального адреса 192.168.3.14 на хосте "Balancer1":

    ```sh
    ip address show
    ```

    Сделаем хост "Balancer1" недоступным (выключим его, поставим виртуальную машину на паузу или отключим сетевой интерфейс) и проверим назначение виртуального адреса на хосте "Balancer2". Затем вернём доступность хоста "Balancer1" и проверим назначение виртаульного адреса на нём и отсутствие этого адреса на "Balancer2".
