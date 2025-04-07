# Создание структуры сети

## Описание

Создаётся структура отказоустойчивой сети с резервированием подключения к внешней сети (интернету), веб-сервера, сервера баз данных, сетевых устройств.

## Выполнение

1. В VirtualBox создадим следующие виртуальные машины для серверов и установим на них Ubuntu Server 24.04 LTS со следующими параметрами:
    * Индивидуальные параметры:
        * "Balancer1" - 1-й балансировщий нагрузки:
            * имя хоста: `balancer1`;
            * устройства хранения:
                * 10 Гб -1 шт. (раздел `/`);
        * "Balancer2" - 1-й балансировщий нагрузки:
            * имя хоста: `balancer2`;
            * устройства хранения:
                * 10 Гб -1 шт. (раздел `/`);
        * "Web1" - 1-й веб-сервер:
            * имя хоста: `web1`;
            * устройства хранения:
                * 2 Гб - 1 шт. (раздел `/boot`);
                * 20 Гб - 2 шт. (RAID 1, LVM, раздел `/`);
        * "Web2" - 2-й веб-сервер:
            * имя хоста: `web2`;
            * устройства хранения:
                * 2 Гб - 1 шт. (раздел `/boot`);
                * 20 Гб - 2 шт. (RAID 1, LVM, раздел `/`);
        * "DB1" - 1-й сервер баз данных:
            * имя хоста: `db1`;
            * устройства хранения:
                * 2 Гб - 1 шт. (раздел `/boot`);
                * 30 Гб - 3 шт. (RAID 5, LVM, раздел `/`);
        * "DB2" - 1-й сервер баз данных:
            * имя хоста: `db2`;
            * устройства хранения:
                * 2 Гб - 1 шт. (раздел `/boot`);
                * 30 Гб - 3 шт. (RAID 5, LVM, раздел `/`);
        * "Backup" - сервер резервного копирования:
            * имя хоста: `backup`;
            * устройства хранения:
                * 2 Гб - 1 шт. (раздел `/boot`);
                * 40 Гб - 4 шт. (RAID 6, LVM, раздел `/`);
        * "Monitoring" - сервер мониторинга:
            * имя хоста: `monitoring`;
            * устройства хранения:
                * 2 Гб - 1 шт. (раздел `/boot`);
                * 20 Гб - 2 шт. (RAID 1, LVM, раздел `/`).
    * Общие параметры:
        * раскладка клавиатуры и её вариант - "Russian";
        * сетевые настройки - автонастройка по DHCP (на время установки);
        * пользователь - "User", имя учётной записи - `user`;
        * установка сервера OpenSSH.

2. Для каждого созданного хоста авторизуемся от имени `user` и выполним обновление пакетов и установку ZSH:

    ```sh
    sudo apt update
    sudo apt upgrade
    sudo apt install -y zsh zsh-syntax-highlighting
    sudo apt dist-upgrade
    sudo apt autoremove
    ```

    Запустим ZSH:

    ```sh
    zsh
    ```

    Согласимся на создание файла настроек со значениями по умолчанию (вариант "2").

    Выполним установку "Oh My Zsh":

    ```sh
    sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
    ```

    Согласимся на установку ZSH как командного интерпретатора для пользователя по умолчанию.

    В конец файла `~/.zshrc` внесём следующие строки для настройки подсветки синтаксиса и отображения имени хоста:

    ```sh
    PROMPT="$fg[cyan]%}$USER@%{$fg[blue]%}%m ${PROMPT}"
    source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
    ```

    Выйдем завершим сеанс пользователя и авторизумеся вновь.

3. В GNS3 создадим проект "Knowledge".

4. Добавим в проект два облака с именами "ISP1" и "ISP2", соединённые с виртуальным сетевым интерфейсом базовой системы, имеющим доступ в интернет (далее - `virbr0`).

5. Создадим в GNS3 шаблон роутера Mikrotik RB450G, используя данные с сервера, по последней доступной версии прошивки, и добавим два таких роутера в проект - с именами "Router1" и "Router2".

