# WORK IN PROGRESS

## Установка Koji на виртуальную машину

[источник раз](https://docs.pagure.org/koji/server_howto/) |
[источник два](devops-blog.net/koji/koji-rpm-build-system-installation-part-1)

#### Из чего состоит Koji

![koji diagram](./img/koji-scheme.png)

Во-первых, есть Koji-Hub, который является сязующим звеном между всеми остальными элементами. В большинстве случаев, вместо того чтобы работать напрямую друг с другом, они взаимодействуют через koji-hub

Далее, Kojira, сервис, занимающийся созданием и обслуживанием yum/dnf репозиториев

KojiD  (должен быть как минимум один) - это демон, которые делает основную работу (собирает rpm). Его можно выделить в отдельную виртуалку, сервер.  Их может быть много, если планируется работать с большим количеством одновременных сборок. 

Koji-Web - это веб-интерфейс Koji. Полного контроля над процессом с его помощью не получишь, но с ним красивее.

Koji-Client - командный интерфейс Koji. Вот тут у нас полный контроль над всеми возможностями

Все эти части могут работать на физически разных серверах, аутентификация соединения будет происходить либо через Kerberos,  либо через SSL.

Но так как мы учимся, то все это добро поставим на одну слабую виртуалку, потому что можем :)

#### Что потребуется от юзера:

Знания:

- Базовое понимание SSL и аутентификации с помощью сертификатов (неплохо почитать про Kerberos, потому как это второй вариант аутентификации в koji)
- Базовые знания по созданию БД в Postgres и импортирование в нее схемы
- Немного уметь работать с psql
- Чуть знаний Apache
- Базовые навыки работы с dnf/yum/createrepo/mock
- Было бы неплохо владение командной строкой
- Основы по сборке RPM пакетов (иначе зачем вам все это)
- Зачатки умения работы с клиентом koji

#### Из пакетов понадобятся:

На серверной стороне (koji-hub/koji-web)

- httpd (Apache http-сервер)
- mod_ssl (SSL модуль для HTTP сервера Apache)
- postgresql-server (Серверная часть СУБД)
- mod_wsgi (Модуль для HTTP сервера который предоставляет питоновский интерфейс)

На сборочной стороне (koji-builder)

- mock
- setarch
- rpm-builder
- createrepo

[comment]: <> (написать про каждую либу)

___

#### Подготовительный этап

1) Ставим RED OS (7.3.1 на момент написания) в варианте минимальный сервер. В принципе можно поставить сервера postgres и apache, но лучше сделать это самим  

2) Для удобства работы настроим вторую сетевую карту на виртуалке и пробросим порты для подключения по SSH (я использую virtualbox, полагаю на других виртуалках схожие возможности имеются)  

[comment]: <> (написать про возможность бриджа и подключения к виртуалке извне)

<img src="./img/ether1.png" width="400" height="250" /> <img src="./img/ether2.png" width="400" height="250" />

3) Подключаемся как root (если включили такую возможность. В противном случае как юзер)  

```
ssh -p 3022 root@127.0.0.1
```

Может получиться так, что у вас уже есть ключи для локалки. В этом случае получите такое сообщение:  
<img src="./img/ssh1.png" width="700" height="250" />

Удаляем все упоминания 127.0.0.1 в файле ~/.ssh/known_hosts и пробуем еще раз

<img src="./img/ssh2.png" width="700" height="250" />

4) Делаем снапшот еще пустой системы, дабы если что-то пойдет сильно не так можно было быстро откатиться

Все, теперь работаем как белые люди, не нужно настраивать clipboard'ы и т.п.



> **NOTE:**
> 
> Аутентификация в Koji работает либо через Kerberos, либо через SSL. В данном гайде рассматривается SSL



С этого и начнем

## Создание сертификатов

*С высокой долей вероятности, если Вы раньше не работали с генерацией сертификатов, то после этого гайда Вы будете их ненавидеть*

> **_NOTE:_** 
> 
> *CA это Certificate Authority.*
> 
> Это пара ключ/сертификат используемая для подписи всех прочих запросов сертификации. Когда мы будем конфигурировать разные куски Koji оба сертификата - и клиентский, и серверный будут копией CA сгенерированного здесь. CA сертификат будет размещен в /etc/pki/koji, а сертификаты для остальных компонентов в /etc/pki/koji/certs. 
> index.txt - это база сгенерированных сертификатов. В нем удобно быстро посмотреть когда, кому и что выдали

