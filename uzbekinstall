#!/bin/bash

NO_FORMAT="\033[0m"
F_UNDERLINED="\033[4m"
C_GREY3="\033[38;5;232m"
C_DARKTURQUOISE="\033[48;5;44m"
F_DIM="\033[2m"
C_WHITE="\033[38;5;15m"
C_GREEN="\033[38;5;46m"   
C_BRIGHT_RED="\033[91m"


FS="ext4"

source /tmp/uzbekinstall.conf

get_partitions() {
    local DISK="$1"
    if [[ "$DISK" =~ nvme ]]; then
        EFI_PART="${DISK}p1"
        ROOT_PART="${DISK}p2"
    else
        EFI_PART="${DISK}1"
        ROOT_PART="${DISK}2"
    fi
}


if [[ "$MODE" == "auto" ]]; then
    if [[ ! -b "$DISK" ]] || [[ "${DISK:0:5}" != "/dev/" ]]; then
        echo -e "\033[41;97mЕБАЛАЙ, такого диска нет!!!\033[0m"
        exit 1
    fi

    parted "$DISK" --script mklabel gpt
    parted "$DISK" --script mkpart primary fat32 1MiB 2048MiB
    parted "$DISK" --script set 1 esp on
    parted "$DISK" --script mkpart primary "$FS" 2048MiB 100%

    get_partitions "$DISK"

    echo "Форматирование $EFI_PART в fat32..."
    mkfs.fat -F 32 -I "$EFI_PART"

    echo "Форматирование $ROOT_PART в $FS..."
    mkfs."$FS" -f "$ROOT_PART" 

elif [[ "$MODE" == "manual" ]]; then
    if [[ ! -b "$EFI_PART" ]] || [[ "${EFI_PART:0:5}" != "/dev/" ]] || \
       [[ ! -b "$ROOT_PART" ]] || [[ "${ROOT_PART:0:5}" != "/dev/" ]]; then
        echo -e "\033[41;97mЕБАЛАЙ, у тебя такого раздела нет!!!\033[0m"
        exit 1
    fi

    echo "Форматирование $ROOT_PART в $FS..."
    mkfs."$FS" -f "$ROOT_PART" 

    echo "Форматирование $EFI_PART в fat32..."
    mkfs.fat -F 32 -I "$EFI_PART"

else
    echo -e "\033[41;97mСук, ты додик или как\033[0m"
    exit 1
fi

mount "/dev/$ROOT_PART" /mnt || { echo -e "\033[41;97mПроизошла ошибка при монтировании root /dev/$ROOT_PART!\033[0m"; exit 1; }
mount --mkdir "/dev/$EFI_PART" /mnt/boot || { echo -e "\033[41;97mПроизошла ошибка при монтировании EFI /dev/$EFI_PART!\033[0m"; exit 1; }
pacstrap -K /mnt base linux linux-firmware || { echo -e "\033[41;97mПроизошла ошибка при установке базовой системы!\033[0m"; exit 1; }
genfstab -U /mnt >> /mnt/etc/fstab

echo 'ставим HALAL время...'
arch-chroot /mnt /bin/bash -c "ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime" || { echo -e "\e[31mЧто-та не так произошло!!1 ты лох. \e[0m"; exit 1; }
arch-chroot /mnt /bin/bash -c "
if ! curl -s https://www.google.com | grep -q '<title>Google</title>'; then
    echo -e '\033[41;97mинтернета нет!!!:( \033[0m'
    exit 1
fi
"
arch-chroot /mnt /bin/bash -c "hwclock --systohc"
echo 'Генерация локалелей (их не будет)...'
arch-chroot /mnt /bin/bash -c "sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen"
arch-chroot /mnt /bin/bash -c "sed -i 's/^#ru_RU.UTF-8 UTF-8/ru_RU.UTF-8 UTF-8/' /etc/locale.gen"
arch-chroot /mnt /bin/bash -c "locale-gen"

arch-chroot /mnt /bin/bash -c "echo '$HOSTNAME' > /etc/hostname"

arch-chroot /mnt /bin/bash -c "echo -e \"$ROOT_PASSWORD\n$ROOT_PASSWORD\" | passwd"


arch-chroot /mnt /bin/bash -c "mount $efi /boot"
arch-chroot /mnt /bin/bash -c "bootctl install"
arch-chroot /mnt /bin/bash -c "mkdir -p /boot/loader/entries"
arch-chroot /mnt /bin/bash -c "touch /boot/loader/entries/uzbek.conf"
arch-chroot /mnt /bin/bash -c "cat > /boot/loader/entries/uzbek.conf <<EOF
title   Uzbek Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options root=/dev/$ROOT_PART rw
EOF"
arch-chroot /mnt /bin/bash -c "mkinitcpio -P"

