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

## Step 2.3: Clone del repository uvc-gadget

**Descrizione:** Cloniamo il repository ufficiale uvc-gadget, che contiene il software per emulare una camera UVC su Raspberry Pi.

**Comandi da eseguire:**

```bash
cd ~
git clone https://gitlab.freedesktop.org/camera/uvc-gadget.git
cd uvc-gadget
git checkout master  # Usa 'master' invece di 'main'
ls -la
```

**Output atteso:**

```bash
Cloning into 'uvc-gadget'...
remote: Enumerating objects: 650, done.
remote: Counting objects: 100% (72/72), done.
remote: Compressing objects: 100% (38/38), done.
remote: Total 650 (delta 37), reused 65 (delta 34), pack-reused 578 (from 1)
Receiving objects: 100% (650/650), 189.92 KiB | 692.00 KiB/s, done.
Resolving deltas: 100% (398/398), done.
total 44
drwxrwxr-x 7 salim salim 4096 Dec 28 21:44 .
drwx------ 5 salim salim 4096 Dec 28 21:44 ..
drwxrwxr-x 8 salim salim 4096 Dec 28 21:44 .git
-rw-rw-r-- 1 salim salim    6 Dec 28 21:44 .gitignore
drwxrwxr-x 5 salim salim 4096 Dec 28 21:44 include
drwxrwxr-x 3 salim salim 4096 Dec 28 21:44 lib
-rw-rw-r-- 1 salim salim 1377 Dec 28 21:44 Makefile
-rw-rw-r-- 1 salim salim 2693 Dec 28 21:44 meson.build
-rw-rw-r-- 1 salim salim  524 Dec 28 21:44 README.md
drwxrwxr-x 2 salim salim 4096 Dec 28 21:44 scripts
drwxrwxr-x 2 salim salim 4096 Dec 28 21:44 src
```

## Step 2.4: Compilazione del software uvc-gadget

**Descrizione:** Compiliamo il software uvc-gadget utilizzando il sistema de build Meson e Ninja.

**Comandi da eseguire:**

```bash
#Pulisci build precedenti
rm -rf build
meson setup build
ninja -C build
```

**Output atteso:**

```bash
The Meson build system
Version: 1.7.0
Source dir: /home/salim/uvc-gadget
Build dir: /home/salim/uvc-gadget/build
Build type: native build
Project name: uvc-gadget
Project version: 0.4.0
C compiler for the host machine: cc (gcc 14.2.0 "cc (Debian 14.2.0-19) 14.2.0")
C linker for the host machine: cc ld.bfd 2.44
C++ compiler for the host machine: c++ (gcc 14.2.0 "c++ (Debian 14.2.0-19) 14.2.0")
C++ linker for the host machine: c++ ld.bfd 2.44
Host machine cpu family: aarch64
Host machine cpu: aarch64
Found pkg-config: YES (/usr/bin/pkg-config) 1.8.1
Found CMake: /usr/bin/cmake (3.31.6)
Run-time dependency libcamera found: NO (tried pkgconfig and cmake)
Run-time dependency libjpeg found: YES 2.1.5
Run-time dependency threads found: YES
Configuring config.h using configuration
Dependency uvcgadget found: YES 0.4.0 (overridden)
Build targets in project: 2
uvc-gadget 0.4.0
  Versions
    Sources: 0.4.0+6-04c18aa6
Found ninja-1.12.1 at /usr/bin/ninja
ninja: Entering directory `build'
[16/16] Linking target src/uvc-gadget
```

## Step 2.5: Verifica dell'eseguibile

**Descrizione:** Verifichiamo che l'eseguibile sia stato creato correttamente e ne controlliamo le informazioni.

**Comandi da eseguire:**

```bash
ls -lh build/src/uvc-gadget
file build/src/uvc-gadget
build/src/uvc-gadget --help | head -30
```

**Output atteso:**

```bash
-rwxrwxr-x 1 salim salim 77K Dec 28 21:44 build/src/uvc-gadget
build/src/uvc-gadget: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, BuildID[sha1]=4b6a74293fe5f3f1bff5e6b1d445663dce58a95e, for GNU/Linux 3.7.0, with debug_info, not stripped
Usage: build/src/uvc-gadget [options] <uvc device>
Available options are
 -d device      V4L2 source device
 -i image       MJPEG image
 -s directory   directory of slideshow images
 -h             Print this help screen and exit
 <uvc device>   UVC device instance specifier
  For ConfigFS devices the <uvc device> parameter can take the form of a shortened
  function specifier such as: 'uvc.0', or if multiple gadgets are configured, the
  gadget name should be included to prevent ambiguity: 'g1/functions/uvc.0'.
  For legacy g_webcam UVC instances, this parameter will identify the UDC that the
  UVC function is bound to.
  The parameter is optional, and if not provided the first UVC function on the first
  gadget identified will be used.
Example usage:
    build/src/uvc-gadget uvc.1
    build/src/uvc-gadget g1/functions/uvc.1
    build/src/uvc-gadget musb-hdrc.0.auto