Для начала, создадим директорию /etc/pki/koji  

```
mkdir -p /etc/pki/koji
```

и поместим туда следующий конфигурационный файл для генерации сертов  

```/etc/pki/koji/ssl.cnf```

```
HOME                            = .
RANDFILE                        = .rand

[ca]
default_ca                      = ca_default

[ca_default]
dir                             = .
certs                           = $dir/certs
crl_dir                         = $dir/crl
database                        = $dir/index.txt
new_certs_dir                   = $dir/newcerts
certificate                     = $dir/%s_ca_cert.pem
private_key                     = $dir/private/%s_ca_key.pem
serial                          = $dir/serial
crl                             = $dir/crl.pem
x509_extensions                 = usr_cert
name_opt                        = ca_default
cert_opt                        = ca_default
default_days                    = 3650
default_crl_days                = 30
default_md                      = sha256
preserve                        = no
policy                          = policy_match

[policy_match]
countryName                     = match
stateOrProvinceName             = match
organizationName                = match
organizationalUnitName          = optional
commonName                      = supplied
emailAddress                    = optional

[req]
default_bits                    = 2048
default_keyfile                 = privkey.pem
default_md                      = sha256
distinguished_name              = req_distinguished_name
attributes                      = req_attributes
x509_extensions                 = v3_ca # The extensions to add to the self signed cert
string_mask                     = MASK:0x2002

[req_distinguished_name]
countryName                     = Country Name (2 letter code)
countryName_default             = RU
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = Vladimir
localityName                    = Locality Name (eg, city)
localityName_default            = Murom
0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = RED-SOFT
organizationalUnitName          = Organizational Unit Name (eg, OS-DEVEL)
commonName                      = insert_hostname
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_max                = 64

[req_attributes]
challengePassword               = strong and complicated password
challengePassword_min           = 4
challengePassword_max           = 20
unstructuredName                = An optional company name

[usr_cert]
basicConstraints                = CA:FALSE
nsComment                       = "OpenSSL Generated Certificate"
subjectKeyIdentifier            = hash
authorityKeyIdentifier          = keyid,issuer:always

[v3_ca]
subjectKeyIdentifier            = hash
authorityKeyIdentifier          = keyid:always,issuer:always
basicConstraints                = CA:true
```

> **_NOTE:_** 
> 
> Если бы мы ставили Koji на систему на базе Centos 6, дефолтное шифрование было бы:
> 
> ```
> default_md = md5
> ```

Секцию *[req_distinguished_name]* можно отредактировать под свои нужды, дабы потом не приходилось менять дефолтные значения (город например) при каждой генерации сертификата. А их будет много

Поехали дальше

```
cd /etc/pki/koji/
mkdir {certs,private,confs}
touch index.txt
echo 01 > serial
openssl genrsa -out private/koji_ca_cert.key 2048
openssl req -config ssl.cnf -new -x509 -days 3650 -key private/koji_ca_cert.key \
-out koji_ca_cert.crt -extensions v3_ca
```

Скорее всего при выполнении последней команды у вас возникнет ошибка следующего характера:

```
[root@localhost koji]# openssl req -config ssl.cnf -new -x509 -days 3650 -key private/koji_ca_cert.key \
-out koji_ca_cert.crt -extensions v3_ca
Can't load .rand into RNG
140571765004096:error:2406F079:random number generator:RAND_load_file:Cannot open file:crypto/rand/randfile.c:98:Filename=.rand
```

Это происходит потому, что openssl ожидает увидеть объявленный в ssl.cnf файл .rand в текущей директории. А его там скорее всего нет. Надо это исправлять

```
openssl rand -writerand .rand
```

После этого повторяем команду

```
openssl req -config ssl.cnf -new -x509 -days 3650 -key private/koji_ca_cert.key \
-out koji_ca_cert.crt -extensions v3_ca
```

На отсутствующий файл ругаться перестанет и мы заполним дефолтные значения сертификата, например:

