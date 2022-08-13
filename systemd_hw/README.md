# Домашняя работа №8

### Задание 1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig

1. Создаём конфигурационный файл в /etc/sysconfig
```
root@ivan-VirtualBox:~# cat /etc/sysconfig/watchlog
# Configuration file for my watchdog service
# Place it to /etc/sysconfig
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```

2. Создаем /var/log/watchlog.log с ключевым словом "alert" внутри.
```
root@ivan-VirtualBox:~# cat /var/log/watchlog.log
Test alert
```

3. Создаем скрипт /opt/watchlog.sh
```
root@ivan-VirtualBox:~# cat /opt/watchlog.sh
#!/bin/bash
WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
```

4. Создаём юнит для сервиса
```
root@ivan-VirtualBox:~# cat /etc/systemd/system/watchlog.service 
[Unit]
Description=My watchlog service
[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchdog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

5. Создаём юнит для таймера 
```
root@ivan-VirtualBox:/var/log# cat /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second
[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service
[Install]
WantedBy=multi-user.target
```

6. Стартуем таймер и убеждаемся в работоспособности.
```
root@ivan-VirtualBox:~# systemctl start watchlog.timer
root@ivan-VirtualBox:~# tail -f /var/log/syslog
Aug 13 22:41:40 ivan-VirtualBox systemd[1]: Starting My watchlog service...
Aug 13 22:41:40 ivan-VirtualBox root: Sat 13 Aug 2022 10:41:40 PM MSK: I found word, Master!
Aug 13 22:41:40 ivan-VirtualBox systemd[1]: watchlog.service: Succeeded.
Aug 13 22:41:40 ivan-VirtualBox systemd[1]: Finished My watchlog service.
```

### Задание №2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Сервис должен называться так же.

1. Устанавливаем spawn-fcgi и необходимые для него пакеты 
```
yum install epel-release -y && yum install spawn-fcgi php php-cli
mod_fcgid httpd -y
```
2. Раскомментируем строки в /etc/sysconfig/spawn-fcgi
```
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
```
3. Создадим юнит файл
```
[root@localhost ~]# cat /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target
[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process
[Install]
WantedBy=multi-user.target
```
4. Убеждаемся в работоспособности
```
[root@localhost ~]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-08-14 00:44:35 MSK; 9s ago
 Main PID: 3103 (php-cgi)
    Tasks: 33
   CGroup: /system.slice/spawn-fcgi.service
           ├─3103 /usr/bin/php-cgi
           ├─3110 /usr/bin/php-cgi
           ├─3111 /usr/bin/php-cgi
           ├─3112 /usr/bin/php-cgi
           ├─3113 /usr/bin/php-cgi
           ├─3114 /usr/bin/php-cgi
           ├─3115 /usr/bin/php-cgi
           ├─3116 /usr/bin/php-cgi
           ├─3117 /usr/bin/php-cgi
           ├─3118 /usr/bin/php-cgi
           ├─3119 /usr/bin/php-cgi
           ├─3120 /usr/bin/php-cgi
           ├─3121 /usr/bin/php-cgi
           ├─3122 /usr/bin/php-cgi
           ├─3123 /usr/bin/php-cgi
           ├─3124 /usr/bin/php-cgi
           ├─3125 /usr/bin/php-cgi
           ├─3126 /usr/bin/php-cgi
           ├─3127 /usr/bin/php-cgi
           ├─3128 /usr/bin/php-cgi
           ├─3129 /usr/bin/php-cgi
           ├─3130 /usr/bin/php-cgi
           ├─3131 /usr/bin/php-cgi
           ├─3132 /usr/bin/php-cgi
           ├─3133 /usr/bin/php-cgi
           ├─3134 /usr/bin/php-cgi
           ├─3135 /usr/bin/php-cgi
           ├─3136 /usr/bin/php-cgi
           ├─3137 /usr/bin/php-cgi
           ├─3138 /usr/bin/php-cgi
           ├─3139 /usr/bin/php-cgi
           ├─3140 /usr/bin/php-cgi
           └─3141 /usr/bin/php-cgi
```

### Задание 3. Дополнить Юнит-файл apache httpd возможностью запустить несколько
инстансов сервера с разными конфигами

1. Правим юнит-файл для сервиса httpd
```
[root@localhost ~]# cat /etc/systemd/system/httpd.service 
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I #ДОБАВЛЯЕМ ПАРАМЕТР -%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
2. Создаем 2 файла конфигов окружения
```
[root@localhost ~]# cat /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf
[root@localhost ~]# cat /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf
```
3. Создаём 2 файла first.conf и second.conf в /etc/httpd/conf. Для удачного запуска, в конфигурационных файлах должны быть указаны
уникальные для каждого экземпляра опции Listen и PidFile.
```
Listen 80
PidFile /var/run/httpd-first.pid

Listen 8080
PidFile /var/run/httpd-second.pid
```
4. Проверяем работоспособность
```
[root@localhost ~]# systemctl start httpd@first
[root@localhost ~]# systemctl start httpd@second
[root@localhost ~]# systemctl status httpd@second
● httpd@second.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-08-14 02:30:15 MSK; 7s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 7039 (httpd)
   Status: "Processing requests..."
   CGroup: /system.slice/system-httpd.slice/httpd@second.service
           ├─7039 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─7042 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─7043 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─7044 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─7045 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─7046 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           └─7047 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND

Aug 14 02:30:15 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
[root@localhost ~]# systemctl status httpd@first
● httpd@first.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-08-14 02:30:10 MSK; 17s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 7018 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@first.service
           ├─7018 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─7022 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─7023 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─7024 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─7025 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─7026 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           └─7027 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND

Aug 14 02:30:10 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
```
