# 🔐 Lab RHCSA – Récupération du Mot de Passe Root sur RHEL 9

![RHEL 9](https://img.shields.io/badge/RHEL-9-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![SELinux](https://img.shields.io/badge/SELinux-Enforcing-FF6600?style=for-the-badge)
![Difficulté](https://img.shields.io/badge/Difficulté-Intermédiaire-orange?style=for-the-badge)
![Durée estimée](https://img.shields.io/badge/Durée-15--20%20min-blue?style=for-the-badge)

---

## 📋 Énoncé du Lab

> **Contexte d'examen — Style EX200**

Le système `server1.lab.local` est inaccessible suite à la perte du mot de passe de l'utilisateur **root**. Aucun autre compte utilisateur ne dispose de privilèges d'administration (groupe `wheel` vide). Le système démarre normalement mais aucune connexion privilégiée n'est possible.

**Votre mission :**

Sans réinstaller le système d'exploitation, rétablissez l'accès administrateur en définissant le mot de passe root à : **`Redhat@2024!`**

Le système doit, à l'issue de cette procédure :
- Démarrer normalement en mode multi-utilisateur
- Permettre l'authentification `root` avec le nouveau mot de passe
- Maintenir SELinux en mode **Enforcing**
- Conserver l'intégrité du contexte SELinux sur tous les fichiers système

---

## 🛠️ Solution Technique

### Vue d'ensemble de la procédure

```
Interruption GRUB → Shell minimal (rd.break) → Remontage /sysroot → 
Changement de mot de passe → Relabeling SELinux → Redémarrage
```

---

### Étape 1 — Interrompre le démarrage via GRUB

Au démarrage de la machine virtuelle, attendez l'apparition du menu GRUB (généralement 5 secondes).

**Action :** Appuyez sur `e` pour éditer l'entrée de démarrage sélectionnée.

> 📸 **[CAPTURE D'ÉCRAN — Menu GRUB : appuyer sur `e` pour éditer]**

---

Repérez la ligne commençant par `linux` (ou `linuxefi` selon le firmware). Elle ressemble à :

```
linux ($root)/vmlinuz-5.14.0-xxx.el9.x86_64 root=/dev/mapper/... ro crashkernel=auto ...
```

**Action :** Naviguez jusqu'à la fin de cette ligne et ajoutez le paramètre suivant :

```
rd.break
```

> ⚠️ **Important :** Supprimez ou remplacez `rhgb quiet` par `rd.break` pour une meilleure lisibilité de la sortie.

La ligne modifiée ressemblera à :

```
linux ($root)/vmlinuz-5.14.0-xxx.el9.x86_64 root=/dev/mapper/... ro crashkernel=auto rd.break
```

> 📸 **[CAPTURE D'ÉCRAN — Ligne `linux` modifiée avec `rd.break`]**

**Action :** Appuyez sur `Ctrl+X` pour démarrer avec ces paramètres.

Le système s'arrête en mode `initramfs` et présente un shell minimal :

```
switch_root:/#
```

> 📸 **[CAPTURE D'ÉCRAN — Invite de commande `switch_root:/#`]**

---

### Étape 2 — Remonter le système de fichiers racine en lecture/écriture

En mode `rd.break`, le système de fichiers racine réel (`/sysroot`) est monté en **lecture seule**. Il faut le remonter en **lecture/écriture** pour pouvoir modifier le fichier des mots de passe.

```bash
# Vérification de l'état actuel du montage
mount | grep sysroot
```

Sortie attendue (ro = read-only) :
```
/dev/mapper/rhel-root on /sysroot type xfs (ro,relatime,attr2,...)
```

```bash
# Remontage en lecture/écriture
mount -o remount,rw /sysroot
```

Vérification :
```bash
mount | grep sysroot
# /dev/mapper/rhel-root on /sysroot type xfs (rw,relatime,...)
```

> 📸 **[CAPTURE D'ÉCRAN — Sortie de `mount | grep sysroot` avant et après le remontage]**

---

### Étape 3 — Effectuer un chroot et changer le mot de passe root

La commande `chroot` permet de "basculer" la racine du système vers `/sysroot`, rendant ainsi le système RHEL réel actif dans ce shell.

```bash
# Bascule dans l'environnement chroot
chroot /sysroot
```

L'invite de commande change pour indiquer que vous êtes dans le système cible :

```
sh-5.1#
```

Changez maintenant le mot de passe root :

```bash
passwd root
```

Entrez et confirmez le nouveau mot de passe :

```
Changing password for user root.
New password: Redhat@2024!
Retype new password: Redhat@2024!
passwd: all authentication tokens updated successfully.
```

> 📸 **[CAPTURE D'ÉCRAN — Exécution de `passwd root` et confirmation du succès]**

---

### Étape 4 — Gestion du relabeling SELinux (étape critique)

> ⚠️ **Cette étape est indispensable.** En mode `rd.break`, les opérations effectuées dans `/sysroot` sont réalisées **hors du contrôle de SELinux**. Le fichier `/etc/shadow` (qui stocke les mots de passe) a été modifié sans que SELinux ait appliqué le bon contexte de sécurité. Si cette étape est omise, SELinux bloquera l'authentification au prochain démarrage.

**Solution :** Forcer un relabeling complet du système de fichiers au prochain démarrage.

```bash
# Créer le fichier de déclenchement du relabeling SELinux
touch /.autorelabel
```

> 💡 La présence de ce fichier vide indique à SELinux de réétiqueter l'intégralité du système de fichiers au prochain démarrage. Le fichier est automatiquement supprimé une fois l'opération terminée.

> 📸 **[CAPTURE D'ÉCRAN — Commande `touch /.autorelabel` exécutée]**

---

### Étape 5 — Quitter et redémarrer

Sortez de l'environnement chroot, puis quittez le shell initramfs pour déclencher le redémarrage.

```bash
# Sortir du chroot
exit

# Quitter l'initramfs (déclenche le redémarrage)
exit
```

> 📸 **[CAPTURE D'ÉCRAN — Processus de relabeling SELinux en cours au redémarrage (peut durer quelques minutes)]**

Le système effectue le relabeling SELinux (une barre de progression peut s'afficher), puis redémarre automatiquement.

---

### Étape 6 — Vérification finale

Une fois le système redémarré, connectez-vous avec les nouvelles credentials :

```
Login: root
Password: Redhat@2024!
```

Vérifiez que SELinux est bien en mode Enforcing :

```bash
getenforce
# Résultat attendu : Enforcing

sestatus
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux mount point:            /sys/fs/selinux
# Loaded policy name:             targeted
# Current mode:                   enforcing
# Mode from config file:          enforcing
```

> 📸 **[CAPTURE D'ÉCRAN — Connexion root réussie + sortie de `getenforce` et `sestatus`]**

---

## 📊 Récapitulatif des commandes

| # | Commande | Objectif |
|---|----------|----------|
| 1 | Édition GRUB → ajout `rd.break` | Interrompre le boot en mode initramfs |
| 2 | `mount -o remount,rw /sysroot` | Remonter `/sysroot` en lecture/écriture |
| 3 | `chroot /sysroot` | Basculer dans l'environnement système cible |
| 4 | `passwd root` | Définir le nouveau mot de passe root |
| 5 | `touch /.autorelabel` | Déclencher le relabeling SELinux |
| 6 | `exit` → `exit` | Quitter et redémarrer |
| 7 | `getenforce` / `sestatus` | Vérifier l'état de SELinux |

---

## 💡 Why This Matters

### Pourquoi cette compétence est-elle essentielle pour un administrateur système ?

**1. Scénario réel fréquent en entreprise**
La perte du mot de passe root est l'un des incidents les plus courants en administration système. Elle peut survenir lors d'une rotation de personnel, d'un changement de politique de sécurité non documenté, ou d'une simple erreur humaine. Savoir y répondre sans réinstaller le système est une compétence de première nécessité.

**2. Compréhension du processus de boot Linux**
Cette procédure implique une maîtrise concrète de la chaîne de démarrage : GRUB → kernel → initramfs → systemd. Comprendre où et comment intervenir dans ce processus est fondamental pour tout administrateur RHEL/Linux.

**3. Maîtrise de SELinux en situation de récupération**
SELinux est activé par défaut sur RHEL et constitue une couche de sécurité critique. La plupart des guides génériques sur la récupération de mot de passe omettent l'étape `touch /.autorelabel`, ce qui entraîne un système inutilisable après redémarrage. Cette connaissance distingue un administrateur averti d'un débutant.

**4. Compétence directement évaluée à l'examen EX200**
Le domaine **"Manage users and groups"** de la certification RHCSA inclut explicitement la capacité à récupérer l'accès root. C'est l'une des tâches les plus mémorisées par les candidats car son échec (oubli du relabeling SELinux) peut invalider plusieurs autres tâches de l'examen.

**5. Principe de moindre interruption**
Récupérer l'accès sans réinstallation préserve les données, la configuration et la disponibilité du service. En environnement de production, chaque minute d'indisponibilité a un coût. Cette procédure peut être exécutée en moins de 15 minutes.

---

## 🔗 Ressources complémentaires

- [Documentation Red Hat — Resetting the Root Password](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/)
- [Red Hat SELinux User's and Administrator's Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/using_selinux/)
- [Objectifs officiels de l'examen RHCSA EX200](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam)

---

## ✍️ Auteur

> Lab réalisé dans le cadre de la préparation à la certification **RHCSA EX200** — Red Hat Enterprise Linux 9.

---

*Dernière mise à jour : 2026 | Environnement : RHEL 9 | SELinux : Enforcing*
