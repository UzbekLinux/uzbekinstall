#!/bin/bash

FS="ext4"

source /tmp/uzbekinstall.conf

TARGET="/mnt"

if [ ! -d /sys/firmware/efi ]; then
    echo "система не в uefi!"
    exit 1
fi

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

umount -R "$TARGET" 2>/dev/null || true

if [[ "$PART_MODE" == "auto" ]]; then
    if [[ ! -b "$DISK" ]] || [[ "${DISK:0:5}" != "/dev/" ]]; then
        echo -e "ЕБАЛАЙ, такого диска нет!!!"
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
    mkfs."$FS" "$ROOT_PART" 

elif [[ "$PART_MODE" == "manual" ]]; then
    if [[ ! -b "$EFI_PART" ]] || [[ "${EFI_PART:0:5}" != "/dev/" ]] || \
       [[ ! -b "$ROOT_PART" ]] || [[ "${ROOT_PART:0:5}" != "/dev/" ]]; then
        echo -e "ЕБАЛАЙ, у тебя такого раздела нет!!!"
        exit 1
    fi

    echo "Форматирование $ROOT_PART в $FS..."
    mkfs."$FS" "$ROOT_PART" 

    echo "Форматирование $EFI_PART в fat32..."
    mkfs.fat -F 32 -I "$EFI_PART"

else
    echo -e "\033[41;97mСук, ты додик или как\033[0m"
    exit 1
fi

mount "$ROOT_PART" "$TARGET" || { echo -e "Произошла ошибка при монтировании root $ROOT_PART!"; exit 1; }
mount --mkdir "$EFI_PART" "$TARGET/boot" || { echo -e "Произошла ошибка при монтировании EFI $EFI_PART!"; exit 1; }
pacstrap -K "$TARGET" base linux linux-firmware || { echo -e "Произошла ошибка при установке базовой системы!"; exit 1; }
genfstab -U "$TARGET" >> $TARGET/etc/fstab

echo 'ставим HALAL время...'
arch-chroot "$TARGET" /bin/bash -c "ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime" || { echo -e "Что-та не так произошло!!1 ты лох."; exit 1; }
arch-chroot "$TARGET" /bin/bash -c "
if ! curl -s https://www.google.com | grep -q '<title>Google</title>'; then
    echo -e 'интернета нет!!!:( '
    exit 1
fi
"
arch-chroot "$TARGET" /bin/bash -c "hwclock --systohc"
echo 'Генерация локалелей (их не будет)...'
arch-chroot "$TARGET" /bin/bash -c "sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen"
arch-chroot "$TARGET" /bin/bash -c "sed -i 's/^#ru_RU.UTF-8 UTF-8/ru_RU.UTF-8 UTF-8/' /etc/locale.gen"
arch-chroot "$TARGET" /bin/bash -c "locale-gen"

arch-chroot "$TARGET" /bin/bash -c "echo '$HOSTNAME' > /etc/hostname"

arch-chroot "$TARGET" /bin/bash -c "echo -e \"$ROOT_PASSWORD\n$ROOT_PASSWORD\" | passwd"

arch-chroot "$TARGET" /bin/bash -c "mount $EFI_PART /boot"
if [ "$BOOTLOADER" = "rEFInd" ]; then
    arch-chroot "$TARGET" /bin/bash -c "pacman -S --noconfirm refind"
    arch-chroot "$TARGET" /bin/bash -c "refind-install"
    
    mkdir -p /mnt/boot/EFI/refind/themes/
    
    git clone --depth 1 \
      https://github.com/UzbekLinux/uzbek-refind-theme \
      /mnt/boot/EFI/refind/themes/uzbek
    
    KERNEL_PARAMS="root=$ROOT_PART rw"
    REFIND_CONF="$TARGET/boot/refind_linux.conf"
    KERNELS=$(ls "$TARGET/boot"/vmlinuz-* 2>/dev/null || echo "")
    
    > "$REFIND_CONF"
    for KERNEL in $KERNELS; do
        BASENAME=$(basename "$KERNEL")
        INITRD="/initramfs-${BASENAME#vmlinuz-}.img"
        echo "\"Uzbek Linux ($BASENAME)\" \"$KERNEL_PARAMS initrd=$INITRD\"" >> "$REFIND_CONF"
    done
    
    arch-chroot "$TARGET" /bin/bash -c "cat > /boot/EFI/refind/refind.conf <<EOF
    timeout 20
    use_nvram false
    
    include themes/uzbek/theme.conf
    EOF"
elif [ "$BOOTLOADER" = "GRUB" ]; then
    arch-chroot /mnt /bin/bash -c "pacman -S --noconfirm grub efibootmgr"
    arch-chroot /mnt /bin/bash -c "grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB"
    arch-chroot /mnt /bin/bash -c "grub-mkconfig -o /boot/grub/grub.cfg"
elif [ "$BOOTLOADER" = "Limine" ]; then
    arch-chroot "$TARGET" /bin/bash -c "pacman -S --noconfirm limine efibootmgr"
    arch-chroot "$TARGET" /bin/bash -c "
        mkdir -p /boot/EFI/arch-limine
        cp /usr/share/limine/BOOTX64.EFI /boot/EFI/arch-limine/
    "

    arch-chroot "$TARGET" /bin/bash -c "
        efibootmgr \
          --create \
          --disk $DISK \
          --part 1 \
          --label 'Uzbek Linux' \
          --loader '\\EFI\\arch-limine\\BOOTX64.EFI' \
          --unicode
    "

    cat > "$TARGET/boot/EFI/arch-limine/limine.conf" <<EOF
timeout: 5

