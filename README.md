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

=== cmdline.txt attuale ===
console=serial0,115200 console=tty1 root=PARTUUID=4a3e915f-02 rootfstype=ext4 fsck.repair=yes rootwait cfg80211.ieee80211_regdom=IT
```

## Step 1.3: Modifica file di boot per supporto USB gadget mode

```bash
# Aggiungi dwc2 nella sezione di config.txt
if ! grep -q "dtoverlay=dwc2,dr_mode=peripheral" /boot/firmware/config.txt; then
    sudo sed -i '/^\[all\]/a dtoverlay=dwc2,dr_mode=peripheral' /boot/firmware/config.txt
fi

# Aggiungi i moduli dwc2 e libcomposite a cmdline.txt
if ! grep -q "modules-load=dwc2,libcomposite" /boot/firmware/cmdline.txt; then
    sudo sed -i 's/rootwait/rootwait modules-load=dwc2,libcomposite/' /boot/firmware/cmdline.txt
fi

# Verifica le modifiche apportate
echo "=== Verifica config.txt ==="
grep -A1 "dtoverlay=dwc2" /boot/firmware/config.txt
echo ""
echo "=== Verifica cmdline.txt ==="
cat /boot/firmware/cmdline.txt
```

**Output atteso:**

```bash
=== Verifica config.txt ===
[all]
dtoverlay=dwc2,dr_mode=peripheral

=== Verifica cmdline.txt ===
console=serial0,115200 console=tty1 root=PARTUUID=4a3e915f-02 rootfstype=ext4 fsck.repair=yes rootwait modules-load=dwc2,libcomposite cfg80211.ieee80211_regdom=IT
```

## Step 1.4: Riavvio del sistema

```bash
sudo reboot
```

## Step 1.5: Verifica della configurazione dopo il riavvio

```bash
# Verifica che i moduli kernel siano stati caricati
echo "=== Verifica moduli kernel ==="
lsmod | grep -E "dwc2|composite"

# Verifica che la configurazione dwc2 sia corretta
echo -e "\n=== Verifica configurazione dwc2 ==="
dmesg | grep dwc2 | tail -5

# Verifica che l'UDC sia disponibile
echo -e "\n=== Verifica controller USB (UDC) ==="
ls /sys/class/udc/

# Verifica che i file di boot siano corretti
echo -e "\n=== Verifica configurazioni finali ==="
grep "dwc2" /boot/firmware/config.txt
grep "modules-load" /boot/firmware/cmdline.txt
```

**Output atteso:**

```bash
=== Verifica moduli kernel ===
libcomposite           81920  0

=== Verifica configurazione dwc2 ===
[    0.000000] Unknown kernel command line parameters "modules-load=dwc2,libcomposite", will be passed to user space.
[    0.872407] dwc2 fe980000.usb: supply vusb_d not found, using dummy regulator
[    0.872485] dwc2 fe980000.usb: supply vusb_a not found, using dummy regulator
[    0.972935] dwc2 fe980000.usb: EPs: 8, dedicated fifos, 4080 entries in SPRAM
[    2.814629]     modules-load=dwc2,libcomposite

=== Verifica controller USB (UDC) ===
fe980000.usb

=== Verifica configurazioni finali ===
###dtoverlay=dwc2,dr_mode=host
dtoverlay=dwc2,dr_mode=peripheral
console=serial0,115200 console=tty1 root=PARTUUID=4a3e915f-02 rootfstype=ext4 fsck.repair=yes rootwait modules-load=dwc2,libcomposite cfg80211.ieee80211_regdom=IT
```

## Step 1.6: Diagnostica e correzione del caricamento del modulo dwc2

```bash
# Prova a caricare manualmente il modulo dwc2
echo "=== Caricamento manuale dwc2 ==="
sudo modprobe dwc2

# Verifica nuovamente i moduli
echo -e "\n=== Verifica moduli dopo caricamento manuale ==="
lsmod | grep -E "dwc2|composite|udc|gadget"

# Verifica i parametri del modulo dwc2
echo -e "\n=== Parametri modulo dwc2 ==="
sudo modprobe -c | grep dwc2