```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.

-----

Country Name (2 letter code) [RU]:
State or Province Name (full name) [Vladimir]:
Locality Name (eg, city) [Murom]:
Organization Name (eg, company) [RED-SOFT]:
Organizational Unit Name (eg, OS-DEVEL) []:os-dev
Common Name (your server's hostname) []:stapel667.red-soft.ru
Email Address []:
```

Там, где после двоеточия ничего не поставлено, будут использоваться дефолтные значения, указанные в квадратных скобках. Именно их мы редактировали в файле ssl.cnf

Результат выполнения команды *tree* (если не поставили - рекомендую, облегчает понимание, что где лежит)

```
[root@localhost koji]# tree -a --dirsfirst
.
├── certs
├── confs
├── private
│   └── koji_ca_cert.key
├── index.txt
├── koji_ca_cert.crt
├── .rand
├── serial
└── ssl.cnf
```

#### Еще про сертификаты

У каждого компонента Koji должен быть сертификат, идентифицирующий его. Два из них (для kojihub и koji-web) используются как сертификаты с серверной стороны, для аутентификации сервера-клиенту. Поэтому желательно, чтобы __Common Name__ (он же __CN__) у обоих этих сертификатов был во-первых одинаковым, а во-вторых, чтобы он был FQDN веб-сервера (fully qualified domain name. В случае этого примера: stapel667.red-soft.ru).  
__Organizational Unit Name__ (он же __OU__) можно сделать kojihub и kojiweb, чтобы их можно было отличить между собой.

Для других сертификатов (kojira, kojid, первичный админский аккаунт, все серты юзеров), используется наоборот, сертификат аутентифицирующий клиента-серверу.  
__Common Name__ для них должен быть логином конкретного компонента. 
Например __CN__ для kojira будет, как ни удивительно, kojira. Причина для этого в том, что __CN__ сертификата будет использоваться для сравнения с именем пользователя в базе данных koji. И если в базе данных не будет имени пользователя совпадающего с конкретным __CN__ - аутентификация не пройдет.

> **_NOTE:_** 
> 
> Кстати, когда мы создадим сертификат для kojiweb, было бы неплохо точно запомнить все данные, которые мы вводили при генерации, так как это потребуется при конфигурировании файла ```/etc/koji-hub/hub.conf```, а именно поля ProxyDNs

Для упрощения себе жизни создадим скрипт-генератор сертификатов  
Обзовем его ```certgen.sh```

```
#!/bin/bash

# if you change your certificate authority name to something else you will
# need to change the caname value to reflect the change.
caname=koji

# user is equal to parameter one or the first argument when you actually
# run the script
user=$1

openssl genrsa -out private/${user}.key 2048

# when you creating an user certificate, it is nice, when script change
# default CN to this username.
# Anyway you have to type it, coz in other case it will be empty string
# insert_hostname keyword (or whatever) should be in your ssl.cnf file
cat ssl.cnf | sed 's/insert_hostname/'${user}'/'> ssl2.cnf

openssl req -config ssl2.cnf -new -nodes -out certs/${user}.csr -key private/${user}.key
openssl ca -config ssl2.cnf -keyfile private/${caname}_ca_cert.key -cert ${caname}_ca_cert.crt \
    -out certs/${user}.crt -outdir certs -infiles certs/${user}.csr
cat certs/${user}.crt private/${user}.key > ${user}.pem
mv ssl2.cnf confs/${user}-ssl.cnf
```

Далее, дадим ему права на запуск

```
chmod +x certgen.sh
```

Кроме того, создадим скрипт поменьше, для генерации сертификата для браузера. Это нам потребуется один раз (если повезет), но лишним не будет.  
Назовем его webcertgen.sh. Так же сделаем его запускаемым

```
#!/bin/bash
#if you change your certificate authority name to something else you will need to change the caname value to reflect the change.
caname=koji

# user is equal to parameter one or the first argument when you actually run the script

user=$1

openssl pkcs12 -export -inkey private/${user}.key -in certs/${user}.crt \
    -CAfile ${caname}_ca_cert.crt -out certs/${user}_browser_cert.p12
```

> **NOTE:** 
> 
> Когда мы сгенерируем сертификат для пользователя, ему потребуются 
> 
> \${user}.pem, \${caname}_ca_cert.crt и \${user}_browser_cert.p12.

Первым делом создадим админа koji и сертификаты для него.
В рамках этого гайда назовем его kojiadmin. Понятное дело, что в реальной системе он может быть назван как угодно

