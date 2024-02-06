# Artix-Dinit-Sway-Doas
Источники: [Оптимизация ArchLinux](https://ventureo.codeberg.page/); [Artix Wiki](https://wiki.artixlinux.org/Main/Installation); [Arch Wiki](https://wiki.archlinux.org/)
## Установка Artix
```bash
artixlinux login: artix
Password: artix

# Заходим под root
su
```
### Разметка диска
Разметка диска утилитой `cfdisk`. Для просмотра информации о всех дисках, точек монтирования и др. воспользуемся `lsblk`, `lsblk -f`.
```bash
lsblk
lsblk -f
cfdisk /dev/sdXY
```
`LWM`:
```bash
+----------------------+----------------------+ +------------------------+
| Logical volume 1     | Logical volume 2     | | Logical volume 3       |
|                      |                      | |                        |
| /                    | /home                | | /mnt/storage           |
|                      |                      | |                        |
| /dev/ArtixVG/root    | /dev/ArtixVG/home    | | /dev/StorageVG/storage |
|----------------------+----------------------| |------------------------|
| disk drive /dev/sda                         | | disk drive /dev/sdb    |
|                                             | |                        |
+---------------------------------------------+ +------------------------+
```

> **Внимание**, если установка происходит в режиме `Dual Boot`, то не размечаем `Efi` раздел и не форматируем его, а используем, уже существующий.

```bash
pvcreate /dev/sda2
vgcreate ArtixVG /dev/sda2
lvcreate -L 50G ArtixVG -n root
lvcreate -l 100%FREE ArtixVG -n home

pvcreate /dev/sdb1
vgcreate StorageVG /dev/sdb1
lvcreate -l 100%FREE StorageVG -n storage
```
Форматирование и монтирование
```bash 
mkfs.fat -F32 /dev/sda1 # Не делайте это если у вас Dual Boot
fatlabel /dev/sda1 BOOT

mkfs.btrfs -f -L ROOT /dev/ArtixVG/root

mkfs.btrfs -f -L HOME /dev/ArtixVG/home

mkfs.ext4 -L STORAGE /dev/StorageVG/storage

# Монтируем
mount /dev/ArtixVG/root /mnt
mount --mkdir /dev/sda1 /mnt/boot/efi
mount --mkdir /dev/ArtixVG/home /mnt/home
mount --mkdir /dev/StorageVG/storage /mnt/mnt/storage
```
Проверка разделов при помощи `lsblk -f`.

Проверка соединения с сетью:
```bash
ping google.com
```
Подключение к беспроводной сети:
```bash
rfkill unblock wifi
connmandctl
>enable wifi
>scan wifi
>services 
>agent on
>connect wifi_...
>quit
```

Активация NTP для синхронизации часов компьютера с реальным временем:
```bash
dinitctl start ntpd
```
### Установка базовой системы
Установка нестандартного ядра [linux-zen](https://wiki.archlinux.org/title/Kernel#Officially_supported_kernels).
> **Внимание**, для любителей `doas` противопоказан пакет `base-devel`.

```bash
basestrap /mnt base dinit linux-zen linux-zen-headers linux-firmware
```

генерация `fstab` при помощи `fstabgen`:
```bash
fstabgen -U /mnt >> /mnt/etc/fstab
```
## Настройка базовой системы
Смена корня:
```bash
artix-chroot /mnt
```
Установка базовых пакетов:
```bash
pacman -S nvim neofetch grub efibootmgr os-prober dhclient doas networkmanager-dinit terminus-font artix-archlinux-support
```
> Если вам нужна поддержка Wi-Fi, то устанавливайте [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant).
```text
nvim                      - легендарный текстовый редактор
neofetch                  - информация о системе
grub                      - загрузчик
efibootmgr                - изменения диспетчера загрузки UEFI
os-prober                 - обнаружение Windows или других, не Linux Os
dhclient                  - демон dhcp (интернет подключение)
networkmanager-dinit      - сетевой менеджер
doas                      - альтернатива sudo
terminus-font             - шрифт для tty, поддерживающий русский язык
artix-archlinux-support   - поддержка репозиториев arch'а
```
Редактор по умолчанию:
```bash
vim /etc/environment
EDITOR=nvim
```
Конфигурация часового пояса. Утилита `hwclock` позволит установить время по аппаратным часам.
```bash
ln -sf /usr/share/zoneinfo/Регион/Город /etc/localtime
hwclock --systohc
```
Поиск и добаление нужных нам локалей в файле `/etc/locale.gen`.
```bash
# /etc/locale.gen
# ...
en_US.UTF-8 UTF-8
# ...
ru_RU.UTF-8 UTF-8
```
Генерация локалей:
```bash
locale-gen
```
Локаль по умолчанию:
```bash
# /etc/locale.conf
LANG=ru_RU.UTF-8
C_COLLATE=C
```
Шрифт:
```bash
# /etc/vconsole.conf
KEYMAP=ru
FONT=ter-v20b
```
### Загрузчик
Установка grub:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub --recheck
```
Если установка в `Dual Boot` режиме, то в файле /etc/default/grub раскомментруйте строку:
```bash
# /etc/default/grub
# ...
GRUB_DISABLE_OS_PROBER=false
```
Генерация конфига grub:
> **Внимание**, если `grub` не нашел `Windows`, то нужно будет перегенерировать конфиг после перезагрузки пк.
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
### Пользователи и сеть
Зададим пороль для root:
```bash
passwd
```
Создание пользователя и выдача ему пароля и присвоение групп:
```bash
useradd -m имя_пользователя
passwd имя_пользователя
usermod -aG wheel,audio,video,storage имя_пользователя
```
Настройка сети:
В файле `/etc/hostname` укажем имя хоста, а в `/etc/hosts` укажем:
```bash
# /etc/hostname
127.0.0.1  localhost
::1        localhost
127.0.1.1  имя_хоста.localdomain  имя_хоста
```
Запуск демона `networkmanager`:
```bash
dinitctl enable NetworkManager
```
Doas:
В файле `/etc/doas.conf`:Настраиваем pacman
```bash
doas vim /etc/pacman.conf
```
```bash
# /etc/doas.conf
permit persist :wheel
```
Проверка поддерживаемой архитектуры:
```bash
/lib/ld-linux-x86-64.so.2 --help | grep "x86-64"
```
Настраиваем pacman:
```bash
# /etc/pacman.conf
[options]
Architecture = x86_64 x86_64_v3
IgnorePkg = sudo                  # Игнорируем пакет sudo. При установки sudo pacman будет предупреждать, что он игнорируемый
Color                             # Раскомментируем
VerbosePkgLists
ParallelDownloads = 5 # Раскомментируем
DisableDownloadTimeout

# Репы Arch'а
[extra]
Include = /etc/pacman.d/mirrorlist-arch

[community]
Include = /etc/pacman.d/mirrorlist-arch

[multilib]
Include = /etc/pacman.d/mirrorlist-arch
```
Флаги компилятора:
```bash
# /etc/makepkg.conf
CFLAGS="-march=native -mtune=native -O3 -pipe -fno-plt -fexceptions \
      -Wp,-D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security \
      -fstack-clash-protection -fcf-protection"
CXXFLAGS="$CFLAGS -Wp,-D_GLIBCXX_ASSERTIONS"
RUSTFLAGS="-C opt-level=3"
MAKEFLAGS="-j$(nproc) -l$(nproc)"
...
OPTIONS=(strip docs !libtool !staticlibs emptydirs zipman purge !debug lto)
```
Данные флаги компилятора выжимают максимум производительности при компиляции, но могут вызывать ошибки сборки в очень редких приложениях. Если такое случится, то отключите ‘lto’ в строке options, добавив символ восклицательного знака ("!lto").

Перезагрузим систему:
```bash
exit
reboot
```
## Пост-настройка
`Grub` не определил вторую систему по этому обновим конфиг `grub`:
```bash
doas grub-mkconfig -o /boot/grub/grub.cfg
```
Обновление ключек для `pacman`:
```bash
doas pacman-key --init
doas pacman-key --populate archlinux
doas pacman-key --populate artix
doas pacman-key --refresh-keys
doas pacman -Sy
```
Добовление зеркал репозиториев (можно вручную найти на сайтах Arch Linux и Artix)
```bash
doas pacman -S reflector
# Страна на выбор
doas reflector --verbose --country 'Germany' -l 25 --sort rate --save /etc/pacman.d/mirrorlist-arch
doas pacman -Suy
```
Установка [Yay](https://github.com/Jguer/yay):
```bash
doas pacman -S autoconf automake bison fakeroot flex gcc make pkgconf git patch # Инструменты для сборки
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd
```

### Софт
```bash
yay alacritty      # Эмулятор терминала
```

```
### Установка ZSH
```bash
doas pacman -Syu zsh
zsh # Настройка
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
chsh -s /bin/zsh
doas chsh -s /bin/zsh 
```
Настраиваем `zsh` под себя.

### Драйверы
```bash
doas pacman -S amd-ucode iucode-tool
doas mkinitcpio -P
```
Устанавливаем недостающие драйвера по [таблице](https://wiki.archlinux.org/title/mkinitcpio#Possibly_missing_firmware_for_module_XXXX):
| Modul            | Package                 |
|------------------|-------------------------|
| aic94xx	       | aic94xx-firmware        | 
| ast	             | ast-firmware            |
| bfa	             | linux-firmware-qlogic   |
| bnx2x	       | linux-firmware-bnx2x    |
| liquidio	       | linux-firmware-liquidio |
| mlxsw_spectrum   | linux-firmware-mellanox |
| nfp	             | linux-firmware-nfp      |
| qed	             | linux-firmware-qlogic   |
| qla1280	       | linux-firmware-qlogic   |
| qla2xxx	       | linux-firmware-qlogic   |
| wd719x	       | wd719x-firmware         |
| xhci_pci	       | upd72020x-fw            |
```bash
doas pacman -S btrfs-progs # Если корень отфоратирован в btrfs
doas mkinitcpio -P
doas pacman -S mesa vulkan-radeon vulkan-icd-loader
doas pacman -S lib32-mesa lib32-vulkan-radeon lib32-vulkan-icd-loader
```
### Еще оптимизация
```bash
yay ananicy-dinit
doas dinitctl enable ananicy

yay haveged-dinit
doas dinitctl enable haveged
```
Ananicy — это демон для распределения приоритета задач, его установка сильно повышает отклик системы.

Haveged — это демон, что следит на энтропией системы.

### Звук
```bash
doas pacman -S pipewire pipewire-jack pipewire-alsa pavucontrol pipewire-pulse alsa-utils
```
Что бы звук появился в системе нужно включить автозапуск в конфиге `Hyprland`:
```bash
exec-once = pipewire
exec-once = pipewire-pulse
exec-once = wireplumber
```

### Swap
Есть 2 варианта: `zram`, `swapfile`:
```bash
yay zramen-dinit
zramen make
doas dinitctl enable zramen
```
```bash
btrfs subvolume create /swap
btrfs filesystem mkswapfile --size 16g --uuid clear /swap/swapfile
swapon /swap/swapfile
# /etc/fstab
/swap/swapfile none swap defaults 0 0
```

### Разгон
Установим [corectrl](https://gitlab.com/corectrl/corectrl):
```bash
yay corectrl
```
```bash
doas vim /etc/polkit-1/rules.d/90-corectrl.rules

polkit.addRule(function(action, subject) {
    if ((action.id == "org.corectrl.helper.init" ||
         action.id == "org.corectrl.helperkiller.init") &&
        subject.local == true &&
        subject.active == true &&
        subject.isInGroup("your-user-group")) {
            return polkit.Result.YES;
    }
});
```
### Параметры ядра
```bash
# Узнаем свой lpj
doas dmesg | grep lpj=

doas vim /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet sysctl.vm.swappiness=10 amdgpu.ppfeaturemask=0xffffffff"
```
lpj= - Уникальный параметр для каждой системы. Его значение автоматически определяется во время загрузки, что довольно трудоемко, поэтому лучше задать вручную. Определить ваше значение для lpj можно через следующую команду: sudo dmesg | grep "lpj="

mitigations=off - Непосредственно отключает все заплатки безопасности ядра (включая Spectre и Meltdown). Подробнее об этом написано здесь.

nowatchdog - Отключает сторожевые таймеры. Позволяет избавиться от заиканий в онлайн играх.

page_alloc.shuffle=1 - Этот параметр рандомизирует свободные списки распределителя страниц. Улучшает производительность при работе с ОЗУ с очень быстрыми накопителями (NVMe, Optane). Подробнее тут.

threadirqs - задействует многопоточную обработку IRQ прерываний.

split_lock_detect=off - Отключаем раздельные блокировки шины памяти. Одна инструкция с раздельной блокировкой может занимать шину памяти в течение примерно 1 000 тактов, что может приводить к кратковременным зависаниям системы.

intel_idle.max_cstate=1 - только для процессоров Intel. Отключает энергосберегательные функции процессора, ограничивая его спящие состояния, не позволяя ему переходить в состояние глубокого сна. Увеличивает (может значительно увеличить) энергопотребление на ноутбуках. Помогает исправлять некоторые странные зависания и ошибки на многих системах.

pci=pcie_bus_perf - Увеличивает значение Max Payload Size (MPS) для родительской шины PCI Express. Это даёт лучшую пропускную способность, т. к. некоторые устройства могут использовать значение MPS/MRRS выше родительской шины.

[https://unix.stackexchange.com/questions/684623/pcie-bus-perf-understanding-the-capping-of-mrrs]

[https://www.programmersought.com/article/74187399630/]

libahci.ignore_sss=1 - Отключает ступенчатое включение жёстких дисков. Ускоряет работу HDD.

### Установка кастомного ядра linux-cachyos
```bash
sudo pacman-key --recv-keys F3B607488DB35A47 --keyserver keyserver.ubuntu.com
sudo pacman-key --lsign-key F3B607488DB35A47
sudo pacman -U 'https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-keyring-3-1-any.pkg.tar.zst' 'https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-mirrorlist-18-1-any.pkg.tar.zst' 'https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-v3-mirrorlist-18-1-any.pkg.tar.zst' 'https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-v4-mirrorlist-6-1-any.pkg.tar.zst' 'https://mirror.cachyos.org/repo/x86_64/cachyos/pacman-6.0.2-14-x86_64.pkg.tar.zst'
```

```bash
# /etc/pacman.conf
[options]
Architecture = x86_64 x86_64_v3

[cachyos-v3]
Include = /etc/pacman.d/cachyos-v3-mirrorlist
```

```bash
doas pacman -Syyuu
doas pacman -S linux-cachyos-bore
doas pacman -S linux-cachyos-bore-headers
```
### Софт
```bash
yay steam
yay heroic-games-launcher-bin
yay btop
yay cmus
yay mpv
yay cava
yay ark
yay p7zip
yay discord
yay betterdiscordctl
yay telegram
# Шрифты
yay noto-fonts
yay noto-fonts-emoji
yay noto-fonts-extra
yay noto-fonts-nerd
```
