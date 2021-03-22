# Домашнее задание. Загрузка системы.

##### 1. Попасть в систему без пароля несколькими способами
##### 2. Установить систему с LVM, после чего переименовать VG
##### 3. Добавить модуль в initrd
##### 4(*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM. Репозиторий с пропатченым grub: https://yum.rumyantsev.com/centos/7/x86_64/. PV необходимо инициализировать с параметром --bootloaderareasize 1m

#### 1. Попасть в систему без пароля несколькими способами:

Способ №1:
- При загрузке машины нажать F12 для выбора загрузочного носителя
- На этапе выбора ядра нажать "e"
- В конце строки начинающейся с linux16 добавить init=/bin/sh и нажать сtrl-x для загрузки системы
- Перемонтировать файловую систему в rw командой: mount -o remount,rw /

Способ №2:
- При загрузке машины нажать F12 для выбора загрузочного носителя
- На этапе выбора ядра нажать "e"
- В конце строки начинающейся с linux16 добавить rd.break и нажать сtrl-x для загрузки системы
- Перемонтировать файловую систему в rw командой: mount -o remount,rw /sysroot 
- Далее, выполнить chroot в смонтированную директорию: chroot /sysroot
- Сменить пароль: passwd root
- После смены пароля необходимо создать скрытый файл .autorelabel в /, выполнив touch /.autorelabel. Файл нужен для того, чтобы выполнить relabel файлов в системе. Без этого загрузка будет невозможной.

Способ №3:
- При загрузке машины нажать F12 для выбора загрузочного носителя
- На этапе выбора ядра нажать "e"
- В конце строки начинающейся с linux16 заменить ro на rw init=/sysroot/bin/sh и нажать сtrl-x для загрузки системы
- Сменить пароль: passwd root
- После смены пароля необходимо создать скрытый файл .autorelabel в /, выполнив touch /.autorelabel. Файл нужен для того, чтобы выполнить relabel файлов в системе. Без этого загрузка будет невозможной.

#### Установить систему с LVM, после чего переименовать VG:

[vagrant@lvm ~]$ sudo vgs
> VG         #PV #LV #SN Attr   VSize   VFree
> VolGroup00   1   2   0 wz--n- <38.97g    0

[vagrant@lvm ~]$ sudo vgrename VolGroup00 OtusRoot
> Volume group "VolGroup00" successfully renamed to "OtusRoot"

Далее правим /etc/fstab, /etc/default/grub, /boot/grub2/grub.cfg. Везде заменить старое название на новое.

[root@lvm vagrant]#  mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

>Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64

>dracut module 'busybox' will not be installed, because command 'busybox' could not be found!

>...

>*** Creating image file ***

>*** Creating image file done ***

>*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

Перезагрузить машину и проверить:

[root@otuslinux ~]# vgs

>VG #PV #LV #SN Attr VSize VFree

 > OtusRoot 1 2 0 wz--n- <38.97g 0

#### Добавить модуль в initrd:

Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Чтобы добавить свой модуль необходимо создать папку с именем 01test:

sudo mkdir /usr/lib/dracut/modules.d/01test

В нее поместим два скрипта:

    module_setup.sh
      #!/bin/bash
      check() {
      return 0
      }
      depends() {
          return 0
      }
      install() {
          inst_hook cleanup 00 "${moddir}/test.sh"
      }
  
_________________________________________________
    test.sh
    #!/bin/bash
    exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
    cat <<'msgend'
    Hello! You are in dracut module!
    ___________________
    < I'm dracut module >
    -------------------
       \
        \
            .--.
           |o_o |
           |:_/ |
          //   \ \
         (|     | )
        /'\_   _/`\
        \___)=(___/
    msgend
    sleep 10
    echo " continuing...."
    __________________________________________________

Пересобрать образ initrd:

sudo mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

sudo lsinitrd -m /boot/initramfs-$(uname -r).img | grep test

Перезагрузить машину отключив при загрузке опции rghb и quiet.

Пингвин)


#### 4(*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM:

- sudo -s
- parted -> /dev/sdb -> mklabel msdos -> mkpart 1 -1
- pvcreate /dev/sdb1/ --bootloaderareasize 1M
- vgcreate testRoot /dev/sdb1
- lvcreate -n root -l 100%FREE testOtus
- pvs

>  PV         VG      Fmt  Attr PSize   PFree

>  /dev/sda1  testRoot lvm2 a--  <10.00g    0
- vgs

>  VG      #PV #LV #SN Attr   VSize   VFree

>  testRoot   1   1   0 wz--n- <10.00g    0

- lvs
 > LV   VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
 
 > root testRoot -wi-ao---- <10.00g
- mkfs.ext4 /dev/mapper/testRoot-root
- mkdir /mnt/root
- mount /dev/mapper/testRoot-root /mnt/root
- cp /* /mnt/root
- mount --rbind /dev/ /mnt/root/dev; mount --rbind /proc /mnt/root/proc; mount --rbind /sys /mnt/root/sys; mount --rbind /run /mnt/root/run
- chroot /mnt/root
- yum-config-manager --add-repo=https://yum.rumyantsev.com/centos/7/x86_64/
- yum install grub2
- В файле /etc/fstab, изменить "/" на  "lvm/dev/mapper/testRoot-root" и закомментировать "/boot"
- В файле /etc/default/grub изменить в строке GRUB_CMDLINE_LINUX значение "rd.lvm.lv" на "rd.lvm.lv=testRoot/root"
- grub2-mkconfig -o /boot/grub2/grub.cfg
- mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
- grub2-install /dev/sdb
- В файле /etc/selinux/config опции SELINUX присвоить Disabled
- reboot