```
./certgen.sh kojiadmin
```

Как мы уже обсуждали, CN для всего, кроме koji-hub и koji-web, должен быть равен имени пользователя, иначе аутентификация не пройдет.  
Например:

```
Country Name (2 letter code) [RU]:
State or Province Name (full name) [Vladimir]:
Locality Name (eg, city) [Murom]:
Organization Name (eg, company) [RED-SOFT]:
Organizational Unit Name (eg, OS-DEVEL) []:os-dev
kojiadmin []:kojiadmin
Email Address []:
```

Видим, что в нужном месте наш скрипт вежливо подсказал, каким должен быть CN

Пошли дальше. Создадим пользователя в системе и скопируем ему сгенерированные сертификаты в домашнюю папку

```
useradd kojiadmin
su kojiadmin
mkdir ~/.koji

# ВАЖНО использовать PEM а не CRT

cp /etc/pki/koji/kojiadmin.pem ~/.koji/client.crt
cp /etc/pki/koji/koji_ca_cert.crt ~/.koji/clientca.crt
cp /etc/pki/koji/koji_ca_cert.crt ~/.koji/serverca.crt
exit
```

> **NOTE:**
> 
> Настройки koji хранятся в файле ```/etc/koji.conf```. Если требуется настройка в зависимости от пользователя, то можно разместить файл koji.conf в ~/.koji каждого конкретного юзера

### База данных

Мы будем использовать postgres, в теории можно любую.

Установим сервер БД и CLI koji

```
dnf install -y postgresql-server koji
```

Инициализируем БД и запускаем сервис

```
postgresql-setup --initdb --unit postgresql
systemctl enable postgresql --now
```

Создаем пользователя koji. Для удобства сделаем без пароля.

```
useradd koji
passwd -d koji
```

Теперь настроим postgres и создадим схему

```
su postgres
createuser --no-superuser --no-createrole --no-createdb koji
createdb -O koji koji
su koji
psql koji koji < /usr/share/doc/koji*/docs/schema.sql
\q
exit
```

> **NOTE:**
> 
> __Важно__: Когда импортируем схему в чистую БД,  необходимо, чтобы это осуществлялось от лица koji юзера. Иначе будут проблемы

Отредактируем ```/var/lib/pgsql/data/pg_hba.conf``` 
Так как мы все делаем на одной машине: 

```
#TYPE   DATABASE    USER    CIDR-ADDRESS      METHOD
local   koji        koji                       trust
local   all         postgres                   peer
```

> **NOTE:**
> 
> - Тип соединения __local__  означает, что postgres использует локальный Unix сокет, поэтому PostgreSQL не открывается по TCP/IP.
> 
> - Локальный юзер __koji__ должен иметь доступ только к базе данных __koji__. Локальный пользователь __postgres__ будет иметь доступ ко всему (это нужно для создания юзеров и таблиц)
> 
> - CIDR-ADDRESS пустой, потому что в нашем случае мы используем локальные Unix сокеты.
> 
> - Метод __trust__ значит, что PosgreSQL разрешит любое соединение от любого локального юзера с таким именем. Мы установили это для юзера __koji__, потому что Apache httpd запускается как  пользователь __apache__ вместо пользователя __koji__ когда подсоединяется к сокету Unix. 
>   
>   __trust__ не особо секьюрно в системах с множеством пользователей , но в нашей однозадачной системе это нормально.
> 
> - Метод __peer__ означает, что PostgreSQL получит юзернейм пользователя и будет выполнять то, что этому пользователю позволено. Это безопаснее, потому как только локальные юзеры postgres смогут подключиться к PostgreSQL с этим уровнем доступа.

В файле ```/var/lib/pgsql/data/postgresql.conf``` блокируем доступ по TCP/IP (мы ж на одной машине все делаем)

```
listen_addresses = ''
```

Перезапускаем postgres

```
systemctl reload postgresql
```

Создадим нашего __kojiadmin__'a в БД

```
su koji
psql
insert into users (name, status, usertype) values ('kojiadmin', 0, 0);
insert into user_perms (user_id, perm_id, creator_id) values (1, 1, 1);
\q
exit
```

С базой данный все.
Теперь опять к сертификатам

### Koji-hub

