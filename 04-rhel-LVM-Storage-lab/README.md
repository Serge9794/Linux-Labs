
# 🗄️ Infrastructure de Stockage Entreprise avec LVM — Rocky Linux / RHEL

<div align="center">

![Linux](https://img.shields.io/badge/OS-Rocky%20Linux%20%7C%20RHEL-10B981?style=for-the-badge&logo=redhat&logoColor=white)
![LVM](https://img.shields.io/badge/Technologie-LVM-3B82F6?style=for-the-badge&logo=linux&logoColor=white)
![RHCSA](https://img.shields.io/badge/Certification-RHCSA%20Prep-EF4444?style=for-the-badge&logo=redhat&logoColor=white)
![Status](https://img.shields.io/badge/Statut-Production%20Ready-22C55E?style=for-the-badge)

**Auteur :** [Serge TOGNON](https://www.linkedin.com/in/serge-tognon) | **Environnement :** `serge@server` (VirtualBox) | **Rôle :** Admin Cloud Azure/ Admin Système Linux

> *Projet réel d'entreprise — Isolation et automatisation de la gestion du stockage pour les applications et les logs via LVM, sans interruption de service.*

</div>

---

## 📋 Table des Matières

1. [Contexte & Problématique](#-contexte--problématique)
2. [Architecture Technique & Concepts LVM](#-architecture-technique--concepts-lvm)
3. [Guide des Commandes — Étape par Étape](#-guide-des-commandes--étape-par-étape)
4. [Cas Pratique : Extension à Chaud (Scénario RHCSA)](#-cas-pratique--extension-à-chaud-scénario-rhcsa)
5. [Sécurité & Vérifications — Check-list RHCSA](#-sécurité--vérifications--check-list-rhcsa)

---

## 🏢 Contexte & Problématique

L'entreprise fait face à une **croissance rapide de ses données**. La partition racine `/dev/mapper/rhel-root` (47 Go) héberge actuellement à la fois le système, les applications et les logs. Cette architecture monolithique présente des risques critiques :

| Risque | Impact |
|--------|--------|
| Saturation de `/` par les logs | Plantage total du système |
| Impossibilité d'étendre à chaud | Interruption de service lors des maintenances |
| Absence d'isolation applicative | Une application peut saturer le stockage de toutes les autres |
| Pas de granularité de gestion | Impossible de quotater ou monitorer finement |

**Solution retenue :** Implémenter **LVM (Logical Volume Manager)** pour isoler les logs et les applications sur des volumes logiques dédiés, extensibles à chaud, sans interruption de service.

---

## 🏛️ Architecture Technique & Concepts LVM

### La Chaîne de Valeur du Stockage Logique

LVM introduit une couche d'abstraction entre le matériel physique et le système de fichiers. La transition se fait en **5 niveaux** :

```
┌─────────────────────────────────────────────────────────────────┐
│                     COUCHE APPLICATIVE                          │
│          /var/log/app_logs      /mnt/applications               │
├──────────────────────┬──────────────────────────────────────────┤
│   SYSTÈME DE FICHIERS│  EXT4 (lv_logs)    XFS (lv_apps)        │
├──────────────────────┴──────────────────────────────────────────┤
│              VOLUMES LOGIQUES (LV)                              │
│         lv_logs (5 Go)        lv_apps (10 Go)                   │
├─────────────────────────────────────────────────────────────────┤
│              GROUPE DE VOLUMES (VG)                             │
│                    vg_data (20 Go)                              │
├──────────────────────┬──────────────────────────────────────────┤
│  VOLUMES PHYSIQUES   │  /dev/sdb (PV)     /dev/sdc (PV)        │
├──────────────────────┴──────────────────────────────────────────┤
│              DISQUES PHYSIQUES                                  │
│         /dev/sdb (10 Go)       /dev/sdc (10 Go)                 │
└─────────────────────────────────────────────────────────────────┘
```

### Tableau Récapitulatif des Composants LVM

| Composant | Acronyme | Commande de création | Commande d'inspection | Rôle |
|-----------|----------|----------------------|-----------------------|------|
| Physical Volume | **PV** | `pvcreate /dev/sdX` | `pvs` / `pvdisplay` | Initialise un disque/partition pour LVM |
| Volume Group | **VG** | `vgcreate vg_name PV...` | `vgs` / `vgdisplay` | Fusionne plusieurs PV en un pool de stockage |
| Logical Volume | **LV** | `lvcreate -L taille -n nom VG` | `lvs` / `lvdisplay` | Découpe le VG en volumes utilisables |
| Système de fichiers | **FS** | `mkfs.ext4` / `mkfs.xfs` | `df -h` / `blkid` | Formate le LV pour stocker des données |
| Point de montage | **Mount** | `mount /dev/VG/LV /chemin` | `mount` / `findmnt` | Rend le LV accessible dans l'arborescence |
| Persistance | **fstab** | `echo "..." >> /etc/fstab` | `cat /etc/fstab` | Assure le remontage automatique au démarrage |

### Concepts Clés à Maîtriser pour le RHCSA

- **Physical Extent (PE)** : Unité de base d'allocation dans un VG (défaut : 4 Mo). Un LV est une collection de PE.
- **Métadonnées LVM** : Stockées au début de chaque PV. Contiennent la configuration complète du VG.
- **Device Mapper** : Couche noyau qui traduit les accès aux LV vers les PV physiques. Les LV apparaissent sous `/dev/mapper/`.
- **Snapshot LVM** : Copie instantanée d'un LV à un instant T. Utile pour les sauvegardes à chaud.

---

## 🛠️ Guide des Commandes — Étape par Étape

> **Prérequis :** Connecté en tant que `root` sur `serge@server`, ou utiliser `sudo` avant chaque commande. Les deux nouveaux disques `/dev/sdb` et `/dev/sdc` de 10 Go chacun ont été ajoutés via l'interface VirtualBox et sont détectés par le noyau.

---

### Étape A — Inspection et Diagnostic Initial

**Objectif :** Photographier l'état du système *avant* toute modification. C'est une bonne pratique irremplaçable en production.

#### A.1 — Vue d'ensemble de l'utilisation des systèmes de fichiers montés

```bash
[serge@server ~]$ df -h
```

**Explication :** `df` (disk free) affiche l'utilisation de l'espace pour chaque système de fichiers monté. L'option `-h` (human-readable) convertit les octets en Ko/Mo/Go lisibles. On observe que `/dev/mapper/rhel-root` fait **47 Go** avec seulement **7% d'utilisation** — mais sans isolation, les logs applicatifs pourraient saturer cette partition.

<img width="473" height="260" alt="A" src="https://github.com/user-attachments/assets/8d836f24-6d13-41ab-a122-ad7a597a6fca" />


---

#### A.2 — Cartographie complète des blocs de périphériques

```bash
[serge@server ~]$ lsblk
```

**Explication :** `lsblk` (list block devices) affiche l'arborescence de tous les périphériques bloc du système : disques, partitions, volumes LVM. On doit voir apparaître `/dev/sdb` et `/dev/sdc` comme nouveaux disques **sans partition ni système de fichiers**, confirmant qu'ils sont disponibles pour l'initialisation LVM.

<img width="649" height="178" alt="B" src="https://github.com/user-attachments/assets/7a761a5c-5643-4788-b2c8-d556d4086f28" />


---

### Étape B — Initialisation des Physical Volumes (PV)

**Objectif :** Écrire les métadonnées LVM au début de chaque disque pour les enregistrer dans l'écosystème LVM.

#### B.1 — Création du PV sur le premier disque

```bash
[serge@server ~]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```

**Explication :** `pvcreate` inscrit un en-tête LVM (PVID, taille des Physical Extents, métadonnées) au début du disque `/dev/sdb`. À partir de ce moment, LVM reconnaît ce disque comme un Physical Volume exploitable. **Attention :** Cette opération détruit toute donnée préexistante sur le disque.

<img width="646" height="92" alt="C" src="https://github.com/user-attachments/assets/f4f9d0fb-aae3-4636-b2a1-7016773d92b5" />


---

#### B.2 — Création du PV sur le second disque

```bash
[serge@server ~]$ sudo pvcreate /dev/sdc
  Physical volume "/dev/sdc" successfully created.
```

**Explication :** Même opération pour `/dev/sdc`. Les deux PV sont maintenant déclarés. La commande `pvs` ou `pvdisplay` permet de vérifier leur état.

```bash
[serge@server ~]$ sudo pvs
  PV         VG      Fmt  Attr PSize   PFree
  /dev/sda3  rhel    lvm2 a--  <48.41g     0
  /dev/sdb           lvm2 ---  <10.00g <10.00g
  /dev/sdc           lvm2 ---  <10.00g <10.00g
```

<img width="649" height="109" alt="D" src="https://github.com/user-attachments/assets/9e7ec779-61d4-4e64-b9b5-4558527f3619" />


---

### Étape C — Création du Groupe de Volumes `vg_data`

**Objectif :** Fusionner les deux PV (20 Go au total) en un seul pool de stockage logique nommé `vg_data`.

```bash
[serge@server ~]$ sudo vgcreate vg_data /dev/sdb /dev/sdc
  Volume group "vg_data" successfully created
```

**Explication :** `vgcreate` crée un Volume Group nommé `vg_data` en agrégeant `/dev/sdb` et `/dev/sdc`. LVM découpe l'espace total en **Physical Extents** de 4 Mo chacun et maintient une table d'allocation centralisée. Le VG dispose désormais d'environ **19,99 Go** d'espace libre (les métadonnées LVM consomment quelques Mo).

```bash
[serge@server ~]$ sudo vgs
  VG       #PV #LV #SN Attr   VSize   VFree
  rhel       1   2   0 wz--n- <48.41g     0
  vg_data    2   0   0 wz--n- <19.99g <19.99g
```

<img width="647" height="116" alt="E" src="https://github.com/user-attachments/assets/85641c16-371d-4e75-97b9-46583c64bc1a" />


---

### Étape D — Création des Logical Volumes

**Objectif :** Découper le VG `vg_data` en deux volumes logiques aux usages distincts.

#### D.1 — Création de `lv_logs` (5 Go pour les logs applicatifs)

```bash
[serge@server ~]$ sudo lvcreate -L 5G -n lv_logs vg_data
  Logical volume "lv_logs" created.
```

**Explication :** `lvcreate` crée un LV nommé `lv_logs` de **5 Go exactement** dans le VG `vg_data`. L'option `-L` spécifie la taille absolue, `-n` le nom. Le LV est immédiatement accessible sous `/dev/vg_data/lv_logs` et `/dev/mapper/vg_data-lv_logs`.

<img width="650" height="78" alt="F" src="https://github.com/user-attachments/assets/821803dc-2702-48b2-b078-1a290087635a" />


---

#### D.2 — Création de `lv_apps` (10 Go pour les applications)

```bash
[serge@server ~]$ sudo lvcreate -L 10G -n lv_apps vg_data
  Logical volume "lv_apps" created.
```

**Explication :** Même logique pour `lv_apps` avec une taille de **10 Go**, dimensionné pour les données applicatives plus volumineuses. Vérification :

```bash
[serge@server ~]$ sudo lvs
  LV      VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_apps vg_data -wi-a-----  10.00g
  lv_logs vg_data -wi-a-----   5.00g
```

<img width="646" height="119" alt="G" src="https://github.com/user-attachments/assets/69182578-7a1f-4a8c-b475-87ac725a7cb4" />


---

### Étape E — Formatage des Systèmes de Fichiers

**Objectif :** Écrire un système de fichiers sur chaque LV pour les rendre utilisables. Le choix du FS est stratégique.

#### E.1 — Formatage de `lv_logs` en EXT4

```bash
[serge@server ~]$ sudo mkfs.ext4 /dev/vg_data/lv_logs
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 1310720 4k blocks and 327680 inodes
Filesystem UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
...
Writing superblocks and filesystem accounting information: done
```

**Explication :** `mkfs.ext4` formate `lv_logs` en **EXT4**, le système de fichiers journalisé de référence sous Linux. EXT4 est choisi pour les logs car il gère efficacement un grand nombre de **petits fichiers**, offre un journaling robuste qui prévient la corruption en cas de crash, et ses outils de réparation (`fsck.ext4`) sont très matures.

<img width="647" height="164" alt="H" src="https://github.com/user-attachments/assets/a55180fc-cbf5-4937-aa4d-b27a0157d784" />


---

#### E.2 — Formatage de `lv_apps` en XFS

```bash
[serge@server ~]$ sudo mkfs.xfs /dev/vg_data/lv_apps
meta-data=/dev/vg_data/lv_apps   isize=512    agcount=4, agsize=655360 blks
...
log      =internal log           bsize=4096   blocks=10240, version=2
realtime =none                   exts=0
```

**Explication :** `mkfs.xfs` formate `lv_apps` en **XFS**, le système de fichiers haute performance adopté par défaut dans RHEL/Rocky Linux. XFS excelle dans la gestion de **fichiers volumineux** (binaires d'applications, bases de données, images Docker), offre un excellent parallélisme I/O et supporte nativement l'extension à chaud avec `xfs_growfs`.

<img width="662" height="184" alt="I" src="https://github.com/user-attachments/assets/bcf2e379-e516-4773-9887-c515f70a3db6" />


---

### Étape F — Configuration du Montage Persistant dans `/etc/fstab`

**Objectif :** Créer les points de montage, intégrer les volumes dan
s `/etc/fstab` pour la persistance au redémarrage, et valider la configuration.

#### F.1 — Création des répertoires de montage

```bash
[serge@server ~]$ sudo mkdir -p /var/log/app_logs
[serge@server ~]$ sudo mkdir -p /mnt/applications
```

**Explication :** `mkdir -p` crée les répertoires cibles. `/var/log/app_logs` est placé sous `/var/log/` pour suivre la convention FHS (Filesystem Hierarchy Standard) des logs système. `/mnt/applications` utilise le répertoire standard pour les montages temporaires ou applicatifs.

<img width="652" height="169" alt="J" src="https://github.com/user-attachments/assets/70237f05-03e4-4d5f-aece-ff3e1a253251" />


---

#### F.2 — Récupération des UUID des volumes

```bash
[serge@server ~]$ sudo blkid /dev/vg_data/lv_logs /dev/vg_data/lv_apps
/dev/vg_data/lv_logs: UUID="aaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" TYPE="ext4"
/dev/vg_data/lv_apps: UUID="11111-2222-3333-4444-555555555555" TYPE="xfs"
```

**Explication :** `blkid` retourne l'UUID unique de chaque système de fichiers. **Il est impératif d'utiliser les UUID plutôt que les noms de périphériques** (`/dev/vg_data/lv_logs`) dans `/etc/fstab`. Les noms de devices peuvent changer lors d'une modification de la configuration des disques ; les UUID, eux, sont immuables.

<img width="644" height="56" alt="K" src="https://github.com/user-attachments/assets/e6e75df6-87d8-4819-8378-73d353351b60" />


--
#### F.3 — Ajout des entrées dans `/etc/fstab`

```bash
[serge@server ~]$ sudo vi /etc/fstab
```

Ajouter les lignes suivantes à la fin du fichier (remplacer les UUID par ceux obtenus via `blkid`) :

```bash
# LVM - Logs applicatifs (EXT4) - Projet Infrastructure Stockage - Serge TOGNON
UUID=aaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee  /var/log/app_logs  ext4  defaults  0 2

# LVM - Applications (XFS) - Projet Infrastructure Stockage - Serge TOGNON
UUID=11111-2222-3333-4444-555555555555   /mnt/applications  xfs   defaults  0 2
```

**Explication des champs `/etc/fstab` :**

| Champ | Valeur | Signification |
|-------|--------|---------------|
| `UUID=...` | UUID du FS | Identifiant unique immuable du périphérique |
| `/var/log/app_logs` | Point de montage | Répertoire cible dans l'arborescence |
| `ext4` / `xfs` | Type de FS | Système de fichiers à utiliser |
| `defaults` | Options de montage | `rw, suid, dev, exec, auto, nouser, async` |
| `0` | Dump | 0 = pas de sauvegarde via `dump` |
| `2` | Pass (fsck) | Ordre de vérification au boot (1=racine, 2=autres) |

<img width="658" height="248" alt="L" src="https://github.com/user-attachments/assets/1c3d1a2b-04f3-43f6-9fec-0cdbb1c18895" />


---

#### F.4 — Validation du fstab avec `mount -a`

```bash
[serge@server ~]$ sudo mount -a
```

**Explication :** `mount -a` lit `/etc/fstab` et monte **tous** les systèmes de fichiers qui ne sont pas encore montés. Si la commande retourne sans erreur, le fichier `fstab` est valide. C'est le test de validation **incontournable** avant tout redémarrage — une erreur dans fstab peut rendre le système non-bootable.

```bash
[serge@server ~]$ df -h | grep -E "app_logs|applications"
/dev/mapper/vg_data-lv_logs  4.9G   24K  4.6G   1% /var/log/app_logs
/dev/mapper/vg_data-lv_apps   10G  104M  9.9G   2% /mnt/applications
```

<img width="650" height="175" alt="M" src="https://github.com/user-attachments/assets/6afb6f82-8a0b-42aa-8435-b5a912a6f8e1" />


---

## 🔥 Cas Pratique : Extension à Chaud (Scénario RHCSA)

### Contexte du Scénario

> **⚠️ ALERTE CRITIQUE — 02:47 du matin**
> 
> Le système de monitoring remonte une alarme : `/var/log/app_logs` est rempli à **95%**. L'application génère des logs à un rythme anormal depuis le déploiement de la v2.3.1 il y a 6 heures. Dans 45 minutes, la partition sera pleine et l'application plantera.
>
> **Contrainte absolue :** Zéro interruption de service. L'application tourne 24h/24. Pas de fenêtre de maintenance possible.

### Inventaire des Ressources Disponibles

```bash
[serge@server ~]$ sudo vgs vg_data
  VG      #PV #LV #SN Attr   VSize   VFree
  vg_data   2   2   0 wz--n- <19.99g <4.99g
```

✅ **Il reste ~5 Go libres dans `vg_data`** — de quoi doubler `lv_logs` sans ajouter de nouveau disque.

```bash
[serge@server ~]$ df -h /var/log/app_logs
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/vg_data-lv_logs  4.9G  4.5G  84M  99% /var/log/app_logs
```


<img width="648" height="165" alt="N" src="https://github.com/user-attachments/assets/09ce1888-e017-4d54-9dc2-0347d45dd5c9" />

---

### Procédure d'Extension à Chaud

#### Étape 1 — Extension du Logical Volume de 4,8 Go supplémentaires

```bash
[serge@server ~]$ sudo lvextend -L +4,8G /dev/vg_data/lv_logs
  Size of logical volume vg_data/lv_logs changed from 5.00 GiB (1280 extents) to 10.00 GiB (2560 extents).
  Logical volume vg_data/lv_logs successfully resized.
```

**Explication :** `lvextend -L +4,8G` ajoute **4,8 Go supplémentaires** au LV existant en prélevant les Physical Extents libres dans `vg_data`. Le signe `+` est crucial : sans lui, `-L 4,8G` définirait une taille absolue de 5G (ce qui réduirait le volume !). L'application continue de tourner pendant cette opération — **aucune interruption**.

<img width="651" height="117" alt="O" src="https://github.com/user-attachments/assets/2d1d9925-b87e-4c6f-9761-813240e1d7a8" />


---

#### Étape 2 — Extension du Système de Fichiers EXT4 à Chaud

```bash
[serge@server ~]$ sudo resize2fs /dev/vg_data/lv_logs
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/vg_data/lv_logs is mounted on /var/log/app_logs; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/vg_data/lv_logs is now 2621440 (4k) blocks long.
```

**Explication :** `lvextend` agrandit le **conteneur** (le LV), mais le système de fichiers EXT4 ne connaît pas encore ce nouvel espace. `resize2fs` étend le FS EXT4 pour occuper **tout l'espace disponible** dans le LV. La mention "on-line resizing required" confirme que l'opération est effectuée **à chaud**, sans démonter le volume. Les données existantes sont intégralement préservées.

<img width="648" height="118" alt="P" src="https://github.com/user-attachments/assets/99f2521b-3ce9-405a-9284-3b7f18cab3dc" />


> **💡 Note RHCSA :** Pour XFS (`lv_apps`), la commande équivalente serait `sudo xfs_growfs /mnt/applications`. XFS ne peut être étendu qu'à chaud (monté). `resize2fs` est spécifique à EXT2/3/4.

---

#### Étape 3 — Vérification Immédiate

```bash
[serge@server ~]$ df -h /var/log/app_logs
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/vg_data-lv_logs  9.6G  4.5G  4.7G  50% /var/log/app_logs
```

```bash
[serge@server ~]$ sudo lvs vg_data
  LV      VG      Attr       LSize
  lv_apps vg_data -wi-ao----  10.00g
  lv_logs vg_data -wi-ao----  9.80g
```

✅ `lv_logs` est maintenant à **9,8 Go**, l'utilisation est tombée à **50%**. L'application n'a subi **aucune interruption**. Les logs existants sont intacts.

<img width="643" height="170" alt="Q" src="https://github.com/user-attachments/assets/b99012b1-7ca9-4e3f-b780-ce3aaf13ff7e" />


---

### Récapitulatif de l'Extension à Chaud

```
AVANT                              APRÈS
──────────────────────────────     ──────────────────────────────
lv_logs : 5 Go   [████████░░] 99%  lv_logs : 9,80 Go  [████░░░░░░] 50%
vg_data free : 5 Go                vg_data free : ~0 Go

Durée totale de l'opération : < 2 minutes
Interruption de service      : 0 seconde
Perte de données             : 0 octet
```

---

## 🔐 Sécurité & Vérifications — Check-list RHCSA

### ✅ Check-list Critique Avant Redémarrage

Avant de redémarrer le serveur après toute modification de `/etc/fstab` ou de la configuration LVM, vérifier **chaque point** de la liste suivante :

---

#### 1. Validation Syntaxique de `/etc/fstab`

```bash
# Vérifier qu'aucun UUID n'est manquant ou mal copié
[serge@server ~]$ sudo cat /etc/fstab | grep -v "^#" | grep -v "^$"

# Simuler le montage de toutes les entrées fstab sans redémarrer
[serge@server ~]$ sudo mount -a && echo "✅ fstab valide"

# Outil de validation avancé (RHEL/Rocky)
[serge@server ~]$ sudo findmnt --verify
```

> ⚠️ **CRITIQUE :** Si `mount -a` retourne une erreur, **NE PAS redémarrer**. Corriger l'entrée fstab immédiatement. Un fstab invalide peut bloquer le démarrage en mode rescue.

---

#### 2. Vérification des UUID

```bash
# Comparer les UUID dans fstab avec ceux retournés par blkid
[serge@server ~]$ sudo blkid | grep -E "lv_logs|lv_apps"

# Lister les montages actuels avec leur UUID
[serge@server ~]$ sudo findmnt -o UUID,TARGET,FSTYPE,OPTIONS | grep -E "app_logs|applications"
```

> ✅ **Attendu :** Les UUID dans `fstab` correspondent exactement à ceux retournés par `blkid`.

---

#### 3. Vérification de l'Intégrité LVM

```bash
# État général de tous les composants LVM
[serge@server ~]$ sudo vgck vg_data
  No volume groups found with physical volumes present.

# Vérification détaillée
[serge@server ~]$ sudo lvmdiskscan
[serge@server ~]$ sudo pvs && sudo vgs && sudo lvs
```

---

#### 4. Vérification de l'Intégrité EXT4 (hors ligne uniquement)

```bash
# ATTENTION : ne peut être exécuté que si le FS est DÉMONTÉ
# Pour une vérification complète lors d'une fenêtre de maintenance :
[serge@server ~]$ sudo umount /var/log/app_logs
[serge@server ~]$ sudo e2fsck -f /dev/vg_data/lv_logs
[serge@server ~]$ sudo mount /var/log/app_logs
```

> ⚠️ **En production :** Utiliser les snapshots LVM pour effectuer une vérification sans démonter le FS en direct.

---

#### 5. Test de Persistance au Redémarrage (Validation Finale)

```bash
# Démonter manuellement les deux volumes
[serge@server ~]$ sudo umount /var/log/app_logs
[serge@server ~]$ sudo umount /mnt/applications

# Vérifier qu'ils sont bien démontés
[serge@server ~]$ df -h | grep -E "app_logs|applications"
# (aucun résultat attendu)

# Remonter via fstab uniquement
[serge@server ~]$ sudo mount -a

# Confirmer le remontage automatique
[serge@server ~]$ df -h | grep -E "app_logs|applications"
```

<img width="666" height="143" alt="R" src="https://github.com/user-attachments/assets/93d42e60-cbc1-4ad4-b85e-acbd52e86eee" />


---

#### 6. Récapitulatif de l'Architecture Finale

```bash
[serge@server ~]$ lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID
```

<img width="665" height="209" alt="S" src="https://github.com/user-attachments/assets/346f24d2-4bc2-464a-8da7-71ef60599dc7" />


---

### 🗂️ Tableau de Bord Final — État de l'Infrastructure

| Élément | Valeur | Statut |
|---------|--------|--------|
| PV `/dev/sdb` | 10 Go | ✅ Actif |
| PV `/dev/sdc` | 10 Go | ✅ Actif |
| VG `vg_data` | 19,99 Go | ✅ Actif |
| LV `lv_logs` | 9,80 Go (EXT4) | ✅ Monté sur `/var/log/app_logs` |
| LV `lv_apps` | 10 Go (XFS) | ✅ Monté sur `/mnt/applications` |
| Persistance `fstab` | UUID-based | ✅ Validé via `mount -a` |
| Extension à chaud | +4,8 Go EXT4 | ✅ Effectuée sans interruption |

---

## 👤 Auteur

**Serge TOGNON**
Admin cloud Azure/Linux | Préparation RHCSA

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Serge%20TOGNON-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/serge-tognon)

---

## 📜 Licence

Ce projet est distribué sous licence [MIT](LICENSE). Libre d'utilisation à des fins éducatives et professionnelles.

---

*Projet réalisé dans le cadre d'une infrastructure de production réelle et d'une préparation rigoureuse à la certification Red Hat RHCSA.*

