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
Для разметки диска воспользуемся утилитой `cfdisk`. Для просмотра информации о всех дисках, точек монтирования и др. воспользуемся `lsblk`, `lsblk -f`.
```bash
lsblk
lsblk -f
cfdisk /dev/sdXY
```
Я буду использовать `LWM`:

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

> **Внимание**, если вы устанавливаете систему в режиме `Dual Boot`, то не размечаем `Efi` раздел и не форматируем его.

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
Проверяем все при помощи `lsblk`.

Проверяем соединение с сетью интернет.

```bash
ping google.com
```
Если вы используете Wi-Fi то подключитесь к сети при помощи:
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

Активируйте NTP для синхронизации часов компьютера с реальным временем:
```bash
dinitctl start ntpd
```
### Установка базовой системы
Изначально я буду устанавливать нестандартное ядро `linux`, а [linux-zen](https://wiki.archlinux.org/title/Kernel#Officially_supported_kernels).
> **Внимание**, если вы не хотите устанавливать `sudo`, то обходите стороной пакет `base-devel`.

```bash
basestrap /mnt base dinit linux-zen linux-zen-headers linux-firmware
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
pacman -S vim neofetch grub efibootmgr os-prober dhclient doas connman-dinit terminus-font artix-archlinux-support
```
> Если вам нужна поддержка Wi-Fi, то устанавливайте [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant).
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
```
Укажем редактор по умолчанию:
```bash
vim /etc/environment
EDITOR=vim
```
Конфигурируем часовой пояс. Утилита `hwclock` позволит установить время по аппаратным часам.
```bash
ln -sf /usr/share/zoneinfo/Регион/Город /etc/localtime
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
LANG=ru_RU.UTF-8
C_COLLATE=C
```
Шрифт:
```bash
vim /etc/vconsole.conf
```
```bash
vim /etc/vconsole.conf
KEYMAP=ru
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
Включим демона `connmand`:
```bash
ln -s ../connmand /etc/dinit.d/boot.d/
```
Doas:

В файле `/etc/doas.conf`:
```bash
# /etc/doas.conf
permit persist :wheel
```
Настраиваем pacman
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
Флаги компилятора
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

### Софт
```bash
yay alacritty      # Эмулятор терминала
yay wofi           # Меню запуска приложений
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

Для автозапуска `Hyprland` добавте в файл `~/.zshrc`
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
yay ark
yay p7zip
yay discord
yay betterdiscordctl
yay telegram
yay noto-fonts
yay noto-fonts-emoji
yay noto-fonts-extra
```
### Thunar
```bash
yay thunar
yay gvfs
yay thunar-archive-plugin
yay thunar-media-tags-plugin
yay thunar-volman
yay tumbler
yay raw-thumbnailer
yay libgsf
yay go-mtpfs
```