6. Добавим в проект два коммутатора ("Open vSwitch") - с именами "Switch1" и "Switch2".

7. Добавим в проект все 8 созданных виртуальных машин в качестве хостов, с соответствующими именами. Для каждого хоста разрешим GNS3 управлять его сетевыми адаптерами.

8. Выполним следующие подключения:
    * роутер "Router1", интерфейс `ether1` - облако "ISP1", интерфейс `virbr0`;
    * роутер "Router2", интерфейс `ether1` - облако "ISP2", интерфейс `virbr0`;
    * роутер "Router1", интерфейс `ether2` - роутер "Router2", интерфейс `ether2`;
    * роутер "Router1", интерфейс `ether3` - коммутатор "Switch1", интерфейс `eth2`;
    * роутер "Router2", интерфейс `ether3` - коммутатор "Switch2", интерфейс `eth2`;
    * коммутатор "Switch1", интерфейс `eth0` - коммутатор "Switch2", интерфейс `eth0`;
    * коммутатор "Switch1", интерфейс `eth1` - коммутатор "Switch2", интерфейс `eth1`;
    * коммутатор "Switch1", интерфейс `eth3` - сервер "Balancer1", интерфейс `Ethernet0`;
    * коммутатор "Switch1", интерфейс `eth4` - сервер "Web1", интерфейс `Ethernet0`;
    * коммутатор "Switch1", интерфейс `eth5` - сервер "DB1", интерфейс `Ethernet0`;
    * коммутатор "Switch1", интерфейс `eth6` - сервер "Backup", интерфейс `Ethernet0`;
    * коммутатор "Switch2", интерфейс `eth3` - сервер "Balancer2", интерфейс `Ethernet0`;
    * коммутатор "Switch2", интерфейс `eth4` - сервер "Web2", интерфейс `Ethernet0`;
    * коммутатор "Switch2", интерфейс `eth5` - сервер "DB2", интерфейс `Ethernet0`;
    * коммутатор "Switch2", интерфейс `eth6` - сервер "Monitoring", интерфейс `Ethernet0`.

9. Включим режим агрегации каналов для коммутаторов:

    * в консоли коммутатора "Switch1" выполним:

        ```config
        ovs-vsctl del-port br0 eth0
        ovs-vsctl del-port br0 eth1
        ovs-vsctl add-bond br0 bond0 eth0 eth1
        ovs-vsctl set port bond0 lacp=active
        ```

    * в консоли коммутатора "Switch2" выполним:

        ```config
        ovs-vsctl del-port br0 eth0
        ovs-vsctl del-port br0 eth1
        ovs-vsctl add-bond br0 bond0 eth0 eth1
        ovs-vsctl set port bond0 lacp=active
        ```

10. Настроим сетевые интерфейсы роутеров:

    * в консоли роутера "Router1" установим пароль администратора (по умолчанию он пустой) и выполнми (`192.168.1.1` и `192.168.1.2` - адреса шлюзов базовой системы):

        ```mikrotik
        interface enable ether1
        interface enable ether2
        interface enable ether3
        ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 check-gateway=ping distance=1
        ip route add dst-address=0.0.0.0/0 gateway=192.168.1.2 check-gateway=ping distance=2
        ip address add address=192.168.2.1/30 interface=ether2
        ip address add address=192.168.3.1/28 interface=ether3
        ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
        ```

    * в консоли роутера "Router2" установим пароль администратора (по умолчанию он пустой) и выполнми (`192.168.1.1` и `192.168.1.2` - адреса шлюзов базовой системы):

        ```mikrotik
        interface enable ether1
        interface enable ether2
        interface enable ether3
        ip route add dst-address=0.0.0.0/0 gateway=192.168.1.2 check-gateway=ping distance=1
        ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 check-gateway=ping distance=2
        ip address add address=192.168.2.2/30 interface=ether2
        ip address add address=192.168.3.2/28 interface=ether3
        ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
        ```

