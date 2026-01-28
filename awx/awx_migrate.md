# Перенос данных/миграция/резервное копирование AWX

Если у Вас происходит обновление AWX или перенос данных на другой сервер - воспользуйтесь данной статьёй.

Будут перенесены все важные данные (доступы, проекты, инвентарные файлы, jobs и прочее), но не временные (подробности выполнения jobs, синхронизации проектов и под.).


Для переноса оба сервера (старый и новый) должны быть в работе

### Перенос

Первое - установите приложения для миграции
```bash
pip install --upgrade ansible-tower-cli
```
### Считывание данных

Далее, необходимо считать все настройки через API старого AWX в файл. Для этого мы указываем адрес AWX-сервера, логин/пароль и запускаем программу считывания.
```bash
tower-cli config host  http://awx.testdomain.com
tower-cli config username admin
tower-cli config password 'PASS'
tower-cli config verify_ssl false
http_proxy= https_proxy= tower-cli receive --all > assets.json
```
В данном json-файле содержатся все основные данные нашего AWX: пользователи, проекты, инвентарные файлы, задачи и пр.

### Заливка данных

Далее, подключаемся к API нового сервера AWX и запускаем уже разворачивание данных
```bash
tower-cli config host http://newawx.testdomain.com
tower-cli config username admin
tower-cli config password 'PASS2'
http_proxy= https_proxy= tower-cli send assets.json
```

 

 
