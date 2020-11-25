Домашнее задание. Загрузка системы.

Работа с загрузчиком
1. Попасть в систему без пароля несколькими способами
2. Установить систему с LVM, после чего переименовать VG
3. Добавить модуль в initrd

4(*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM
Репозиторий с пропатченым grub: https://yum.rumyantsev.com/centos/7/x86_64/
PV необходимо инициализировать с параметром --bootloaderareasize 1m

1. Попасть в систему без пароля несколькими способами:
Способ №1:
При загрузке машины нажать F12 для выбора загрузочного носителя
На этапе выбора ядра нажать "e"
В конце строки начинающейся с linux16 добавить init=/bin/sh и нажать сtrl-x для загрузки системы
Перемонтировать файловую систему в rw командой: mount -o remount,rw /

Способ №2:
При загрузке машины нажать F12 для выбора загрузочного носителя
На этапе выбора ядра нажать "e"
В конце строки начинающейся с linux16 добавить rd.break и нажать сtrl-x для загрузки системы
Перемонтировать файловую систему в rw командой: mount -o remount,rw /sysroot 
Далее, выполнить chroot в смонтированную директорию: chroot /sysroot
Сменить пароль: passwd root
После смены пароля необходимо создать скрытый файл .autorelabel в /, выполнив touch /.autorelabel. Файл нужен для того, чтобы выполнить relabel файлов в системе. Без этого загрузка будет невозможной.

Способ #3:
При загрузке машины нажать F12 для выбора загрузочного носителя
На этапе выбора ядра нажать "e"
В конце строки начинающейся с linux16 заменить ro на rw init=/sysroot/bin/sh и нажать сtrl-x для загрузки системы
Сменить пароль: passwd root
После смены пароля необходимо создать скрытый файл .autorelabel в /, выполнив touch /.autorelabel. Файл нужен для того, чтобы выполнить relabel файлов в системе. Без этого загрузка будет невозможной.

Установить систему с LVM, после чего переименовать VG:

[vagrant@lvm ~]$ sudo vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0
[vagrant@lvm ~]$ sudo vgrename VolGroup00 OtusRoot
  Volume group "VolGroup00" successfully renamed to "OtusRoot"
Далее правим /etc/fstab, /etc/default/grub, /boot/grub2/grub.cfg. Везде заменить старое название на новое.
[root@lvm vagrant]#  mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
Перезагрузить машину и проверить:
[root@otuslinux ~]# vgs


Добавить модуль в initrd:

Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Чтобы добавить свой модуль необходимо создать папку с именем 01test:
mkdir /usr/lib/dracut/modules.d/01test
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