Установим apache, koji-hub и ssl-модуль для апача

```
dnf install -y httpd koji-hub mod_ssl
```

Сгенерируем сертификаты для kojihub и kojiweb. Как мы помним, CN в этом случае равен FQDN (в моем случае, stapel667.red-soft.ru), а OU - kojihub или kojiweb соответственно. Тут наша подсказка, которая заменяет CN, будет немного мешать, поэтому будьте внимательны и осторожны

```
cd /etc/pki/koji
./certgen.sh kojiweb
./certgen.sh kojihub
```

Что в нашей любимой папке:

```
[root@localhost koji]# tree -a --dirsfirst
.
├── certs
│   ├── 01.pem
│   ├── 02.pem
│   ├── 03.pem
│   ├── 04.pem
│   ├── kojiadmin.crt
│   ├── kojiadmin.csr
│   ├── kojihub.crt
│   ├── kojihub.csr
│   ├── kojiweb.crt
│   └── kojiweb.csr
├── confs
│   ├── kojiadmin-ssl.cnf
│   ├── kojihub-ssl.cnf
│   └── kojiweb-ssl.cnf
├── private
│   ├── kojiadmin.key
│   ├── koji_ca_cert.key
│   ├── kojihub.key
│   └── kojiweb.key
├── certgen.sh
├── index.txt
├── index.txt.attr
├── index.txt.attr.old
├── index.txt.old
├── kojiadmin.pem
├── koji_ca_cert.crt
├── kojihub.pem
├── kojiweb.pem
├── .rand
├── serial
├── serial.old
├── ssl.cnf
└── webcertgen.sh
```

Можем посмотреть, какие сертификаты мы на текущий момент создали в ```index.txt```

```
[root@localhost koji]# cat index.txt
V	320805071659Z		01	unknown	/C=RU/ST=Vladimir/O=RED-SOFT/OU=os-dev/CN=kojiadmin
V	320805075641Z		02	unknown	/C=RU/ST=Vladimir/O=RED-SOFT/OU=kojiweb/CN=stapel667.red-soft.ru
V	320805075713Z		03	unknown	/C=RU/ST=Vladimir/O=RED-SOFT/OU=kojihub/CN=stapel667.red-soft.ru
```



В ```/etc/httpd/conf.d/kojihub.conf``` раскомментируем следующее:

```
# uncomment this to enable authentication via SSL client certificates

# <Location /kojihub/ssllogin>
# SSLVerifyClient require
# SSLVerifyDepth  10
# SSLOptions +StdEnvVars
# </Location>
```

В ```/etc/httpd/conf.d/ssl.conf``` укажем требуемые пути до ключей и сертификатов

```
ServerName stapel667.red-soft.ru:443

SSLCertificateFile /etc/pki/koji/certs/kojihub.crt
SSLCertificateKeyFile /etc/pki/koji/private/kojihub.key
SSLCertificateChainFile /etc/pki/koji/koji_ca_cert.crt
SSLCACertificateFile /etc/pki/koji/koji_ca_cert.crt

SSLVerifyClient require
SSLVerifyDepth 10
```

В ```/etc/koji-hub/hub.conf```

```
DBName = koji
DBUser = koji

KojiDir = /mnt/koji
LoginCreatesUser = On
KojiWebURL = http://stapel667.red-soft.ru/koji

DNUsernameComponent = CN
ProxyDNs = CN=stapel667.red-soft.ru,OU=kojiweb,O=RED-SOFT,ST=Vladimir,C=RU
```

Чтобы позволить апачу подсоединиться к БД

```
setsebool -P httpd_can_network_connect_db=1
```

#### Подготовка файловой системы

```
cd /mnt
mkdir koji
cd koji
mkdir {packages,repos,work,scratch,repos-dist}
chown apache.apache *
```

Нужно поставить нехватающую либу (semanage)

```
dnf install -y policycoreutils-python-utils
```

Разрешим апачу писать в созданную файловую систему

```
setsebool -P allow_httpd_anon_write=1
semanage fcontext -a -t public_content_rw_t "/mnt/koji(/.*)?"
restorecon -r -v /mnt/koji
```

Запустим Apache

```
systemctl enable httpd.service --now
```

### Koji CLI

Зайдем в ```/etc/koji.conf```

