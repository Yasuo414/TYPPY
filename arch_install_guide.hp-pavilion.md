# Arch Linux — Instalační průvodce
## HP Pavilion Gaming Laptop 15-ec2xxx
### Ryzen 5 5600H · RTX 3050 · 476GB NVMe · 16GB RAM

---

## Příprava

### 1. Stáhni Arch ISO
- https://archlinux.org/download/
- Vyber nejbližší mirror (CZ: ftp.sh.cvut.cz)

### 2. Zapiš na USB
```bash
# Na Manjaro (před smazáním):
sudo dd if=archlinux-*.iso of=/dev/sdX bs=4M status=progress
sync
# Nebo použij Ventoy / balenaEtcher
```

### 3. Bootni z USB
- Restartuj → F9 (HP boot menu) → vyber USB
- Vyber "Arch Linux install medium"

---

## Instalace (v live prostředí)

### Krok 1 — Klávesnice a internet

```bash
# České rozložení
loadkeys cz-qwertz

# WiFi (tvůj RTL8852AE)
iwctl
# Uvnitř iwctl:
#   station wlan0 scan
#   station wlan0 get-networks
#   station wlan0 connect "NázevSítě"
#   exit

# Ověř internet
ping -c 3 archlinux.org

# Synchronizuj hodiny
timedatectl set-ntp true
```

### Krok 2 — Partice

```bash
# Zkontroluj disk
lsblk
# Uvidíš: nvme0n1  476.9G

# Smaž vše a vytvoř nové partice
fdisk /dev/nvme0n1
```

V `fdisk`:
```
g          ← nová GPT tabulka (SMAŽE VŠE)
n          ← nová partice
1          ← číslo 1
[Enter]    ← výchozí start
+1G        ← EFI partice (1 GB)
t          ← změnit typ
1          ← EFI System

n          ← nová partice
2          ← číslo 2
[Enter]    ← výchozí start
[Enter]    ← zbytek disku (~475 GB)

w          ← zapsat a ukončit
```

### Krok 3 — Formátování

```bash
# EFI partice
mkfs.fat -F32 /dev/nvme0n1p1

# Root partice (ext4)
mkfs.ext4 /dev/nvme0n1p2
```

### Krok 4 — Mountování

```bash
# Root
mount /dev/nvme0n1p2 /mnt

# EFI
mount --mkdir /dev/nvme0n1p1 /mnt/boot
```

### Krok 5 — Základní systém

```bash
# Vyber nejrychlejší mirrory
reflector --country Czechia,Germany,Poland --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Nainstaluj základ
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers \
    amd-ucode \
    networkmanager \
    nano \
    git \
    sudo
```

### Krok 6 — Konfigurace systému

```bash
# Generuj fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Chroot do nového systému
arch-chroot /mnt
```

**Odtud jsi UVNITŘ nového systému:**

```bash
# Časová zóna
ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime
hwclock --systohc

# Locale
nano /etc/locale.gen
# Odkomentuj tyto řádky (smaž # na začátku):
#   cs_CZ.UTF-8 UTF-8
#   en_US.UTF-8 UTF-8

locale-gen

echo "LANG=cs_CZ.UTF-8" > /etc/locale.conf
echo "KEYMAP=cz-qwertz" > /etc/vconsole.conf

# Hostname
echo "voxia" > /etc/hostname

# Hosts
cat > /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   voxia.localdomain voxia
EOF
```

### Krok 7 — Swap file

```bash
# 8 GB swap (polovina RAM — dost pro hibernaci nepotřebuješ)
dd if=/dev/zero of=/swapfile bs=1M count=8192 status=progress
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Přidej do fstab
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
```

### Krok 8 — Uživatel

```bash
# Root heslo
passwd

# Tvůj uživatel
useradd -m -G wheel,video,audio,input -s /bin/bash onnx4144
passwd onnx4144

# Sudo přístup
EDITOR=nano visudo
# Najdi a odkomentuj řádek:
#   %wheel ALL=(ALL:ALL) ALL
```

