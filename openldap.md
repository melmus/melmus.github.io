# OpenLDAP on Oracle Linux


Устанавливаем необходимые пакеты для настройки OpenLDAP
```bash
yum install openldap openldap-servers openldap-clients nss-pam-ldapd
```
Меняем владельца в /var/lib/ldap:
```bash
cd /var/lib/ldap 
chown ldap:ldap ./*
```
Создаем зашифрованный пароль для администратора LDAP
```bash
slappasswd -h {SSHA} 
New password: password 
Re-enter new password:
password {SSHA}lkMShz73MZBic19Q4pfOaXNxpLN3wLRy
```
Создаем конфигурационный файл для LDAP

Основной конфигурационный файл, который как правило располагается в /etc/openldap/slap.d/mydomain.ldif
```bash
include file:///etc/openldap/schema/cosine.ldif
include file:///etc/openldap/schema/nis.ldif
include file:///etc/openldap/schema/inetorgperson.ldif

dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulepath: /usr/lib64/openldap
olcModuleload: back_hdb


# Configure the database settings
dn: olcDatabase=hdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcHdbConfig
olcDatabase: {1}hdb
olcSuffix: dc=mydomain,dc=com
olcDbDirectory: /var/lib/ldap
olcRootDN: cn=admin,dc=mydomain,dc=com

olcRootPW: {SSHA}lkMShz73MZBic19Q4pfOaXNxpLN3wLRy

olcDbConfig: set_cachesize 0 10485760 0
olcDbConfig: set_lk_max_objects 2000
olcDbConfig: set_lk_max_locks 2000
olcDbConfig: set_lk_max_lockers 2000
olcDbIndex: objectClass eq
olcLastMod: TRUE
olcDbCheckpoint: 1024 10

# Set up access control

olcAccess: to attrs=userPassword
  by dn="cn=admin,dc=mydomain,dc=com"
  write by anonymous auth
  by self write
  by * none

olcAccess: to attrs=shadowLastChange
  by self write
  by * read

olcAccess: to dn.base=""
  by * read

olcAccess: to *
  by dn="cn=admin,dc=mydomain,dc=com"
  write by * read
```
Добавляем конфигурацию:
```bash
# ldapadd -Y EXTERNAL -H ldapi:/// -f mydomain.ldif 
SASL/EXTERNAL authentication started 
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth 
SASL SSF: 0 adding new entry "cn=module,cn=config" adding new entry "olcDatabase=hdb,cn=config"
```
Конфигурация об организации

Создаем конфигурацию об организации:
```bash
dn: dc=mydomain,dc=com
dc: mydom
objectclass: dcObject
objectclass: organizationalUnit
ou: mydomain.com

# Users
dn: ou=People,dc=mydomain,dc=com
objectClass: organizationalUnit
ou: people

# Groups
dn: ou=Groups,dc=mydomain,dc=com
objectClass: organizationalUnit
ou: groups
```
Добавляем конфигурацию об организации:
```bash
ldapadd -cxWD "cn=admin,dc=mydomain,dc=com" -f mydomaincom.ldif
Enter LDAP Password: admin_password 
adding new entry "dc=mydomain,dc=com"
adding new entry "ou=People,dc=mydomain,dc=com"
adding new entry "ou=Groups,dc=mydomain,dc=com"
```
## Создание групп

Создаем файл с настройкой для группы devops:
```bash
# Groups
dn: cn=employees,ou=Groups,dc=mydomain,dc=com
cn: devops
gidNumber: 626
objectClass: top
objectclass: posixGroup
```
Применяем файл с настройкой группы:
```bash
ldapadd -cxWD "cn=admin,dc=mydomain,dc=com" -f devops-group.ldif
Enter LDAP Password: admin_password
adding new entry "cn=devops,ou=Groups,dc=mydomain,dc=com"
```
Проверяем наличие группы по gid = 626
```bash
ldapsearch -LLL -x -b "dc=mydomain,dc=com" gidNumber=626
dn: cn=devops,ou=Groups,dc=mydomain,dc=com
cn: devops
gidNumber: 626
objectClass: top
objectClass: posixGroup
```
Создание пользователя в LDAP

Создание пользователя на сервере LDAP:
```bash
useradd -b /nethome -s /sbin/nologin -u 5159 -U user1

id user1 uid=5159(user1) gid=5159(user1) groups=5159(user1)
```
Создаём файл с настройкой для user1:
```bash
# UPG user1 
dn: cn=user1,ou=Groups,dc=mydomain,dc=com
cn: user1
gidNumber: 5159
objectclass: top
objectclass: posixGroup
# User user1 
dn: uid=suer1,ou=People,dc=mydomain,dc=com
cn: User 1
givenName: User
sn: 1
uid: user1
uidNumber: 5159
gidNumber: 5159
homeDirectory: /home/user1
loginShell: /bin/bash
mail: user1@mydomain.com
objectClass: top
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
userPassword: {SSHA}x
```
Применяем:
```bash
ldapadd -cxWD cn=admin,dc=mydomain,dc=com -f user1-user.ldif
Enter LDAP Password: admin_password
adding new entry "cn=user1,ou=Groups,dc=mydomain,dc=com"
adding new entry "uid=user1,ou=People,dc=mydomain,dc=com"
```
Создание и смена пароля пользователю

Задаем пароль, этой же командой его можно сменить:
```bash
ldappasswd -xWD "cn=admin,dc=mydomain,dc=com" -S "uid=user1,ou=people,dc=mydomain,dc=com"
New password: user_password
Re-enter new password: user_password
Enter LDAP Password: admin_password
```
Проверяем наличие пользователя:
```bash
ldapsearch -LLL -x -b "dc=mydomain,dc=com" '(|(uid=iser1)(cn=user1))' 
```
Добавление пользователя в группу
```bash
dn: cn=company,ou=Groups,dc=mydomain,dc=com
changetype: modify
add: memberUid
memberUid: user1

dn: cn=company,ou=Groups,dc=mydomain,dc=com
changetype: modify
add: memberUid
memberUid: user2
```
Применяем конфиг добавления пользователя в группу:
```bash
ldapmodify -xcWD "cn=admin,dc=mydomain,dc=com" -f add-users.ldif
Enter New Password:
Enter LDAP Password: user_password modifying entry "cn=ngenie,ou=Groups,dc=mydomain,dc=com" 
```
Проверяем состав пользователей в группе devops
```bash
ldapsearch -LLL -x -b "dc=mydomain,dc=com" gidNumber=626
dn: cn=devops,ou=Groups,dc=mydomain,dc=com
cn: ngenie gidNumber: 626
objectClass: top
objectClass: posixGroup
memberUid: user1
memberUid: user2
```

Редактирование пользователя

Например, мы хотим изменить поле "mail" пользователя. Подготавливаем LDIF-файл:
```bash
dn: cn=user1,ou=Groups,dc=mydomain,dc=com
changetype: modify
replace: mail
mail: user1@mydomain.com
```
Применяем файл:
```bash
ldapmodify -xWD "cn=admin,dc=mydomain,dc=com" -f user1_change_mail.ldif
```
