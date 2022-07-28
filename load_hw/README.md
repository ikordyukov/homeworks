# Домашнее задание № 7
## Попасть в систему без пароля несколькими способами
### Способ №1. init=/bin/sh

1. В строку, начинающуюся с linux добавляем init=/bin/sh, затем нажимаем ctrl+x.
Отдельно стоит обратить внимание на то, что таким способом система монтируется в режиме Read-Only.

2. Для перемонтирования ее в режим Read-Write воспользуемся командой:
```
mount -o remount,rw /
```

3. Убедимся, что перемонтирование прошло успешно
```
sh-4.2# mount | grep root
/dev/mapper/centos-root on / type xfs (rw,relatime,attr2,inode64,noquota)
```
### Способ № 2. rd.break
1. В конце строки начинающейся с linux16 добавляем rd.break и нажимаем сtrl-x для 
загрузки в систему
2. Попадаем в emergency mode. Корневая файловая система смонтирована в режиме Read-Only, к тому же мы не в ней.
3. Производим перемонтирование
```
mount -o remount,rw /sysroot
```
4. Изменяем корневой каталог на /sysroot
```
chroot /sysroot
```
5. Устанавливаем пароль root
```
sh-4.2# passwd root
Changing password for user root.
New password:
Retype new password:
passwd: all authentication tokens updated successfully
```
6. Создаем скрытый файл .autorelabel
```
touch /.autorelabel
```
7. После перезагрузки можно заходить в систему под новым паролем.

### Способ № 3. rw init=/sysroot/bin/sh
1. В строке начинающейся с linux16 заменяем ro на rw init=/sysroot/bin/sh и нажимаем сtrl-x 
для загрузки в систему.
2. При входе в систему данным способом, файловая система смонтирована сразу в режиме Read-Write.
## Установить систему с LVM, после чего переименовать VG
1. Проверяем состояние системы 
```
[root@localhost ~]# vgs
 VG      #PV #LV #SN Attr    VSize   VFree
 centos    1   2   0 wz--n-  <7.00g     0
```
2. Переименуем группу томов centos
```
[root@localhost ~]# vgrename centos OtusRoot
 Volume group "centos" successfully renamed to "OtusRoot"
```
3. Далее правим /etc/fstab, /etc/default/grub, /boot/grub2/grub.cfg. Везде заменяем старое 
название на новое.
4. Пересоздаем initrd image, чтобы он знал новое название группы томов.
```
[root@localhost ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-1160.el7.x86_64.img' done ***
```
5. Перезагружаемся и проверяем, применились ли изменения
```
 VG        #PV #LV #SN Attr    VSize   VFree
 OtusRoot    1   2   0 wz--n-  <7.00g     0
```
### Добавить модуль initrd
1. Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того чтобы 
добавить свой модуль создаем там папку с именем 01test
```
[root@localhost ~]# mkdir /usr/lib/dracut/modules.d/01test
```
2. В нее поместим два скрипта:
 - module-setup.sh - который устанавливает модуль и вызывает скрипт test.sh
 - test.sh - собственно сам вызываемый скрипт, в нём у нас рисуется пингвинчик
3. Пересобираем образ initrd
```
[root@localhost ~]# dracut -f -v
```
4. Проверяем, какие модули загружены в образ
```
[root@localhost ~]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test
```
5. Перезагружаемся и убираем опции rghb и quiet. В результате при загрузке увидим пингвина.
