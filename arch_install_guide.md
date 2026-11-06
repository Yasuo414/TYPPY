# Arch Linux — Instalační průvodce
## Lenovo ThinkPad T460 (20FMS5DX18)
### i5-6300U · Intel HD 520 · 238GB SATA SSD · 15GiB RAM · 2× baterie

> Původní verze průvodce pro HP Pavilion Gaming (Ryzen + RTX 3050) zůstala
> vedle jako `arch_install_guide.hp-pavilion.md`.

---

## ⚠ NEŽ ZAČNEŠ — záloha

Instalace **smaže celý `/dev/sda`**, včetně současného NixOS. Na disku je teď
věci, které v zálohe z 6. 11. **nejsou**:

- `~/.claude/` — obnovené paměti ze čtyř projektů (TCTM/Voxia, škola, DSP-SVC),
  sloučená `history.jsonl`, transkripty, `~/.claude/restore/`
- `~/ukui-nix/` — rozpracované balení UKUI pro Nix (5 hotových derivací + README
  s porting poznatky)
- `~/CLAUDE.md`

```bash
# Připoj Ventoy USB a zazipuj to, co je nového
cd ~
tar czf /run/media/onnx4144/Ventoy/zaloha-pred-archem-$(date +%F).tar.gz \
    .claude .claude.json CLAUDE.md ukui-nix

# Ověř, že to tam je a dá se to otevřít
tar tzf /run/media/onnx4144/Ventoy/zaloha-pred-archem-*.tar.gz | wc -l
sync
```

Zálohu si **ověř ještě před** `fdisk`em. Po `w` už není cesta zpět.

---

## Stav UKUI na Archu — čti dřív, než budeš plánovat desktop