```

# 3: Configurazione UVC con g_webcam

Dopo aver tentato vari approcci con ConfigFS, abbiamo scoperto che la soluzione più semplice e funzionante è utilizzare il modulo kernel g_webcam, che configura automaticamente il Raspberry Pi come dispositivo UVC.

## Step 3.1: Caricamento modulo g_webcam
**Descrizione:** Dopo il riavvio del sistema, carichiamo il modulo kernel g_webcam che configura automaticamente il Raspberry Pi come dispositivo UVC compatibile.

```bash
sudo modprobe -r libcomposite usb_f_uvc 2>/dev/null || true
sudo modprobe g_webcam
```

**Verifica del caricamento:**

```bash
lsmod | grep g_webcam
```

**Output atteso:**

```bash
g_webcam               16384  0
libcomposite           81920  2 usb_f_uvc,g_webcam
```

## Step 3.2: Verifica del dispositivo video creato
**Descrizione:** Verifichiamo che il modulo abbia creato il dispositivo video virtuale.

```bash
ls -la /dev/video* 2>/dev/null | tail -5
```

**Verifica con v4l2-ctl:**

```bash
v4l2-ctl --list-devices 2>/dev/null | head -10
```

**Output atteso:**

```bash
fe980000.usb (gadget.0):
        /dev/video0
```

## Step 3.3: Creazione immagine di test

Descrizione: Creiamo un'immagine di test in formato MJPEG che verrà trasmessa come stream video.

```bash
convert -size 640x480 gradient:blue-cyan \
  -fill white -stroke black -strokewidth 2 \
  -draw "rectangle 20,20 620,460" \
  -fill black -pointsize 36 -gravity center \
  -draw "text 0,-50 'UVC WEBCAM TEST'" \
  -draw "text 0,0 'Raspberry Pi'" \
  -draw "text 0,50 '640x480 MJPEG'" \
  -draw "text 0,100 '$(date +"%H:%M:%S")'" \
  -quality 90 ~/webcam_test.jpg
echo "Immagine creata:"
ls -lh ~/webcam_test.jpg
```

**Output atteso:**

```bash
Immagine creata:
-rw-rw-r-- 1 salim salim 32K Dec 28 22:54 /home/salim/webcam_test.jpg
```

## Step 3.4: Avvio di uvc-gadget

**Descrizione:** Avviamo il programma uvc-gadget per inviare l'immagine di test al dispositivo video virtuale.

```bash
#Ferma processi precedenti
sudo pkill uvc-gadget 2>/dev/null || true
sleep 1
#Cerca il nome UDC corretto
UDC_DEVICE=$(ls /sys/class/udc/)
echo "UDC trovato: $UDC_DEVICE"
#Avvia uvc-gadget
cd ~/uvc-gadget
sudo ./build/src/uvc-gadget -i ~/webcam_test.jpg $UDC_DEVICE &
sleep 3
#Verifica che il processo sia in esecuzione
ps aux | grep uvc-gadget | grep -v grep
```

**Output atteso :**

```bash
root        1508  0.0  0.2   4528  2408 pts/0    S    22:54   0:00 sudo ./build/src/uvc-gadget -i /home/salim/webcam_test.jpg fe980000.usb
root        1509  0.3  0.5  18080  5532 pts/0    Sl   22:54   0:00 ./build/src/uvc-gadget -i /home/salim/webcam_test.jpg fe980000.usb
```

## Step 3.5: Verifica finale

**Descrizione:** Verifichiamo che il dispositivo video sia configurato correttamente e pronto per l'uso.

```bash
echo "=== Verifica dispositivo /dev/video0 ==="
if [ -e /dev/video0 ]; then
    v4l2-ctl -d /dev/video0 --info 2>/dev/null | head -10
else
    echo "/dev/video0 non trovato"
fi

echo -e "\n=== Messaggi del kernel recenti ==="
dmesg | tail -5
```

**Output atteso:**

```bash
=== Verifica dispositivo /dev/video0 ===
Driver Info:
        Driver name      : g_uvc
        Card type        : fe980000.usb
        Bus info         : gadget.0
        Driver version   : 6.12.47
        Capabilities     : 0x84200002
                Video Output
                Streaming
                Extended Pix Format
                Device Capabilities
        Device Caps      : 0x04200002
                Video Output
                Streaming
                Extended Pix Format
=== Messaggi del kernel recenti ===
[  363.519393] g_webcam gadget.0: uvc: uvc_function_bind()
[  363.519698] g_webcam gadget.0: Webcam Video Gadget
[  363.519709] g_webcam gadget.0: g_webcam ready
[  363.519775] dwc2 fe980000.usb: bound driver g_webcam
```

## Step 3.6: Istruzioni per il collegamento al PC Procedura:

Scollega il Raspberry Pi dall'alimentazione corrente

Attendi 5 secondi

Collega il Raspberry Pi al PC con un cavo USB-C DATI (non solo alimentazione)

Su Windows:

Apri "Gestione dispositivi"

Cerca "USB Video Class Device" nella sezione "Telecamere" o "Dispositivi di acquisizione immagini"

Prova con l'app Fotocamera di Windows, Zoom, Teams, OBS Studio, ecc.

Comandi per riavviare il servizio (se necessario):

```bash
#Ferma uvc-gadget
sudo pkill uvc-gadget
#Ricarica il modulo g_webcam
sudo modprobe -r g_webcam
sudo modprobe g_webcam
#Riavvia uvc-gadget
cd ~/uvc-gadget
sudo ./build/src/uvc-gadget -i ~/webcam_test.jpg &
```

Per un setup permanente (avvio automatico all'accensione), aggiungi al file /etc/rc.local (prima di exit 0):

```bash
modprobe g_webcam
cd /home/salim/uvc-gadget && ./build/src/uvc-gadget -i /home/salim/webcam_test.jpg &
```
