### Настройка NFS сервера

1. Устанавливаем утилиты для отладки NFS
```
[root@nfss ~]# yum install nfs-utils

 Dependencies Resolved

======================================================================================================================
 Package                   Arch                   Version                               Repository               Size
======================================================================================================================
Updating:
 nfs-utils                 x86_64                 1:1.3.0-0.68.el7.2                    updates                 413 k

Transaction Summary
======================================================================================================================
Upgrade  1 Package

Total download size: 413 k
Is this ok [y/d/N]: y
Downloading packages:
No Presto metadata available for updates
warning: /var/cache/yum/x86_64/7/updates/packages/nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm is not installed
nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm                                                          | 413 kB  00:00:00     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Is this ok [y/N]: y
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                1/2 
  Cleanup    : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                  2/2 
  Verifying  : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                1/2 
  Verifying  : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                  2/2 

Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2 
```
2. Включаем firewall
```
[root@nfss ~]# systemctl enable firewalld --now
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
```
```
[root@nfss ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2022-06-07 14:53:34 UTC; 13s ago
     Docs: man:firewalld(1)
 Main PID: 3442 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─3442 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Jun 07 14:53:34 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Jun 07 14:53:34 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
Jun 07 14:53:34 nfss firewalld[3442]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure co... now.
Hint: Some lines were ellipsized, use -l to show in full.
```
3. Разрешаем firewall доступ к сервисам NFS
```
[root@nfss ~]# firewall-cmd --add-service="nfs3" \
> --add-service="rpc-bind" \
> --add-service="mountd" \
> --permanent
success
[root@nfss ~]# firewall-cmd --reload
success
[root@nfss ~]# 
```
4. Включаем сервер NFS
```
[root@nfss ~]# systemctl enable nfs --now
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.
[root@nfss ~]# 
```
5. Проверяем наличие слушаемых портов 2049/udp, 2049/tcp, 20048/udp,
20048/tcp, 111/udp, 111/tcp
```
[root@nfss ~]# ss -tnplu
Netid  State      Recv-Q Send-Q           Local Address:Port                          Peer Address:Port              
udp    UNCONN     0      0                    127.0.0.1:823                                      *:*                   users:(("rpc.statd",pid=3615,fd=5))
udp    UNCONN     0      0                    127.0.0.1:323                                      *:*                   users:(("chronyd",pid=339,fd=5))
udp    UNCONN     0      0                            *:68                                       *:*                   users:(("dhclient",pid=2603,fd=6))
udp    UNCONN     0      0                            *:20048                                    *:*                   users:(("rpc.mountd",pid=3624,fd=7))
udp    UNCONN     0      0                            *:50263                                    *:*                  
udp    UNCONN     0      0                            *:111                                      *:*                   users:(("rpcbind",pid=337,fd=6))
udp    UNCONN     0      0                            *:54397                                    *:*                   users:(("rpc.statd",pid=3615,fd=8))
udp    UNCONN     0      0                            *:931                                      *:*                   users:(("rpcbind",pid=337,fd=7))
udp    UNCONN     0      0                            *:2049                                     *:*                  
udp    UNCONN     0      0                        [::1]:323                                   [::]:*                   users:(("chronyd",pid=339,fd=6))
udp    UNCONN     0      0                         [::]:57930                                 [::]:*                   users:(("rpc.statd",pid=3615,fd=10))
udp    UNCONN     0      0                         [::]:20048                                 [::]:*                   users:(("rpc.mountd",pid=3624,fd=9))
udp    UNCONN     0      0                         [::]:111                                   [::]:*                   users:(("rpcbind",pid=337,fd=9))
udp    UNCONN     0      0                         [::]:931                                   [::]:*                   users:(("rpcbind",pid=337,fd=10))
udp    UNCONN     0      0                         [::]:2049                                  [::]:*                  
udp    UNCONN     0      0                         [::]:56074                                 [::]:*                  
tcp    LISTEN     0      128                          *:111                                      *:*                   users:(("rpcbind",pid=337,fd=8))
tcp    LISTEN     0      128                          *:20048                                    *:*                   users:(("rpc.mountd",pid=3624,fd=8))
tcp    LISTEN     0      64                           *:41238                                    *:*                  
tcp    LISTEN     0      128                          *:22                                       *:*                   users:(("sshd",pid=608,fd=3))
tcp    LISTEN     0      100                  127.0.0.1:25                                       *:*                   users:(("master",pid=828,fd=13))
tcp    LISTEN     0      128                          *:35581                                    *:*                   users:(("rpc.statd",pid=3615,fd=9))
tcp    LISTEN     0      64                           *:2049                                     *:*                  
tcp    LISTEN     0      128                       [::]:111                                   [::]:*                   users:(("rpcbind",pid=337,fd=11))
tcp    LISTEN     0      128                       [::]:20048                                 [::]:*                   users:(("rpc.mountd",pid=3624,fd=10))
tcp    LISTEN     0      128                       [::]:32977                                 [::]:*                   users:(("rpc.statd",pid=3615,fd=11))
tcp    LISTEN     0      128                       [::]:22                                    [::]:*                   users:(("sshd",pid=608,fd=4))
tcp    LISTEN     0      100                      [::1]:25                                    [::]:*                   users:(("master",pid=828,fd=14))
tcp    LISTEN     0      64                        [::]:2049                                  [::]:*                  
tcp    LISTEN     0      64                        [::]:43587                                 [::]:*
```
6. Создаем и настраиваем экспортироваемую директорию
```
[root@nfss ~]# mkdir -p /srv/share/upload
[root@nfss ~]# chown -R nfsnobody: /srv/share
[root@nfss ~]# chmod 0777 /srv/share/upload
```
7. Создаем в файле /etc/exports структуру, позволяющую эскпортировать ранее созданную директорию
```
[root@nfss ~]# vi /etc/fstab 
[root@nfss ~]# cat << EOF > /etc/exports
> /srv/share 192.168.50.11/32(rw,sync,root_squash)
> EOF
[root@nfss ~]# 
```
8. Экспортируем созданную директорию
```
[root@nfss ~]# exportfs -r
```
9. Проверяем экспортированную директорию
```
[root@nfss ~]# exportfs -s
/srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
[root@nfss ~]# 
```
### Настройка NFS клиента
1. Устанавливаем утилиты для отладки NFS
```
Dependencies Resolved

======================================================================================================================
 Package                   Arch                   Version                               Repository               Size
======================================================================================================================
Updating:
 nfs-utils                 x86_64                 1:1.3.0-0.68.el7.2                    updates                 413 k

Transaction Summary
======================================================================================================================
Upgrade  1 Package

Total download size: 413 k
Is this ok [y/d/N]: y
Downloading packages:
No Presto metadata available for updates
warning: /var/cache/yum/x86_64/7/updates/packages/nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm is not installed
nfs-utils-1.3.0-0.68.el7.2.x86_64.rpm                                                          | 413 kB  00:00:00     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Is this ok [y/N]: y
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                1/2 
  Cleanup    : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                  2/2 
  Verifying  : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                1/2 
  Verifying  : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                  2/2 

Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2                                                                                 

Complete!
```
2. Включаем firewall и проверяем, что он работает
```
[root@nfsc ~]# systemctl enable firewalld --now
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
```
```
[root@nfsc ~]# systemctl status firewalld      
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2022-06-07 15:29:58 UTC; 9s ago
     Docs: man:firewalld(1)
 Main PID: 22121 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─22121 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Jun 07 15:29:57 nfsc systemd[1]: Starting firewalld - dynamic firewall daemon...
Jun 07 15:29:58 nfsc systemd[1]: Started firewalld - dynamic firewall daemon.
Jun 07 15:29:58 nfsc firewalld[22121]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure c... now.
Hint: Some lines were ellipsized, use -l to show in full.
[root@nfsc ~]# 
```
3. Добавляем в /etc/fstab строку
```
[root@nfsc ~]# echo "192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
```
4. Выполняем 
```
[root@nfsc ~]# systemctl daemon-reload
[root@nfsc ~]# systemctl restart remote-fs.target
```
5. Проверяем успешность монтирования
```
[root@nfsc ~]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=46,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=45778)
```
### Проверка работоспособности

