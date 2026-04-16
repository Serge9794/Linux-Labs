
# 🔐 Lab Linux — Tunneling SSH & Redirection de Port Local

![Linux](https://img.shields.io/badge/OS-Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![VirtualBox](https://img.shields.io/badge/Hyperviseur-VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)
![SSH](https://img.shields.io/badge/Protocole-SSH-4A90D9?style=for-the-badge&logo=openssh&logoColor=white)
![Python](https://img.shields.io/badge/Serveur_Web-Python_3-3776AB?style=for-the-badge&logo=python&logoColor=white)

> **Objectif :** Démontrer la mise en place d'un tunnel SSH avec redirection de port local (*Local Port Forwarding*) pour accéder de manière sécurisée à un service web hébergé sur un serveur distant, depuis une machine cliente, sans exposer le service sur le réseau.

---

## 📋 Table des matières

1. [Architecture du réseau](#-architecture-du-réseau)
2. [Prérequis](#-prérequis)
3. [Guide pas à pas](#-guide-pas-à-pas)
   - [Étape 1 — Préparation du serveur](#étape-1--préparation-du-serveur)
   - [Étape 2 — Lancement du serveur web](#étape-2--lancement-du-serveur-web)
   - [Étape 3 — Création du tunnel SSH](#étape-3--création-du-tunnel-ssh)
   - [Étape 4 — Validation depuis le client](#étape-4--validation-depuis-le-client)
4. [Concepts clés](#-concepts-clés)
5. [Résultats & Captures d'écran](#-résultats--captures-décran)
6. [Compétences démontrées](#-compétences-démontrées)

---

## 🗺️ Architecture du réseau

L'environnement de lab repose sur deux machines virtuelles Ubuntu interconnectées via un **réseau interne VirtualBox** (`intnet`), isolé du réseau physique hôte.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Réseau Interne VirtualBox                   │
│                      (192.168.10.0/24)                          │
│                                                                 │
│   ┌─────────────────────┐        ┌────────────────────────┐    │
│   │   💻 Machine Cliente │        │   🖥️  Machine Serveur   │    │
│   │   Utilisateur: anne  │        │   Utilisateur: serge   │    │
│   │   IP: 192.168.10.1  │        │   IP: 192.168.10.2     │    │
│   │                     │        │                        │    │
│   │  Firefox            │        │  index.html            │    │
│   │  localhost:9999 ────┼──SSH───┼──> Port 8989           │    │
│   │  (port local)       │  :22   │  (python http.server)  │    │
│   └─────────────────────┘        └────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘

  Flux du tunnel : CLIENT:9999 ──[chiffré SSH]──> SERVEUR:8989
```

| Rôle | Machine | Utilisateur | Adresse IP |
|------|---------|-------------|------------|
| 🖥️ Serveur | Ubuntu Server | `serge` | `192.168.10.2` |
| 💻 Client | Ubuntu Desktop | `anne` | `192.168.10.1` |

---

## ✅ Prérequis

- [x] VirtualBox installé sur la machine hôte
- [x] Deux VMs Ubuntu configurées sur un réseau interne
- [x] Service SSH (`openssh-server`) installé et actif sur le **serveur**
- [x] Python 3 disponible sur le **serveur**
- [x] Navigateur Firefox disponible sur le **client**

Vérifier que SSH est actif sur le serveur :
```bash
sudo systemctl status ssh
```

---

##  Guide pas à pas

### Étape 1 — Préparation du serveur

Se connecter sur la **machine serveur** (`serge@192.168.10.2`) et créer la page web à exposer.

```bash
# Créer un répertoire de travail
mkdir -p ~/weblab && cd ~/weblab

# Créer la page HTML
echo "Bienvenue sur l'interface privee de serge" > index.html

# Vérifier le contenu du fichier
cat index.html
```

**Résultat attendu :**
```
Bienvenue sur l'interface privee de serge
```

> 📸 <img width="1596" height="974" alt="1" src="https://github.com/user-attachments/assets/676f92eb-6a21-47fa-82c6-71dee41335c1" />


---

### Étape 2 — Lancement du serveur web

Toujours sur la **machine serveur**, démarrer un serveur HTTP avec Python 3 sur le port `8989`.

```bash
# Se placer dans le répertoire contenant index.html
cd ~/weblab

# Lancer le serveur web Python sur le port 8989
python3 -m http.server 8989
```

**Résultat attendu dans le terminal :**
```
Serving HTTP on 0.0.0.0 port 8989 (http://0.0.0.0:8989/) ...
```

> ⚠️ **Important :** Le service écoute sur `0.0.0.0`, mais grâce au tunnel SSH, il ne sera accessible que via une connexion authentifiée. Aucun port n'est ouvert publiquement.

> 📸 <img width="1920" height="974" alt="2" src="https://github.com/user-attachments/assets/cb41432d-26a8-4fd7-bde3-0300bfb209e2" />


---

### Étape 3 — Création du tunnel SSH

Depuis la **machine cliente** (`anne@192.168.10.1`), ouvrir un **nouveau terminal** et établir le tunnel SSH avec redirection de port local.

```bash
# Syntaxe générale :
# ssh -L [port_local]:[hôte_distant]:[port_distant] [utilisateur]@[serveur]

ssh -L 9999:localhost:8989 serge@192.168.10.2
```

**Décomposition de la commande :**

| Paramètre | Valeur | Signification |
|-----------|--------|---------------|
| `-L` | — | Active le *Local Port Forwarding* |
| `9999` | Port local | Port ouvert sur la machine cliente |
| `localhost:8989` | Cible distante | Le serveur web vu depuis le serveur SSH |
| `serge@192.168.10.2` | Destination SSH | Connexion SSH vers le serveur |

Saisir le mot de passe de `serge` lorsque demandé. **Laisser ce terminal ouvert** : le tunnel reste actif tant que la session SSH est ouverte.

> 📸 <img width="1920" height="974" alt="3" src="https://github.com/user-attachments/assets/95a7b867-1fb2-4c61-85c2-3a688bb45ceb" />


---

### Étape 4 — Validation depuis le client

Sans fermer le terminal SSH, ouvrir **Firefox** sur la machine cliente et naviguer vers :

```
http://localhost:9999
```

> 📸 <img width="1920" height="974" alt="4" src="https://github.com/user-attachments/assets/6ee9b91a-8586-4376-b38e-82cd8e1f99d6" />


✅ **Succès !** Le contenu hébergé sur le serveur (`port 8989`) est accessible depuis le client (`port 9999`) via le tunnel SSH chiffré.

---

## 🧠 Concepts clés

### 🔒 Qu'est-ce que le Local Port Forwarding SSH ?

Le **Local Port Forwarding** (ou redirection de port local) est une fonctionnalité du protocole SSH qui permet de **rediriger le trafic d'un port local vers un port distant**, en faisant transiter les données à travers le **canal SSH chiffré**.

```
Machine Cliente                    Machine Serveur
─────────────                      ───────────────
  Application  ──► localhost:9999
                         │
                   [Tunnel SSH chiffré sur le port 22]
                         │
                         └──────────────► localhost:8989
                                              │
                                         Serveur Web
                                         (Python)
```

### 🛡️ Pourquoi est-ce sécurisé ?

| Aspect | Explication |
|--------|-------------|
| **Chiffrement de bout en bout** | Toutes les données transitent dans le tunnel SSH (AES-256), rendant l'interception inutile |
| **Authentification forte** | Seul un utilisateur authentifié sur le serveur SSH peut créer le tunnel |
| **Pas d'exposition directe** | Le port `8989` n'est pas accessible depuis l'extérieur — uniquement via SSH |
| **Isolation du service** | Le service web reste lié à `localhost` du serveur, invisible sur le réseau |

### 🆚 Comparaison : Avec vs Sans tunnel SSH

| Scénario | Accès direct (sans SSH) | Tunnel SSH |
|----------|------------------------|------------|
| Exposition réseau | ✅ Port 8989 ouvert sur le réseau | ❌ Port 8989 non exposé |
| Chiffrement | ❌ Trafic HTTP en clair | ✅ Trafic chiffré SSH |
| Authentification requise | ❌ Aucune (selon config) | ✅ Credentials SSH obligatoires |
| Risque d'interception | ⚠️ Élevé | ✅ Très faible |

---

## 👤 Auteur

**[Serge TOGNON]**
- 🔗 GitHub : [@votre-username](https://github.com/Serge9794)
- 💼 LinkedIn : [Votre Profil](https://linkedin.com/in/Serge_TOGNON)

---

*Lab réalisé dans le cadre d'une formation en administration système Linux — © 2026*
