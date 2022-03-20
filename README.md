# Инструкция по ЛР №16 Протокол Kerberos
## Конфигурация сетевых карт ВМ:
 - Виртуальные сети VBox:
   - Host-Network:
     - IP: 192.168.0.254
     - Mask: 255.255.255.0
     - DHCP: disabled
 - ВМ1:
   - enp0s3(NAT/bridge) -> интернет для скачивания пакетов
   - enp0s8(подключен к HostNetwork)
     - IP: 192.168.0.1/24
     - Gate: 192.168.0.254
 - ВМ2:
   - enp0s3(NAT/bridge) -> интернет для скачивания пакетов
   - enp0s8(подключен к HostNetwork)
     - IP: 192.168.0.2/24
     - Gate: 192.168.0.254
 - ВМ3:
   - enp0s3(NAT/bridge) -> интернет для скачивания пакетов
   - enp0s8(подключен к HostNetwork)
     - IP: 192.168.0.3/24
     - Gate: 192.168.0.254
---
## 0. Подготовка машин.
### Инфраструктура:
 - ВМ1 => vm1.mad.net (Клиент)
 - ВМ2 => vm2kbs.mad.net (Сервер LDAP+KBS)
 - ВМ3 => vm3.mad.net (Клиент)
---
### Действия ВМ1:
 - Переименовать машину:\
 `hostnamectl set-hostname "vm1.mad.net"`
 - Настраиваем IP адреса: `nano /etc/network/interfaces`
    ```conf
    auto enp0s3
    iface enp0s3 inet dhcp

    auto enp0s8
    iface enp0s8 inet static
    address 192.168.0.2/24
    gateway 192.168.0.254
    ```
 - Разрешаем `root` логин по `ssh`:\
 `sed -i -e 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config`
---
### Действия ВМ2:
 - Переименовать машину:\
 `hostnamectl set-hostname "vm2kbs.mad.net"`
 - Настраиваем IP адреса: `nano /etc/network/interfaces`
    ```conf
    auto enp0s3
    iface enp0s3 inet dhcp

    auto enp0s8
    iface enp0s8 inet static
    address 192.168.0.1/24
    gateway 192.168.0.254
    ```
---
### Действия ВМ3:
 - Переименовать машину:\
 `hostnamectl set-hostname "vm3.mad.net"`
 - Настраиваем IP адреса: `nano /etc/network/interfaces`
    ```conf
    auto enp0s3
    iface enp0s3 inet dhcp

    auto enp0s8
    iface enp0s8 inet static
    address 192.168.0.3/24
    gateway 192.168.0.254
    ```
 - Разрешаем `root` логин по `ssh`:\
 `sed -i -e 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config`
---
### Действия для всех машин по очереди:
 - Перезагружаем:\
 `reboot`
 - Проверим подключение к интернету:\
 `ping 8.8.8.8`
 - Проверим подключение к Host-Network:\
 `ping 192.168.0.254`
 - Обновляемся:\
 `apt update && apt upgrade -y`
   - `/dev/sda` - место установки GRUB
---
### Действия ВМ2:
 - На этот раз, чтобы не вводить постоянно IP адреса, добавим все машины в файл: `nano /etc/hosts`
    ```conf
    192.168.0.1 vm2kbs.mad.net vm2kbs
    192.168.0.2 vm1.mad.net vm1
    192.168.0.3 vm3.mad.net vm3
    ```
 - Скопируем этот файл на ВМ1 и ВМ3:\
 `scp /etc/hosts root@vm1:/etc/`\
 `scp /etc/hosts root@vm3:/etc/`
## 1. Запустить и настроить сервер OpenLDAP и сервер Mit Kerberos на ВМ2.
### Основные значения:
 - Домен: mad.net
 - Данные LDAP администратора: admin:secret
 - IP адрес LDAP/KBS сервера: 192.168.0.1
---
### Действия ВМ2:
 - Установим Kerberos на сервер:\
 `apt install krb5-kdc krb5-admin-server krb5-config -y`
   - `MAD.NET` - область по умолчанию
   - `vm2kbs.mad.net` - сервера Kerberos в области
   - `vm2kbs.mad.net` - сервер админа Kerberos
 - Добавим новую область:\
 `krb5_newrealm`
   - `secretk` - пароль админа Kerberos
   - `secretk` - подтверждение
 - Выдадим полные права админу в ACL:\
 `sed -i -e 's/# \*\/admin \*/\*\/admin \*/' /etc/krb5kdc/kadm5.acl`
 - Проверим, что статус kerberos сервисов running:\
 `systemctl status krb5-*`
 - Установим теперь openLDAP на сервер:\
 `apt install slapd ldap-utils -y`
   - `secretl` - пароль админа openLDAP
   - `secretl` - подтверждение
 - Теперь настроим SLAPD:\
 `dpkg-reconfigure slapd`
   - `No` - создаем новую директорию
   - `mad.net` - доменное имя
   - `mad` - название организации
   - `secretl` - пароль админа
   - `secretl` - подтверждаем
   - `MDB` - тип ББД
   - `No` - если удалить slapd, оставить БД
   - `Yes` - подтверждение
 - Проверяем статус openLDAP:\
 `systemctl status slapd`
