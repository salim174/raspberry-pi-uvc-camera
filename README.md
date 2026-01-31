# Setup Iniziale

## Step 1.1: Controllo sistema

Esegui questo comando per verificare la versione del sistema operativo e le informazioni sul kernel:

```bash
cat /etc/os-release && uname -a
```

**Output atteso:** 

```bash
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixie)"
VERSION_CODENAME=trixie
DEBIAN_VERSION_FULL=13.2
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
Linux raspberry 6.12.47+rpt-rpi-v8 #1 SMP PREEMPT Debian 1:6.12.47-1+rpt1 (2025-09-16) aarch64 GNU/Linux
```
## Step 1.2: Backup e visualizzazione configurazioni

Prima di modificare qualsiasi configurazione, crea un backup dei file critici del boot e visualizza il loro contenuto attuale:

```bash
sudo cp /boot/firmware/config.txt /boot/firmware/config.txt.backup
sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.backup
echo "=== config.txt attuale ==="
head -30 /boot/firmware/config.txt
echo ""
echo "=== cmdline.txt attuale ==="
cat /boot/firmware/cmdline.txt
```

**Output atteso:** 

```bash
=== cmdline.txt attuale ===
console=serial0,115200 console=tty1 root=PARTUUID=4a3e915f-02 rootfstype=ext4 fsck.repair=yes rootwait cfg80211.ieee80211_regdom=IT
```

```bash
echo "=== config.txt attuale ==="
head -30 /boot/firmware/config.txt
```

**Output atteso:** 

```bash
salim@raspberry:~ $ echo "=== config.txt attuale ===" echo "=== config.txt attuale ==="
head -30 /boot/firmware/config.txt
=== config.txt attuale ===
# For more options and information see
# http://rptl.io/configtxt
# Some settings may impact device functionality. See link above for details

# Uncomment some or all of these to enable the optional hardware interfaces
#dtparam=i2c_arm=on
#dtparam=i2s=on
#dtparam=spi=on

# Enable audio (loads snd_bcm2835)
dtparam=audio=on

# Additional overlays and parameters are documented
# /boot/firmware/overlays/README

# Automatically load overlays for detected cameras
camera_auto_detect=1

# Automatically load overlays for detected DSI displays
display_auto_detect=1

# Automatically load initramfs files, if found
auto_initramfs=1

# Enable DRM VC4 V3D driver
dtoverlay=vc4-kms-v3d
max_framebuffers=2

# Don't have the firmware create an initial video= setting in cmdline.txt.
# Use the kernel's default instead.
```

## Step 1.3: Modifica file di boot

```bash
# Aggiungi dwc2 a config.txt (se non già presente)
if ! grep -q "dtoverlay=dwc2" /boot/firmware/config.txt; then
    echo "dtoverlay=dwc2,dr_mode=peripheral" | sudo tee -a /boot/firmware/config.txt
fi

# Modifica cmdline.txt SOLO se non già modificato
if ! grep -q "modules-load=dwc2,libcomposite" /boot/firmware/cmdline.txt; then
    sudo sed -i 's/rootwait/rootwait modules-load=dwc2,libcomposite/' /boot/firmware/cmdline.txt
fi

# Verifica
echo "=== Ultime righe config.txt ==="
tail -3 /boot/firmware/config.txt
echo ""
echo "=== cmdline.txt modificato ==="
cat /boot/firmware/cmdline.txt
```

**Output atteso:** 

```bash
=== cmdline.txt modificato ===
console=serial0,115200 console=tty1 root=PARTUUID=4a3e915f-02 rootfstype=ext4 fsck.repair=yes rootwait modules-load=dwc2,libcomposite cfg80211.ieee80211_regdom=ITsalim@raspberry:~ $
```

```bash
echo "=== Ultime righe config.txt ==="
tail -5 /boot/firmware/config.txt
```

**Output atteso:** 

```bash
salim@raspberry:~ $ echo "=== Ultime recho "=== Ultime righe config.txt ==="
tail -5 /boot/firmware/config.txt
=== Ultime righe config.txt ===
[cm5]
dtoverlay=dwc2,dr_mode=host

[all]
enable_uart=1
salim@raspberry:~ $
```

**Correzione della configurazione dwc2:**

```bash
# Prima, rimuoviamo la riga sbagliata nella sezione [cm5]
sudo sed -i '/\[cm5\]/,/^\[/ s/dtoverlay=dwc2,dr_mode=host/#dtoverlay=dwc2,dr_mode=host/' /boot/firmware/config.txt

# Ora aggiungiamo la riga corretta nella sezione [all]
if ! grep -q "dtoverlay=dwc2,dr_mode=peripheral" /boot/firmware/config.txt; then
    sudo sed -i '/^\[all\]/a dtoverlay=dwc2,dr_mode=peripheral' /boot/firmware/config.txt
fi

# Verifichiamo le modifiche
echo "=== Config.txt dopo correzione ==="
grep -A2 -B2 "dwc2\|\[all\]" /boot/firmware/config.txt
```

## Step 1.4: Riavvio del sistema

```bash
sudo reboot
```
