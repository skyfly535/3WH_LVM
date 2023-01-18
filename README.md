# Файловые системы и LVM
## Подготовка к выполнению ДЗ

Для уменьшения тома `VolGroup00-LogVol00` (занимает 37.5G) под каталог `/` до 8G необходимо в VM добавить дополнительный диск (временный том) объемом больше чем `VolGroup00-LogVol00`.
Редактируем исходный Vagrantfile, добавляем следующий раздел:
```
:sata5 => {
    :dfile => home + '/VirtualBox VMs/sata5.vdi',
    :size => 40960,
    :port => 5
    }
```
Стартуем с отредактированным Vagrantfile
```
vagrant up
```
Проверяем подмонтированный диск
```
[root@lvm vagrant]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
sdf                       8:80   0   40G  0 disk 
```
## Уменьшить том под `/` до 8G

Подготовка временного тома для `/` раздела: 
```
pvcreate /dev/sdf
  
vgcreate vg_root /dev/sdf 

lvcreate -n lv_root -l +100%FREE /dev/vg_root 

```` 
Создадание файловой системы и монтирование для переноса данных: 
```
df -T /

mkfs.xfs /dev/vg_root/lv_root 

mount /dev/vg_root/lv_root /mnt
```
Устанавливаем пакет для создания копии тома `/`
```
yum install xfsrestore xfsdump
```
Копируем все данные с `/` раздела в `/mnt`
```
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Переконфигурирование grub для старта при с копии каталога `\` 
```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done 

chroot /mnt/ 

grub2-mkconfig -o /boot/grub2/grub.cfg
```
Обновление образа `initrd`
```
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```
Чтобы при загрузке был смонтирован нужный `root` нужно в файле  `/boot/grub2/grub.cfg` заменить `rd.lvm.lv=VolGroup00/LogVol00` на `rd.lvm.lv=vg_root/lv_root`
```
sed -i 's@VolGroup00/LogVol00@vg_root/lv_root@g' /boot/grub2/grub.cfg

exit
```
Дальше в `[root@lvm /]`  перегружаемся 
```
reboot
```
Проверяем результат "перезда"
```
lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   40G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01   253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00   253:2    0 37.5G  0 lvm  
sdb                         8:16   0   10G  0 disk 
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk 
sdf                         8:80   0   40G  0 disk 
└─vg_root-lv_root 253:0            0   40G  0 lvm  /
```
Уменьшаем размер исходного тома `VolGroup00-LogVol00` до 8G и монтируем с новой фаловой системой обратно
```
lvremove /dev/VolGroup00/LogVol00

lvcreate -n LogVol00 -L 8G VolGroup00

mkfs.xfs /dev/VolGroup00/LogVol00

mount /dev/VolGroup00/LogVol00 /mnt
```
Копируем данные обратно
```
xfsdump -J - /dev/vg_root/lv_root | sudo xfsrestore -J - /mnt
```
Производим обратное переконфигурирование grub

```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do sudo mount --bind $i /mnt/$i; done

sudo chroot /mnt/

grub2-mkconfig -o /boot/grub2/grub.cfg

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

exit
```
Далее в `[vagrant@lvm ~]` перегружаемся
```
reboot
```
Смотрим результат - каталог `/` в уменьшеном томе
```
lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   40G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00   253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01   253:1    0  1.5G  0 lvm  [SWAP]
sdb                         8:16   0   10G  0 disk 
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk 
sdf                         8:80   0   40G  0 disk 
└─vg_root-lv_root 253:2            0   40G  0 lvm
```
Удаляем временную VG
```
lvremove /dev/vg_root/lv_root

sudo vgremove vg_root

sudo pvremove /dev/sdf
```

## Выделить том под /home

В `[vagrant@lvm ~]` создаём LV (для нового каталога `/home`), создаем в нем файлову. систему и монтируем
```
lvcreate -n LogVol_HOME_OTUS -L 2G /dev/VolGroup00  

mkfs.xfs /dev/VolGroup00/LogVol_HOME_OTUS

