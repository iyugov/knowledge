# Настройка резервного копирования базы данных

## Описание

Настраивается периодическое резервное копирование базы данных wiki.

## Требования

* [Настройка мониторинга](monitoring.md)

## Выполнение

1. На хосте "Backup" создадим каталог для резервного копирования wiki:

    ```sh
    sudo mkdir /var/backups/wiki
    ```

2. Создадим файл `/home/user/backup-wiki.sh` следующего содержания (вместо `PGPASS` укажем пароль пользователя `postgres` серверов баз данных:

    ```sh
    #!/bin/bash

    # Параметры базы данных
    DB_HOST=192.168.3.14
    DB_USER="postgres"
    DB_NAME="doc_wiki"
    DB_PASSWORD="PGPASS"
    BACKUP_DIR="/var/backups/wiki"
    PG_PASS_FILE="/tmp/pgpass"

    # Ограничения на резервные копии
    MAX_BACKUPS=30
    MIN_FREE_SPACE_PERCENT=10
    MIN_FREE_INODES_PERCENT=10

    # Лог-файл
    LOG_FILE="/var/log/backup-wiki.log"

    DATE=$(date +%Y-%m-%d_%H-%M-%S)
    BACKUP_FILE="${BACKUP_DIR}/backup_${DB_NAME}_${DATE}.sql"

    log_message() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
    }

    log_message "Резервное копирование базы данных MediaWiki начато."

    if [ -z "$DB_HOST" ]; then
        echo "Ошибка: хост базы данных недоступен."
        exit 1
    fi

    echo "$DB_HOST:5432:$DB_NAME:$DB_USER:$DB_PASSWORD" > $PG_PASS_FILE
    chmod 600 $PG_PASS_FILE


    USED_SPACE_PERCENT=$(df $BACKUP_DIR | awk 'NR==2 {print $5}' | sed 's/%//')
    USED_INODES_PERCENT=$(df -i $BACKUP_DIR | awk 'NR==2 {print $5}' | sed 's/%//')
    FREE_SPACE_PERCENT=$((100 - USED_SPACE_PERCENT))
    FREE_INODES_PERCENT=$((100 - USED_INODES_PERCENT))

    if [ $FREE_SPACE_PERCENT -lt $MIN_FREE_SPACE_PERCENT ] || [ $FREE_INODES_PERCENT -lt $MIN_FREE_INODES_PERCENT ]; then
        log_message "Предупреждение: недостаточно свободного места или инодов. Резервное копирование не произведено."
        rm -f $PG_PASS_FILE
        exit 1
    fi

    PGPASSWORD="$DB_PASSWORD" pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME -F c -b -v -f "$BACKUP_FILE"

    if [ $? -eq 0 ]; then
        log_message "Резервное копирование базы данных выполнено успешно: $BACKUP_FILE"
    else
        log_message "Ошибка резервного копирования базы данных."
        rm -f $PG_PASS_FILE
        exit 1
    fi

    BACKUP_COUNT=$(ls $BACKUP_DIR | grep -E '^backup_' | wc -l)

    if [ $BACKUP_COUNT -gt $MAX_BACKUPS ]; then
        log_message "Количество резервных копий превышает максимальное значение ($MAX_BACKUPS). Удаляем старые копии..."
        ls -t $BACKUP_DIR/backup_* | tail -n +$((MAX_BACKUPS)) | xargs rm -f
        log_message "Старые копии удалены."
    fi

    rm -f $PG_PASS_FILE
    ```

    Установим файлу права на выполнение:

    ```sh
    sudo chmod +x /home/user/backup-wiki.sh
    ```

3. Проверим и отладим работу скрипта:

    ```sh
    sudo /home/user/backup-wiki.sh
    ls /var/backups/wiki
    cat /var/log/backup-wiki.log
    ```

4. Создадим файл службы systemd `/etc/systemd/system/backup-wiki.service` следующего содержания:

    ```ini
    [Unit]
    Description=MediaWiki batabase backup
    StartLimitIntervalSec=12h
    StartLimitBurst=3

    [Service]
    Type=oneshot
    ExecStart=/home/user/backup-wiki.sh
    Restart=on-failure
    RestartSec=1h
    ```

5. Создадим файл таймера systemd `/etc/systemd/system/backup-wiki.timer` следующего содержания:

    ```ini
    [Unit]
    Description=MediaWiki batabase backup timer

    [Timer]
    OnCalendar=daily
    Unit=backup-wiki.service

    [Install]
    WantedBy=timers.target
    ```

6. Учтём изменения в службах, включим новую службу:

    ```sh
    sudo systemctl daemon-reload
    sudo systemctl enable --now backup-wiki.timer
    ```

7. Проверим работу службы:

    ```sh
    sudo systemctl start backup-wiki
    ls /var/backups/wiki
    cat /var/log/backup-wiki.log
    ```