1. Заходим на сервер
2. Заходим в каталог /srv/share/upload
```
[root@nfss ~]# cd /srv/share/upload
[root@nfss upload]# 
```
3. Создаем тестовый файл
```
[root@nfss upload]# touch check_file
[root@nfss upload]# 
```
4. Заходим на клиент
5. Заходим в каталог /mnt/upload
```
[root@nfsc ~]# cd /mnt/upload
[root@nfsc upload]# 
```
6. Проверяем наличие ранее созданного файла
```
[root@nfsc upload]# ll
total 0
-rw-r--r--. 1 root root 0 Jun  7 15:43 check_file
[root@nfsc upload]# 
```
7. Создаем тестовый файл
```
[root@nfsc upload]# touch client_file
[root@nfsc upload]# 
```
8. Проверяем наличие ранее созданного файла на сервере
```
[root@nfss upload]# ll
total 0
-rw-r--r--. 1 root      root      0 Jun  7 15:43 check_file
-rw-r--r--. 1 nfsnobody nfsnobody 0 Jun  7 15:46 client_file
[root@nfss upload]# 
```
#### Предварительно проверяем клиент
1. Перезагружаем клиент
2. Заходим на клиент
3. Переходим в директорию /mnt/upload
```
[root@nfsc ~]# cd /mnt/upload
[root@nfsc upload]#
```
4. Проверяем наличие ранее созданных файлов
```
[root@nfsc upload]# ll
total 0
-rw-r--r--. 1 root      root      0 Jun  7 15:43 check_file
-rw-r--r--. 1 nfsnobody nfsnobody 0 Jun  7 15:46 client_file
[root@nfsc upload]#
```
#### Проверяем сервер
1. Заходим на сервер в отдельном окне терминала
2. Перезагружаем сервер
3. Заходим на сервер
4. Проверяем наличие файлов в каталоге /srv/share/upload/
```
[root@nfss ~]# cd /srv/share/upload/
[root@nfss upload]# ll
total 0
-rw-r--r--. 1 root      root      0 Jun  7 15:43 check_file
-rw-r--r--. 1 nfsnobody nfsnobody 0 Jun  7 15:46 client_file
[root@nfss upload]#
```
5. Проверяем статус сервера NFS
```
[root@nfss upload]# systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Tue 2022-06-07 15:53:57 UTC; 2min 12s ago
  Process: 824 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
  Process: 799 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 794 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 799 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Jun 07 15:53:56 nfss systemd[1]: Starting NFS server and services...
Jun 07 15:53:57 nfss systemd[1]: Started NFS server and services.
[root@nfss upload]# 
```
6. Проверяе статус firewall
```
[root@nfss upload]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2022-06-07 15:53:53 UTC; 3min 29s ago
     Docs: man:firewalld(1)
 Main PID: 400 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─400 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Jun 07 15:53:53 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Jun 07 15:53:53 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
Jun 07 15:53:54 nfss firewalld[400]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure co...t now.
Hint: Some lines were ellipsized, use -l to show in full.
[root@nfss upload]# 
```
7. Проверяем экспорты
```
[root@nfss upload]# exportfs -s
/srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
[root@nfss upload]#
```
8. Проверяем работу RPC
```
[root@nfss upload]# showmount -a 192.168.50.10
All mount points on 192.168.50.10:
192.168.50.11:/srv/share
[root@nfss upload]# 
```
#### Проверяем клиент