mount /dev/VolGroup00/LogVol_HOME_OTUS /mnt/
```
Переносим данные и удаляем "старый" `/home`
```
cp -aR /home/* /mnt/

rm -rf /home/*

umount /mnt
```
Монтируем созданный каталог с перенесёнными данными, как `/home`
```
mount /dev/VolGroup00/LogVol_HOME_OTUS /home/
```
Результат
```
lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                               8:0    0   40G  0 disk 
├─sda1                            8:1    0    1M  0 part 
├─sda2                            8:2    0    1G  0 part /boot
└─sda3                            8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00         253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01         253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_HOME_OTUS 253:2    0    2G  0 lvm  /home
sdb                               8:16   0   10G  0 disk 
sdc                               8:32   0    2G  0 disk 
sdd                               8:48   0    1G  0 disk 
sde                               8:64   0    1G  0 disk 
sdf                               8:80   0   40G  0 disk
```


Правим fstab для автоматического монтирования `/home`
```
echo "`sudo blkid | grep HOME_OTUS | awk '{print $2}'` /home xfs defaults 0 0" | sudo tee -a /etc/fstab
```

## Выделить том под /var (/var - сделать в mirror)

В `[vagrant@lvm ~]` создаём зеркало на двухблочных устройствах

```
pvcreate /dev/sdd /dev/sde

vgcreate VolGroupVar /dev/sdd /dev/sde

lvcreate -L 950M -m1 -n var VolGroupVar
```
Создаем на нем ФС, перемещаем туда `/var`
```
mkfs.ext4 /dev/VolGroupVar/var

mount /dev/VolGroupVar/var /mnt/

cp -aR /var/* /mnt/
```
Монтируем новый каталог `var` в `/var`

```
umount /mnt

mount /dev/VolGroupVar/var /var
```
Редактируем fstab
```
echo "` blkid | grep "VolGroupVar-var:" | awk '{print $2}'` /var xfs defaults 0 0" | tee -a /etc/fstab
```
Результат
```
lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                               8:0    0   40G  0 disk 
├─sda1                            8:1    0    1M  0 part 
├─sda2                            8:2    0    1G  0 part /boot
└─sda3                            8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00         253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01         253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_HOME_OTUS 253:2    0    2G  0 lvm  /home
sdb                               8:16   0   10G  0 disk 
sdc                               8:32   0    2G  0 disk 
sdd                               8:48   0    1G  0 disk 
├─VolGroupVar-var_rmeta_0       253:3    0    4M  0 lvm  
│ └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
└─VolGroupVar-var_rimage_0      253:4    0  952M  0 lvm  
  └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
sde                               8:64   0    1G  0 disk 
├─VolGroupVar-var_rmeta_1       253:5    0    4M  0 lvm  
│ └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
└─VolGroupVar-var_rimage_1      253:6    0  952M  0 lvm  
  └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
sdf                               8:80   0   40G  0 disk 

```
## Для /home - сделать том для снэпшотов

создаем каталоги для проверки работы снэпшотов
```
mkdir /home/otus_{1..20}

ls -l /home
total 0
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_1
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_10
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_11
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_12
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_13
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_14
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_15
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_16
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_17
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_18
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_19
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_2
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_20
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_3
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_4
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_5
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_6
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_7
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_8
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_9
drwx------. 3 vagrant vagrant 95 Jan 18 12:06 vagrant
```
Создаем LV для снэпшотов

```
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                               8:0    0   40G  0 disk 
├─sda1                            8:1    0    1M  0 part 
├─sda2                            8:2    0    1G  0 part /boot
└─sda3                            8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00         253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01         253:1    0  1.5G  0 lvm  [SWAP]
  ├─VolGroup00-LogVol_Home-real 253:8    0    2G  0 lvm  
  │ ├─VolGroup00-LogVol_Home    253:2    0    2G  0 lvm  /home
  │ └─VolGroup00-home_snap      253:10   0    2G  0 lvm  
  └─VolGroup00-home_snap-cow    253:9    0  128M  0 lvm  
    └─VolGroup00-home_snap      253:10   0    2G  0 lvm  
sdb                               8:16   0   10G  0 disk 
sdc                               8:32   0    2G  0 disk 
sdd                               8:48   0    1G  0 disk 
├─VolGroupVar-var_rmeta_0       253:3    0    4M  0 lvm  
│ └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
└─VolGroupVar-var_rimage_0      253:4    0  952M  0 lvm  
  └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
sde                               8:64   0    1G  0 disk 
├─VolGroupVar-var_rmeta_1       253:5    0    4M  0 lvm  
│ └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
└─VolGroupVar-var_rimage_1      253:6    0  952M  0 lvm  
  └─VolGroupVar-var             253:7    0  952M  0 lvm  /var
sdf                               8:80   0   40G  0 disk 
```

Удалим некоторые каталоги

```
rm -r /home/otus_{15..20}

ls -l /home
total 0
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_1
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_10
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_11
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_12
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_13
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_14
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_2
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_3
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_4
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_5
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_6
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_7
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_8
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_9
drwx------. 3 vagrant vagrant 95 Jan 18 12:06 vagrant
```
Произведем восстановление с снэпшота в `/home`

```
umount /home

lvconvert --merge /dev/VolGroup00/home_snap

lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk 
├─sda1                       8:1    0    1M  0 part 
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  
sdb                          8:16   0   10G  0 disk 
sdc                          8:32   0    2G  0 disk 
sdd                          8:48   0    1G  0 disk 
├─VolGroupVar-var_rmeta_0  253:3    0    4M  0 lvm  
│ └─VolGroupVar-var        253:7    0  952M  0 lvm  /var
└─VolGroupVar-var_rimage_0 253:4    0  952M  0 lvm  
  └─VolGroupVar-var        253:7    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk 
├─VolGroupVar-var_rmeta_1  253:5    0    4M  0 lvm  
│ └─VolGroupVar-var        253:7    0  952M  0 lvm  /var
└─VolGroupVar-var_rimage_1 253:6    0  952M  0 lvm  
  └─VolGroupVar-var        253:7    0  952M  0 lvm  /var
sdf                          8:80   0   40G  0 disk 

mount /home
```
Результат

```
ls -l /home
total 0
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_1
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_10
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_11
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_12
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_13
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_14
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_15
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_16
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_17
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_18
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_19
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_2
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_20
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_3
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_4
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_5
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_6
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_7
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_8
drwxr-xr-x. 2 root    root     6 Jan 18 12:41 otus_9
drwx------. 3 vagrant vagrant 95 Jan 18 12:06 vagrant
```
