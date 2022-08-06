# Установка Koji на виртуальную машину

[источник раз](https://docs.pagure.org/koji/server_howto/)  
[источник два](devops-blog.net/koji/koji-rpm-build-system-installation-part-1)

![koji diagram](http://www.devops-blog.net/wp-content/uploads/2013/03/Zeichnung0.png)

[comment]: <> (сделать локальную ссылку на изображение, либо нарисовать свое)


#### Что потребуется:

Знания:
- Базовое понимание SSL и аутентификации с помощью сертификатов (неплохо почитать про Kerberos, потому как это второй вариант аутентификации в koji)
- Базовые знание по созданию БД в Postgres и импортирование в нее схемы
- Немного работать с psql
- Чуть знаний Apache
- Базовые навыки работы с dnf/yum/createrepo/mock
- Было бы неплохо владение командной строкой
- Основы по сборке RPM пакетов (иначе зачем вам все это)
- Зачатки умения работы с клиентом koji

#### Из пакетов понадобятся:

На серверной стороне(koji-hub/koji-web)
- httpd (Apache http-сервер)
- mod_ssl (SSL модуль для HTTP сервера Apache)
- postgresql-server (Серверная часть СУБД)
- mod_wsgi (Модуль для HTTP сервера который предоставляет питоновский интерфейс)

На сборочной стороне(koji-builder)
- mock
- setarch
- rpm-builder
- createrepo



