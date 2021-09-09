# otus_linux_3
В данной работе будут выполнены следующие пункты:
На имеющемся образе centos/7 - v. 1804.2
1)Уменьшить том под / до 8G
2)Выделить том под /home
3)Выделить том под /var - сделать в mirror
4)/home - сделать том для снапшотов
5)Прописать монтирование в fstab. Попробовать с разными опциями и разными
файловыми системами ( на выбор)

Будет произведена следующая Работа со снапшотами:
- сгенерить файлы в /home/
- снять снапшот
- удалить часть файлов
- восстановится со снапшота
- залоггировать работу можно с помощью утилиты script

Для выполнения разворачиваем Vagrantfile и подключаемся к машине lvm:
```
vagrant up
vagrant ssh
```
На ВМ устанавливаем утилиты, которые будем использовать в дальнейшем:
```
sudo yum install xfsdump nano
```

С помощью следующей команды выведем список дисков и их разделов сразу после старта ВМ:
```
lsblk
```
Подготовка временного тома для / раздела:
```
pvcreate /dev/sdb
vgcreate vg_root /dev/sdb
lvcreate -n lv_root -l +100%FREE /dev/vg_root
```
Создание файловой системы и монтирование, чтобы перенести туда данные (Выводы команд на скриншоте №1):

```
mkfs.xfs /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/1.png)

Следующей командой сдампим все данные с / раздела в /mnt:
```
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Убедимся в успешности копирования:
```
ls /mnt
```
Выполним переконфигурирвоание grub для того, чтобы при старте перейти в новый /
Сделаем имитацию текущего root, для этого сделаем в него chroot и обновим grub:
```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
```
Обновим образ initrd:
```
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/2.png)

Чтобы при загрузке был смонтирован нужный root: в файле /boot/grub2/grub.cfg заменим rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root(Скриншот №3). 
```
nano /etc/default/grub
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/3.png)
После изменений производим обновление grub(Скриншот №4):
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/4.png)

Выполним перезагрузку системы. После подключения vagrant ssh проверим, что новый root том успешно загружен(Скриншот №5):
```
lsblk
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/5.png)

Теперь изменим размер старой VG и вернём на него root. Для этого удалим старый LV размером 40G и создим новый на 8G:
```
lvremove /dev/VolGroup00/LogVol00
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
```
Создим файловую систему:
```
mkfs.xfs /dev/VolGroup00/LogVol00
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/6.png)
Смонтируем файловую систему и скопируем данные:
```
mount /dev/VolGroup00/LogVol00 /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/7.png)

Выполним переконфигурацию grub, за исключением правки /etc/grub2/grub.cfg
```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/8.png)
Отложим перезагрузку и выход из под chroot до момента перенеса /var.
Сделаем зеркало из свободных дисков:
```
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
```
Создадим на нем файловую систему и переместим туда /var:
```
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/      # rsync -avHPSAX /var/ /mnt/
```
Cохраним содержимое старого var:
```
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```
Смонтируем новый var в каталог /var:
```
umount /mnt
mount /dev/vg_var/lv_var /var
```
Исправим fstab для автоматического монтирования /var:
```
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
Внесём правки в grub, заменив rd.lvm.lv=vg_root/lv_root на rd.lvm.lv=VolGroup00/LogVol00:
```
grub2-mkconfig -o /boot/grub2/grub.cfg
vim /etc/default/grub
```
После изменений обновим grub:
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/91.png)
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/92.png)

Перезагрузимся в новый (уменьшенный root) и удалим временную Volume Group:
```
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb
```
Выделим том под /home по тому же принципу как для /var:
```
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00 
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
```
Исправим fstab для автоматического монтирования /home:
```
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
Сгенерируем файлы в /home/:
```
touch /home/file{1..20}
```
Сделаем снапшот:
```
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
```
Удалить часть файлов:
```
rm -f /home/file{11..20}
```
Процесс восстановления со снапшота:
```umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home
```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/11.png)

После выхода и перезагрузки проверим, что все действия выполнены корректно

 ```
 lsblk
 ```
![Image alt](https://github.com/danoque/otus_linux_3/raw/main/12.png)
