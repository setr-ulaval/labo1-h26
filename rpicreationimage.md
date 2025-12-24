---
title: "Laboratoire 1 : Création de l'image du Raspberry Pi Zero W"
---

> **Note importante** : il est **fortement recommandé** d'utiliser directement l'image Raspbian fournie. Cette procédure n'est à réaliser **que si vous n'utilisez pas cette image Raspbian**, ce qui est déconseillé puisque nous n'offrirons *aucun support* pour les installations personnalisées et que l'évaluation des laboratoires sera faite sur l'image Raspbian du cours : tout problème ou dysfonctionnement sur cette image sera considéré comme une erreur, *même si le laboratoire fonctionne correctement sur votre image personnalisée*. Cette page est fournie seulement à titre indicatif, vous ne devriez **pas** avoir à suivre ces instructions en temps normal.

## 1. Téléchargement et validation de l'image Raspberry Pi OS

Nous utilisons Raspberry Pi OS comme base du système. Il faut télécharger la version **32-bit** (le Raspberry Pi Zero W n'est pas compatible avec la version 64 bit), version Debian Trixie. Dans le cadre du cours, nous avons utilisé l'image du *2025-12-04*, disponible directement [via ce lien](https://downloads.raspberrypi.com/raspios_lite_armhf/images/raspios_lite_armhf-2025-12-04/2025-12-04-raspios-trixie-armhf-lite.img.xz). Les versions subséquentes *peuvent* fonctionner, mais ce n'est pas garanti.

Une fois l'image téléchargée, validez sa source en utilisant une somme de contrôle [SHA-1](https://downloads.raspberrypi.com/raspios_lite_armhf/images/raspios_lite_armhf-2025-12-04/2025-12-04-raspios-trixie-armhf-lite.img.xz.sha1) ou [SHA-256](https://downloads.raspberrypi.com/raspios_lite_armhf/images/raspios_lite_armhf-2025-12-04/2025-12-04-raspios-trixie-armhf-lite.img.xz.sha256). Par exemple, sous Linux, les commandes `sha1sum` et `sha256sum` permettent d'obtenir la somme de contrôle du fichier téléchargé, qui peut ensuite être comparée à la valeur présente dans les fichiers liés ici. Par la suite, extraire le fichier `.img` du `.xz`.

Copiez ce fichier image *directement* sur la carte Micro-SD.

> **Note importante** : copier le fichier lui-même n'est *pas* suffisant. Il faut remplacer l'entièreté du contenu de la carte SD, y compris la table de partition, avec le contenu du fichier .img. Par exemple, sous Linux, la commande `dd` peut être utilisée. Sur Windows, des utilitaires comme `Rufus` offrent la même fonctionnalité.

## 2. Premier démarrage et configuration initiale

Insérez la carte Micro-SD dans le Raspberry Pi Zero W et démarrez-le. 

> **Note importante** : assurez-vous qu'un clavier et un écran sont branchés _avant_ de brancher l'alimentation sur le Raspberry Pi Zero W, sinon le Raspberry Pi désactive sa sortie graphique et ne la réactive pas si vous branchez un écran subséquemment.

Lorsque demandé, modifiez le nom d'utilisateur pour `pi` et le mot de passe pour `setrh2026` (si vous créez votre propre image, vous pouvez bien sûr tout de suite choisir un autre mot de passe).

Exécutez par la suite le programme `raspi-config` en mode superutilisateur :
```
sudo raspi-config
```

Vous verrez alors un menu en mode terminal, qui vous permet de configurer certaines fonctions du Raspberry Pi. En particulier, faites les manipulations suivantes :

- Dans la section `Section 1 System Options`, modifiez le *hostname* (`S4 Set hostname`) pour *rpisetr* et activez l'option `S6 Autologin in the console`
- Dans la section `Section 3 Interface Options`, activez l'option `I1 SSH`
- Dans la section `Section 5 Localization Options`, choisissez *Canada* dans la liste des pays de l'option `L4 WLAN Country` (important pour que le Wifi s'active)

Terminez en sauvegardant les changements, puis redémarrez le Raspberry Pi.

## 3. Désactivation du *swap* et installation de paquets logiciels requis

> *Note* : à partir de cette étape, la configuration peut se faire à distance, via SSH. Connectez d'abord le Raspberry Pi à un réseau wifi (par exemple en utilisant `nmtui`) et notez son adresse IP (par exemple en utilisant `ip addr`) pour pouvoir vous connecter à distance.

Nous voulons éviter le *swap* sur la carte Micro-SD en cas de dépassement de la mémoire vive. Pour ce faire, effectuez la manipulation suivante :
```
sudo mkdir -p /etc/rpi/swap.conf.d/
sudo tee /etc/rpi/swap.conf.d/90-disable-swap.conf > /dev/null <<EOF
[Main]
Mechanism=none
EOF
```

Le cours nécessite également l'installation de trois paquets logiciels supplémentaires (et leur dépendances):
```
sudo apt install libfuse-dev libcurl4-openssl-dev gdbserver
```

## 4. Ajustement de la configuration de démarrage

Le Raspberry Pi possède un fichier de configuration bas niveau, qui permet de configurer certains éléments comme le pilote graphique à utiliser ou la fréquence du processeur principal. Ce fichier est situé dans `/boot/firmware/config.txt` (attention! il ne s'agit *plus* de `/boot/config.txt` comme dans les versions précédentes). Éditez-le pour que son contenu soit le suivant :
```
# For more opwtions and information see
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
#dtoverlay=vc4-kms-v3d
#max_framebuffers=2

# Don't have the firmware create an initial video= setting in cmdline.txt.
# Use the kernel's default instead.
disable_fw_kms_setup=1

# Disable compensation for displays with overscan
disable_overscan=1

# Run as fast as firmware / board allows
arm_boost=1
force_turbo=1


[cm4]
# Enable host mode on the 2711 built-in XHCI USB controller.
# This line should be removed if the legacy DWC2 controller is required
# (e.g. for USB device mode) or if USB support is not required.
otg_mode=1

[cm5]
dtoverlay=dwc2,dr_mode=host

[all]

```

Validez que les changements sont fonctionnels en redémarrant.

## 5. Préconfiguration du réseau eduroam2

Copiez le fichier [eduroam2.nmconnection)](https://setr-ulaval.github.io/labo1-h26/eduroam2.nmconnection) dans `/etc/NetworkManager/system-connections/`. Si vous créez votre propre image personnalisée, vous pouvez immédiatement y ajouter vos identifiants.

## 6. Configuration de DuckDNS

Créez le fichier `/usr/local/bin/duckdns.sh` avec le contenu suivant :
```
#!/bin/bash
DUCKDNS_LOCALIP=`hostname -I`
DUCKDNS_TOKEN=ECRIRE VOTRE TOKEN DUCKDNS ICI
DUCKDNS_DOMAINS=ECRIRE VOTRE DOMAINS DUCKDNS ICI
DUCKDNS_LOGFILE=/var/log/duckdns.log
echo url="https://www.duckdns.org/update?domains=$DUCKDNS_DOMAINS&token=$DUCKDNS_TOKEN&ip=$DUCKDNS_LOCALIP" | curl -k -o $DUCKDNS_LOGFILE -K -
```

> *Note* : si vous créez votre propre image personnalisée, vous pouvez immédiatement modifier `DUCKDNS_TOKEN` et `DUCKDNS_DOMAINS` avec vos identifiants.

Créez ensuite le fichier `/etc/systemd/system/updateIP.service` avec le contenu suivant :
```
[Unit]
Description=Mise a jour de l'IP sur DuckDNS
Wants=network-online.target
After=network-online.target
[Service]
Type=oneshot
ExecStartPre=/bin/sh -c 'until ping -c1 8.8.8.8; do sleep 1; echo "Pas de connexion active..."; done;'
ExecStart=/usr/local/bin/duckdns.sh
[Install]
WantedBy=multi-user.target
```