```
[koji]

;url of XMLRPC server
server = http://stapel667.red-soft.ru/kojihub

;url of web interface
weburl = http://stapel667.red-soft.ru/koji

;url of package download site
topurl = http://stapel667.red-soft.ru/kojifiles

;path to the koji top directory
topdir = /mnt/koji

; configuration for SSL athentication

;client certificate
cert = ~/.koji/client.crt

;certificate of the CA that issued the HTTP server certificate
serverca = ~/.koji/serverca.crt
```

Проверим

```
su kojiadmin
koji moshimoshi
```

Может возникнуть следующая проблема:

```
[kojiadmin@localhost koji]$ koji moshimoshi
2022-08-08 11:34:11,550 [ERROR] koji: ConnectionError: HTTPSConnectionPool(host='stapel667.red-soft.ru', port=443): Max retries exceeded with url: /kojihub/ssllogin (Caused by NewConnectionError('<urllib3.connection.HTTPSConnection object at 0x7f7eb4dfc070>: Failed to establish a new connection: [Errno -2] Name or service not known'))
```

Это происходит потому, что koji понятия не имеет, куда идти по адресу __CN__

Решается она просто, идем в файл ```/etc/hosts```

```
127.0.0.1 stapel667.red-soft.ru
```

В результате, должно получиться следующее:

```
[kojiadmin@localhost koji]$ koji moshimoshi
olá, kojiadmin!

You are using the hub at https://stapel667.red-soft.ru/kojihub
Authenticated via client certificate /home/kojiadmin/.koji/client.crt
```

Можно проверить и так:

```
koji call getLoggedInUser
```

```
[kojiadmin@localhost koji]$ koji call getLoggedInUser
{'authtype': 2,
 'id': 1,
 'krb_principal': None,
 'krb_principals': [],
 'name': 'kojiadmin',
 'status': 0,
 'usertype': 0}
```



> **NOTE:**
> 
> На самом деле это только один из вариантов проблем. Остальные 150 штук в основном возникают из-за косяков с сертификатами и путей к ним в конфигах.



#### Koji-Web

Установим koji-web. mod_ssl мы уже ставили ранее

```
dnf install -y koji-web
```

Идем в ```/etc/httpd/conf.d/kojiweb.conf``` и раскомментируем

```
# uncomment this to enable authentication via SSL client certificates

# <Location /koji/login>
#     SSLVerifyClient require
#     SSLVerifyDepth  10
#     SSLOptions +StdEnvVars
# </Location>
```

Файл ```/etc/httpd/conf.d/ssl.conf``` мы уже откорректировали ранее



Пошли в ```/etc/kojiweb/web.conf```

```
[web]
SiteName = koji
# KojiTheme =

# Necessary urls
KojiHubURL = https://stapel667.red-soft.ru/kojihub
KojiFilesURL = http://stapel667.red-soft.ru/kojifiles

## SSL authentication options
WebCert = /etc/pki/koji/kojiweb.pem
ClientCA = /etc/pki/koji/koji_ca_cert.crt
KojiHubCA = /etc/pki/koji/koji_ca_cert.crt

LoginTimeout = 72

# This must be set before deployment
Secret = CHANGE_ME

LibPath = /usr/share/koji-web/lib
```

Перезапустим Apache

```
systemctl restart httpd.service
```

В принципе, веб-интерфейс уже должен работать. Но залогиниться не получится, так как мы не сгенерировали сертификат для браузера

Идем в ```/etc/pki/koji``` и запускаем ```webcertgen.ch``` для __kojiadmin__

```
cd /etc/pki/koji
./webcertgen.sh kojiadmin
```



Попробуем зайти на веб-интерфейс.

Можем получить следующее сообщение

```
An error has occurred while processing your request.
ConnectionError: HTTPConnectionPool(host='stapel667.red-soft.ru', port=80): Max retries exceeded with url: /kojihub (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7fa47ff8b4d0>: Failed to establish a new connection: [Errno 13] Permission denied',))
```

Решается она несложно

```
setsebool -P httpd_can_network_connect on
```

После этого все должно быть в порядке.

Для того, чтобы залогиниться, нам потребуется импортировать `kojiadmin_browser_cert.p12` в браузер. Если все пройдет удачно, то мы сможем зайти под kojiadmin