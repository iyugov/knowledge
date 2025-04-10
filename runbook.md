# План восстановления

## Восстановление базы данных

### Организация тестового сбоя базы данных

1. Убедимся, что резервные копии на хосте `backup` существуют, доступны и актуальны. Если нет, то создадим резервную копию:

    ```sh
    ssh backup
    sudo systemctl start backup-wiki
    ```

    Убедимся, что резервная копия создана.

2. Определим лидера кластера Patroni:

    ```sh
    ssh backup
    patronictl -c /etc/patroni.yml list
    ```

3. На хосте-лидере удалим базу данных:

    ```sh
    sudo -u postgres dropdb doc_wiki
    ```

4. Убедимся, что MediaWiki не работает, сообщая об ошибке подключения к базе данных.

### Восстановление базы данных

1. На хосте `backup` в `/var/backups/wiki` определим резервную копию для восстановления из неё. Пусть она располагается в каталоге `backup_X`.

2. Пусть лидер кластера - `db1`. Скопируем файл резервной копии базы данных на него:

    ```sh
    ssh monitoring
    scp user@backup:/var/backups/wiki/backup_X/db.sql db1:/home/user/wiki_restore.sql
    ```

3. Создадим базу данных на лидере и восстановим её:

    ```sh
    ssh db1
    sudo -u postgres createdb -O postgres doc_wiki
    mkdir wiki_restore
    mv wiki_restore.sql wiki_restore
    sudo chown -R postgres:postgres wiki_restore
    cd wiki_restore
    sudo -u postgres pg_restore -d doc_wiki wiki_restore.sql
    ```

4. Убедимся, что MediaWiki снова работает.

5. Дождёмся окончания репликации восстановленной базы данных:

    ```sh
    ssh db1
    patronictl -c /etc/patroni.yml list
    ```

6. Удалим файл, из которого восстанавливалась резервная копия:

    ```sh
    ssh db1
    rm -rf /home/user/wiki_restore
    ```

## Восстановление каталога файлов MediaWiki

### Организация тестового сбоя каталога файлов

1. Убедимся, что резервные копии на хосте `backup` существуют, доступны и актуальны. Если нет, то создадим резервную копию:

    ```sh
    ssh backup
    sudo systemctl start backup-wiki
    ```

    Убедимся, что резервная копия создана.

2. На обоих веб-серверах удалим каталоги файлов MediaWiki:

    ```sh
    ssh web1
    sudo rm -rf /var/www/mediawiki
    ssh web2
    sudo rm -rf /var/www/mediawiki
    ```

3. Убедимся, что MediaWiki не работает.

### Восстановление каталога файлов

1. На хосте `backup` в `/var/backups/wiki` определим резервную копию для восстановления из неё. Пусть она располагается в каталоге `backup_X`.

2. Остановим синхронизацию файлов на обоих веб-серверах:

    ```sh
    ssh web1
    sudo systemctl stop unison-images
    ssh web2
    sudo systemctl stop unison-images
    ```

3. Скопируем файл резервной копии базы данных на каждый веб-сервер:

    ```sh
    ssh monitoring
    scp user@backup:/var/backups/wiki/backup_X/files.tar.gz web1:/home/user/wiki_restore.tar.gz
    scp user@backup:/var/backups/wiki/backup_X/files.tar.gz web2:/home/user/wiki_restore.tar.gz
    ```

4. Восстановим каталог файлов:

    ```sh
    ssh web1
    sudo tar -xzf wiki_restore.tar.gz -C /
    ssh web2
    sudo tar -xzf wiki_restore.tar.gz -C /
    ```

5. Запустим синхронизацию каталогов:

    ```sh
    ssh web1
    sudo systemctl start unison-images
    ssh web2
    sudo systemctl start unison-images
    ```

    Дождёмся окончания синхронизации (см. лог `/var/log/unison-images.log` на каждом сервере).

6. Убедимся, что MediaWiki снова работает на каждом веб-сервере, используя их индивидуальные IP-адреса (не через балансировщик), в том числе отображаются изображения.

7. Удалим файл, из которого восстанавливалась резервная копия:

    ```sh
    ssh web1
    rm wiki_restore.tar.gz
    ssh web2
    rm wiki_restore.tar.gz
    ```
