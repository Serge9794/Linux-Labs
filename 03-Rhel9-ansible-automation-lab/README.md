# 🚀 Ansible Automation Lab — RHCSA Training

<div align="center">

```
╔═══════════════════════════════════════════════════════════════╗
║        🤖  ANSIBLE AUTOMATION LAB — RHCSA TRAINING  🎯       ║
║        Automatisation Linux | Red Hat | Infrastructure        ║
╚═══════════════════════════════════════════════════════════════╝
```

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![RHEL](https://img.shields.io/badge/RHEL-CC0000?style=for-the-badge&logo=red-hat&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![SSH](https://img.shields.io/badge/SSH-4D4D4D?style=for-the-badge&logo=gnubash&logoColor=white)
![RHCSA](https://img.shields.io/badge/RHCSA-Exam_Ready-009639?style=for-the-badge)

*Laboratoire pratique d'automatisation pour la préparation à la certification RHCSA*

</div>

---

## 📋 Table des Matières

1. [🏗️ Architecture](#️-architecture)
2. [👤 Étape 1 — Création de l'utilisateur `ansible`](#-étape-1--création-de-lutilisateur-ansible)
3. [⚙️ Étape 2 — Installation d'Ansible](#️-étape-2--installation-dansible)
4. [🔑 Étape 3 — Configuration SSH par clé](#-étape-3--configuration-ssh-par-clé)
5. [📦 Étape 4 — Création de l'inventaire](#-étape-4--création-de-linventaire)
6. [📜 Étape 5 — Premier Playbook](#-étape-5--premier-playbook)
7. [👨‍💻 Auteur](#-auteur)
8. [🎓 Conclusion RHCSA](#-conclusion-rhcsa)

---

## 🏗️ Architecture

Ce lab repose sur **2 machines virtuelles RHEL** interconnectées en réseau local, avec un utilisateur système dédié `ansible` présent sur les deux machines :

| 🖥️ Rôle | 🏷️ Hostname | 🌐 Adresse IP | 👤 Utilisateur | 📌 Fonction |
|---|---|---|---|---|
| **Control Node** | `client.lab.local` | `192.168.10.1` | `ansible` | Exécute Ansible, lance les playbooks |
| **Managed Node** | `server.lab.local` | `192.168.10.2` | `ansible` | Cible gérée par Ansible via SSH |

### Schéma de l'architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     Réseau : 192.168.10.0/24                   │
│                                                                 │
│   ┌──────────────────────┐  SSH (clé RSA)  ┌────────────────┐  │
│   │    VM1 — Control     │ ──────────────► │  VM2 — Managed │  │
│   │  client.lab.local    │                 │server.lab.local│  │
│   │   192.168.10.1       │ ◄──────────────  │  192.168.10.2 │  │
│   │  user: ansible 🤖    │    Réponses     │ user: ansible  │  │
│   │  [Ansible Engine]    │                 │  [Apache/App]  │  │
│   └──────────────────────┘                 └────────────────┘  │
│                                                                 │
│   💡 L'utilisateur "ansible" dispose des droits sudo sans      │
│      mot de passe sur les deux machines.                        │
└────────────────────────────────────────────────────────────────┘
```

<img width="182" height="50" alt="A" src="https://github.com/user-attachments/assets/101b6863-e514-4018-acde-ee7b0cf7ee25" />
<img width="176" height="43" alt="B" src="https://github.com/user-attachments/assets/f4079ade-ea72-4fa9-88bc-3c0a52a0717a" />


---

## 👤 Étape 1 — Création de l'utilisateur `ansible`

> 📍 **Effectuer sur : LES DEUX VMs** — `client.lab.local` ET `server.lab.local`

L'utilisateur `ansible` est le compte système **dédié à toutes les opérations d'automatisation**. Il doit exister sur chaque machine du lab et disposer des droits `sudo` sans mot de passe pour que les playbooks s'exécutent de façon non interactive.

---

### 1.1 — Créer l'utilisateur `ansible`

```bash
# Créer l'utilisateur ansible avec un répertoire home et un shell bash
sudo useradd -m -s /bin/bash ansible

# Définir un mot de passe (nécessaire pour la première connexion SSH)
sudo passwd ansible
# → Saisir et confirmer le mot de passe souhaité
```

### 1.2 — Accorder les droits sudo sans mot de passe

```bash
# Créer un fichier dédié dans sudoers.d (bonne pratique, évite de modifier /etc/sudoers)
sudo tee /etc/sudoers.d/ansible << EOF
# Permissions sudo sans mot de passe pour l'automatisation Ansible
ansible ALL=(ALL) NOPASSWD: ALL
EOF

# Appliquer les permissions correctes (obligatoire)
sudo chmod 440 /etc/sudoers.d/ansible

# Vérifier la syntaxe du fichier pour éviter tout blocage sudo
sudo visudo -c
# Résultat attendu : /etc/sudoers: parsed OK
<img width="421" height="275" alt="C" src="https://github.com/user-attachments/assets/a57b542a-135d-442a-b558-3386ee003863" />
<img width="452" height="217" alt="F" src="https://github.com/user-attachments/assets/952fc020-9faa-44b3-baf6-00fae15dc66a" />


```

### 1.3 — Vérifier les droits sudo de l'utilisateur `ansible`

```bash
# Basculer vers l'utilisateur ansible
su - ansible

# Tester l'élévation sudo (aucun mot de passe ne doit être demandé)
sudo whoami
# Résultat attendu : root

# Tester une commande système avec droits root
sudo systemctl status sshd

# Revenir à votre utilisateur initial
exit
<img width="618" height="293" alt="D" src="https://github.com/user-attachments/assets/2915f5c8-6bfc-4093-84ce-57c8dd672dee" />
<img width="640" height="270" alt="G" src="https://github.com/user-attachments/assets/0d17003e-7797-450e-8a00-833a11c1fb79" />


```

### 1.4 — Vérifier la création de l'utilisateur

```bash
# Vérifier l'UID, GID et les groupes
id ansible
# Résultat attendu :
# uid=1001(ansible) gid=1001(ansible) groups=1001(ansible)

# Vérifier l'entrée dans /etc/passwd
grep ansible /etc/passwd
# ansible:x:1001:1001::/home/ansible:/bin/bash

# Vérifier l'existence du répertoire home
ls -la /home/ansible/
```

> <img width="506" height="247" alt="E" src="https://github.com/user-attachments/assets/313a88c9-52a4-4677-b3a9-3dedec0f5add" />
<img width="553" height="196" alt="H" src="https://github.com/user-attachments/assets/46745694-d7a3-4e61-8ce8-c78ec2efa2c4" />



> ⚠️ **Important :** Répéter toutes les commandes de cette étape sur **les deux machines** avant de continuer.

---

## ⚙️ Étape 2 — Installation d'Ansible

> 📍 **Effectuer sur :** `client.lab.local` (VM1 — Control Node) **en tant qu'utilisateur `ansible`**

```bash
# Basculer vers l'utilisateur ansible
su - ansible

# Vérifier l'identité active
whoami
# → ansible
```

### 2.1 — Activer le dépôt EPEL et installer Ansible

```bash
# Installer le dépôt EPEL (Extra Packages for Enterprise Linux)
sudo dnf install -y epel-release

# Mettre à jour le système
sudo dnf update -y

# Installer Ansible
sudo dnf install -y ansible

# Vérifier l'installation
ansible --version
```

### 2.2 — Vérifier la version installée

```bash
ansible --version
# Résultat attendu :
# ansible [core 2.x.x]
#   config file = /etc/ansible/ansible.cfg
#   configured module search path = ['/home/ansible/.ansible/plugins/modules']
#   python version = 3.x.x
```

> [IMAGE_ICI — Capture d'écran : Sortie de `ansible --version` exécuté sous l'utilisateur `ansible`]

---

## 🔑 Étape 3 — Configuration SSH par Clé

> 📍 **Effectuer sur :** `client.lab.local` (VM1) **en tant qu'utilisateur `ansible`**

```bash
# S'assurer d'être connecté en tant qu'ansible
whoami   # → ansible
```

### 3.1 — Générer la paire de clés SSH pour `ansible`

```bash
# Générer une paire de clés RSA 4096 bits
ssh-keygen -t rsa -b 4096 -C "ansible@client.lab.local"

# Appuyer sur Entrée pour accepter le chemin par défaut :
#   /home/ansible/.ssh/id_rsa      ← clé privée
#   /home/ansible/.ssh/id_rsa.pub  ← clé publique

# Laisser la passphrase VIDE (l'automatisation ne peut pas saisir de mot de passe)
```

### 3.2 — Configurer `/etc/hosts` (si DNS non disponible)

```bash
# Sur les DEUX VMs, ajouter les résolutions de noms
sudo tee -a /etc/hosts << EOF
192.168.10.1  client.lab.local  client
192.168.10.2  server.lab.local  server
EOF

# Vérifier les entrées ajoutées
grep lab /etc/hosts
```

### 3.3 — Copier la clé publique vers le Managed Node

```bash
# Copier la clé publique de l'utilisateur ansible vers server.lab.local
# (le mot de passe d'ansible sur VM2 sera demandé une dernière fois)
ssh-copy-id -i /home/ansible/.ssh/id_rsa.pub ansible@192.168.10.2

# Résultat attendu :
# Number of key(s) added: 1
# Now try logging into the machine: "ssh ansible@192.168.10.2"
```

### 3.4 — Tester la connexion SSH sans mot de passe

```bash
# Connexion SSH (aucun mot de passe ne doit être demandé)
ssh ansible@server.lab.local

# Une fois connecté sur VM2, vérifier l'identité et les droits sudo
whoami          # → ansible
sudo whoami     # → root

# Revenir sur le control node
exit
```

> [IMAGE_ICI — Capture d'écran : Connexion `ssh ansible@server.lab.local` sans mot de passe et `sudo whoami` retournant `root`]

---

## 📦 Étape 4 — Création de l'Inventaire

> 📍 **Effectuer sur :** `client.lab.local` (VM1) **en tant qu'utilisateur `ansible`**

### 4.1 — Créer le répertoire du projet

```bash
# Créer le répertoire de travail dans le home de l'utilisateur ansible
mkdir -p /home/ansible/ansible-lab && cd /home/ansible/ansible-lab
```

### 4.2 — Créer le fichier d'inventaire

```bash
nano inventory.ini
```

```ini
# inventory.ini — Inventaire Ansible du lab RHCSA
# Utilisateur de connexion dédié : ansible

[webservers]
server.lab.local ansible_host=192.168.10.2

[all:vars]
ansible_user=ansible
ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

### 4.3 — Créer le fichier de configuration Ansible

```bash
nano ansible.cfg
```

```ini
# ansible.cfg — Configuration locale du projet Ansible Lab RHCSA

[defaults]
inventory        = ./inventory.ini
remote_user      = ansible
private_key_file = /home/ansible/.ssh/id_rsa
host_key_checking = False

[privilege_escalation]
become          = True
become_method   = sudo
become_user     = root
become_ask_pass = False   # Pas de mot de passe grâce à la config sudoers
```

### 4.4 — Tester la connectivité avec un ping Ansible

```bash
# Lancer depuis /home/ansible/ansible-lab
ansible all -m ping

# Résultat attendu :
# server.lab.local | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

### 4.5 — Vérifier les facts du Managed Node

```bash
# Récupérer les informations système de distribution
ansible all -m setup -a "filter=ansible_distribution*"
```

> [IMAGE_ICI — Capture d'écran : Résultat du `ansible all -m ping` avec la réponse "pong" sous l'utilisateur `ansible`]

---

## 📜 Étape 5 — Premier Playbook

> 📍 **Effectuer sur :** `client.lab.local` (VM1) **en tant qu'utilisateur `ansible`**

```bash
cd /home/ansible/ansible-lab
```

### 5.1 — Créer le playbook d'installation d'Apache

```bash
nano install_apache.yml
```

```yaml
---
# install_apache.yml — Playbook d'installation du serveur web Apache
# Auteur    : ansible@client.lab.local
# Objectif  : Installer, configurer et démarrer Apache sur les webservers
# Exécution : ansible-playbook install_apache.yml

- name: 🌐 Installation et Configuration du Serveur Web Apache
  hosts: webservers
  remote_user: ansible      # Connexion SSH avec l'utilisateur dédié ansible
  become: true              # Élévation en root via sudo (sans mot de passe)
  become_method: sudo
  become_user: root

  vars:
    http_port: 80
    web_root: /var/www/html
    server_name: "{{ inventory_hostname }}"

  tasks:

    - name: 📦 Installer le package httpd (Apache)
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: 🔥 Ouvrir le port HTTP dans le pare-feu
      ansible.posix.firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true

    - name: 📄 Déployer la page d'accueil personnalisée
      ansible.builtin.copy:
        dest: "{{ web_root }}/index.html"
        content: |
          <!DOCTYPE html>
          <html>
            <head>
              <meta charset="UTF-8">
              <title>Ansible Lab — RHCSA</title>
            </head>
            <body>
              <h1>🚀 Déployé automatiquement par Ansible !</h1>
              <hr>
              <p>👤 Exécuté par l'utilisateur : <strong>ansible</strong></p>
              <p>🖥️ Control Node : <strong>client.lab.local (192.168.10.1)</strong></p>
              <p>🎯 Managed Node : <strong>{{ server_name }} (192.168.10.2)</strong></p>
              <p>📅 Déployé le : {{ ansible_date_time.date }}</p>
            </body>
          </html>
        owner: apache
        group: apache
        mode: '0644'

    - name: ✅ Démarrer et activer le service Apache au démarrage
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true

    - name: 🔍 Vérifier qu'Apache répond sur le port {{ http_port }}
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}"
        status_code: 200
      register: http_result

    - name: 📊 Afficher le résultat de la vérification HTTP
      ansible.builtin.debug:
        msg: "✅ Apache opérationnel sur {{ server_name }} — Code HTTP : {{ http_result.status }}"
```

### 5.2 — Valider la syntaxe avant exécution

```bash
ansible-playbook install_apache.yml --syntax-check
# Résultat attendu :
# playbook: install_apache.yml   ← Aucune erreur
```

### 5.3 — Exécuter le playbook

```bash
# Simulation (dry-run) — aperçu des changements sans les appliquer
ansible-playbook install_apache.yml --check

# Exécution réelle
ansible-playbook install_apache.yml

# Exécution verbeuse (recommandé pour l'apprentissage)
ansible-playbook install_apache.yml -v
```

### 5.4 — Structure finale du projet

```
/home/ansible/ansible-lab/
│
├── 📄 ansible.cfg           # Config : user ansible, sudo sans mdp, clés SSH
├── 📋 inventory.ini         # Inventaire : managed nodes et variables
├── 📜 install_apache.yml    # Playbook : installation et déploiement Apache
│
└── 📁 roles/                # (répertoire pour les labs avancés)
```

> [IMAGE_ICI — Capture d'écran : Sortie complète d'`ansible-playbook install_apache.yml` avec tous les tasks en vert (OK / Changed)]

> [IMAGE_ICI — Capture d'écran : Page web Apache affichée dans un navigateur à l'adresse `http://192.168.10.2`]

---

## 🔄 Récapitulatif du Flux Complet

```
     VM1 & VM2                  VM1 (Control)             VM2 (Managed)
─────────────────────────────────────────────────────────────────────────
[Étape 1]
Créer user ansible    →   useradd ansible           useradd ansible
Configurer sudoers    →   sudoers.d/ansible          sudoers.d/ansible
Tester sudo           →   sudo whoami → root ✅       sudo whoami → root ✅

[Étape 2]
Installer Ansible     →   dnf install ansible        (non requis)

[Étape 3]
Générer clé SSH       →   ssh-keygen (ansible)
Copier clé publique   →   ssh-copy-id ────────────►  ~/.ssh/authorized_keys
Tester connexion      →   ssh ansible@server ──────►  connexion sans mdp ✅

[Étape 4]
Inventaire & Config   →   inventory.ini
                          ansible.cfg (user: ansible)
Test ping             →   ansible all -m ping ──────► pong ✅

[Étape 5]
Exécuter playbook     →   ansible-playbook ──────────► Install httpd
(user: ansible)                                         Ouvre firewall HTTP
(become: root)                                          Deploy index.html
                                                        Start httpd ✅
                                                        Vérif HTTP 200 ✅
```

---

## 👨‍💻 Auteur

<div align="center">

| | |
|---|---|
| 👨‍💻 **Nom** | Serge TOGNON |
| 💼 **LinkedIn** | https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BDxJkd3wUQV2zaekLKp2JnQ%3D%3D |
| 🎯 **Objectif** | Certification RHCSA — Red Hat Certified System Administrator |
| 🛠️ **Stack** | Ansible · RHEL · Linux · SSH · Bash · YAML |

</div>

---

## 🎓 Conclusion RHCSA

Ce laboratoire couvre des compétences **directement alignées sur les objectifs de l'examen RHCSA (EX200)** de Red Hat :

| ✅ Compétence RHCSA | 🔧 Couverte dans ce Lab | 📍 Étape |
|---|---|---|
| Création et gestion d'utilisateurs | `useradd`, `passwd`, `id` | Étape 1 |
| Gestion des privilèges (`sudo`) | `sudoers.d`, `NOPASSWD` | Étape 1 |
| Gestion des paquets avec `dnf` | Installation d'Ansible & Apache | Étapes 2 & 5 |
| Configuration SSH par clé | `ssh-keygen`, `ssh-copy-id` | Étape 3 |
| Automatisation avec des outils | Ansible (inventaire, playbook) | Étapes 4 & 5 |
| Gestion des services (`systemctl`) | Module `service` — Apache | Étape 5 |
| Configuration du pare-feu | Module `firewalld` — port HTTP | Étape 5 |
| Création et gestion de fichiers | Module `copy` — index.html | Étape 5 |



---

<div align="center">

**⭐ Si ce lab vous a été utile, n'hésitez pas à mettre une étoile sur le repo !**


</div>