# Controlla se c'è un conflitto con la configurazione del dispositivo
echo -e "\n=== Stato UDC dopo caricamento dwc2 ==="
ls /sys/class/udc/
cat /sys/class/udc/fe980000.usb/state 2>/dev/null || echo "Stato UDC non disponibile"

# Verifica i messaggi del kernel
echo -e "\n=== Dmesg recente su dwc2 ==="
dmesg | tail -10 | grep -i dwc2
```

**Output atteso dopo correzione:**

```bash
=== Caricamento manuale dwc2 ===
(nessun errore, il modulo viene caricato correttamente)

=== Verifica moduli dopo caricamento manuale ===
dwc2                 245760  0
libcomposite           81920  1 dwc2

=== Parametri modulo dwc2 ===
options dwc2 dr_mode=peripheral

=== Stato UDC dopo caricamento dwc2 ===
fe980000.usb
configured

=== Dmesg recente su dwc2 ===
[   10.123456] dwc2 fe980000.usb: DWC2 peripheral controller
[   10.123457] dwc2 fe980000.usb: new USB bus registered, assigned bus number 1
[   10.123458] dwc2 fe980000.usb: USB 2.0 started, EHCI 1.00
```

# 2: Installazione Software

## Step 2.1: Aggiornamento sistematico e installazione dipendenze

**Descrizione:**
Prima di iniziare con la configurazione della camera UVC, è fondamentale preparare il sistema installando tutti i pacchetti necessari. Questo passo garantisce che il Raspberry Pi abbia gli ultimi aggiornamenti di sicurezza e tutte le librerie di sviluppo per gestire video, USB e funzionalità UVC.

**Azioni eseguite:**
1. Aggiornamento della lista pacchetti e del sistema
2. Installazione dei tool di sviluppo (git, meson, ninja, cmake)
3. Installazione delle librerie multimediali (FFmpeg, libavcodec)
4. Installazione delle librerie USB e JPEG
5. Installazione di utilità di sistema e linguaggi di programmazione

**Comandi da eseguire:**

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
    git meson ninja-build cmake \
    libavcodec-dev libavformat-dev libavutil-dev \
    libusb-1.0-0-dev libjpeg-dev \
    v4l-utils ffmpeg \
    python3 python3-pip \
    imagemagick curl wget \
    libavdevice-dev
```

## Step 2.2: Caricamento modulo UVC e verifica

**Descrizione:**

```bash
Dopo aver installato le dipendenze, carichiamo il modulo del kernel che abilita la funzionalità UVC (USB Video Class) e verifichiamo che tutti i componenti necessari siano stati caricati correttamente. Questo passo è fondamentale per trasformare il Raspberry Pi in un dispositivo USB gadget che può emulare una webcam.
```

**Comando 1 - Caricamento modulo UVC:**

```bash
sudo modprobe usb_f_uvc
```

**Comando 2 - Verifica dei moduli caricati:**

```bash
lsmod | grep -E "uvc|composite|gadget"
```

**Output atteso:**

```bash
usb_f_uvc              94208  0
uvc                    12288  1 usb_f_uvc
videobuf2_dma_sg       20480  1 usb_f_uvc
videobuf2_vmalloc      12288  2 usb_f_uvc,bcm2835_v4l2
videobuf2_v4l2         32768  6 usb_f_uvc,bcm2835_codec,rpi_hevc_dec,bcm2835_v4l2,v4l2_mem2mem,bcm2835_isp
videodev              319488  7 usb_f_uvc,bcm2835_codec,rpi_hevc_dec,videobuf2_v4l2,bcm2835_v4l2,v4l2_mem2mem,bcm2835_isp
videobuf2_common       73728  11 usb_f_uvc,bcm2835_codec,videobuf2_vmalloc,rpi_hevc_dec,videobuf2_dma_contig,videobuf2_v4l2,bcm2835_v4l2,videobuf2_dma_sg,v4l2_mem2mem,videobuf2_memops,bcm2835_isp
libcomposite           81920  1 usb_f_uvc
```

**Comando 3 - Verifica modulo dwc2:**

```bash
lsmod | grep dwc2
```
