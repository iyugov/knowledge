# Настройка синхронизации каталога загрузки файлов

## Описание

Настраивается синхронизация каталогов загрузки файлов wiki на двух веб-серверах. Используется Unison.

## Требования

* [Настройка MediaWiki](mediawiki-database.md)

## Выполнение

1. На каждом из хостов "Web1" и "Web2" установим пакеты для синхронизации:

    ```sh
    sudo apt update
    sudo apt install -y unison inotify-tools
    ```

2. На хосте "Monitoring" выполним копирование ключей SSH на хосты "Web1" и "Web2":

    ```sh
    scp ~/.ssh/id_knowledge* web1:/home/user/.ssh
    scp ~/.ssh/id_knowledge* web2:/home/user/.ssh
    ```

    Укажем путь к файлу для сохранения ключа - по умолчанию (`/home/user/.ssh/id_ed25519`). Зададим пустой пароль.

3. На хосте "Web1" создадим файл `/home/user/.ssh/config` следующего содержания:

    ```config
    Host 192.168.3.6
        Hostname 192.168.3.6
        Port 22
        IdentityFile ~/.ssh/id_knowledge
        IdentitiesOnly yes
    ```

    Выполним тестовое подключение к хосту "Web2":

    ```sh
    ssh web2
    ```

4. На хосте "Web2" создадим файл `/home/user/.ssh/config` следующего содержания:

    ```config
    Host 192.168.3.5
        Hostname 192.168.3.5
        Port 22
        IdentityFile ~/.ssh/id_knowledge
        IdentitiesOnly yes
    ```

    Выполним тестовое подключение к хосту "Web1":

    ```sh
    ssh web1
    ```

5. На хосте "Web1" создадим ссылку на каталог файлов wiki:

    ```sh
    ln -s /var/www/mediawiki/images /home/user/wiki-images
    ```

    Включим пользователя в группу для права использовать каталог и дадим права группе:

    ```sh
    sudo usermod user -aG www-data
    sudo chmod g+w /var/www/mediawiki/images
    ```

    Завершим сеансы пользователя `user` и авторизуемся повторно.

    Создадим каталог `/home/user/.unison` и в нём - файл `images.prf` со следующим содержанием:

    ```config
    root = /home/user/wiki-images
    root = ssh://user@192.168.3.6//home/user/wiki-images
    auto = true
    batch = true
    prefer = newer
    fastcheck = true
    times = true
    repeat = watch
    confirmbigdel = false
    log = true
    logfile = /var/log/unison-images.log
    ```

    Создадим лог-файл:

    ```sh
    sudo touch /var/log/unison-images.log
    sudo chown user:user /var/log/unison-images.log
    ```

6. На хосте "Web2" создадим ссылку на каталог файлов wiki:

    ```sh
    ln -s /var/www/mediawiki/images /home/user/wiki-images
    ```

    Включим пользователя в группу для права использовать каталог и дадим права группе:

    ```sh
    sudo usermod user -aG www-data
    sudo chmod g+w /var/www/mediawiki/images
    ```

    Завершим сеансы пользователя `user` и авторизуемся повторно.

    Создадим каталог `/home/user/.unison` и в нём - файл `images.prf` со следующим содержанием:

    ```config
    root = /home/user/wiki-images
    root = ssh://user@192.168.3.5//home/user/wiki-images
    auto = true
    batch = true
    prefer = newer
    fastcheck = true
    times = true
    repeat = watch
    confirmbigdel = false
    log = true
    logfile = /var/log/unison-images.log
    ```

    Создадим лог-файл:

    ```sh
    sudo touch /var/log/unison-images.log
    sudo chown user:user /var/log/unison-images.log
    ```

7. На каждом из хостов "Web1" и "Web2" создадим скрипт `/home/user/sync-images.sh` следующего содержания:

    ```sh
    #!/bin/bash
    WATCHDIR="/home/user/wiki-images"
    UNISON_PROFILE="images"

    inotifywait -m -r -e modify,create,delete,move --format '%w%f' "$WATCHDIR" | while read FILE
    do
        logger -t unison-sync "Detected change in $FILE, running unison"
        unison "$UNISON_PROFILE"
    done
    ```

    Установим для скрипта право на исполнение:

    ```sh
    chmod +x /home/user/sync-images.sh
    ```

8. На каждом из хостов "Web1" и "Web2" создадим юнит службы systemd `/etc/systemd/system/unison-images.service` со следующим содержанием:

    ```ini
    [Unit]
    Description=Unison image directory sync watcher
    After=network.target

    [Service]
    User=user
    ExecStart=/home/user/sync-images.sh
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```

    На каждом из хостов активируем службу и проверим её состояние:

    ```sh
    sudo systemctl daemon-reexec
    sudo systemctl daemon-reload
    sudo systemctl enable unison-images
    sudo systemctl start unison-images
    sudo systemctl status unison-images
    ```
