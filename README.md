# Artix-Dinit-Hyprland-Doas

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

### Интернет
Проверяем соединение:

```bash
ping google.com
```
Если вы используете Wi-Fi то подключитесь к сети при помощи утилиты [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant) или [iw](https://wireless.wiki.kernel.org/en/users/documentation/iw).

### Cистемные часы
Активируйте NTP для синхронизации часов компьютера c реальным временем:
```bash
dinitctl start ntpd
```
### Установка базовой системы
Я буду устанавливать не стандартное ядро `linux`, а [linux-zen](https://wiki.archlinux.org/title/Kernel#Officially_supported_kernels).
> **Внимание**, если вы не хотите устанавливать `sudo`, то обходите стороной пакет `base-devel`.

```bash
basestrap /mnt base dinit elogind-dinit linux-zen linux-zen-headers linux-firmware
```

### Закончим монтирование, создадим swap file и сгенерируем fstab
Монтируем все остальные разделы в /mnt/mnt/...
```bash
mkdir -p /mnt/mnt/hd
mount /dev/sdb1 /mnt/mnt/hd
```
Создадим [swap file](https://wiki.archlinux.org/title/swap#Swap_file) на 4Gib
```bash
btrfs subvolume create /mnt/swap
btrfs filesystem mkswapfile --size 4g --uuid clear /mnt/swap/swapfile
swapon /mnt/swap/swapfile
```

Cгенерируем конфиг `fstab` при помощи `fstabgen`:
```bash
fstabgen -U /mnt >> /mnt/etc/fstab
```
## Настройка базовой системы

### Меняем корень
```bash
artix-chroot /mnt
```

### Устанавливаем нужные пакеты
```bash
pacman -S vim neofetch grub efibootmgr os-prober dhclient doas connman-dinit terminus-font artix-archlinux-support ranger
```
> Если вам нужна поддержка Wi-Fi, то устанавливайте [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant) или [iw](https://wireless.wiki.kernel.org/en/users/documentation/iw)
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
Укажем редактор по умолчанию
```bash
EDITOR=vim
```
### Конфигурируем часовой пояс. Утилита `hwclock` позволит установить время по аппаратным часам

```bash
ln -sf /usr/share/zoneinfo/Регион/Город /etc/timezone
hwclock --systohc
```

### Локали и шрифт tty
Расскоментируем нужные нам локали в файле `/etc/locale.gen`
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
Сгенерируем локали
```bash
locale-gen
```
Укажем локаль по умолчанию
```bash
vim /etc/locale.conf
```
```bash
# /etc/locale.conf
LANG=en_US.UTF-8
C_COLLATE=C
```
Шрифт
```bash
vim /etc/vconsole.conf
```
```bash
vim /etc/vconsole.conf
KEYMAP=en
FONT=ter-v20b
```
### Загрузчик
Устанавливаем grub
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
Сгенерируем конфиг grub
> **Внимание**, если `grub` не нашел `Windows`, то нужно будет перегенерировать конфиг после перезагрузки пк
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
### Настройка пользователей
Зададим пороль для root
```bash
passwd
```
Создадим пользователя и зададим ему пороль и группы
```bash
useradd -m имя_пользователя
passwd имя_пользователя
usermod -aG wheel,audio,video,storage имя_пользователя
```
### Настройка сети
В файле `/etc/hostname` укажем имя хоста, а в `/etc/hosts` укажем:
```bash
127.0.0.1  localhost
::1        localhost
127.0.1.1  имя_хоста.localdomain  имя_хоста
```
Включим демона `connmand`  и `elogind`
```bash
ln -s ../connmand /etc/dinit.d/boot.d/
dinitctl enable elogind
```
### Настройка doas
В файле `/etc/doas.conf`:
```bash
# /etc/doas.conf
permit persist :wheel
```
### Первая перезагрузка
```bash
exit
reboot
```
## Послуящая настройка 
Обновим конфиг `grub`
```bash
doas grub-mkconfig -o /boot/grub/grub.cfg
```
### Пропишем в системный `enviroment`
```bash
echo "LIBSEAT_BACKEND=logind" >> /etc/enviroment
```
### Настройка pacman



