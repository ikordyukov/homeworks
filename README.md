### Определение алгоритма с наилучшим сжатием 

1. Проверим наличие дисков, которые есть на виртуальной машине
```
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk
```
2. Создадим 4 пула по 2 диска в режиме RAID 1
```
[root@zfs ~]# zpool create otus1 mirror /dev/sdb /dev/sdc
[root@zfs ~]# zpool create otus2 mirror /dev/sdd /dev/sde
[root@zfs ~]# zpool create otus3 mirror /dev/sdf /dev/sdg
[root@zfs ~]# zpool create otus4 mirror /dev/sdh /dev/sdi
```
3. Вывод информации о пулах
```
[root@zfs ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M   106K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
4. Добавим разные алгоритмы сжатия в каждую ФС
```
[root@zfs ~]# zfs set compression=lzjb otus1
[root@zfs ~]# zfs set compression=lz4 otus2
[root@zfs ~]# zfs set compression=gzip-9 otus3
[root@zfs ~]# zfs set compression=zle otus4
```
5. Убеждаемся, что установлены разные алгоритмы сжатия
```
[root@zfs ~]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
6. Скачиваем тестовый файл
```
[root@zfs ~]# for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
--2022-06-06 17:47:40--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40817341 (39M) [text/plain]
Saving to: '/otus1/pg2600.converter.log'

100%[============================================================================>] 40,817,341  1.07MB/s   in 91s    

2022-06-06 17:49:13 (436 KB/s) - '/otus1/pg2600.converter.log' saved [40817341/40817341]

--2022-06-06 17:49:13--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40817341 (39M) [text/plain]
Saving to: '/otus2/pg2600.converter.log'
...
```
7. Убеждаемся, что файл скачался во все пулы 
```
[root@zfs ~]# ls -l /otus*
/otus1:
total 22022
-rw-r--r--. 1 root root 40817341 Jun  2 08:39 pg2600.converter.log

/otus2:
total 17974
-rw-r--r--. 1 root root 40817341 Jun  2 08:39 pg2600.converter.log

/otus3:
total 10949
-rw-r--r--. 1 root root 40817341 Jun  2 08:39 pg2600.converter.log

/otus4:
total 39889
-rw-r--r--. 1 root root 40817341 Jun  2 08:39 pg2600.converter.log
```
8. Проверяем степень сжатия и сколько места занимает один и тот же файл в разных пулах
```
[root@zfs ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.6M   330M     21.5M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.1M   313M     39.0M  /otus4

[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                  -
otus2  compressratio         2.22x                  -
otus3  compressratio         3.64x                  -
otus4  compressratio         1.00x                  -
```
9. Таким образом, видим, что наиболее эффективный по сжатию алгоритм - gzip-9.

### Определение настроек пула
1. Скачиваем архив в домашний каталог
```
[root@zfs ~]# wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
--2022-06-06 18:05:49--  https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
Resolving drive.google.com (drive.google.com)... 142.250.181.238, 2a00:1450:4001:82f::200e
Connecting to drive.google.com (drive.google.com)|142.250.181.238|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download [following]
--2022-06-06 18:05:49--  https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
Reusing existing connection to drive.google.com:443.
HTTP request sent, awaiting response... 303 See Other
Location: https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/kr36dta6uv22p66tfg9aahop8t9tsbja/1654538700000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download [following]
Warning: wildcards not supported in HTTP.
--2022-06-06 18:05:55--  https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/kr36dta6uv22p66tfg9aahop8t9tsbja/1654538700000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download
Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 142.250.185.65, 2a00:1450:4001:80e::2001
Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|142.250.185.65|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/x-gzip]
Saving to: 'archive.tar.gz'

100%[============================================================================>] 7,275,140   6.58MB/s   in 1.1s   

2022-06-06 18:05:57 (6.58 MB/s) - 'archive.tar.gz' saved [7275140/7275140]
```
2. Разорхивируем его
```
[root@zfs ~]# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```
3. Проверяем, возможно ли импротировать данный каталог в пул
```
[root@zfs ~]# zpool import -d zpoolexport
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
```
Видим имя пула, его состав и тип рейда
4. Импортируем пул в свою ОС
```
[root@zfs ~]# zpool import -d zpoolexport/ otus
[root@zfs ~]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
```
Определяем настройки
5. Запросить сразу все параметры пула и файловой системы можно командами zpool get all otus и zfs get all otus.
Но мы выведем только определенные параметры
```
[root@zfs ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -

[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
```

### Работа со снапшотом, поиск сообщения от преподавателя

1. Скачиваем файл, указанный в задании
```
[root@zfs ~]# wget -O otus_task2.file --no-check-certificate 'https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download'
--2022-06-06 18:19:36--  https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
Resolving drive.google.com (drive.google.com)... 142.250.181.238, 2a00:1450:4001:82f::200e
Connecting to drive.google.com (drive.google.com)|142.250.181.238|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download [following]
--2022-06-06 18:19:37--  https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
Reusing existing connection to drive.google.com:443.
HTTP request sent, awaiting response... 303 See Other
Location: https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/43nu4rnfgo4escvs72gtimel6f9642m3/1654539525000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download [following]
Warning: wildcards not supported in HTTP.
--2022-06-06 18:19:40--  https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/43nu4rnfgo4escvs72gtimel6f9642m3/1654539525000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download
Resolving doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)... 142.250.185.65, 2a00:1450:4001:80e::2001
Connecting to doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)|142.250.185.65|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5432736 (5.2M) [application/octet-stream]
Saving to: 'otus_task2.file'

100%[============================================================================>] 5,432,736   6.14MB/s   in 0.8s   

2022-06-06 18:19:42 (6.14 MB/s) - 'otus_task2.file' saved [5432736/5432736]
```
2. Восстановим файловую систему из снапшота командой: zfs receive otus/test@today < otus_task2.file

3. Ищем в каталоге /tus/test файл с именем "secret_message"
```
[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
```
4. Смотрим содержимое найденного файла
```
[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome
```
