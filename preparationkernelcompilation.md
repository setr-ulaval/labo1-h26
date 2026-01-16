---
title: "Laboratoire 1 : Préparation de la configuration de compilation du noyau"
---

> **Note importante** : **ces instructions ne devraient pas avoir à être suivies en temps normal**. Nous vous fournissons déjà une configuration du noyau valide et nous vous **recommandons fortement** de l'utiliser. Cette page détaille la procédure de création de la configuration noyau pour permettre sa reproductibilité et fournir une base de connaissance, mais _nous n'offrirons **aucun support** pour ces configurations personnalisées_. Par ailleurs, l'évaluation des laboratoires sera faite sur l'image Raspbian du cours, avec le noyau configuré avec la configuration fournie : tout problème ou dysfonctionnement sera considéré comme une erreur, *même si le laboratoire fonctionne correctement sur votre noyau Linux personnalisé*. Cette page est fournie seulement à titre indicatif, vous ne devriez **pas** avoir à suivre ces instructions en temps normal.

## 1. Téléchargement des sources du noyau sur le Raspberry Pi

Cette étape doit être effectuée directement sur le Raspberry Pi Zero W. Clonez le dépôt Git du noyau de Raspberry Pi OS (utilisez `apt` pour installer `git` si ce n'est pas déjà fait) :
```
git clone -b rpi-6.12.y --shallow-since=2025-12-03 https://github.com/raspberrypi/linux
```

Pour s'assurer d'être exactement sur la même version que celle utilisée lors de la préparation du cours, modifiez le `HEAD` :
```
git reset --hard a8c4c464b753ef2273ae23cb79de4f9f05ce4ec7
```

## 2. Génération de la configuration du noyau par défaut

> **Note importante** : globalement, cette commande désactive tous les modules qui ne sont pas utilisés lors de son exécution. Il est donc crucial qu'un écran soit branché (et allumé) ainsi qu'un clavier USB standard et que le wifi soit activé. Dans le cas contraire, le noyau résultant ne permettra pas d'utiliser ces périphériques!

Installez les dépendances de compilation sur le Raspberry Pi Zero W :
```
sudo apt install bc bison flex libssl-dev make
```

Exécutez la commande suivante dans le dossier contenant les sources de Linux pour générer un fichier `.config` correspondant aux modules *actuellement* chargés :
```
make localmodconfig
```

Conservez par la suite le fichier `.config` généré, il servira de base à la compilation du noyau. Transférez le sur la machine virtuelle.


## 3. Récupération des sources de Linux et préconfiguration

> *Note* : à partir d'ici, les instructions doivent être exécutées sur la machine virtuelle, équipée d'une chaîne de compilation croisée fonctionnelle.

La configuration par défaut n'est pas suffisante pour le cours. En particulier, elle n'inclut pas le *patch* `PREEMPT_RT` et certains modules sont manquants (par exemple le `USB_GADGET`, permettant au Raspberry Pi Zero W de se faire passer pour un périphérique USB). Pour terminer la configuration, passez sur la machine virtuelle configurée avec la chaîne de compilation croisée. Par la suite, clonez sur celle-ci le dépôt Git du noyau de Raspberry Pi OS :

```
git clone -b rpi-6.12.y --shallow-since=2025-12-03 https://github.com/raspberrypi/linux linux-build
```

Pour s'assurer d'être exactement sur la même version que celle utilisée lors de la préparation du cours, modifiez le `HEAD` :
```
cd linux-build
git reset --hard a8c4c464b753ef2273ae23cb79de4f9f05ce4ec7
```

> Note : l'option `shallow-since` évite de télécharger l'entièreté de l'historique Git du noyau Linux; toutefois, la taille téléchargée ira en augmentant au fil du temps. Si votre exécutable `git` est à la version 2.49.0 ou plus récente, l'option `--revision` peut être directement passée à `git clone` et éviter complètement d'avoir à recourir à `git reset` :
```
git clone --depth=1 --revision=a8c4c464b753ef2273ae23cb79de4f9f05ce4ec7 https://github.com/raspberrypi/linux linux-build
```

Par la suite, téléchargez puis appliquez le *patch* `PREEMPT_RT`, en exécutant ces commandes dans le dossier créé par le clone du dépôt Git (par défaut, `linux`) :
```
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/6.12/patch-6.12.57-rt14.patch.gz
gzip -d patch-6.12.57-rt14.patch.gz
patch -p1 -i patch-6.12.57-rt14.patch
```

Copiez le fichier `.config` généré à l'étape 2 dans la racine du dépôt.


## 4. Configuration d'options supplémentaires et test de compilation

Quelques paramètres doivent être manuellement configurés. Bien qu'il soit possible d'utiliser `make menuconfig` pour le faire, les manipulations sont subtiles et peuvent facilement faire en sorte de regénérer toute la configuration. Il est donc plus simple de modifier directement le fichier `.config`. Il peut être ouvert dans un éditeur standard. Attention toutefois, les lignes commençant par `#` ne sont _pas_ toutes des commentaires. Par exemple, si on demande de modifier le fichier pour y ajouter `CONFIG_USB_DWC2=m`, il faut _également_ y **retirer** toute référence à CONFIG_USB_DWC2, telle que `# CONFIG_USB_DWC2 is not set`. Dans la liste suivante, ces retraits ne sont pas mentionnés, mais ils _doivent_ être faits. À la fin du processus, il ne doit y avoir _qu'une seule_ occurrence de chaque symbole dans le fichier `.config`.

Trouvez les lignes contenant les symboles suivants et modifiez-les pour activer la compilation de ces modules :
```
CONFIG_USB_DWC2=m
CONFIG_USB_GADGETFS=m
CONFIG_USB_FUNCTIONFS=m
CONFIG_USB_MASS_STORAGE=m
CONFIG_USB_G_SERIAL=m
CONFIG_USB_CDC_COMPOSITE=m
CONFIG_USB_G_HID=m
CONFIG_LOCALVERSION="-SETR-H2026"
```

Lancez ensuite une première compilation :
```
KERNEL=kernel
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs
```

Le processus s'interrompera rapidement parce que certaines options ne sont pas configurées à cause de l'ajout du *patch* `PREEMPT_RT`. La liste suivante présente les options à configurer. La plupart peuvent être laissées à leur valeur par défaut, mais certaines doivent être modifiées (par exemple pour activer `PREEMPT_RT`) :
```
Fully Preemptible Kernel (Real-Time) (PREEMPT_RT) [N/y/?] (NEW)                             # Répondre y <----- DIFFÉRENT DE LA VALEUR PAR DÉFAUT
Enable RCU priority boosting (RCU_BOOST) [Y/n/?]                                            # Répondre y (ou laisser la valeur par défaut)
Milliseconds to delay boosting after RCU grace-period start (RCU_BOOST_DELAY) [500] (NEW)   # Laisser la valeur par défaut de 500
USB GPIO Hub on HiSilicon Hikey 960/970 Platform (HISI_HIKEY_USB) [N/m/?] (NEW)             # Répondre N (valeur par défaut)
Generic GPIO-based CEC driver (CEC_GPIO) [N/m/y/?] (NEW)                                    # Répondre N (valeur par défaut)
DWC2 Mode Selection
        1. Host only mode (USB_DWC2_HOST) (NEW)
      * Gadget/Dual-role mode requires USB Gadget support to be enabled
        2. Gadget only mode (USB_DWC2_PERIPHERAL) (NEW)
      > 3. Dual Role mode (USB_DWC2_DUAL_ROLE) (NEW)
      choice[1-3?]:                                                                         # Répondre 2 <----- DIFFÉRENT DE LA VALEUR PAR DÉFAUT
Enable Debugging Messages (USB_DWC2_DEBUG) [N/y/?] (NEW)                                    # Répondre N (valeur par défaut)
Enable Missed SOF Tracking (USB_DWC2_TRACK_MISSED_SOFS) [N/y/?] (NEW)                       # Répondre N (valeur par défaut)
S5P/Exynos SoC series USB 2.0 PHY driver (PHY_SAMSUNG_USB2) [N/m/?] (NEW)                   # Répondre N (valeur par défaut)
Debug preemptible kernel (DEBUG_PREEMPT) [N/y/?] (NEW)                                      # Répondre N (valeur par défaut)
RCU CPU stall timeout in seconds (RCU_CPU_STALL_TIMEOUT) [21] (NEW)                         # Laisser la valeur par défaut de 21
Expedited RCU CPU stall timeout in milliseconds (RCU_EXP_CPU_STALL_TIMEOUT) [0] (NEW)       # Laisser la valeur par défaut de 0
Provide additional RCU stall debug information (RCU_CPU_STALL_CPUTIME) [N/y/?] (NEW)        # Répondre N (valeur par défaut)
Preemption-off Latency Tracer (PREEMPT_TRACER) [N/y/?] (NEW)                                # Répondre N (valeur par défaut)
GCC plugins (GCC_PLUGINS) [Y/n/?] (NEW) n                                                   # Répondre N <----- DIFFÉRENT DE LA VALEUR PAR DÉFAUT
Randomize layout of sensitive kernel structures
> 1. Disable structure layout randomization (RANDSTRUCT_NONE)
    2. Limit randomization of structure layout to cache-lines (RANDSTRUCT_PERFORMANCE) (NEW)
choice[1-2?]: 1                                                                             # Répondre 1
```

Par la suite, la compilation devrait commencer (environ 10 minutes sur 4 coeurs).

> Note : si vous obtenez une erreur du type _certs/extract-cert.c:21:10: fatal error: openssl/bio.h: No such file or directory_, installez `libssl-dev` sur la VM.

La compilation devrait se terminer avec succès, vous pouvez alors copier le fichier `.config` pour le conserver comme fichier de configuration final; dans les artefacts fournis, nous l'avons renommé `kernel_config`.
