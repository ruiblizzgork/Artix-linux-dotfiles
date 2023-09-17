# Artix-Dinit-Hyprland-Doas
Установка `Artix` в `Dual Boot` режиме. Мое железо:
| Comp | Name                  |
|------|-----------------------|
| CPU  | AMD Ryzen 1600        |
| GPU  | AMD Radeon RX580 8Gib |
| RAM  | 16Gib                 |

Источники: [Оптимизация ArchLinux](https://www.ixbt.com/live/offtopic/optimizaciya-archlinux-dlya-raboty-i-igr.html#con_1fnoe4half); [Artix Wiki](https://wiki.artixlinux.org/Main/Installation)
## Установка Artix
```bash
artixlinux login: artix
Password: artix

# Заходим под root
su
```
### Разметка диска
Для разметки диска воспользуемся утилитой `cfdisk` (утилиты `fdisk` нет в коробке с `Artix`). Для просмотра информации о всех дисках, точек монтирования и др. воспользуемся `lsblk`.
```bash
lsblk
cfdisk /dev/sdXY
```
Я разбиваю диски на разделы в таком варианте:

> **Внимание**, если вы устанавливаете систему в режиме `Dual Boot`, то не размечаем `Efi` раздел и не форматируем его.

| Раздел          | Тип              | Обьем                         |
|-----------------|------------------|-------------------------------|
| Root + SwapFile | Linux filesystem | 30Gib + 4Gib = 34Gib          |
| Efi             | EFI System       | 512Mib                        |
| Home            | Linux filesystem | Оставшееся пространство       |
| Other           | Linux filesystem | Пространство на других дисках |

Я ставлю `Artix` параллельно `Microsoft Windows`, поэтому мои разделы выглядят так:

| Раздел          | Тип              | Устройство | Файловая система |
|-----------------|------------------|------------|------------------|
| Root + SwapFile | Linux filesystem | /dev/sda4  | btrfs            |
| Efi             | EFI System       | /dev/sda1  | fat32            |
| Home            | Linux filesystem | /dev/sda5  | btrfs            |
| Other           | Linux filesystem | /dev/sdb1  | ext4             |

```bash
# Форматируем основные разделы. Все остальные будем монтировать позже 
mkfs.fat -F32 /dev/sda1 # Не делайте это если у вас Dual Boot
fatlabel /dev/sda1 BOOT

mkfs.btrfs -f -L ROOT /dev/sda4

mkfs.btrfs -f -L HOME /dev/sda5

mkfs.ext4 -L STORAGE /dev/sdb1

# Монтируем
mount /dev/sda4 /mnt

mkdir -p /mnt/boot/efi
mkdir -p /mnt/home

mount /dev/sda1 /mnt/boot/efi
mount /dev/sda5 /mnt/home
```
Проверяем все при помощи `lsblk`.
```bash
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 223,6G  0 disk
├─sda1   8:1    0   100M  0 part /boot/efi
├─--
├─--
├─sda4   8:4    0    34G  0 part /
├─sda5   8:5    0   136G  0 part /home
└─--
sdb      8:16   0 465,8G  0 disk
└─sdb1   8:17   0 465,8G  0 part
```

Проверяем соединение:

```bash
ping google.com
```
Если вы используете Wi-Fi то подключитесь к сети при помощи утилиты [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant) или [iw](https://wireless.wiki.kernel.org/en/users/documentation/iw).

Активируйте NTP для синхронизации часов компьютера с реальным временем:
```bash
dinitctl start ntpd
```
### Установка базовой системы
Я буду устанавливать нестандартное ядро `linux`, а [linux-zen](https://wiki.archlinux.org/title/Kernel#Officially_supported_kernels).
> **Внимание**, если вы не хотите устанавливать `sudo`, то обходите стороной пакет `base-devel`.

```bash
basestrap /mnt base dinit elogind-dinit linux-zen linux-zen-headers linux-firmware
```
Монтируем все остальные разделы в /mnt/mnt/...
```bash
mkdir -p /mnt/mnt/hd
mount /dev/sdb1 /mnt/mnt/hd

lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 223,6G  0 disk
├─sda1   8:1    0   100M  0 part /boot/efi
├─--
├─--
├─sda4   8:4    0    34G  0 part /
├─sda5   8:5    0   136G  0 part /home
└─--
sdb      8:16   0 465,8G  0 disk
└─sdb1   8:17   0 465,8G  0 part /mnt/hd
```
Создадим [swap file](https://wiki.archlinux.org/title/swap#Swap_file) на 4Gib:
```bash
btrfs subvolume create /mnt/swap
btrfs filesystem mkswapfile --size 4g --uuid clear /mnt/swap/swapfile
swapon /mnt/swap/swapfile
```

Генерируем конфиг `fstab` при помощи `fstabgen`:
```bash
fstabgen -U /mnt >> /mnt/etc/fstab
```
## Настройка базовой системы
Меняем корень:
```bash
artix-chroot /mnt
```
Устанавливаем нужные пакеты:
```bash
pacman -S vim neofetch grub efibootmgr os-prober dhclient doas connman-dinit terminus-font artix-archlinux-support ranger
```
> Если вам нужна поддержка Wi-Fi, то устанавливайте [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant) или [iw](https://wireless.wiki.kernel.org/en/users/documentation/iw).
```text
vim                       - легендарный текстовый редактор
neofetch                  - информация о системе
grub                      - загрузчик
efibootmgr                - изменения диспетчера загрузки UEFI
os-prober                 - обнаружение Windows или других, не Linux Os
dhclient                  - демон dhcp (интернет подключение)
connman-dinit             - сетевой менеджер
doas                      - альтернатива sudo
terminus-font             - шрифт для tty, поддерживающий русский язык
artix-archlinux-support   - поддержка репозиториев arch'а
ranger                    - консольный файловый менеджер
```
Укажем редактор по умолчанию:
```bash
EDITOR=vim
```
Конфигурируем часовой пояс. Утилита `hwclock` позволит установить время по аппаратным часам.
```bash
ln -sf /usr/share/zoneinfo/Регион/Город /etc/timezone
hwclock --systohc
```
Расскоментируем нужные нам локали в файле `/etc/locale.gen`.
```bash
vim /etc/locale.gen
```
```bash
# /etc/locale.gen
# ...
en_US.UTF-8 UTF-8
# ...
ru_RU.UTF-8 UTF-8
```
Сгенерируем локали:
```bash
locale-gen
```
Укажем локаль по умолчанию:
```bash
vim /etc/locale.conf
```
```bash
# /etc/locale.conf
LANG=en_US.UTF-8
C_COLLATE=C
```
Шрифт:
```bash
vim /etc/vconsole.conf
```
```bash
vim /etc/vconsole.conf
KEYMAP=en
FONT=ter-v20b
```
### Загрузчик
Устанавливаем grub:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub --recheck
```
Если вы хотите `Dual Boot`, то в файле /etc/default/grub раскомментруйте строку:
```bash
vim /etc/default/grub
```
```bash
# /etc/default/grub
# ...
GRUB_DISABLE_OS_PROBER=false
```
Сгенерируем конфиг grub:
> **Внимание**, если `grub` не нашел `Windows`, то нужно будет перегенерировать конфиг после перезагрузки пк.
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
### Пользователи и сеть
Зададим пороль для root:
```bash
passwd
```
Создадим пользователя и зададим ему пароль и группы:
```bash
useradd -m имя_пользователя
passwd имя_пользователя
usermod -aG wheel,audio,video,storage имя_пользователя
```
Настроим сеть:
В файле `/etc/hostname` укажем имя хоста, а в `/etc/hosts` укажем:
```bash
127.0.0.1  localhost
::1        localhost
127.0.1.1  имя_хоста.localdomain  имя_хоста
```
Включим демона `connmand`  и `elogind`:
```bash
ln -s ../connmand /etc/dinit.d/boot.d/
dinitctl enable elogind
```
Doas:

В файле `/etc/doas.conf`:
```bash
# /etc/doas.conf
permit persist :wheel
```
Перезагрузим систему:
```bash
exit
reboot
```
## Пост-настройка и установка Hyprland
`Grub` не определил вторую систему по этому обновим конфиг `grub`:
```bash
doas grub-mkconfig -o /boot/grub/grub.cfg
```
Пропишем в системный `enviroment`
```bash
echo "LIBSEAT_BACKEND=logind" >> /etc/enviroment
```
### Настраиваем pacman, чтобы тот летал
```bash
doas vim /etc/pacman.conf
```
Раскомментируем и добавим следующие строки:
```bash
[options]
IgnorePkg = sudo      # Игнорируем пакет sudo. При установки sudo pacman будет предупреждать, что он игнорируемый
Color                 # Раскомментируем
ParallelDownloads = 5 # Раскомментируем

# Репы Arch'а
[extra]
Include = /etc/pacman.d/mirrorlist-arch

[community]
Include = /etc/pacman.d/mirrorlist-arch

[multilib]
Include = /etc/pacman.d/mirrorlist-arch
```
```bash
doas pacman-key --init
doas pacman-key --populate archlinux
doas pacman-key --populate artix
doas pacman-key --refresh-keys
doas pacman -Sy
```
```bash
doas pacman -S reflector
# Страна на выбор
doas reflector --verbose --country 'Germany' -l 25 --sort rate --save /etc/pacman.d/mirrorlist-arch
doas pacman -Suy
```
Установим [Yay](https://github.com/Jguer/yay):
```bash
doas pacman -S autoconf automake bison fakeroot flex gcc make pkgconf git patch # Инструменты для сборки
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd
```
### Флаги компилятора
```bash
doas vim /etc/makepkg.conf
```
```bash
CFLAGS="-march=native -mtune=native -O3 -pipe -fno-plt -fexceptions \
      -Wp,-D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security \
      -fstack-clash-protection -fcf-protection"
CXXFLAGS="$CFLAGS -Wp,-D_GLIBCXX_ASSERTIONS"
RUSTFLAGS="-C opt-level=3"
MAKEFLAGS="-j$(nproc) -l$(nproc)"
OPTIONS=(strip docs !libtool !staticlibs emptydirs zipman purge !debug lto)
```
Данные флаги компилятора выжимают максимум производительности при компиляции, но могут вызывать ошибки сборки в очень редких приложениях. Если такое случится, то отключите ‘lto’ в строке options, добавив символ восклицательного знака ("!lto").

### Софт
```bash
yay alacritty      # Эмулятор терминала
yay hyprland
yay ew-wayland     # Бар
yay google-chrome
yay wofi           # Меню запуска приложений
```
Для автологина редактируем файл `/etc/dinit.d/getty.d/tty1`:
```bash
command = /sbin/agetty -a имя_пользователя --noclear tty1 38400 linux
```
### Установка ZSH
```bash
doas pacman -Syu zsh
zsh # Настройка
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
Настраиваем `zsh` под себя.

Для автозапуска `Hyprland` добавте в файл `~/.zshrc.pre-oh-my-zsh`
```bash
if test -z "${XDG_RUNTIME_DIR}"; then
  UID="$(id -u)"
  export XDG_RUNTIME_DIR=/tmp/"${UID}"-runtime-dir
    if ! test -d "${XDG_RUNTIME_DIR}"; then
        mkdir "${XDG_RUNTIME_DIR}"
        chmod 0700 "${XDG_RUNTIME_DIR}"
    fi
fi

if [ -z "${WAYLAND_DISPLAY}" ] && [ "${XDG_VTNR}" -eq 1 ]; then
    dbus-run-session Hyprland
fi
```
### Драйверы
```bash
doas pacman -S amd-ucode iucode-tool
doas mkinitcpio -P
```
Устанавливаем недостающие драйвера по [таблице](https://wiki.archlinux.org/title/mkinitcpio#Possibly_missing_firmware_for_module_XXXX):
| Modul          | Package                 |
|----------------|-------------------------|
| aic94xx	       | aic94xx-firmware        | 
| ast	           | ast-firmware            |
| bfa	           | linux-firmware-qlogic   |
| bnx2x	         | linux-firmware-bnx2x    |
| liquidio	     | linux-firmware-liquidio |
| mlxsw_spectrum | linux-firmware-mellanox |
| nfp	           | linux-firmware-nfp      |
| qed	           | linux-firmware-qlogic   |
| qla1280	       | linux-firmware-qlogic   |
| qla2xxx	       | linux-firmware-qlogic   |
| wd719x	       | wd719x-firmware         |
| xhci_pci	     | upd72020x-fw            |
```bash
doas pacman -S btrfs-progs # Если корень отфоратирован в btrfs
doas mkinitcpio -P
doas pacman -S mesa lib32-mesa vulkan-radeon lib32-vulkan-radeon vulkan-icd-loader lib32-vulkan-icd-loader
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

```bash
# Узнаем свой lpj
doas dmesg | grep lpj=

doas vim /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet splash noibrs tsx_async_abort=off rootfstype=btrfs selinux=0 lpj=ваш_lpj raid=noautodetect mitigations=off preempt=none amdgpu.ppfeaturemask=0xffffffff"
```
lpj = это уникальный параметр для каждой системы. Самоопределяется во время загрузки, что довольно трудоёмко, поэтому лучше задать вручную. Определить ваше значение lpj можно через следующую команду: doas dmesg | grep lpj=.

raid=noautodetect — отключает проверку на RAID во время загрузки. Если вы его используете RAID массив, то не прописывайте параметр.

rootfstype=btrfs — Здесь указываем название ФС в которой у вас форматирован корень.

elevator=noop — указывает для всех дисков планировщик ввода NONE. **Не использовать, если у вас жёсткий диск**.
### Оптимальные флаги монтирования
```bash
doas vim /etc/fstab
```
Указываем флаги для разделов, которые находятся на SSD(/, home) 
```bash
rw,noatime,ssd,ssd_spread,discard=async,space_cache=v2,max_inline=256,commit=600,nodatacow,suvolid=5,subvol=/
```
И заменим везде realtime -> noatime
### Собираем ядро Xanmod
Только для amd, для intel собирайть linux-lqx
```bash
doas pacman -S clang llvm lld
export _microarchitecture=99 use_numa=n use_tracers=n _compiler=clang
yay xanmod (linux-xanmod, linux-xanmod-headers)
doas pacman -Rns linux-zen linux-zen-headers
doas mkinitcpio -P
doas grub-mkconfig -o /boot/grub/grub.cfg
```

### Софт
```bash
yay steam
yay lutris
yay btop
yay cmus
yay cava
yay discord
yay betterdiscordctl
yay telegram
```