/Uzbek Linux
    protocol: linux
    path: boot():/vmlinuz-linux
    module_path: boot():/initramfs-linux.img
    cmdline: root=$ROOT_PART rw
EOF
else
    exit 1
fi
arch-chroot "$TARGET" /bin/bash -c "mkinitcpio -P"

arch-chroot "$TARGET" /bin/bash -c "pacman -S --noconfirm git python-pip labwc python3 tk swaybg nwg-panel nwg-drawer nwg-menu python-pyqt6 jq --needed"
arch-chroot "$TARGET" /bin/bash -c "mkdir /tmp"
arch-chroot "$TARGET" /bin/bash -c "cd /tmp && git clone https://github.com/ZDesktopEnvironment/ZDE && cd ZDE && cp -rfv tree/* /"
arch-chroot "$TARGET" /bin/bash -c "cd /tmp && git clone https://github.com/ZDesktopEnvironment/ZSysConf && cd ZSysConf && cp -rfv tree/* /"

if [ "$DM" = "SDDM" ]; then
    arch-chroot "$TARGET" /bin/bash -c "pacman -S --noconfirm sddm"
    arch-chroot "$TARGET" /bin/bash -c "systemctl enable sddm.service"

elif [ "$DM" = "LightDM" ]; then
    arch-chroot "$TARGET" /bin/bash -c "pacman -S --noconfirm lightdm lightdm-gtk-greeter"
    arch-chroot "$TARGET" /bin/bash -c "systemctl enable lightdm.service"

else
    exit 1
fi


arch-chroot "$TARGET" /bin/bash -c "useradd -m -g users -G wheel,video,audio -s /bin/bash $USERNAME"
arch-chroot "$TARGET" /bin/bash -c "echo -e \"$USER_PASSWORD\n$USER_PASSWORD\" | passwd $USERNAME"

echo 'Установка UZBEK-APPS...'

arch-chroot "$TARGET" /bin/bash -c "pacman -S --noconfirm python-pip sudo python"
echo 'Установка SPM...'
arch-chroot "$TARGET" /bin/bash -c "curl -s https://zenusus.serv00.net/dl/installSPM.sh | bash"
arch-chroot "$TARGET" /bin/bash -c "spm add https://raw.githubusercontent.com/lutit/UzbekGramDesktop/refs/heads/uzbekgram/repo.json"
arch-chroot "$TARGET" /bin/bash -c "spm add https://msh356.ru/spm/"

echo 'Установка HALAL софт...'
PACKAGES=("halalIDE" "320totalsecurity" "eblan-editor" "eblan-music-editor" "eblanoffice" "uzbekgram-desktop" "uzbeknetwork")

for pkg in "${PACKAGES[@]}"; do
    arch-chroot $TARGET /bin/bash -c "
        timeout --signal=SIGKILL 30s bash -c '
            python -m venv /tmp/venv_$pkg &&
            source /tmp/venv_$pkg/bin/activate &&
            spm install $pkg &&
            deactivate
        '
    "
done

arch-chroot "$TARGET" /bin/bash -c "pacman -S --noconfirm firefox alacritty mako wlr-randr nano micro pipewire pipewire-pulse libnotify python-pyqt5 swaybg nwg-drawer nwg-menu jq dhcpcd iw wpa_supplicant"
arch-chroot "$TARGET" /bin/bash -c "echo '%wheel ALL=(ALL:ALL) ALL' >> /etc/sudoers"


echo 'ПРОИЗВОДСТВО HALAL.NET...'

echo "Удаление харам labwc.desktop (чтобы только ZDE был)..."
arch-chroot "$TARGET" /bin/bash -c "rm /usr/share/wayland-sessions/labwc.desktop"

echo "Копирование халяль компонентов из LiveCD..."

mkdir -p $TARGET/etc/xdg/zde
arch-chroot "$TARGET" /bin/bash -c 'cat > /etc/xdg/zde/config.json <<EOF
{
    "wallpaper_type": "image",
    "wallpaper": {
        "path": "/usr/share/wallpapers/uzbek-linux-3.png",
        "mode": "stretch"
    },
    "interface_menu": "drawer",
    "autostart": [
        "echo привет как дела"
    ],
    "session_terminal": "alacritty"
}
EOF'

ln -sf /run/systemd/resolve/stub-resolv.conf $TARGET/etc/resolv.conf
arch-chroot "$TARGET" /bin/bash -c "systemctl enable systemd-resolved uzbeknetwork"
arch-chroot "$TARGET" /bin/bash -c "systemctl disable systemd-networkd.service"
arch-chroot "$TARGET" /bin/bash -c "systemctl disable dhcpcd"
arch-chroot "$TARGET" /bin/bash <<'EOF'
cat > /etc/systemd/resolved.conf <<'EOC'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9
DNSStubListener=yes
EOC
EOF


cp /etc/os-release $TARGET/etc/os-release
mkdir -p $TARGET/usr/local/bin
cp /usr/local/bin/halal $TARGET/usr/local/bin/halal
cp /usr/local/bin/halalfetch $TARGET/usr/local/bin/halalfetch
cp /usr/local/bin/uzupdate $TARGET/usr/local/bin/uzupdate
cp /usr/local/bin/sing-box $TARGET/usr/local/bin/sing-box

chmod +x $TARGET/usr/local/bin/halal
chmod +x $TARGET/usr/local/bin/sing-box
chmod +x $TARGET/usr/local/bin/halalfetch
chmod +x $TARGET/usr/local/bin/uzupdate

arch-chroot "$TARGET" /bin/bash -c "uzupdate --force-installed"

echo 'Da.'

echo
echo -e "---------------------------------------------------------------"
echo -e "УСТАНОВКА UZBEK LINUX ЗАВЕРШЕНА."