Arch **má** skupinu `ukui` (https://archlinux.org/groups/x86_64/ukui/), takže
`sudo pacman -S ukui` projde. Ale těch 11 balíčků ve skupině jsou **periferní
utility, ne desktop**:

| V `ukui` skupině je | Co to je |
|---|---|
| `kylin-burner`, `kylin-scanner`, `kylin-process-manager` | vypalovačka, skener, správce procesů |
| `kylin-virtual-keyboard`, `libinput-touch-translator` | virtuální klávesnice, gesta |
| `kylin-nm-plugin`, `libkylin-nm-base` | NetworkManager applet |
| `biometric-authentication` | framework pro čtečku prstů |
| `indicator-china-weather` | počasí v panelu |
| `ukui-input-gather`, `ukui-service-manager` | pomocné služby |

Ve skupině **není** session manager, window manager, panel, menu ani file
manager. Ani jinde v repozitářích: `ukui-session-manager`, `ukui-panel`,
`ukui-menu`, `ukwm`, `ukui-kwin`, `peony`, `ukui-settings-daemon`,
`ukui-control-center`, `ukui-greeter` — žádný z nich v `core`/`extra` ani v AUR
není (ověřeno 6. 11. 2026). V AUR jsou z UKUI jen tři drobky: `parchives`,
`qt5-ukui-platformtheme-git`, `ukui-notification-daemon`.

Instrukce `pacman -S ukui xorg-server && systemctl enable lightdm` v README
projektu UKUI jsou tedy **zavádějící** — příkaz funguje, ale desktop ti
nenaběhne, protože lightdm nebude mít jakou session nabídnout.

**Dobrá zpráva:** na Archu je dopsání zbytku *výrazně* snazší než na NixOS, a to
ze tří konkrétních důvodů:

| | NixOS 26.05 | Arch |
|---|---|---|
| KF5 (`kwindowsystem`, `kidletime`) | veřejně odstraněno, `kidletime` jsem musel vendorovat z Attic | `kwindowsystem5` a `kidletime5` **5.116.0 v `extra`** |
| QtWebKit (potřebuje ho panelový plugin s hodinami) | není vůbec → panel bez hodin | `qt5-webkit` **v AUR** → hodiny půjdou |
| Zadrátované `/usr/share`, `/etc/xdg/autostart`, `$$[QT_INSTALL_*]` | musí se patchovat ve každé komponentě | **funguje bez patchů** |

Chybí jen dvě závislosti (`gsettings-qt` není ani v repo, ani v AUR;
`libqtxdg` je 4.5.0 = Qt6, UKUI 3.x chce Qt5) plus PKGBUILDy samotných
komponent. Postup je v sekci [UKUI ze zdrojů](#krok-13--ukui-ze-zdrojů) dole.

**Proto tento průvodce nainstaluje nejdřív funkční desktop, aby byl stroj po
rebootu použitelný, a UKUI se staví až na něm.**

---

## Příprava

### 1. Stáhni Arch ISO
- https://archlinux.org/download/
- CZ mirror: ftp.sh.cvut.cz

### 2. Zapiš na USB
Ventoy už na tom USB máš — stačí ISO **nakopírovat** na Ventoy partici, nic
přepisovat nemusíš (a nepřijdeš tím o zálohy, které na tom USB leží).

```bash
cp archlinux-*.iso /run/media/onnx4144/Ventoy/
sync
```

### 3. Bootni z USB — ThinkPad klávesy
- Restartuj → **F12** = boot menu → vyber USB
- **F1** = BIOS setup (pokud potřebuješ nastavení změnit)

V BIOSu ověř:
- `Security → Secure Boot → Disabled`
- `Startup → UEFI/Legacy Boot → UEFI Only`
- `Config → Thunderbolt/USB → USB UEFI BIOS Support → Enabled`

Pak vyber „Arch Linux install medium".

---

## Instalace (v live prostředí)

### Krok 1 — Klávesnice a internet

```bash
# České rozložení
loadkeys cz-qwertz

# WiFi — Intel Wireless 8260 (PCI 8086:24F3, driver iwlwifi).
# Firmware je součástí linux-firmware, takže funguje out-of-the-box.
iwctl
# Uvnitř iwctl:
#   device list                      ← ověř jméno (v live ISO bývá wlan0)
#   station wlan0 scan
#   station wlan0 get-networks
#   station wlan0 connect "NázevSítě"
#   exit

# Nebo kabelem — ethernet je Intel e1000e, taky funguje hned
# (v nainstalovaném systému se rozhraní jmenují wlp4s0 a enp0s31f6)

ping -c 3 archlinux.org
timedatectl set-ntp true
```

### Krok 2 — Partice

Pozor: disk je **SATA, tedy `/dev/sda`** — ne `nvme0n1` jako u HP.

```bash
lsblk
# Uvidíš: sda  238.5G   (SanDisk SD8TB8U2)
# A současné NixOS partice sda1/sda2/sda3 — ty zmizí.

fdisk /dev/sda
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
[Enter]    ← zbytek disku (~237 GB)

w          ← zapsat a ukončit
```

### Krok 3 — Formátování

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
```

### Krok 4 — Mountování

```bash
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

### Krok 5 — Základní systém

```bash
reflector --country Czechia,Germany,Poland --protocol https --sort rate \
    --save /etc/pacman.d/mirrorlist

# intel-ucode místo amd-ucode (Skylake i5-6300U)
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers \
    intel-ucode \
    networkmanager \
    nano \
    git \
    sudo
```

> **Tip pro 2016 hardware:** přihoď `linux-lts linux-lts-headers` jako záložní
> kernel. Když ti nový kernel rozbije i915 nebo Wi-Fi, nabootuješ do LTS
> a nemusíš řešit chroot z USB.

### Krok 6 — Konfigurace systému

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

**Odtud jsi UVNITŘ nového systému:**

```bash
ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime
hwclock --systohc

nano /etc/locale.gen
# Odkomentuj:
#   cs_CZ.UTF-8 UTF-8
#   en_US.UTF-8 UTF-8
locale-gen

# Na NixOS jsi měl LANG=en_US.UTF-8. Když chceš zachovat anglický systém
# s českým formátováním, použij druhou variantu.
echo "LANG=cs_CZ.UTF-8" > /etc/locale.conf
#   nebo:
#   printf 'LANG=en_US.UTF-8\nLC_TIME=cs_CZ.UTF-8\nLC_PAPER=cs_CZ.UTF-8\nLC_MEASUREMENT=cs_CZ.UTF-8\n' > /etc/locale.conf

echo "KEYMAP=cz-qwertz" > /etc/vconsole.conf

echo "voxia" > /etc/hostname
cat > /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   voxia.localdomain voxia
EOF
```

### Krok 7 — Swap file

Máš 15 GiB RAM. 8 GB swapu stačí na běžný provoz i na to, aby ti nespadla
binarizace datasetu.

```bash
# Na ext4 použij fallocate — rychlejší než dd
fallocate -l 8G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
```

<details>
<summary><b>Chceš hibernaci?</b> (u notebooku se dvěma bateriemi se to hodí)</summary>

Swap musí být **aspoň jako RAM**, tedy 16 GB, a kernel musí vědět, kam se
probouzet:

```bash
fallocate -l 17G /swapfile      # místo 8G výše
# …mkswap/swapon/fstab stejně…

# Offset a UUID pro resume
filefrag -v /swapfile | awk 'NR==4 {gsub("\\.\\.",""); print "offset:", $4}'
findmnt -no UUID -T /swapfile

# Do /etc/mkinitcpio.conf přidej hook `resume` ZA `filesystems`:
nano /etc/mkinitcpio.conf
#   HOOKS=(base udev autodetect microcode modconf kms keyboard keymap \
#          consolefont block filesystems resume fsck)
mkinitcpio -P

# A do boot entry (Krok 9) přidej:
#   resume=UUID=<uuid-root-fs> resume_offset=<offset>
```
</details>

### Krok 8 — Uživatel

```bash
passwd

# Skupina video/input kvůli backlightu a trackpointu
useradd -m -G wheel,video,audio,input -s /bin/bash onnx4144
passwd onnx4144

EDITOR=nano visudo
# Odkomentuj:
#   %wheel ALL=(ALL:ALL) ALL
```

### Krok 9 — Bootloader (systemd-boot)

```bash
bootctl install

cat > /boot/loader/loader.conf << EOF
default arch.conf
timeout 3
console-mode max
editor no
EOF

# PARTUUID je robustnější než /dev/sda2 — přežije přehození disku i přidání
# druhého (T460 má Ultrabay, kam se dá vložit HDD caddy).
ROOT_PARTUUID=$(blkid -s PARTUUID -o value /dev/sda2)

cat > /boot/loader/entries/arch.conf << EOF
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=PARTUUID=$ROOT_PARTUUID rw quiet
EOF

# Pokud jsi nainstaloval linux-lts, přidej i záložní entry:
cat > /boot/loader/entries/arch-lts.conf << EOF
title   Arch Linux (LTS)
linux   /vmlinuz-linux-lts
initrd  /intel-ucode.img
initrd  /initramfs-linux-lts.img
options root=PARTUUID=$ROOT_PARTUUID rw quiet
EOF
```

> `intel-ucode.img` **musí** být uvedený před `initramfs-linux.img` — jinak se
> mikrokód nenahraje včas.

### Krok 10 — Grafika (Intel HD 520, žádná dedikovaná)

Celá NVIDIA sekce z původního průvodce **padá**. T460 má jen integrovanou
Skylake Gen9 grafiku (`/sys/class/drm` → jen `card1`).

```bash
pacman -S mesa vulkan-intel intel-media-driver libva-utils vulkan-tools

# Pro 32bit hry (Steam/Wine) — vyžaduje povolený [multilib] v /etc/pacman.conf
# pacman -S lib32-mesa lib32-vulkan-intel
```

> **Neinstaluj `xf86-video-intel`.** Ten starý DDX driver na Skylake dělá
> tearing a zamrzání; kernelový `modesetting` (součást Xorgu) je spolehlivější.
> Na Waylandu se stejně nepoužije.

### Krok 11 — ThinkPad specifika

Tohle v původním průvodci vůbec nebylo a na T460 to dělá velký rozdíl.

```bash
# Správa napájení. tlp-rdw = automatické vypínání Wi-Fi při zapojení kabelu
pacman -S tlp tlp-rdw powertop acpi

# Nabíjecí prahy pro OBĚ baterie (BAT0 interní + BAT1 v Ultrabay).
# Potřebuje tpacpi-bat, který je součástí tlp, a modul acpi_call.
pacman -S acpi_call-dkms     # nebo acpi_call, pokud nejedeš dkms kernel
echo "acpi_call" > /etc/modules-load.d/acpi_call.conf

# Firmware/BIOS update přímo z Linuxu — Lenovo T460 je ve LVFS
pacman -S fwupd

# Bluetooth
pacman -S bluez bluez-utils

systemctl enable tlp
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable fstrim.timer    # TRIM pro ten SanDisk SSD (rotational=0)
```

Prahy nabíjení se pak ladí v `/etc/tlp.conf`:
```
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
START_CHARGE_THRESH_BAT1=75
STOP_CHARGE_THRESH_BAT1=80
```

### Krok 12 — Desktop, zvuk, fonty

```bash
# Zvuk (Skylake HDA — snd_hda_intel, žádný SOF firmware netřeba)
pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber

# Fonty — čeština + CJK (japonština/čínština/korejština, co se učíš,
# a CJK labely v SVS datasetech)
pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji ttf-liberation \
    ttf-dejavu adobe-source-han-sans-otc-fonts

pacman -S firefox flatpak
```

**Desktop.** Protože UKUI session se musí doteprve postavit (viz sekce výše),
nainstaluj něco funkčního, na co se po rebootu přihlásíš. Dvě rozumné volby:

```bash
# A) LXQt — Qt5, lehké, nejblíž tomu, jak UKUI vypadá a funguje.
#    Navíc tím předinstaluješ půlku Qt5 závislostí, které UKUI potřebuje.
pacman -S lxqt breeze-icons xorg-server sddm
systemctl enable sddm

# B) KDE Plasma — co máš teď na NixOS, takže žádná adaptace
# pacman -S plasma-meta konsole dolphin sddm
# systemctl enable sddm
```

> `lightdm` z UKUI README nech na později — `ukui-greeter` v repozitářích není,
> takže by ti lightdm nic navíc nepřinesl. SDDM zvládne přidat UKUI session,
> jakmile ji postavíš.

### Krok 13 — Hotovo, restartuj

```bash
exit
umount -R /mnt
reboot        # a vytáhni USB
```

---

## Po prvním bootu

### AUR helper

```bash
nmcli device wifi connect "NázevSítě" password "heslo"

cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

### Ověř, že hardware jede

```bash
# Grafická akcelerace
vainfo | head                       # VA-API přes intel-media-driver
vulkaninfo --summary | head -20
glxinfo | grep "OpenGL renderer"    # čekej: Mesa Intel(R) HD Graphics 520

# Obě baterie
upower -e
cat /sys/class/power_supply/BAT{0,1}/capacity

# Prahy nabíjení se aplikovaly
sudo tlp-stat -b | head -30

# Firmware updaty
fwupdmgr refresh && fwupdmgr get-updates
```

### Software

```bash
# Python ML stack — CPU-only! Tenhle stroj nemá dedikovanou GPU,
# takže žádná cu124/CUDA větev (a `cuda`/`cudnn` z původního průvodce
# by tu byly jen mrtvá váha).
sudo pacman -S python-pytorch        # repo verze, CPU, přes OpenBLAS
#   nebo přes pip do venv:
#   pip install torch torchvision torchaudio \
#       --index-url https://download.pytorch.org/whl/cpu

# Conda, když chceš prostředí jako Voxia-TCTM z macOS
yay -S miniconda3

# Pro TCTM/Voxia práci
sudo pacman -S python-numpy python-scipy python-librosa ffmpeg sox
yay -S python-pyworld

sudo pacman -S code              # nebo: yay -S visual-studio-code-bin
sudo pacman -S prismlauncher     # Minecraft na HD 520 pojede

flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install flathub com.usebottles.bottles
```

> **DaVinci Resolve vynech.** V původním průvodci byl kvůli RTX 3050. Resolve
> vyžaduje podporovanou OpenCL/CUDA GPU a na Intel HD 520 buď nenastartuje,
> nebo bude nepoužitelný. Na stříhání tady posluž `kdenlive` nebo `shotcut`.

> **`prime-run` a `nvidia` alias** z původního průvodce taky nemají smysl —
> není co přepínat, je tu jedna GPU.

---

## Krok 13 — UKUI ze zdrojů

Až bude systém stát a budeš mít čas. Pořadí je zdola nahoru, každý krok ověř
`makepkg` / spuštěním, ne až na konci.

### Co je potřeba dodat

```
1. gsettings-qt          ← PKGBUILD od nuly (není v repo ani v AUR)
                            zdroj: Launchpad / Ubuntu source package
2. libqtxdg 3.12.0       ← PKGBUILD; repo má 4.5.0, což je Qt6-only.
                            3.12.0 je poslední Qt5 verze.
   + lxqt-build-tools 0.13.0 (poslední Qt5-era, kvůli find_package jménu)
3. qt5-webkit            ← yay -S qt5-webkit   (je v AUR, 5.212.0alpha4)
4. qt5-ukui-platformtheme ← yay -S qt5-ukui-platformtheme-git (už v AUR!)
```

Z repozitářů si vezmeš zdarma to, co jsem na NixOS musel vendorovat:
`kwindowsystem5`, `kidletime5` (oba 5.116.0), `mate-desktop`, `libmatekbd`,
`libmatemixer`, `libxklavier`, `uchardet`, `libdbusmenu-qt5`, `qt5-x11extras`,
`qt5-multimedia`.

### Komponenty UKUI, v tomto pořadí

| # | Komponenta | Tag | Build systém |
|---|---|---|---|
| 1 | `ukui-themes` | v3.0.1 | meson (data) |
| 2 | `ukui-wallpapers` | v20.04.4 | data |
| 3 | `ukui-settings-daemon` | v3.1.1.1 | qmake |
| 4 | `ukwm` | v1.2.3 | autotools (fork mutteru) |
| 5 | `ukui-session-manager` | v3.0.5 | cmake |
| 6 | `ukui-menu` | v3.0.3 | qmake |
| 7 | `ukui-panel` | v3.0.6.4 | cmake |
| 8 | `peony` | v3.2.4 | qmake (file manager + ikony na ploše) |

```bash
curl -sfL https://github.com/ukui/<repo>/archive/refs/tags/<tag>.tar.gz
```

### Co z mých NixOS poznatků platí i na Archu

Většina patchů **odpadá** (cesty `/usr` a `/etc/xdg/autostart` tam prostě jsou,
`$$[QT_INSTALL_*]` míří do skutečného Qt prefixu). Tyto tři ale narazíš i tady,
protože Arch má stejně nový toolchain:

1. **CMake 4 odmítá `cmake_minimum_required < 3.5`** →
   `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`
2. **GCC 15 udělal z `-Wincompatible-pointer-types` chybu.** Potká to vendorovaný
   cogl v `ukwm` (`cogl-path.c` + `gluTessCallback`) →
   `CFLAGS+=" -Wno-error=incompatible-pointer-types"`
3. **`ukwm` configure má obrácenou logiku u dvou flagů.**
   `--disable-remote-desktop` a `--disable-egl-device` je paradoxně **zapnou**
   (třetí argument `AC_ARG_ENABLE` je „akce pokud zadáno", ne „pokud ano").
   Ty flagy prostě nepředávej. Dál: `--enable-compile-warnings=minimum`
   (jinak `-Werror`) a introspection nech **zapnutou**, protože vendorovaný
   clutter chce `Clutter-1.typelib`.

A dva upstream bugy, nezávislé na distru:
- `ukui-session-manager` používá `${GSETTINGS_QT_INCLUDES}`, ale
  `pkg_search_module` plní `_INCLUDE_DIRS`. Na Archu to projde náhodou
  (hlavičky budou v `/usr/include`), na jiném prefixu ne.
- `ukui-panel` a `ukui-menu` linkují `-lukui-log4qt`, což je Kylin SDK knihovna,
  která na GitHubu **není**. Je to jen Qt message handler — vyhoď ji
  z `target_link_libraries` / `LIBS`, smaž `#include <ukui-log4qt.h>`
  a volání `initUkuiLog4qt(...)`.

Detailní zápis (včetně hotových derivací a toho, co přesně se kde patchovalo)
je v záloze v `ukui-nix/README.md`.

### Zprovoznění session

`ukui-session` hledá komponenty **jen v `/etc/xdg/autostart`** podle jmen
z gsettings `org.ukui.session.required-components`
(default `ukwm` / `ukui-panel` / `peony-qt-desktop`). Na Archu tam ty
`.desktop` soubory komponenty nainstalují samy, takže by to mělo naskočit
bez zasahování. WM má navíc fallback na
`/usr/share/applications/ukwm.desktop` — taky standardní cesta.

Session se v SDDM objeví přes `/usr/share/xsessions/ukui.desktop`, který
instaluje `ukui-session-manager`.

---

## Shrnutí partičního layoutu

```
/dev/sda1      1 GB      EFI (vfat)   /boot
/dev/sda2    ~237 GB     ext4         /
/swapfile      8 GB      swap         (17 GB, pokud chceš hibernaci)
```

## Klíčové příkazy po instalaci

```bash
sudo pacman -Syu           # aktualizuj systém
yay -S balíček             # instaluj z AUR
sudo pacman -Rns balíček   # odinstaluj + závislosti
sudo pacman -Qtdq | sudo pacman -Rns -   # ukliď osiřelé závislosti
sudo tlp-stat -b           # stav baterií a prahů
fwupdmgr get-updates       # BIOS/firmware updaty
```

## Troubleshooting pro tenhle hardware

| Problém | Řešení |
|---|---|
| Blikající/černající displej (Skylake PSR bug) | přidej `i915.enable_psr=0` do `options` v boot entry |
| Tearing na Xorgu | `/etc/X11/xorg.conf.d/20-intel.conf` → `Option "TearFree" "true"` u modesetting driveru |
| Wi-Fi se odpojuje | `/etc/modprobe.d/iwlwifi.conf` → `options iwlwifi power_save=0` |
| Trackpoint nejede | ověř, že jsi ve skupině `input`; `libinput list-devices` |
| Druhá baterie se nevidí | `lsmod \| grep acpi_call`, pak `sudo tlp-stat -b` |
| Po update kernelu nenaběhne X | nabootuj `Arch Linux (LTS)` entry |
