# Установка MediaWiki

## Описание

На веб-серверах устанавливается MediaWiki.

## Требования

* [Настройка веб-серверов и балансировщиков нагрузки](balancers-web.md)

## Выполнение

1. На хостах "Web1" и "Web2" установим модули PHP:

    ```sh
    sudo apt update
    sudo apt install -y php-fpm php-pgsql php-xml php-intl php-mbstring php-apcu unzip
    ```

    Проверим версии nginx и PHP:

    ```sh
    nginx -v
    php -v
    ```

2. Загрузим архив файлов MediaWiki, извлечём из него файлы:

    ```sh
    cd /var/www
    sudo wget https://releases.wikimedia.org/mediawiki/1.43/mediawiki-1.43.0.tar.gz
    sudo tar -xvzf mediawiki-*.tar.gz
    sudo mv mediawiki-1.43.0 mediawiki
    sudo chown -R www-data:www-data /var/www/mediawiki
    ```

3. Создадим файл конфигурации сайта и откроем его в редакторе:

    ```sh
    sudo vim /etc/nginx/sites-available/mediawiki
    ```

    В редакторе обеспечим следующие содержимое файла:

    ```config
    server {
        listen 80;
        server_name wiki.example.com;

        root /var/www/mediawiki;
        index index.php;

        location / {
            try_files $uri $uri/ /index.php;
        }

        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass unix:/run/php/php-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

4. Выключим сайт по умолчанию, включим новый сайт и перезапустим службу nginx:

    ```sh
    sudo rm  /etc/nginx/sites-enabled/default
    sudo ln -s /etc/nginx/sites-available/mediawiki /etc/nginx/sites-enabled/
    sudo systemctl restart nginx php8.3-fpm
    ```

5. Проверим работу сайтов: откроем адреса http://192.168.3.5 и http://192.168.3.6 в веб-браузере. Должна отобразиться стартовая страница MediaWiki с предложением настройки.
