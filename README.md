---
title: TrueNAS SCALE - Guide Complet
aliases:
  - TrueNAS SCALE
  - NAS Configuration
tags:
  - truenas
  - scale
  - zfs
  - raid
  - sftp
  - ssh
  - vaultwarden
  - docker
  - debian
  - réseau
  - stockage
  - virtualisation
  - homelab
  - sysadmin
date: 2026-03-31
status: actif
type: guide
projet: HexaLab
---

# 🖥️ TrueNAS SCALE — Guide Complet d'Installation et Configuration

> [!abstract] Résumé rapide
> Ce guide couvre l'installation complète de TrueNAS SCALE, la configuration d'un pool RAIDZ2 (RAID 6 ZFS), les services réseau (SSH, HTTPS, SFTP), le déploiement de Vaultwarden via Docker, la création d'une VM Debian et le dépannage des problèmes courants.

---

## 📋 Table des matières

- [[#Configuration de TrueNAS SCALE]]
- [[#Configuration du RAID 6 (RAIDZ2)]]
- [[#Configuration des services réseau]]
  - [[#SSH]]
  - [[#HTTPS]]
  - [[#SFTP]]
- [[#Installation de Vaultwarden via Docker]]
- [[#Création d'une VM Debian sur TrueNAS SCALE]]
- [[#Dépannage et problèmes courants]]

---

## ⚙️ Configuration de TrueNAS SCALE

### 1. Premier accès à l'interface web

- Depuis la VM Debian, ouvrez un navigateur web
- Accédez à `https://[IP-TRUENAS]`
- Acceptez l'avertissement de sécurité lié au certificat auto-signé
- Connectez-vous avec les identifiants créés lors de l'installation :

| Champ | Valeur |
|---|---|
| Utilisateur | `root` |
| Mot de passe | _(défini à l'installation)_ |

### 2. Mise à jour du système

1. Allez dans **System > Update**
2. Cliquez sur **Check for Updates**
3. Si des mises à jour sont disponibles → cliquez sur **Update**
4. Redémarrez via **Reboot Now** quand demandé
5. Reconnectez-vous à l'interface après le redémarrage

> [!tip] Bonne pratique
> Toujours mettre à jour TrueNAS avant toute configuration avancée pour éviter les bugs connus.

---

## 💾 Configuration du RAID 6 (RAIDZ2)

> [!info] RAIDZ2 = RAID 6 en ZFS
> RAIDZ2 tolère la panne simultanée de **2 disques**. Avec 5 disques de 2 To, la capacité utile est environ **6 To** (3 disques de données + 2 de parité).

### 1. Création du pool de stockage

1. Allez dans **Storage > Pools**
2. Cliquez sur **Add**
3. Nommez le pool `Stockage`
4. Sélectionnez les **5 disques de 2 To** disponibles
5. Choisissez la configuration **RAIDZ2**
6. Vérifiez la capacité ≈ **6 To**
7. Cliquez sur **Create**

### 2. Vérification du pool

- [ ] Le pool apparaît dans la liste des pools
- [ ] Son état est **ONLINE**
- [ ] L'espace disponible est ≈ **6 To**
- [ ] Créer des datasets si nécessaire pour organiser le stockage

```dataview
table status, capacity
from #stockage
where type = "pool"
```

---

## 🌐 Configuration des services réseau

### SSH

#### Configuration du service SSH

1. Allez dans **Services**
2. Trouvez **SSH** dans la liste
3. Cliquez sur l'icône ⚙️ pour configurer
4. Activez **Enable SSH**
5. Changez le port `22` → port personnalisé (ex : `2222`)
6. Activez l'authentification par mot de passe si nécessaire
7. Activez **Allow root login with password** _(temporairement pour les tests)_
8. Sauvegardez et démarrez le service

> [!warning] Sécurité SSH
> Désactivez `Allow root login with password` en production. Préférez l'authentification par clé SSH.

#### Test de la connexion SSH

```bash
ssh -p 2222 root@[IP-TRUENAS]
```

- Acceptez l'empreinte du serveur SSH
- Entrez le mot de passe administrateur
- Vérifiez que vous êtes connecté à la console TrueNAS

---

### HTTPS

Le service HTTPS est **activé par défaut** pour l'interface d'administration web.

#### Configuration HTTPS avancée

1. Allez dans **System > General**
2. Onglet **GUI**, vérifiez les paramètres :

| Paramètre | Valeur |
|---|---|
| Adresse d'écoute | `0.0.0.0` (toutes les interfaces) |
| Port | `443` (par défaut) |
| Protocole | `HTTPS` |

3. Pour un certificat personnalisé → **System > Certificates**
4. Créez ou importez un certificat et assignez-le à l'interface web

---

### SFTP

#### 1. Création des utilisateurs

1. Allez dans **Credentials > Local Users**
2. Cliquez sur **Add** pour chaque utilisateur :

| Champ | Valeur |
|---|---|
| Nom d'utilisateur | _(à définir)_ |
| Mot de passe | _(sécurisé)_ |
| Répertoire personnel | `/mnt/Stockage/homes/username` |
| Shell | `/usr/sbin/nologin` _(SFTP uniquement)_ |

#### 2. Création des dossiers utilisateurs

1. **Storage > Pools > Stockage**
2. Créez un dataset `homes` pour les répertoires personnels
3. Pour chaque utilisateur → créez un sous-dataset avec son nom
4. Configurez les permissions :
   - ⚙️ > **Edit Permissions**
   - Définissez le propriétaire = utilisateur correspondant
   - Appliquez récursivement

#### 3. Création d'un dossier public

1. **Storage > Pools > Stockage** → créez un dataset `public`
2. Permissions `775` (`rwxrwxr-x`)
3. Groupe : `users` (ou groupe personnalisé)

#### 4. Configuration du service FTP

1. **Services > FTP** → icône ⚙️
2. Activez **Enable FTP**
3. Changez le port `21` → `2121`
4. Options à configurer :

| Option | Recommandation |
|---|---|
| Connexions maximum | Selon besoin |
| Connexions par IP | Limiter pour la sécurité |
| Timeout | 300s recommandé |
| Root login | ❌ Désactivé |

5. Sauvegardez et démarrez le service

#### 5. Test de la connexion SFTP (FileZilla)

```bash
apt update && apt install filezilla
```

Configuration FileZilla :

| Champ | Valeur |
|---|---|
| Hôte | `sftp://[IP-TRUENAS]` |
| Port | `2121` |
| Protocole | SFTP - SSH File Transfer Protocol |
| Type d'auth | Normale |
| Utilisateur | _(utilisateur créé)_ |
| Mot de passe | _(mot de passe de l'utilisateur)_ |

- [ ] Naviguer dans le répertoire personnel
- [ ] Accéder au dossier public
- [ ] Uploader et télécharger des fichiers

---

## 🔐 Installation de Vaultwarden via Docker

> [!abstract] Qu'est-ce que Vaultwarden ?
> Vaultwarden est une implémentation alternative open source du serveur **Bitwarden** (gestionnaire de mots de passe). Léger, auto-hébergeable, compatible avec tous les clients Bitwarden officiels.

### 1. Préparation

1. Allez dans **Apps**
2. Première utilisation → suivez l'assistant :
   - Sélectionnez le pool `Stockage` pour les applications
   - Attendez que le système configure **Kubernetes**

### 2. Installation de Vaultwarden

1. Dans **Discover Apps**, recherchez `Vaultwarden`
2. Si absent, ajoutez le catalogue **TrueCharts** :

| Champ | Valeur |
|---|---|
| Nom | `TrueCharts` |
| Repository | `https://github.com/truecharts/catalog` |
| Preferred trains | `stable` |

3. Relancez la recherche → cliquez sur **Install**

### 3. Configuration de Vaultwarden

| Section | Paramètre | Valeur |
|---|---|---|
| Général | Application Name | `Vaultwarden` |
| Général | Version | Dernière stable |
| Réseau | Service Type | `ClusterIP` |
| Réseau | Port d'écoute | `80` |
| Stockage | Chemin de montage | `/data` |

**Variables d'environnement :**

```env
WEBSOCKET_ENABLED=true
SIGNUPS_ALLOWED=false
DOMAIN=https://[IP-TRUENAS]:[PORT]
```

> [!warning] Sécurité Vaultwarden
> Désactivez les inscriptions publiques (`SIGNUPS_ALLOWED=false`) dès le départ. En production, utilisez un reverse proxy HTTPS devant Vaultwarden.

### 4. Vérification de l'installation

- [ ] Statut de l'application = **Running**
- [ ] Accéder à `http://[IP-TRUENAS]:[PORT]`
- [ ] La page de connexion Vaultwarden s'affiche
- [ ] Créer un compte administrateur à la première connexion

### 5. Configuration supplémentaire (optionnelle)

- Configurez `DOMAIN` pour la synchronisation mobile
- Ajoutez un **reverse proxy** avec HTTPS pour la production
- Configurez des **sauvegardes régulières** des données Vaultwarden

---

## 🖥️ Création d'une VM Debian sur TrueNAS SCALE

### 1. Préparation

- Téléchargez l'ISO de Debian depuis le [site officiel](https://www.debian.org/distrib/)
- Vérifiez que la **virtualisation est activée** dans le BIOS/UEFI
- Dans TrueNAS → **System > Boot Environments** (vérification)

### 2. Téléchargement de l'ISO

1. **Virtual Machines > Resources > ISO Files**
2. Cliquez sur **Upload**
3. Sélectionnez l'ISO Debian ou fournissez une URL
4. Attendez la fin du téléchargement

### 3. Création de la VM

**Configuration de base :**

| Paramètre | Valeur |
|---|---|
| Nom | `Debian-VM` |
| Description | VM Debian pour le projet |
| Système d'exploitation | Linux |
| Boot Method | UEFI |

**Configuration CPU/Mémoire :**

| Paramètre | Valeur |
|---|---|
| Cœurs CPU | 2 |
| Threads | 2 |
| Mémoire | 2048 MB (2 GB) |

**Configuration disque :**

| Paramètre | Valeur |
|---|---|
| Taille | 20 GB |
| Type | ZVOL |
| Mode | VIRTIO |

**Configuration réseau :**
- Type : Bridge ou NAT selon votre configuration

### 4. Installation de Debian sur la VM

1. Démarrez la VM → **Start**
2. Ouvrez la console → **VNC**
3. Suivez l'installateur Debian :
   - [ ] Langue, pays, clavier
   - [ ] Configuration réseau
   - [ ] Nom d'hôte et domaine
   - [ ] Créer l'utilisateur `LaPlateforme` — mot de passe : `LaPlateforme13`
   - [ ] Partitionnement (tout le disque)
   - [ ] Environnement de bureau au choix (GNOME, KDE, XFCE…)
   - [ ] Terminer et redémarrer

### 5. Vérification post-installation

- [ ] Connexion avec l'utilisateur `LaPlateforme`
- [ ] Environnement graphique fonctionnel
- [ ] Connectivité réseau depuis la VM

---

## 🔧 Dépannage et problèmes courants

### Tableau récapitulatif

| Problème | Symptôme | Solutions |
|---|---|---|
| **Interface web** | Impossible d'accéder à TrueNAS | Vérifier IP, essayer HTTP, vérifier pare-feu |
| **Pool de stockage** | Statut DEGRADED / erreurs ZFS | Vérifier disques, lancer un scrub, consulter les logs |
| **Services réseau** | SSH/SFTP inaccessible | Vérifier service démarré, port correct, permissions dossiers |
| **Applications Docker** | Vaultwarden ne démarre pas | Vérifier pod dans Apps, consulter logs, vérifier réseau |
| **Machines virtuelles** | VM ne démarre pas / lente | Vérifier ressources allouées, virtualisation BIOS, logs VM |

### Connexion à l'interface web

> [!bug] Symptôme : Impossible d'accéder à l'interface web
> - Vérifiez que TrueNAS est bien démarré
> - Vérifiez que l'adresse IP est correcte
> - Essayez `http://` au lieu de `https://`
> - Vérifiez les pare-feu sur votre réseau

### Pool de stockage

> [!bug] Symptôme : Erreurs dans le pool ZFS ou statut DEGRADED
> - **Storage > Disks** → vérifier l'état des disques
> - **Storage > Pools** → lancer un scrub du pool
> - Consulter les logs système pour les erreurs de disque

### Services réseau

> [!bug] Symptôme : Impossible de se connecter via SSH ou SFTP
> - Vérifiez que le service est **démarré**
> - Vérifiez que vous utilisez le **bon port**
> - Consultez les **logs du service**
> - Vérifiez les **permissions** des dossiers utilisateurs

### Applications Docker

> [!bug] Symptôme : Vaultwarden ne démarre pas ou n'est pas accessible
> - **Apps > Installed** → vérifier l'état du pod
> - Consulter les **logs de l'application**
> - Vérifier la **configuration réseau**

### Machines virtuelles

> [!bug] Symptôme : La VM ne démarre pas ou est lente
> - Vérifiez les **ressources allouées** à la VM
> - Vérifiez que la **virtualisation est activée** dans le BIOS
> - Consultez les **logs de la VM**

---

> [!success] Récapitulatif des services configurés
> | Service | Port | Statut |
> |---|---|---|
> | Interface Web HTTPS | 443 | ✅ Actif par défaut |
> | SSH | 2222 | ✅ À configurer |
> | FTP/SFTP | 2121 | ✅ À configurer |
> | Vaultwarden | Variable | ✅ Via Docker/K8s |

---

*Guide réalisé dans le cadre du projet HexaLab — Formation Administration Système & Réseau*