## 2. Добавить 2х пользователей и 2 группы (числовые идентификаторы выбрать больше 10000) в LDAP.
### Дерево LDAP:
  ```
  dc[mad.net]
    |      
    ou[Users]
    | |
    | uid[{10011}ldapuser1:Password1] <-+
    | uid[{10022}ldapuser2:Password2] <-|-+
    |                                   | |
    ou[Groups]                          | |
      |                                 | |
      gid[{10010}office] <--------------+ |
      gid[{10020}direct] <----------------+
  ```
---
### Действия ВМ2:
 - Процедура добавления пользователей точно такая же, как и в 15 ЛР.
 - Создаем файл `nano ounits.ldif` с юнитами `Users` и `Groups`:
    ```ldif
    dn: ou=Groups,dc=mad,dc=net
    objectClass: organizationalUnit
    ou: Groups
    
    dn: ou=Users,dc=mad,dc=net
    objectClass: organizationalUnit
    ou: Users
    ```
 - Создаем файл `nano pgroups.ldif` с группами `office` и `direct`:
    ```ldif
    dn: cn=office,ou=Groups,dc=mad,dc=net
    objectClass: posixGroup
    cn: office
    gidNumber: 10010
    
    dn: cn=direct,ou=Groups,dc=mad,dc=net
    objectClass: posixGroup
    cn: direct
    gidNumber: 10020
    ```
 - Создаем файл `nano pusers.ldif` с юзерами `ldapuser1` и `ldapuser2`:
    ```ldif
    dn: uid=ldapuser1,ou=Users,dc=mad,dc=net
    objectClass: inetOrgPerson
    objectClass: posixAccount
    objectClass: shadowAccount
    uid: ldapuser1
    sn: Users
    cn: Office
    uidNumber: 10011
    gidNumber: 10010
    homeDirectory: /home/ldapuser1
    loginShell: /bin/bash
    userPassword: {crypt}x
    
    dn: uid=ldapuser2,ou=Users,dc=mad,dc=net
    objectClass: inetOrgPerson
    objectClass: posixAccount
    objectClass: shadowAccount
    uid: ldapuser2
    sn: Users
    cn: Direct
    uidNumber: 10022
    gidNumber: 10020
    homeDirectory: /home/ldapuser2
    loginShell: /bin/bash
    userPassword: {crypt}x
    ```
  ВАЖНО: перепроверь каждый файл дважды, перед тем, как вводить его, он полностью должен соответствовать, каждый блок отделен отступом, не должно быть опечаток.\
  Если не уверен, сделай снапшот!
 - Файлы готовы, вводим их поочередно в БД (пароль: `secretl`):\
 `ldapadd -x -D cn=admin,dc=mad,dc=net -W -f ounits.ldif` - юниты\
 `ldapadd -x -D cn=admin,dc=mad,dc=net -W -f pgroups.ldif` - группы\
 `ldapadd -x -D cn=admin,dc=mad,dc=net -W -f pusers.ldif` - пользователи
 - Пароли задавать не нужно, т.к. мы их зададим в Kerberos.
 - Чтобы проверить БД можно ввести:\
 `ldapsearch -x -LLL -b dc=mad,dc=net`
## 3. Настроить ВМ1, ВМ2 и ВМ3 так, чтобы они идентифицировали и аутентифицировали пользователей через LDAP+Kerberos с ВМ2.
### Топология сети:
  ```
  Server:VM2[vm2kbs.mad.net]
        |
        |
  Gateway:vboxnet0[192.168.0.1/24]
    |      |
    | Client:VM1[vm1.mad.net]
    |   
  Client:VM3[vm3.mad.net]
  ```
