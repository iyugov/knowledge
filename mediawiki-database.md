# Настройка MediaWiki

## Описание

На веб-серверах настраивается MediaWiki.

## Требования

* [Настройка балансировщиков нагрузки для серверов баз данных](balancers-database.md)

## Выполнение

1. В веб-браузере на хосте, имеющим доступ к хосту "Web1", откроем ресурс по соответствующеу адресу и порту в каталоге `/wiki`. Нажмём ссылку "set up the wiki". Выберем язык и язык справки - русский.

2. Укажем следующие параметры:

    * тип базы данных - PostgreSQL;
    * хост базы данных - `192.168.3.14`;
    * порт базы данных - 5432;
    * имя базы данных - `doc_wiki`;
    * схема для MediaWiki - `mediawiki`;
    * имя пользователя базы данных - `postgres`;
    * пароль базы данных - ранее установленный пароль пользователя `postgres`.

3. Подтвердим, что пользователь базы данных будет тот же, что и для установки. Зададим название вики - "Система документации".

4. Создадим учётную запись участника. Выберем вариант: "Хватит уже, просто установите вики".

5. Загрузим файл LocalSettings.php, установим в нём значение параметра `$wgEnableUploads = true;` поместим его на хосты "Web1" и "Web2" в каталог `/var/www/mediawiki`, установим ему владельца `www-data` и группу `www-data`.

6. Откроем в браузере сайт Wiki, убедимся в его работоспособности.
