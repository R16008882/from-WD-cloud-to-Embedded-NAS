# 🚀 Projet MyCloud – Phase 2 : Système Embarqué Complet avec OP-TEE  
*Buildroot + OP-TEE + Nextcloud sur ARM: Ce projet consiste à étendre l'image minimale de la **Phase 1** (basée sur un initramfs) vers une image **Phase 2** complète, fonctionnelle et sécurisée, intégrant des services réseau (SSH, Nextcloud) et un environnement d'exécution de confiance (**OP-TEE**). L'ensemble est construit à l'aide de **Buildroot** pour une intégration fine et maîtrisée des composants logiciels.*

---

## 📌 Sommaire
1. [Présentation](#-présentation)  
2. [Matériel & prérequis](#-matériel--prérequis)  
3. [Feuille de route rapide](#-feuille-de-route-rapide)  
4. [Déploiement USB](#-déploiement-usb)  
5. [Vérification post-boot](#-vérification-post-boot)  
6. [Points de vigilance](#-points-de-vigilance)  
7. [Améliorations possibles](#-améliorations-possibles)  
8. [Arborescence finale](#-arborescence-finale)

---

## 🧭 Présentation
MyCloud Phase 2 transforme votre **Linux embarqué** en :
- **NAS domestique** (Nextcloud + Lighttpd + PHP)  
- **Serveur SSH léger** (dropbear)  
- **Plateforme sécurisée** (OP-TEE)  

---

## 🧰 Matériel & prérequis
| Élément | Détail |
|---------|--------|
| **Board** | ARM 64 bits (ex : Zidoo X9S) |
| **Toolchain** | Cross-compilation déjà configurée (aarch64-linux-gnu) |
| **Sources** | Buildroot 2025.05 + overlay `board/mycloud/` |
| **Stockage** | Clé USB ≥ 8 Go (boot + rootfs) |

---

## 🚀 Feuille de route rapide

### Étape 0 – Préparation
```bash
# Récupérer la config de base
cp configs/mycloud_phase1_defconfig configs/mycloud_phase2_defconfig
make mycloud_phase2_defconfig
```

### Étape 1 – Migration EXT4
```bash
make menuconfig
# Target options  → désactiver BR2_TARGET_ROOTFS_INITRAMFS
# Filesystem images → [*] ext4 (500 M)
```

### Étape 2 – Services réseau
```bash
# Packages
[*] e2fsprogs
[*] dropbear
[*] lighttpd
[*] php-cgi
```

### Étape 3 – Nextcloud
```bash
[*] nextcloud
[*] sqlite
```
Overlay personnalisé :
```
board/mycloud/rootfs-overlay/
├── var/www/nextcloud/
└── etc/lighttpd/lighttpd.conf
```
Script post-build :
```bash
#!/bin/sh
# board/mycloud/post-build.sh
cp -r ../nextcloud-files ${TARGET_DIR}/var/www/
chown -R www-data:www-data ${TARGET_DIR}/var/www/nextcloud
chmod +x board/mycloud/post-build.sh
```

### Étape 4 – OP-TEE
```bash
make menuconfig
Security  → [*] optee-os
            [*] optee-client
            [*] optee-examples
```
Trusted Application maison :
```c
// ta/nas_encrypt.c
TEE_Result encrypt_file(TEE_Param params[4]) {
    /* AES-256 via OP-TEE */
}
```

### Étape 5 – Boot via U-Boot
```bash
# boot.txt
setenv bootargs "root=/dev/sda2 rootwait console=ttyS0,115200 tee=optee"
fatload usb 0:1 0x1c000000 optee.bin
fatload usb 0:1 0x2000000 Image
fatload usb 0:1 0x2100000 rtd1295-zidoo-x9s.dtb
booti 0x2000000 - 0x2100000
```
Générer le script :
```bash
mkimage -A arm64 -O linux -T script -C none -d boot.txt boot.scr
```

---

## 📀 Déploiement USB
1. **Partitionner** la clé :  
   - p1 FAT32 (boot)  
   - p2 ext4 (rootfs)  

2. **Copier** les fichiers :
```bash
sudo dd if=output/images/rootfs.ext4 of=/dev/sdX2 bs=4M
sudo cp output/images/Image boot.scr optee.bin *.dtb /media/$USER/p1/
```

3. **Brancher** la clé et démarrer.

---

## ✅ Vérification post-boot

| Service | Commande / URL |
|---------|----------------|
| **SSH** | `ssh root@<IP>` (mdp vide → changer !) |
| **Nextcloud** | `http://<IP>/nextcloud` |
| **OP-TEE** | `ls /lib/optee_armtz/` doit lister `nas_encrypt.ta` |
| **Logs** | `dmesg | grep optee` et `journalctl -u tee-supplicant` |

---

## ⚠️ Points de vigilance
- **Performances** : mesurer le débit NAS (`iperf3`, `dd`).  
- **Sécurité** : désactiver SSH root via le post-build script.  
- **Espace disque** : augmenter `BR2_ROOTFS_SIZE` si Nextcloud grossit.  

---

## 🔮 Améliorations possibles
| Fonctionnalité | Lien / Notes |
|----------------|--------------|
| **Chiffrement LUKS** | Utiliser OP-TEE pour l’unlock à partir de l’initramfs. |
| **CI/CD** | Workflow GitHub Actions pour générer l’image à chaque push. |
| **Monitoring** | Prometheus + node_exporter pour CPU, RAM, disque. |

---

## 🗂️ Arborescence finale
```
buildroot/
├── configs/mycloud_phase2_defconfig
├── board/mycloud/
│   ├── rootfs-overlay/
│   ├── post-build.sh
│   └── patches/
optee/
├── ta/
│   ├── nas_encrypt.c
│   └── Makefile
docs/
├── DEPLOY.md
└── OP-TEE_INTEGRATION.md
```

---

## 🎉 C’est prêt !
Vous disposez maintenant d’un **système NAS sécurisé** prêt pour une utilisation domestique ou une démonstration OP-TEE.  
Des questions ? Ouvrez une issue ou consultez le dossier `docs/`.

*Happy hacking !*