### Krok 9 — Bootloader (systemd-boot)

```bash
# Nainstaluj bootloader
bootctl install

# Loader konfigurace
cat > /boot/loader/loader.conf << EOF
default arch.conf
timeout 3
console-mode max
editor no
EOF

# Boot entry
cat > /boot/loader/entries/arch.conf << EOF
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options root=/dev/nvme0n1p2 rw quiet
EOF
```

### Krok 10 — NVIDIA + GNOME

```bash
# NVIDIA proprietary driver
pacman -S nvidia nvidia-utils nvidia-settings lib32-nvidia-utils

# GNOME (vanilla — přesně jako Ubuntu)
pacman -S gnome gdm

# Zvuk
pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber

# Fonty (čeština + CJK pro SVC datasety)
pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji ttf-liberation

# Síťové nástroje
pacman -S firefox flatpak

# Povol služby
systemctl enable gdm
systemctl enable NetworkManager
```

### Krok 11 — Dolaď NVIDIA

```bash
# Kernel moduly při bootu
cat > /etc/mkinitcpio.conf.d/nvidia.conf << EOF
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
EOF

# Regeneruj initramfs
mkinitcpio -P

# DRM modeset (pro Wayland)
# Uprav boot entry:
nano /boot/loader/entries/arch.conf
# K řádku options přidej: nvidia_drm.modeset=1
# Výsledek:
#   options root=/dev/nvme0n1p2 rw quiet nvidia_drm.modeset=1

# Pacman hook — automaticky regeneruj initramfs při NVIDIA updatu
mkdir -p /etc/pacman.d/hooks
cat > /etc/pacman.d/hooks/nvidia.hook << 'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = nvidia

[Action]
Description = Regenerating initramfs after NVIDIA driver update...
Depends = mkinitcpio
When = PostTransaction
Exec = /usr/bin/mkinitcpio -P
EOF
```

### Krok 12 — Hotovo, restartuj

```bash
# Opusť chroot
exit

# Odmontuj
umount -R /mnt

# Restartuj (vytáhni USB)
reboot
```

---

## Po prvním bootu

### Přihlásit se a doladit

```bash
# Připoj se k WiFi přes GNOME Settings (nebo nmcli)
nmcli device wifi connect "NázevSítě" password "heslo"

# AUR helper (yay)
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si

# Conda / Miniconda (pro ML)
yay -S miniconda3
# Nebo:
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh

# Flatpak aplikace
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

# Bottles (pro Wine/SynthV2)
flatpak install flathub com.usebottles.bottles

# DaVinci Resolve
yay -S davinci-resolve

# Prism Launcher (Minecraft)
sudo pacman -S prismlauncher

# VS Code
yay -S visual-studio-code-bin

# CUDA toolkit (pro ML trénink)
sudo pacman -S cuda cudnn

# Python ML stack
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

### prime-run pro NVIDIA

```bash
# Ověř že obě GPU fungují
lspci -k | grep -A 2 VGA

# Spouštěj aplikace na NVIDIA
prime-run glxinfo | grep "OpenGL renderer"
# Mělo by ukázat: NVIDIA GeForce RTX 3050

# Alias pro pohodlí
echo 'alias nv="prime-run"' >> ~/.bashrc
# Pak: nv davinci-resolve, nv prismlauncher, atd.
```

---

## Shrnutí partičního layoutu

```
/dev/nvme0n1p1    1 GB     EFI    /boot
/dev/nvme0n1p2  ~475 GB    ext4   /
/swapfile         8 GB     swap
```

## Klíčové příkazy po instalaci

```bash
sudo pacman -Syu          # aktualizuj systém
yay -S balíček             # instaluj z AUR
sudo pacman -Rns balíček   # odinstaluj + závislosti
```
