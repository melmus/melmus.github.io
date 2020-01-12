# Уменьшение LVM раздела

Порядок действий.

В облаке Vmware делаем обязательно Snapshot

Грузимся с livecd и заходим terminal

Выполняем проверку разделов, которые планируем уменьшить и увеличить, в данном примере мы уменьшаем раздел /var и увеличиваем раздел /root
```bash
e2fsck -f /dev/INTVG/var
e2fsck -f /dev/INTVG/root
```

Уменьшаем раздел /var
```bash 
lvresize --resizefs --size -190G /dev/INTVG/var
```

Увеличиваем раздел /root
```bash 
lvresize --resizefs --size +190G /dev/INTVG/root
```
Перезагружаем сервер. После перезапуска проверяем, что диски размечены правильно.
