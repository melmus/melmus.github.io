# Шифрование паролей Maven
Для большей безопасности хранить пароли в открытом виде в конфигах maven запрещается!!!
Для создания закодированного пароля, необходимо воспользоваться возможностью maven "Password Encryption" доступна с версии 2.1 и выше.
 
1) Создаем мастер пароль
```bash
mvn --encrypt-master-password
# вводим необходимый пароль после чего на выходе получим его в закодированном виде.

{jSMOWnoPFgsHVpMvz5VrIt5kRbzGpI8u+wqeRc1iFQyJQ=}
```
Внимание забывать оригинальный пароль не стоит его нужно сохранить.
 
2) Говорим maven использовать данный мастер пароль .
Для этого выполняем следующие действия: создаем файл с настройкой security обычное расположение 
```bash
vim ${user.home}/.m2/settings-security.xml
# вставляем следующие строки в него 
<settingsSecurity>
  <master>{jSMOWnoPFgsHVpMvz5VrIt5kRbzGpI8u+wqeRc1iFQyJQ=}</master>
</settingsSecurity>
#Сохраняем
{jSMOWnoPFgsHVpMvz5VrIt5kRbzGpI8u+wqeRc1iFQyJQ=} - создали на первом шаге
``` 
3) Создаем закодированный пароль для сервисов
```bash
mvn --encrypt-password
# вводим пароль,который хотим зашифровать.
# На выходе получаем что-то типа 
{COQLCE6DU34XtcS5P=}
```
 
4) Прописываем этот пароль в настройки maven, открываем на запись файл настроек(стандартный путь)
```bash
vim ${user.home}/.m2/settings.xml 
# Добавляем в секцию с сервером наш закодированный пароль
    <server>
      <id>id_server</id>
      <username>test</username>
      <password>{COQLCE6DU34XtcS5P=}</password>
    </server>
где, {COQLCE6DU34XtcS5P=} полученный на 3 шаге пароль
```
Дополнение.
Можно хранить мастер пароль на внешнем накопителе.
Для этого нужно создать файл как в первом шаге, но прописать следущее
<settingsSecurity>
  <relocation>/path/secretplace/maven/settings-security.xml</relocation>
</settingsSecurity>
где /path/secretplace/maven/settings-security.xml - путь до файла с с мастер паролем. выглядет как на шаге один.
