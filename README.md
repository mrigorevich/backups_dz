# Настраиваем бэкапы<br><br>
Описание домашнего задания<br><br>
1.	Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client.<br>
2.	Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:<br>
•	директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;<br>
•	репозиторий для резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;<br>
•	имя бекапа должно содержать информацию о времени снятия бекапа;<br>
•	глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех.
Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;<br>
•	резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;<br>
•	написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;<br>
•	настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.<br>
3.	Запустите стенд на 30 минут.<br>
4.	Убедитесь что резервные копии снимаются.<br>
5.	Остановите бэкап, удалите (или переместите) директорию /etc и восстановите ее из бэкапа.<br>
_____________________________________________________________________________
<br><br>
1.	Машины поднимаем при помощи [Vagrantfile](https://github.com/mrigorevich/backups_dz/blob/main/Vagrantfile). При этом на ВМ backup-server добавляем диск для бэкапов с монтированием в каталог /var/backup, на обеих машинах синхронизируем время и устанавливаем borgbackup<br><br>
2.	На ВМ backup-server:<br><br>
`adduser borg` # добавляем пользователя, права, ключи <br>
`chown borg:borg /var/backup/`<br>
`su borg`<br>
`cd ~`<br>
`mkdir .ssh`<br>
`chmod 700 .ssh/`<br>
`touch .ssh/authorized_keys`<br>
`chmod 600 .ssh/authorized_keys`<br><br>
На ВМ clent:<br><br>
`ssh-keygen`<br><br>
 <img width="974" height="556" alt="image" src="https://github.com/user-attachments/assets/324e3b14-1eda-476d-88a0-b2c44e74d187" /><br><br>
`ssh-copy-id -i ~/.ssh/id_rsa.pub borg@192.168.11.150` # копируем ключ на backup-server<br>
`nano ~/.ssh/config` # добавляем ключ в конфиг ssh для авторизации<br><br>
<img width="627" height="203" alt="image" src="https://github.com/user-attachments/assets/45896956-5828-49d5-a398-41200d3c2e4b" /><br><br>
`borg init --encryption=repokey borg@192.168.11.150:/var/backup/` # инициализируем borg-репозиторий<br>
`borg create --stats --list borg@192.168.11.150:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc` # проверяем создание бэкапа<br><br>
<img width="974" height="83" alt="image" src="https://github.com/user-attachments/assets/6f5d86df-e0b0-4d10-a22f-7862315e5a4f" /><br><br>

`sudo cp /home/vagrant/.ssh/id_rsa ~/.ssh/ && cp /home/vagrant/.ssh/config ~/.ssh` # Для автоматизации скопируем созданный ключ и конфиг в домашний каталог root<br>
`sudo nano /etc/systemd/system/borg-backup.service` # создадим сервис<br><br>
<img width="974" height="718" alt="image" src="https://github.com/user-attachments/assets/c1ac1cb9-3827-4ba2-aba8-884c74ea3d40" /><br><br>
`sudo nano /etc/systemd/system/borg-backup.timer` # и таймер <br><br>
<img width="613" height="411" alt="image" src="https://github.com/user-attachments/assets/865011b3-6fcc-4e24-8b52-ac0133c4d38b" /><br><br>
Включим службу и таймер.
Проверяем:<br><br>
<img width="974" height="133" alt="image" src="https://github.com/user-attachments/assets/b7c30e2b-5cd2-4704-9d7f-c28cee490f81" /><br><br>
Настраиваем логирование<br><br>
В файл /etc/systemd/system/borg-backup.service добавим строку `SyslogIdentifier=borg-backup`<br>
`sudo nano /etc/rsyslog.d/70-borg-backup.conf` # создадим конфиг для rsyslog<br><br>
<img width="974" height="135" alt="image" src="https://github.com/user-attachments/assets/319d3003-c27c-461d-83c8-99d593bce1ad" /><br><br>
`sudo nano /etc/logrotate.d/borg-backup` # настроим ротацию логов<br><br>
<img width="974" height="208" alt="image" src="https://github.com/user-attachments/assets/85b20282-d399-4c1d-a9b0-35e4a82634aa" /><br><br>
`sudo systemctl restart rsyslog` # перезапускаем syslog и ждем ~полчаса.<br><br>

Смотрим файл [/var/log/borg-backup.log](https://github.com/mrigorevich/backups_dz/blob/main/borg-backup.log)<br><br>
`sudo systemctl stop borg-backup.timer` # останавливаем таймер бэкапов<br>
`sudo mv /etc /etcbroken` # ломаем каталог /etc<br><br>
<img width="948" height="81" alt="image" src="https://github.com/user-attachments/assets/219b11ea-6679-4d6b-986a-98ae89b9745e" /><br><br>
Выключаем ВМ client для манипуляций с диском.<br><br>
На ВМ backup-server:<br>
Цепляем в гипервизоре диск ВМ client к ВМ backup-server<br><br>
<img width="974" height="369" alt="image" src="https://github.com/user-attachments/assets/f45220dc-7ecb-4631-b7fd-52e34bdc7703" /><br><br>
`mkdir /mnt/client && mount /dev/sdd1 /mnt/client` # монтируем<br><br>
<img width="974" height="77" alt="image" src="https://github.com/user-attachments/assets/5158cda2-acc1-4b28-89d5-229de428d609" /><br><br>
`borg list /var/backup` # смотрим список бэкапов<br><br>
<img width="974" height="95" alt="image" src="https://github.com/user-attachments/assets/545fee16-b2b7-49d7-96e5-5c1e68f155bc" /><br><br>
`cd /mnt/client`<br>
`borg extract /var/backup::etc-2026-10-11_15:27:47 etc` # извлекаем каталог etc из бэкапа на прежнее место<br><br>
<img width="974" height="263" alt="image" src="https://github.com/user-attachments/assets/44005dde-cd0d-4b9a-84e5-1122de022191" /><br><br>
Отмонтируем диск от backup-server и запускаем client<br><br>
<img width="974" height="178" alt="image" src="https://github.com/user-attachments/assets/3e6d6c32-2d97-4172-8799-0d8e47b4ad73" /><br><br>
Работоспособность ВМ client восстановлена из бэкапа.
