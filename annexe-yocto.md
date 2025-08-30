# Annexe A – Aperçu de Yocto (Poky + meta-raspberrypi)

Remarque sur la numérotation des figures
- Définir N = numéro de la dernière figure du document principal.
- Remplacer dans cette annexe: «Figure F1» → «Figure N+1», «Figure F2» → «Figure N+2», etc. La numérotation reste continue à travers l’annexe.

A.1 Présentation rapide
Objectif Yocto vs Buildroot
- Yocto Project (Poky) fournit une chaîne complète et reproductible pour construire des distributions Linux embarquées, avec gestion de paquets, SDK, signatures, couches et variantes. Il cible des produits de production, le partage et la traçabilité.
- Buildroot est plus minimal et plus rapide pour générer des rootfs statiques; il ne gère pas nativement un gestionnaire de paquets ni des SDK aussi intégrés. Idéal pour images petites/sans mises à jour sophistiquées.

Composants Poky
- BitBake: moteur d’exécution qui parse les recettes (.bb/.bbappend) et exécute les tâches.
- OpenEmbedded Core (OE-Core): recettes et classes de base (distro features, toolchains, etc.).
- Couches (layers): empilement hiérarchique pour activer des BSP, des paquets, une distro personnalisée.

Schéma des couches Yocto
![Schéma général Yocto](/project/workspace/assets/yocto/downloaded/yocto-how-it-works.png)
Figure F1 — Flux global Yocto/BitBake (Source: Yocto Project Documentation, https://docs.yoctoproject.org)

A.2 Pré‑requis et récupération des sources
Paquets hôte (Ubuntu/Debian)
```
sudo apt update && sudo apt install -y \
  gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat cpio \
  python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping \
  python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3 xterm bc \
  locales zstd rsync
```
Clonage des dépôts (branche LTS «kirkstone»)
```
mkdir -p ~/yocto && cd ~/yocto
git clone -b kirkstone https://git.yoctoproject.org/poky
cd poky
git clone -b kirkstone https://github.com/openembedded/meta-openembedded
git clone -b kirkstone https://github.com/agherzan/meta-raspberrypi
```
![Clonage des sources](/project/workspace/assets/yocto/generated/terminal-clone.png)
Figure F2 — Clonage des dépôts Poky, meta-openembedded et meta-raspberrypi

Arborescence des couches
![Arborescence des couches](/project/workspace/assets/yocto/generated/tree-layers.png)
Figure F3 — Vue rapide des répertoires (poky, meta-openembedded, meta-raspberrypi)

A.3 Initialisation de l’environnement
```
source oe-init-build-env
```
- Création du dossier build/ et de build/conf/
- Fichiers générés: conf/bblayers.conf et conf/local.conf

![Initialisation de l’environnement](/project/workspace/assets/yocto/generated/oe-init-build-env.png)
Figure F4 — Initialisation de l’environnement de build

A.4 Configuration de la cible et de l’image
Éditer conf/bblayers.conf pour ajouter les couches nécessaires
- Ajouter: meta-raspberrypi et meta-openembedded (meta-oe, meta-networking, meta-python)

Éditer conf/local.conf (exemple minimal)
```
MACHINE ?= "raspberrypi3-64"        # ou "raspberrypi3" pour 32‑bit
IMAGE_FEATURES += "ssh-server-dropbear"
DISTRO_FEATURES:append = " wifi bluetooth"
ENABLE_UART = "1"                     # via config.txt
GPU_MEM = "64"
IMAGE_INSTALL:append = " linux-firmware-bcm43430 wpa-supplicant iw "
```
![Configuration des conf/](/project/workspace/assets/yocto/generated/bblayers-localconf-editor.png)
Figure F5 — Extraits de bblayers.conf et local.conf pour Raspberry Pi 3 (64‑bit)

Notes
- Le firmware Broadcom (linux-firmware-bcm43430), wpa-supplicant et iw assurent le Wi‑Fi.
- Les overlays et paramètres GPU/ UART sont gérés par meta-raspberrypi; ENABLE_UART active la console série.

A.5 Construction de l’image
Construire une image de base
```
bitbake core-image-base
# (alternatif BSP) bitbake rpi-test-image
```
![Progression BitBake](/project/workspace/assets/yocto/generated/bitbake-progress.png)
Figure F6 — Progression d’une construction core-image-base

Artefacts générés
- tmp/deploy/images/raspberrypi3-64/*.rpi-sdimg
- bootfiles/, dtb (ex. bcm2710-rpi-3-b-plus.dtb), kernel Image, u-boot selon configuration

![Déploiement des artefacts](/project/workspace/assets/yocto/generated/deploy-images-listing.png)
Figure F7 — Contenu du dossier deploy/images/raspberrypi3-64

A.6 Écriture sur carte SD et démarrage
Écriture (attention au bon périphérique)
```
sudo dd if=core-image-base-raspberrypi3-64.rpi-sdimg of=/dev/sdX \
  bs=4M conv=fsync status=progress
```
![Écriture avec dd](/project/workspace/assets/yocto/generated/dd-flash-sd.png)
Figure F8 — Écriture de l’image sur carte SD avec dd

Alternative GUI: balenaEtcher
![Interface balenaEtcher](/project/workspace/assets/yocto/downloaded/balenaetcher-ui.png)
Figure F9 — Interface de balenaEtcher (Source: StarWind Blog, https://www.starwindsoftware.com)

Console série et boot
- Dans config.txt: enable_uart=1, gpu_mem=64; overlay «disable-bt» si nécessaire.
- cmdline.txt: console=ttyAMA0,115200 root=/dev/mmcblk0p2 rootwait

![config.txt et cmdline.txt](/project/workspace/assets/yocto/generated/config-txt-editor.png)
Figure F10 — Extraits de config.txt et cmdline.txt (Raspberry Pi)

A.7 Réseau et SSH
- Récupérer l’IP (ip addr), tester la connectivité (ping), se connecter en SSH (Dropbear si activé).
- Identifiants: selon l’image choisie; core-image-base propose généralement l’accès root sans mot de passe en réseau local (à sécuriser).

![Vérification réseau et SSH](/project/workspace/assets/yocto/generated/ip-addr-ssh.png)
Figure F11 — Vérification réseau et session SSH sur la cible

A.8 SDK et développement applicatif
Générer et installer le SDK
```
bitbake core-image-base -c populate_sdk
sh tmp/deploy/sdk/poky-glibc-x86_64-core-image-base-cortexa53-toolchain-*.sh
source /opt/poky/kirkstone/environment-setup-aarch64-poky-linux
```
Compiler et déployer un «hello.c»
```
${CC} hello.c -o hello
scp hello root@raspberrypi.local:/usr/bin/
```
![SDK et compilation croisée](/project/workspace/assets/yocto/generated/sdk-installation.png)
Figure F12 — Génération du SDK, installation et compilation croisée

A.9 Paquet applicatif et service au boot
Créer une couche et une recette minimaliste
```
yocto-layer create meta-myapp
```
Ajouter la couche à bblayers.conf, puis créer la recette:
- recipes-myapp/myapp/myapp_1.0.bb
- Service systemd (si activé) dans recipes-core/myapp/myapp.service
- Ajouter à l’image: `IMAGE_INSTALL:append = " myapp "`

![Couche et recette MyApp](/project/workspace/assets/yocto/generated/myapp-layer-tree-recipe.png)
Figure F13 — Création de meta-myapp, recette myapp_1.0.bb et service systemd

Astuce
- Avec meta-raspberrypi, l’option `RPI_USE_UBOOT` peut être activée pour booter via U‑Boot.

Crédits et sources
- Yocto Project Documentation: schémas et concepts (https://docs.yoctoproject.org)
- Bootlin: schémas de formation Yocto (https://bootlin.com)
- Interface balenaEtcher: capture illustrant l’outil (https://www.starwindsoftware.com)
- Les autres captures d’écran sont générées pour illustrer les étapes du flux (cohérentes avec le style du document).