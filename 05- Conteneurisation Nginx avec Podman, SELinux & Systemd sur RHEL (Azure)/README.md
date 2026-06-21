# 🐳            Conteneurisation Nginx avec Podman, SELinux & Systemd sur RHEL (Azure)

![RHEL](https://img.shields.io/badge/OS-RHEL_9%2F10-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![Azure](https://img.shields.io/badge/Cloud-Microsoft_Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Podman](https://img.shields.io/badge/Conteneur-Podman-892CA0?style=for-the-badge&logo=podman&logoColor=white)
![Systemd](https://img.shields.io/badge/Init-Systemd-2B2E83?style=for-the-badge&logo=linux&logoColor=white)
![SELinux](https://img.shields.io/badge/Sécurité-SELinux-CC0000?style=for-the-badge&logo=redhat&logoColor=white)

> **Objectif :** Déployer un serveur web Nginx conteneurisé avec **Podman** (rootless, sans Docker) sur une VM **RHEL 9** hébergée sur **Azure**, en assurant la persistance des données via un volume local, le respect des contraintes **SELinux**, et l'automatisation complète du cycle de vie via **systemd**.

---

## 📋 Table des matières

1. [Présentation du projet](#-présentation-du-projet)
2. [Architecture technique](#️-architecture-technique)
3. [Guide d'installation étape par étape](#-guide-dinstallation-étape-par-étape)
4. [Gestion de la sécurité (SELinux)](#-gestion-de-la-sécurité-selinux)
5. [Automatisation avec Systemd](#️-automatisation-avec-systemd)
6. [Section Dépannage (Troubleshooting)](#️-section-dépannage-troubleshooting)
7. [Compétences démontrées](#-compétences-démontrées)
8. [Comment aller plus loin](#-comment-aller-plus-loin)

---

## 📌 Présentation du projet

Ce projet démontre la mise en place d'un environnement de conteneurisation **rootless** et **production-ready** sur une VM RHEL 9/10 hébergée sur Azure, en s'appuyant sur **Podman** plutôt que Docker.

L'objectif n'est pas seulement de faire tourner un conteneur Nginx, mais de répondre aux exigences réelles d'un environnement d'entreprise sous RHEL :

- **Sécurité** : exécution rootless et contraintes SELinux respectées (pas de désactivation de SELinux).
- **Persistance** : les données du site web survivent à la suppression/recréation du conteneur.
- **Résilience** : le conteneur redémarre automatiquement avec la machine grâce à `systemd`, sans dépendre d'un démon tiers.
- **Industrialisation** : génération du service via Podman lui-même (`podman generate systemd`), conforme aux pratiques RHEL.

<img width="960" height="212" alt="B" src="https://github.com/user-attachments/assets/9327de1c-8e33-43a0-803b-cd0bf2631513" />



---

## 🏗️ Architecture technique

### Pourquoi Podman et pas Docker ?

Sur RHEL, **Podman est l'outil de conteneurisation officiellement supporté par Red Hat** depuis RHEL 8, et c'est un choix structurant pour plusieurs raisons :

| Critère | Podman | Docker |
|---|---|---|
| Démon (daemon) | Aucun (architecture "daemonless") | `dockerd` requis en arrière-plan |
| Exécution rootless | Native et complète | Limitée / configuration complexe |
| Intégration systemd | Native (`podman generate systemd`) | Nécessite des contournements |
| Support Red Hat | Officiel (RHEL/OpenShift) | Non supporté nativement |

L'absence de démon signifie qu'**il n'y a pas de processus root permanent** qui pourrait constituer une surface d'attaque. Chaque conteneur Podman est un processus enfant directement rattaché à l'utilisateur qui l'a lancé , ce qui s'aligne parfaitement avec le principe de moindre privilège appliqué en administration système RHEL.

### Pourquoi Systemd plutôt qu'un script de démarrage ?

`systemd` est le gestionnaire de services natif de RHEL. En générant une unité directement depuis Podman permet de :

- Bénéficier du redémarrage automatique en cas de crash (`Restart=on-failure`).
- Gérer le conteneur avec les commandes standards (`systemctl start/stop/status`).
- Centraliser les logs dans `journald`, déjà utilisé pour tout le reste du système.
- Garantir le démarrage du conteneur **au boot**, sans dépendre d'une session utilisateur ouverte (via les services "rootless" en mode `--user` + `loginctl enable-linger`).


<img width="1286" height="832" alt="IM IMM" src="https://github.com/user-attachments/assets/066c3197-1b7a-40cb-8222-9ce147c7a9ef" />

---

## 🚀 Guide d'installation étape par étape

### Prérequis

- Un abonnement Azure actif et Azure CLI (`az`) installé/connecté en local (`az login`).
- Le dépôt BaseOS/AppStream activé sur la VM (Podman y est inclus par défaut sur RHEL 9/10).

### 0. Provisioning de l'infrastructure Azure (Resource Group + VM RHEL)

Si la VM n'existe pas encore, voici la séquence complète pour la créer depuis zéro :

**Variables**
```bash

RG="rg-podman-nginx-demo"
LOCATION="canadacentral"
VM_NAME="vm-rhel-podman"
ADMIN_USER="serge"

```

 **1. Création du groupe de ressources**
az group create --name $RG --location $LOCATION

**2. Création de la VM RHEL 9 avec clé SSH générée automatiquement**
```bash

az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --image RedHat:rhel:9_8-arm64:latest \
  --size  Standard_B2ps_v2\
  --admin-username $ADMIN_USER \
  --generate-ssh-keys \
  --public-ip-sku Standard
```
 **3. Ouverture du port 8080 (Nginx via Podman) dans le NSG**
```bash
 
az vm open-port \
  --resource-group $RG \
  --name $VM_NAME \
  --port 8080 \
  --priority 1001
```
  **4. Récupération de l'IP publique pour la connexion SSH**
  ```bash
az vm show -d --resource-group $RG --name $VM_NAME --query publicIps -o tsv
```

**5. Connexion SSH à la VM**
```bash

ssh $ADMIN_USER@<IP_PUBLIQUE>
```

<img width="917" height="389" alt="1" src="https://github.com/user-attachments/assets/ea739b20-a5aa-43da-9ca6-cde0652c94d8" />


**1. Installation de Podman**

```bash

sudo dnf install -y podman
podman --version

```

<img width="960" height="381" alt="2" src="https://github.com/user-attachments/assets/f6981a91-ce18-4207-acf2-50b3eee786a3" />


**2. Création du répertoire de persistance des données**

On crée un répertoire local qui contiendra les fichiers HTML du site, **avant** de lancer le conteneur :

```bash

mkdir -p ~/nginx-data/html

echo "<h1>Bienvenue sur mon serveur Nginx conteneurise !</h1>" > ~/nginx-data/html/index.html

```

<img width="960" height="114" alt="3" src="https://github.com/user-attachments/assets/0bde159d-e918-4c5e-9304-62fb1c08bf96" />


**3. Création d'un pod Podman**

Un pod regroupe un ou plusieurs conteneurs partageant le même espace réseau (utile pour ajouter facilement un sidecar plus tard, comme un exporter Prometheus) :

```bash

podman pod create --name webserver-pod -p 8080:80

```

**4. Lancement du conteneur Nginx avec volume monté**

La commande suivante permet d'instancier le serveur web au sein de l'infrastructure Pod définie précédemment. Elle respecte les principes de conteneurisation rootless (sans privilèges root) et assure la conformité avec la politique de sécurité SELinux de RHEL.

```bash

podman run -d \
  --pod webserver-pod \
  --name nginx-web \
  -v ~/nginx-data/html:/usr/share/nginx/html:Z \
  nginx:latest

```
  
  **Analyse de la commande:**

  --pod : Rattache le conteneur au Pod.

-v ... :Z : Monte le volume avec un étiquetage SELinux privé (container_file_t) pour autoriser l'accès en mode Enforcing.


**5. Vérification**

```bash

podman ps
curl http://localhost:8080

```

<img width="952" height="163" alt="A" src="https://github.com/user-attachments/assets/5d305a49-5ef8-44c2-a5ce-327ffb09138f" />

<img width="960" height="212" alt="B" src="https://github.com/user-attachments/assets/41fc7a86-e4ce-4af9-b883-f762d7ceddcc" />

---

## 🔒 Gestion de la sécurité (SELinux)

### Pourquoi le flag `:Z` est-il indispensable ?

Sur RHEL, **SELinux reste activé en mode `enforcing`** (bonne pratique que ce projet respecte intégralement ,il n'est jamais question de passer en `permissive` ou `disabled`).

Par défaut, SELinux applique un **étiquetage (labeling) strict** aux fichiers. Le répertoire `~/nginx-data/html` possède un contexte SELinux de type `user_home_t`, alors que les processus conteneurisés s'exécutent avec un contexte de type `container_file_t`. Sans intervention, SELinux **bloque l'accès** du conteneur à ce volume, car les contextes ne correspondent pas. c'est le comportement attendu, pas un bug.

Le flag `:Z` (majuscule) demandé lors du montage du volume indique à Podman de :

1. Ré-étiqueter automatiquement le contenu du répertoire avec le contexte `container_file_t`.
2. Appliquer un label **privé**, exclusif à ce conteneur (par opposition au `:z` minuscule, qui autoriserait le partage du volume entre plusieurs conteneurs).

```bash

- v ~/nginx-data/html:/usr/share/nginx/html:Z

```


**Sans ce flag**, la commande `curl` renverrait une erreur `403 Forbidden`, et les logs `journalctl` afficheraient des `AVC denied` , la preuve que SELinux fonctionne correctement en empêchant un accès non autorisé.

### Vérifier les refus SELinux 

```bash
sudo ausearch -m avc -ts recent
```

<img width="942" height="153" alt="C" src="https://github.com/user-attachments/assets/e41a663d-3e49-4c54-9db2-11bb3b55628e" />


 La commande **sudo ausearch -m avc -ts recent** renvoie **no matches**, ce qui prouve que notre étiquetage est correct et que SELinux ne bloque aucun accès légitime.

---

## ⚙️ Automatisation avec Systemd (Méthode Robuste)

Voici une version mise à jour, propre et professionnelle pour votre `README.md`. Comme nous avons découvert que **Quadlet** n'est pas présent sur votre version actuelle de RHEL, j'ai adapté la documentation pour la **méthode robuste et universelle** (`podman generate systemd`). Cette version garantit que votre projet fonctionnera sur n'importe quel RHEL 9.

---

## ⚙️ Automatisation avec Systemd (Méthode Robuste)

L'automatisation est une étape clé pour transformer un conteneur en un service de production. Contrairement aux scripts manuels, l'utilisation de `systemd` permet à votre serveur web de redémarrer automatiquement après un crash ou un redémarrage de la machine, tout en centralisant les logs dans `journald`.


### 1. Génération automatique du service Systemd

Podman peut générer automatiquement le fichier de configuration `systemd` requis à partir de l'infrastructure déjà en place. Cela évite les erreurs de syntaxe manuelle.

```bash
# Création du répertoire de service pour l'utilisateur
mkdir -p ~/.config/systemd/user/

# Génération des fichiers .service
podman generate systemd --name webserver-pod --files --new

```

### 2. Activation du mode « Linger »

Pour garantir que nos services se lancent automatiquement au démarrage du système, même si nous ne sommes pas connectés (`rootless`), nous activons le mode "linger" pour notre utilisateur :

```bash
# Activation du linger pour l'utilisateur
sudo loginctl enable-linger $(whoami)

# Vérification
loginctl show-user $(whoami) --property=Linger

```
<img width="957" height="111" alt="E" src="https://github.com/user-attachments/assets/76d2a238-8706-48b7-90ce-97429b32ee37" />

### 4. Installation et démarrage du service

Déplaçons maintenant les fichiers générés vers le répertoire de configuration `systemd` et activons le service :

```bash
# Déplacement des fichiers générés
mv pod-webserver-pod.service container-nginx-web.service ~/.config/systemd/user/

# Recharger la configuration système
systemctl --user daemon-reload

# Activer et démarrer le service immédiatement
systemctl --user enable --now pod-webserver-pod.service

```
<img width="944" height="80" alt="image" src="https://github.com/user-attachments/assets/1a9e86d4-0312-4987-a5a5-5ef5c78b6adc" />

### 5. Vérification du statut

Vérifions que notre infrastructure est correctement prise en charge par `systemd`. Le statut doit indiquer `active (running)`.

```bash
# Vérification du statut du service
systemctl --user status pod-webserver-pod.service

```
<img width="953" height="301" alt="F" src="https://github.com/user-attachments/assets/ac285b22-3dd6-41ef-9607-907a281cd730" />


<img width="960" height="175" alt="G" src="https://github.com/user-attachments/assets/873b07bb-59c7-4b30-bca8-f186e733a025" />


---

### 💡 Note sur les méthodes d'automatisation

*Note technique : Bien que **Quadlet** soit la méthode déclarative moderne recommandée dans les versions récentes de Podman, la méthode `podman generate systemd` utilisée ici reste le standard industriel sur RHEL pour garantir la compatibilité et la robustesse de vos services en environnement d'entreprise.*

---

## 🛠️ Section Dépannage (Troubleshooting)

### Consulter les logs du service via `journalctl`

C'est le réflexe n°1 en administration système RHEL face à un service qui ne démarre pas :

```bash
journalctl --user -u pod-webserver-pod.service -f
```

- `-u` cible l'unité spécifique.
- `-f` suit les logs en temps réel (équivalent de `tail -f`).
- `--user` est indispensable ici car le service tourne en mode utilisateur (rootless).

### Cas fréquents rencontrés

| Symptôme | Cause probable | Commande de diagnostic |
|---|---|---|
| `403 Forbidden` sur `curl` | Volume non étiqueté SELinux (`:Z` manquant) | `sudo ausearch -m avc -ts recent` |
| Service inactif après reboot | Linger non activé | `loginctl show-user $(whoami)` |
| Port 8080 inaccessible depuis l'extérieur | NSG Azure non ouvert | `az vm open-port` (voir étape 0.3) |
| Port 8080 inaccessible même en local sur la VM | Pare-feu RHEL (`firewalld`) bloque le port | `sudo firewall-cmd --list-ports` |
| Conteneur "Exited" immédiatement | Erreur de configuration Nginx | `podman logs nginx-web` |

### Ouvrir le port dans le pare-feu local RHEL (si nécessaire)

```bash
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

> ⚠️ Sur Azure, **deux couches** filtrent le trafic : le NSG (au niveau réseau Azure) et `firewalld` (au niveau de l'OS). Les deux doivent autoriser le port 8080 pour un accès externe complet.


---

## 🎯 Compétences démontrées

Ce projet illustre une maîtrise concrète des compétences suivantes, directement transférables à un poste d'**Ingénieur Cloud & Infrastructure Linux** :

- ✅ Conteneurisation rootless avec **Podman** (alternative supportée par Red Hat à Docker).
- ✅ Compréhension fine et application correcte des **contextes SELinux** (`:Z` / `:z`), sans jamais désactiver le module de sécurité.
- ✅ Industrialisation du cycle de vie applicatif via **systemd** (génération d'unité, linger, gestion utilisateur).
- ✅ Diagnostic méthodique via `journalctl` et `ausearch`.
- ✅ Gestion réseau et pare-feu (`firewalld`) sur RHEL.
- ✅ Capacité à documenter un projet technique de manière claire et reproductible,compétence clé pour le travail en équipe DevOps.

---

## 🔭 Comment aller plus loin

Pour industrialiser davantage ce projet et démontrer une approche **Infrastructure as Code**, l'étape naturelle suivante consisterait à :

- Écrire un **playbook Ansible** automatisant l'installation de Podman, la création du volume, le lancement du pod et la génération/activation du service systemd ,rendant le déploiement entièrement idempotent et reproductible sur plusieurs VMs.
- Utiliser le module `containers.podman` de la collection Ansible officielle (`ansible-galaxy collection install containers.podman`).
- Stocker la configuration Nginx et les playbooks dans ce même dépôt Git pour une traçabilité complète.




## 👤 Auteur

**Serge TOGNON**
Cloud & Linux Infrastructure Engineer , AZ-104 Certified · RHCSA (EX200) Candidate
- 🔗 GitHub : [Serge9794](https://github.com/Serge9794)
- 💼 LinkedIn : [Serge TOGNON](https://linkedin.com/in/serge-tognon-a63443187)

---

*Lab réalisé dans le cadre de la formation en administration système Linux & Cloud Azure — © 2026*

