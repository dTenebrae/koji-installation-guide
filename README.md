# WORK IN PROGRESS



## Установка Koji на виртуальную машину

[источник раз](https://docs.pagure.org/koji/server_howto/) |
[источник два](devops-blog.net/koji/koji-rpm-build-system-installation-part-1)

#### Из чего состоит Koji

![koji diagram](./img/koji-scheme.png)

Во-первых, есть Koji-Hub, который является сязующим звеном между всеми остальными элементами. В большинстве случаев, вместо того чтобы работать напрямую друг с другом они взаимодействуют через koji-hub

Далее, Kojira, сервис, занимающийся созданием и присмотром за yum/dnf репозиториями

KojiD - должен быть как минимум один. Это демон, которые делает основную работу (собирает rpm). Его можно выделить в отдельный инстанс, их может быть много, если вы планируете работать с большим количеством одновременных сборок

Koji-Web - это веб-интерфейс Koji. Полного контроля над процессом с его помощью не получишь, но с ним красивее. В общем то можно обойтись и без него, но мы не будем

Koji-Client - командный интерфейс Koji. Вот тут у нас полный контроль над всеми возможностями

Все эти части могут работать на физически разных серверах, аутентификация соединения происходил либо через Kerberos,  либо через SSL.

Но мы все это добро поставим на одну слабую виртуалку, потому что можем :)

[comment]: <> (нарисовать свое)


#### Что потребуется:

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

___


#### *Аутентификация в Koji работает либо через Kerberos, либо через SSL. В данном гайде рассматривается SSL*

С этого и начнем

___

## Создание сертификатов

*С высокой долей вероятности, если Вы раньше не работали с генерацией сертификатов, то после этого гайда Вы будете их ненавидеть*

*CA это Certificate Authority.*

Это пара ключ/сертификат используемая для подписи всех прочих запросов сертификации. Когда мы будем конфигурировать разные куски Koji оба сертификата - и клиентский, и серверный будут копией CA сгенерированного здесь. CA сертификат будет размещен в /etc/pki/koji, а сертификаты для остальных компонентов в /etc/pki/koji/certs.  

index.txt - это база сгенерированных сертификатов. В нем удобно быстро посмотреть когда, кому и что выдали

Для начала, создадим директорию /etc/pki/koji  

```
mkdir -p /etc/pki/koji
```

и поместим туда следующий конфигурационный файл для генерации сертов  

*ssl.cnf*
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
commonName                      = Common Name (your server\'s hostname)
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

Если бы мы ставили Koji на систему на базе Centos 6, дефолтное шифрование было бы:
```
default_md = md5
```

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

<img src="./img/error1.png" width="750" height="200" />

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

Там, гда после двоеточия ничего не поставлено - будут использоваться дефолтные значения, указанные в квадратных скобках. Именно их мы редактировали в файле ssl.cnf

Результат выполнения команды *tree* (если не поставили - рекомендую, облегчает понимание, что где лежит)

```
[root@localhost koji]# tree -a
.
├── certs
├── confs
├── index.txt
├── koji_ca_cert.crt
├── private
│   └── koji_ca_cert.key
├── .rand
├── serial
└── ssl.cnf
```

#### Еще про сертификаты

У каждого компонента Koji должен быть сертификат, идентифицирующий его. Два из них (для kojihub и koji-web) используются как сертификаты с серверной стороны, для аутентификации сервера-клиенту. Поэтому желательно, чтобы __Common Name__ (он же CN) у обоих этих сертификатов был во-первых одинаковым, а во-вторых, чтобы он был FQDN веб-сервера (fully qualified domain name. В случае этого примера: stapel667.red-soft.ru).  
__Organizational Unit Name__ (он же OU) можно сделать kojihub и kojiweb, чтобы их можно было отличить между собой.

Для других сертификатов (kojira, kojid, первичный админский аккаунт, все серты юзеров), используется наоборот, сертификат аутентифицирующий клиента-серверу.  
__Common Name__ для них должен быть логином конкретного компонента. 
Например CN для kojira будет, как ни удивительно, kojira. Причина для этого в том, что CN сертификата будет использоваться для сравнения с именем пользоваеля к базе данных koji. И если в базе данных не будет имени пользователя совпадающего с конкретным CN - аутентификация не пройдет.

Когда мы создадим сертификат для kojiweb, было бы неплохо точно запомнить все данные, которые мы вводили при генерации, так как это потребуется при конфигурировании файла 
/etc/koji-hub/hub.conf, а именно поля ProxyDNs

В рамках этого гайда админский аккаунт назовем kojiadmin. Понятное дело, что в реальной системе он может быть назван как угодно