arch-chroot /mnt /bin/bash -c "pacman -S --noconfirm git python-pip labwc python3 tk swaybg nwg-panel nwg-drawer nwg-menu python-pyqt6 jq --needed"
arch-chroot /mnt /bin/bash -c "mkdir /tmp"
arch-chroot /mnt /bin/bash -c "cd /tmp && git clone https://github.com/ZDesktopEnvironment/ZDE && cd ZDE && cp -rfv tree/* /"
arch-chroot /mnt /bin/bash -c "cd /tmp && git clone https://github.com/ZDesktopEnvironment/ZSysConf && cd ZSysConf && cp -rfv tree/* /"

arch-chroot /mnt /bin/bash -c "pacman -S --noconfirm sddm"
arch-chroot /mnt /bin/bash -c "systemctl enable sddm.service"

arch-chroot /mnt /bin/bash -c "useradd -m -g users -G wheel,video,audio -s /bin/bash $USER_NAME"
arch-chroot /mnt /bin/bash -c "echo -e \"$USER_PASSWORD\n$USER_PASSWORD\" | passwd $USER_NAME"

echo 'Установка UZBEK-APPS...'

arch-chroot /mnt /bin/bash -c "pacman -S --noconfirm python-pip sudo python"
echo 'Установка SPM...'
arch-chroot /mnt /bin/bash -c "curl -s https://zenusus.serv00.net/dl/installSPM.sh | bash"
echo 'Установка HALAL Eblan софт...'
PACKAGES=("halalIDE" "320totalsecurity" "eblan-editor" "eblan-music-editor" "eblanoffice")

for pkg in "${PACKAGES[@]}"; do
    arch-chroot /mnt /bin/bash -c "
        python -m venv /tmp/venv_$pkg &&
        source /tmp/venv_$pkg/bin/activate &&
        spm install $pkg &&
        deactivate
    "
done

echo 'я хочу вам установить: firefox, kitty'

arch-chroot /mnt /bin/bash -c "pacman -S --noconfirm firefox alacritty mako wlr-randr nano micro pipewire pipewire-pulse"
arch-chroot /mnt /bin/bash -c "echo '$USER_NAME ALL=(ALL:ALL) ALL' >> /etc/sudoers"

echo 'ПРОИЗВОДСТВО HALAL.NET...'

arch-chroot /mnt /bin/bash -c "systemctl enable systemd-networkd.service systemd-resolved.service"

echo "Удаление харам labwc.desktop (чтобы только ZDE был)..."
arch-chroot /mnt /bin/bash -c "rm /usr/share/wayland-sessions/labwc.desktop"

echo "Копирование халяль компонентов из LiveCD..."
cp -r /etc/systemd/network /mnt/etc/systemd/
ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
arch-chroot /mnt /bin/bash -c "systemctl enable systemd-resolved"
arch-chroot /mnt /bin/bash -c "systemctl enable systemd-networkd.service systemd-resolved.service"
arch-chroot /mnt /bin/bash -c "systemctl disable dhcpcd"
arch-chroot /mnt /bin/bash <<'EOF'
cat > /etc/systemd/resolved.conf <<'EOC'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9
DNSStubListener=yes
EOC
EOF


cp /etc/os-release /mnt/etc/os-release
mkdir -p /mnt/usr/local/bin
cp /usr/local/bin/halal /mnt/usr/local/bin/halal
cp /usr/local/bin/halalfetch /mnt/usr/local/bin/halalfetch
cp /usr/local/bin/uzupdate /mnt/usr/local/bin/uzupdate
cp /usr/local/bin/sing-box /mnt/usr/local/bin/sing-box

chmod +x /mnt/usr/local/bin/halal
chmod +x /mnt/usr/local/bin/sing-box
chmod +x /mnt/usr/local/bin/halalfetch
chmod +x /mnt/usr/local/bin/uzupdate

arch-chroot /mnt /bin/bash -c "uzupdate"

echo 'Da.'

echo
echo -e "${F_DIM}---------------------------------------------------------------${NO_FORMAT}"
echo -e "${C_GREEN}УСТАНОВКА UZBEK LINUX ЗАВЕРШЕНА.${NO_FORMAT}"