1. Возвращаемся на клиент
2. Перезагружаем клиент
3. Заходим на клиент
4. Проверяем работу RPC
```
[vagrant@nfsc ~]$ showmount -a 192.168.50.10
All mount points on 192.168.50.10:
```
5. Заходим в каталог /mnt/upload
6. Проверяем статус монтирования
```
[vagrant@nfsc upload]$ mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=21,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=10875)
192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)
[vagrant@nfsc upload]$ 
```
7. Проверяем наличие ранее созданных файлов
```
[vagrant@nfsc upload]$ ll
total 0
-rw-r--r--. 1 root      root      0 Jun  7 15:43 check_file
-rw-r--r--. 1 nfsnobody nfsnobody 0 Jun  7 15:46 client_file
[vagrant@nfsc upload]$ 
```
8. Создаем тестовый файл
```
[vagrant@nfsc upload]$ touch final_check
```
9. Проверяем наличие созданного файла на сервере 
```
[root@nfss upload]# ll
total 0
-rw-r--r--. 1 root      root      0 Jun  7 15:43 check_file
-rw-r--r--. 1 nfsnobody nfsnobody 0 Jun  7 15:46 client_file
-rw-rw-r--. 1 vagrant   vagrant   0 Jun  7 16:04 final_check
[root@nfss upload]# 
```
