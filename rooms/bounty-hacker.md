# 🤠 Bounty Hacker — TryHackMe

**Catégorie :** CTF / Services réseau (FTP, SSH)
**Difficulté annoncée :** Facile
**Compétences mobilisées :** Énumération de services (nmap), FTP, OSINT léger (contenu web), brute-force SSH (Hydra), privilege escalation (abus sudo sur `tar`)

---

## 🎯 Objectif

Obtenir un accès root sur la machine en suivant les indices laissés par l'équipage du Bebop (thème Cowboy Bebop).

---

## 🔍 Reconnaissance

Scan initial avec `nmap -sV --script vuln <ip>` → 3 ports ouverts :
- **21** (FTP)
- **22** (SSH)
- **80** (HTTP)

**Fausse piste initiale :** recherche de vulnérabilités directement via `msfconsole` (search + exploits connus) sur les services détectés — aucun résultat exploitable. Le scan `--script vuln` n'a rien remonté de directement utilisable non plus.

---

## 📂 Énumération FTP

- Connexion en **anonyme** sur le service FTP (port 21), sans credentials
- Récupération de deux fichiers :
  - Une **wordlist** de mots de passe (variantes autour de "Red Dragon Syndicate")
  - Une **note** contenant deux tâches et signée :
    ```
    1.) Protect Vicious.
    2.) Plan for Red Eye pickup on the moon.
    -lin
    ```

**Point clé :** la signature "-lin" en bas de la note a d'abord été mal interprétée comme faisant partie d'un flag de commande (`-lin` au lieu de `lin`). En réalité, le tiret est juste une signature, le username est **`lin`**.

---

## 🌐 Énumération Web (port 80)

Page contenant un dialogue entre les personnages de l'équipage (Spike, Jet, Ed, Faye), évoqué comme du texte d'ambiance à première vue.

**Erreur de parcours :** la page web a été jugée "vide d'intérêt" lors d'une première lecture rapide. En réalité, le dialogue contenait des informations utiles (les noms des personnages, pistes de username) qu'il fallait croiser avec les fichiers FTP plutôt que de les lire isolément.

---

## 🔓 Accès SSH — Brute-force avec Hydra

- Username confirmé : `lin` (signature de la note FTP)
- Wordlist : fichier récupéré sur le FTP

```
hydra -l lin -P wordlist.txt ssh://<ip>
```

➡️ Connexion SSH réussie en tant que `lin`.

---

## 🔼 Privilege Escalation — Abus de sudo sur `tar`

Vérification des droits sudo :

```
sudo -l
```

Résultat :
```
User lin may run the following commands on ip-10-128-137-224:
    (root) /bin/tar
```

→ `lin` peut exécuter **uniquement** `/bin/tar` en root (pas de `NOPASSWD: ALL` cette fois, contrairement à Pickle Rick — l'autorisation est limitée à ce seul binaire).

**Technique utilisée :** abus du mécanisme de **checkpoint** de `tar`, normalement prévu pour exécuter une action personnalisée à intervalles réguliers pendant un archivage (ex: affichage de progression). En détournant l'action de checkpoint pour lancer un shell, on obtient une exécution de commande arbitraire avec les privilèges root accordés à `tar` par sudo.

```
sudo tar -cf essai.tar <fichier_existant> --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Le fichier passé en argument n'a pas besoin d'avoir un contenu pertinent : `tar` a juste besoin d'un argument fichier valide pour démarrer son exécution, et le checkpoint se déclenche avant la fin réelle de l'archivage.

➡️ Shell root obtenu.

---

## 🔄 Erreurs de parcours

| Avant | Correction | Pourquoi |
|---|---|---|
| Chercher des exploits via Metasploit directement après le scan nmap, sans énumérer manuellement chaque service | Tester d'abord un accès simple (FTP anonyme) avant de chercher des vulnérabilités complexes | Sur une room "facile", l'accès initial passe souvent par une mauvaise configuration basique (service anonyme, credentials faibles) plutôt que par un exploit technique répertorié dans Metasploit |
| Lire le dialogue de la page web comme un simple texte d'ambiance sans intérêt technique | Relire le contenu en cherchant des noms propres ou indices à croiser avec d'autres sources (ex : fichiers FTP) | Le contenu narratif d'une room CTF contient quasi systématiquement des indices fonctionnels (ici : noms de personnages comme pistes de username) ; rien n'est purement décoratif |
| Interpréter la signature "-lin" d'une note comme un flag de commande (`-lin`) plutôt que comme un simple nom signé | Identifier le tiret comme une signature ("— lin"), et utiliser uniquement `lin` comme username | Le contexte (note signée à la fin d'un message) doit primer sur la ressemblance visuelle avec une syntaxe de commande |
| Penser que `sudo` autorisé sur `tar` permet d'exécuter n'importe quelle autre commande en root (ex: tenter `sudo ls`) | Comprendre que `sudo -l` limite précisément les binaires autorisés — ici seulement `/bin/tar`, donc toute élévation doit passer **à travers** ce binaire précis | `sudo` n'accorde des privilèges que pour les commandes explicitement listées dans `sudo -l` ; un binaire autorisé doit être détourné (GTFOBins) pour obtenir une exécution de commande arbitraire si on n'a pas un accès root direct |

---

## 📝 Conclusion

Room complétée en solo. Contrairement à Pickle Rick (NOPASSWD: ALL sur tous les binaires), cette room oblige à exploiter un binaire **spécifique** (`tar`) de manière détournée pour obtenir une élévation de privilèges — une logique plus proche de ce qui sera rencontré sur des rooms/CTF de niveau supérieur, où l'accès sudo est rarement aussi permissif que sur Pickle Rick.

**Compétences à retenir pour la suite du parcours (Jr Pentester) :**
- Toujours tester l'accès **anonyme** sur FTP avant toute autre piste
- Croiser systématiquement les indices trouvés sur différents services (web, FTP) plutôt que de les traiter isolément
- Quand `sudo -l` limite l'accès à un binaire précis, chercher comment ce binaire spécifique peut être détourné pour obtenir une exécution de commande arbitraire (réflexe GTFOBins)
- `tar --checkpoint-action=exec=` est un vecteur classique d'abus sudo à connaître
