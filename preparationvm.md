---
title: "Laboratoire 1 : Préparation de la machine virtuelle VirtualBox avec l'environnement de compilation croisée"
---

> **Note importante** : **ces instructions ne devraient pas avoir à être suivies en temps normal**. Nous vous fournissons déjà une machine virtuelle et nous vous **recommandons fortement** de l'utiliser. Cette page détaille sa procédure de création pour permettre sa reproductibilité et fournir une base de connaissance, mais _nous n'offrirons **aucun support** pour ces VM personnalisées_. Par ailleurs, l'évaluation des laboratoires sera faite sur en compilant les programmes sur la VM fournie : tout problème ou dysfonctionnement sera considéré comme une erreur, *même si le laboratoire compile correctement et sans warnings sur votre installation personnalisé*. Cette page est fournie seulement à titre indicatif, vous ne devriez **pas** avoir à suivre ces instructions en temps normal.

> *Note* : si vous voulez seulement créer l'environnement de compilation croisée, passez directement à la section 3.

## 1. Création de la machine virtuelle VirtualBox

Installez VirtualBox (la procédure a été testée avec VirtualBox 7.2.4, mais peut fonctionner avec d'autres versions). Téléchargez [Ubuntu 24.04.3](https://releases.ubuntu.com/noble/ubuntu-24.04.3-desktop-amd64.iso) et lancez une machine avec cet ISO. La configuration devrait comporter au moins les éléments suivants :
- Disque dur virtuel **au format VDI** et d'une capacité (virtuelle) d'au moins 50 Go;
- Allouer 8192 Mo de RAM et 4 coeurs (ajustable par la suite, mais bon point de départ);
- Allouer 128 Mo de VRAM (mémoire vidéo). Ne PAS activer l'accélération 3D.

Installez par la suite Ubuntu selon la procédure standard, en utilisant `setr` comme nom d'utilisateur et `setrh2026` comme mot de passe. Sélectionnez *Français (Canada)* pour le clavier et activez le login automatique (auto-login).

Une fois le premier démarrage effectué, installez les "VirtualBox Additions" (peut requérir certains paquets supplémentaires comme `build-essential`) et redémarrez à nouveau.

## 2. Préparation et configuration de la machine virtuelle

Installez les paquets suivants (avec apt) :
```
sudo apt install autoconf bison flex git cmake build-essential texinfo help2man libtool libtool-bin libncurses-dev libssl-dev
```

Installez ensuite Visual Studio Code (testé avec la version 1.107)
Télécharger le .deb [via ce lien](https://update.code.visualstudio.com/1.107.0/linux-deb-x64/stable) puis utilisez `sudo apt ./nomdupackage.deb` pour l'installer.

> Note : l'installation via un snap package n'est pas recommandée puisque la mise à jour est faite automatiquement à des moments arbitraires

Exécutez VSCode puis installez les extensions suivantes (via le menu de gauche) :
- C/C++ Extension Pack
- Native Debug
- CMake
- Cmake Tools

**Désactivez** la mise à jour automatique de ces extensions pour éviter les problèmes durant le cours. Ils peuvent toujours être mis à jour manuellement si nécessaire.

## 3. Compilation et installation de Crosstool-NG

Installez Crosstool-NG 1.28, et assurez-vous qu'il est présent dans le `PATH` :
```
cd $HOME
wget http://crosstool-ng.org/download/crosstool-ng/crosstool-ng-1.28.0.tar.bz2
tar -xvf crosstool-ng-1.28.0.tar.bz2
cd crosstool-ng-1.28.0
./bootstrap
./configure --prefix=$HOME/crosstool-bin
make && make install
export PATH=$PATH:$HOME/crosstool-bin/bin
```

## 4. Création de l'environnement de compilation croisée

Clonez d'abord les sources de Linux, plus spécifiquement celles utilisées pour compiler le noyau plus tard (la commande `git reset --hard` permet de garantir que les sources utilisées seront toujours les mêmes que celles qui ont servi à préparer ce guide) :
```
git clone -b rpi-6.12.y --shallow-since=2025-12-03 https://github.com/raspberrypi/linux linux-build
cd linux-kernel-6.12-headers
git reset --hard a8c4c464b753ef2273ae23cb79de4f9f05ce4ec7
```

> Note : l'option `shallow-since` évite de télécharger l'entièreté de l'historique Git du noyau Linux; toutefois, la taille téléchargée ira en augmentant au fil du temps. Si votre exécutable `git` est à la version 2.49.0 ou plus récente, l'option `--revision` peut être directement passée à `git clone` et éviter complètement d'avoir à recourir à `git reset` :
```
git clone -b rpi-6.12.y --depth=1 --revision=a8c4c464b753ef2273ae23cb79de4f9f05ce4ec7 https://github.com/raspberrypi/linux linux-build
```

Appliquez la _patch_ `PREEMPT_RT` :
```
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/6.12/patch-6.12.57-rt14.patch.gz
gzip -d patch-6.12.57-rt14.patch.gz
patch -p1 -i patch-6.12.57-rt14.patch
```

Nous sommes maintenant prêts à configurer la création de l'environnement de Crosstool-NG. Créez d'abord le dossier de travail :
```
cd $HOME
mkdir ct-config-rpi-zero && cd ct-config-rpi-zero
```

Puis créez la configuration Crosstool-NG initiale :
```
ct-ng armv6-unknown-linux-gnueabihf
```

Il faut ensuite modifier plusieurs paramètres en utilisant `ct-ng menuconfig` :
```
Path and misc options -> Try features marked as EXPERIMENTAL -> Activer (sinon certaines des options suivantes ne s'afficheront pas!)
Path and misc options -> Render the toolchain Read-only -> Désactiver
Target options -> Suffix to the arch part -> Supprimer et laisser vide
Target options -> append 'hf' to the tuple -> Activer
C-library -> Version of glibc = 2.41 (pour correspondre à celle de Raspbian 13)
C-compiler -> Version of gcc = 14.3.0 (pour correspondre à celle de Raspbian 13)
Binary utilities -> Version of binutils = 2.44
Operating system -> Version of linux = 6.12.41
Operating system -> Source of linux = Custom location
Operating system -> Custom location = chemin vers les sources du noyau récupérées plus haut (devrait être ${HOME}/linux-kernel-6.12-headers)
Debug utilities -> Activer strace
```

Sauvegardez les changements avant de quitter `menuconfig`.

Certains paramètres ne peuvent pas être modifiés avec `menuconfig`, il faut donc éditer manuellement le fichier `.config` créé dans le répertoire pour y utiliser les valeurs suivantes :
```
CT_TARGET_ALIAS="arm-linux-gnueabihf"
CT_TARGET_VENDOR="raspbian"
```

Par la suite, l'environnement de compilation croisé peut être compilé :
```
ct-ng build
```

> Note : cette étape requiert environ 50 minutes sur un ordinateurs x86-64 avec 4 coeurs. Un Macbook avec processeur M1 Pro requiert environ 35 minutes.


## 5. Synchronisation de l'environnement de compilation croisée

À ce point-ci, l'environnement de compilation croisée est prêt, mais il n'est pas synchronisé avec les versions spécifiques des librairies (.so et .h) installées sur le Raspberry Pi Zero W. Pour le faire, démarrez un Raspberry Pi avec l'image par défaut du cours installée et un accès SSH activé et exécutez les commandes suivantes dans la machine virtuelle :
```
cd ~/arm-cross-comp-env/arm-raspbian-linux-gnueabihf/arm-raspbian-linux-gnueabihf
rsync -av --numeric-ids --exclude "*.ko" --exclude "*.fw" --exclude "*.bin" --exclude "*.ko.xz" --exclude "/opt/vc/src" --exclude "*/lib/python*" --delete pi@ADRESSE_IP_OU_NOM_DHOTE_DU_RPI:{/lib,/opt} sysroot
rsync -av --numeric-ids --exclude "*.ko" --exclude "*.fw" --exclude "*.bin" --exclude "*.ko.xz" --exclude "/usr/lib/.debug" --exclude "lib/firmware" --exclude "lib/python*" --delete pi@ADRESSE_IP_OU_NOM_DHOTE_DU_RPI:{/usr/lib,/usr/include} sysroot/usr
```

> Note : pour la seconde commande, l'erreur _rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1865) generator=3.2.7_ n'est pas un problème

Corrigez les liens rendus invalides par la copie :
```
cd ~/arm-cross-comp-env/arm-raspbian-linux-gnueabihf/arm-raspbian-linux-gnueabihf/sysroot
find . -lname '/*' | while read l ; do   echo ln -sf $(echo $(echo $l | sed 's|/[^/]*|/..|g')$(readlink $l) | sed 's/.....//') $l; done | sh
```

> Note : les erreurs concernant _/usr/lib/ssl/private/private_ et _/usr/lib/ssl/certs/certs_ ne sont pas un problème

## 6. Création du fichier de configuration CMake et ajout au `PATH`

Ajoutez les répertoires `$HOME/crosstool-bin/bin` et `$HOME/arm-cross-comp-env/arm-raspbian-linux-gnueabihf/bin/` au `PATH` en ajoutant la ligne suivante à la fin du fichier `~/.bashrc` :
```
export PATH=$PATH:$HOME/crosstool-bin/bin:$HOME/arm-cross-comp-env/arm-raspbian-linux-gnueabihf/bin/
```

Finalement créez un fichier nommé `rpi-zero-w-toolchain.cmake` dans le dossier `~/arm-cross-comp-env/` et ajoutez-y le contenu suivant :
```
SET(CMAKE_SYSTEM_NAME Linux)
SET(CMAKE_SYSTEM_VERSION 6.12)
set(CMAKE_SYSTEM_PROCESSOR arm)

# Localisation du sysroot
set(CMAKE_SYSROOT $ENV{HOME}/arm-cross-comp-env/arm-raspbian-linux-gnueabihf/arm-raspbian-linux-gnueabihf/sysroot)

# Selection du compilateur
set(tools $ENV{HOME}/arm-cross-comp-env/arm-raspbian-linux-gnueabihf)
set(CMAKE_C_COMPILER ${tools}/bin/arm-raspbian-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER ${tools}/bin/arm-raspbian-linux-gnueabihf-g++)

# On ajoute des options au compilateur pour lui indiquer ou aller chercher les librairies
SET(FLAGS "-Wl,-rpath-link,${CMAKE_SYSROOT}/opt/vc/lib -Wl,-rpath-link,${CMAKE_SYSROOT}/lib/arm-linux-gnueabihf -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/arm-linux-gnueabihf -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/local/lib -Wl,-rpath-link,${CMAKE_SYSROOT}/lib/gcc/arm-linux-gnueabihf/14 -B${CMAKE_SYSROOT}/usr/lib/arm-linux-gnueabihf -L${CMAKE_SYSROOT}/usr/lib/arm-linux-gnueabihf -B${CMAKE_SYSROOT}/lib/gcc/arm-linux-gnueabihf/14 -I${CMAKE_SYSROOT}/usr/include/arm-linux-gnueabihf")
SET(CMAKE_CXX_FLAGS ${FLAGS} CACHE STRING "" FORCE)
SET(CMAKE_C_FLAGS ${FLAGS} CACHE STRING "" FORCE)

# Quoi aller chercher dans la sysroot (on ne veut pas aller chercher les programmes puisqu'ils sont
# compiles en ARM et ne peuvent donc etre directement executes le processeur de la machine hote)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

## 7. Derniers ajustements et finalisation de la machine virtuelle

Les ajustements suivants doivent être faits pour terminer la configuration avant de livrer la machine virtuelle aux étudiants :

1. Désactiver le verrouillage automatique de l'écran (Settings / Privacy & security / Screen lock)
2. Supprimer l'historique du terminal (history -c)
3. Régler la résolution d'écran à une valeur raisonnable pour tous les ordinateurs (ex. 1280x720)
4. Attacher ("pin") VSCode et le terminal à la barre de lancement à gauche

## A1. Ajustements à apporter pour la création d'une VM ARM64 sur MacOS

Ubuntu 24.0.3 peut être obtenu en version ARM64, mais seulement pour la version serveur. Nous installons donc d'abord cette version avant de la "transformer" en version desktop.

1. Installer [UTM](https://mac.getutm.app/) (testé avec la version 4.6.4)
2. Télécharger [Ubuntu Server for ARM 24.04.3](https://cdimage.ubuntu.com/releases/24.04/release/ubuntu-24.04.3-live-server-arm64.iso)
3. Suivre le [tutoriel suivant](https://itslinuxfoss.com/install-ubuntu-24-04-lts-macbook/) pour installer Ubuntu 24.04 et le transformer en version desktop (ne _pas_ activer l'accélération hardware OpenGL contrairement à ce qui est suggéré, cela pose problème avec VScode)
4. La suite des étapes est la même que celles présentées plus haut pour la version x86-64, sauf que le package `.deb` à utiliser pour installer Visual Studio Code est [celui-ci](https://update.code.visualstudio.com/1.107.0/linux-deb-arm64/stable)
5. Pour pouvoir utiliser les dossiers partagés entre VM et hôte, il faut en plus installer (avec apt) les paquets `spice-vdagent`, `spice-webdavd`, et `davfs2`. Voir [cette page](https://dev.to/ajeebkp23/how-to-access-shared-folders-in-utm-ubuntu-vm-590n) pour plus de détails. Il faut également modifier, dans les propriétés de la machine virtuelle UTM, section `Sharing`, la configuration `Directory Share Mode` pour `SPICE WebDAV`.

> Note : pour une raison inconnue, le démarrage bloque pendant 2 minutes à l'activation du réseau ("NetworkManager-wait-online"). Bien que cela n'affecte pas la machine virtuelle (le réseau est bel et bien actif une fois le démarrage complété), il est possible de réduire le délai en ajustant la variable `Environment=NM_ONLINE_TIMEOUT` dans `/etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service`.
