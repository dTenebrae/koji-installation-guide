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

<img src="./img/ether1.png" width="500" height="350" /> <img src="./img/ether2.png" width="500" height="350" />

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

Для начала, создадим директорию /etc/pki/koji  

```
mkdir -p /etc/pki/koji
```

и поместим туда следующий конфигурационный файл для генерации сертов  

*ssl.conf*
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