---
### Действия ВМ2:
 - Добавим принципалы для клиентов (`vm1`,`vm2kbs` `vm3`) с рандомным ключом, нам его не нужно знать:\
 `kadmin.local`
   - Добавляем принципалы:
   `addprinc -randkey host/vm1.mad.net`\
   `addprinc -randkey host/vm2kbs.mad.net`\
   `addprinc -randkey host/vm3.mad.net`
   - Удостоверимся, что все добавилось:\
   `listprincs`
   - Создаем таблицы ключей, их нужно будет отправить клиентам:\
   `ktadd -k vm1.keytab host/vm1.mad.net`\
   `ktadd -k vm2kbs.keytab host/vm2kbs.mad.net`\
   `ktadd -k vm3.keytab host/vm3.mad.net`
   - Выходим из `kadmin.local`: `quit`
 - Теперь таблицы нужно отправить на соответствующие машины:\
 `scp vm1.keytab vm1:/tmp/` -> `root`\
 `scp vm3.keytab vm3:/tmp/` -> `root`
 - Для удобства отправим в папку /tmp и ключ клиента для сервера:\
 `cp vm2kbs.keytab /tmp/`
### Действия для всех машин по очереди:
 - Настроим клиентские машины, установим для начала пакеты клиента Kerberos:\
 `apt install krb5-user libpam-krb5 -y`
   - `MAD.NET` - область по умолчанию
   - `vm2kbs.mad.net` - сервера области
   - `vm2kbs.mad.net` - основной сервер области
 - Установим пакеты клиента openLDAP:\
 `apt install libnss-ldap libpam-ldap nscd ldap-utils -y`
   - `ldap://vm2kbs.mad.net` - сервер LDAP
   - `dc=mad,dc=net` - БД
   - `3` - версия LDAP
   - `cn=admin,dc=mad,dc=net` - админ LDAP
   - `secretl` - его пароль
   - `Yes` - использовать PAM
   - `No` - необязательный логин
   - `cn=admin,dc=mad,dc=net` - админ сервера
   - `secretl` - его пароль
 - Установим соединение с БД:\
 `echo 'URI ldap://vm2kbs.mad.net' >> /etc/ldap/ldap.conf`
 - Добавим в `nsswitch.conf` поиск по LDAP:
   - `echo 'passwd: files ldap' >> /etc/nsswitch.conf`
   - `echo 'shadow: files ldap' >> /etc/nsswitch.conf`
   - `echo 'group: files ldap' >> /etc/nsswitch.conf`
 - Добавим автоматическое создание папки юзера:\
 `echo 'session required pam_mkhomedir.so skel=/etc/skel umask=077' >> /etc/pam.d/common-session`
 - Перезапустим NSCD:\
 `systemctl restart nscd`
 - На этом этапе `getent passwd | grep ldapuser` уже работает, но мы не задали пароли для ldap аккаунтов. Исправим это, используя Kerberos.
---
### Действия ВМ1:
 - Запускаем утилиту управления таблицами ключей:\
 `ktutil`
   - `rkt /tmp/vm1.keytab` - читаем таблицу, полученную с сервера
   - `wkt /etc/krb5.keytab` - записываем ее в файл
   - `list` - проверяем наличие
   - `quit` - выходим
 - `pam-auth-update` -> `Ok` - обновляем PAM
---
### Действия ВМ2:
 - Запускаем утилиту управления таблицами ключей:\
 `ktutil`
   - `rkt /tmp/vm2kbs.keytab` - читаем таблицу
   - `wkt /etc/krb5.keytab` - записываем ее в файл
   - `list` - проверяем наличие
   - `quit` - выходим
 - `pam-auth-update` -> `Ok` - обновляем PAM
---
### Действия ВМ3:
 - Запускаем утилиту управления таблицами ключей:\
 `ktutil`
   - `rkt /tmp/vm3.keytab` - читаем таблицу, полученную с сервера
   - `wkt /etc/krb5.keytab` - записываем ее в файл
   - `list` - проверяем наличие
   - `quit` - выходим
 - `pam-auth-update` -> `Ok` - обновляем PAM
## 4. Проверить работу билетов Kerberos при подключении по ssh -k.
---
### Действия ВМ2:
 - Вход по SSH без пароля невозможен, зададим пароли для пользователей на сервере:\
 `kadmin.local`:
   - `addprinc ldapuser1`
     - `Password1` - пароль
     - `Password1` - подтверждение
   - `addprinc ldapuser2` -
     - `Password2` - пароль
     - `Password2` - подтверждение
 - Теперь вход доступен и тикеты выдаются, проверяем, зайдем по SSH на vm1, пользователь ldapuser2:\
 `ssh -k ldapuser2@vm1` -> `Password2`
 - `klist` - мы получили наш тикет
### Действия ВМ1:
 - Вход по SSH на vm3, пользователь ldapuser1:\
 `ssh -k ldapuser1@vm3` -> `Password1`
 - `klist` - мы получили наш тикет
### Действия ВМ3:
 - Вход по SSH на vm2kbs, пользователь ldapuser1:\
 `ssh -k ldapuser1@vm2kbs` -> `Password1`
 - `klist` - мы получили наш тикет