11. На серверах произведём следующие настройки сетевых подключений (серверы DNS во внешней сети имеют адреса 192.168.1.1 и 192.168.1.2):

    * "Balancer1":
        * IP-адрес: 192.168.3.3/28;
        * маршрут по умолчанию: 192.168.3.1;
        * серверы DNS: 192.168.1.1, 192.168.1.2;
    * "Balancer2":
        * IP-адрес: 192.168.3.4/28;
        * маршрут по умолчанию: 192.168.3.2;
        * сервер DNS: 192.168.1.2, 192.168.1.1;
    * "Web1":
        * IP-адрес: 192.168.3.5/28;
        * маршрут по умолчанию: 192.168.3.1;
        * сервер DNS: 192.168.1.1, 192.168.1.2;
    * "Web2":
        * IP-адрес: 192.168.3.6/28;
        * маршрут по умолчанию: 192.168.3.2;
        * сервер DNS: 192.168.1.2, 192.168.1.1;
    * "DB1":
        * IP-адрес: 192.168.3.7/28;
        * маршрут по умолчанию: 192.168.3.1;
        * сервер DNS: 192.168.1.1, 192.168.1.2;
    * "DB2":
        * IP-адрес: 192.168.3.8/28;
        * маршрут по умолчанию: 192.168.3.2;
        * сервер DNS: 192.168.1.2, 192.168.1.1;
    * "Backup":
        * IP-адрес: 192.168.3.9/28;
        * маршрут по умолчанию: 192.168.3.1;
        * сервер DNS: 192.168.1.1, 192.168.1.2;
    * "Monitoring":
        * IP-адрес: 192.168.3.10/28;
        * маршрут по умолчанию: 192.168.3.2;
        * сервер DNS: 192.168.1.2, 192.168.1.1;

12. На роутере "Router2" выполним перенаправление порта из внешней сети для подключения к серверу "Monitoring" по SSH для удобства настройки (нестандартное значение внешнего порта для безопасности):

    ```mikrotik
    ip firewall nat add chain=dstnat in-interface=ether3 protocol=tcp dst-port=4224 action=dst-nat to-addresses=192.168.3.10 to-ports=22
    ```

    Выясним адрес роутера на внешнем интерфейсе `ether1`:

    ```mikrotik
    ip address print
    ```

    Проверим подключение по SSH на указанный порт через роутер из внешней сети.

13. Создадим ключ для доступа по SSH из внешней сети и скопируем его на сервер "Monitoring". Включим авторизацию только по ключу.

14. На сервере "Monitoring" откроем файл настроек сервера SSH:

    ```sh
    sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```

    Установим следующие значения параметров:

    ```config
    PermitRootLogin no
    StrictModes yes
    MaxAuthTries 6
    PubkeyAuthentication yes
    PasswordAuthentication no
    AllowUsers user
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

    Перезапустим сервер SSH:

    ```sh
    sudo systemctl restart ssh
    ```

15. На сервере "Monitoring" установим *fail2ban*:

    ```sh
    sudo apt update
    sudo apt install -y fail2ban
    ```

    Создадим файл локальной конфигурации копированием имеющегося:

    ```sh
    sudo cp /etc/fail2ban/jail.{conf,local}
    ```

    Откроем файл локальной конфигурации:

    ```sh
    sudo vim /etc/fail2ban/jail.local
    ```

    Приведём файл к следующему виду:

    ```config
    [sshd]
    enabled = true
    port = 22
    logpath = %(sshd_log)s
    backend = %(sshd_backend)s
    maxretry = 3
    findtime = 10m
    bantime = 30m
    bantime.increment = true
    bantime.ratio = 1
    bantime.factor = 2
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).  

    Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

16. На сервере "Monitoring" сгенерируем ключ для доступа по SSH к другим серверам:

    ```sh
    ssh-keygen -t ed25519 -C "admin@knowledge"
    ```

    Укажем имя файла для ключа: `/home/user/.ssh/id_knowledge`. Зададим пустой пароль.

17. С помощью `ssh-copy-id` скопируем ключ на каждый сервер, выполним подключение, установим на каждом сервере параметры безопасности SSH и fail2ban, аналогичные серверу "Monitoring".
