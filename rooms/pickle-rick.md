# 🥒 Pickle Rick — TryHackMe

**Catégorie :** CTF / Web Exploitation
**Difficulté annoncée :** Facile
**Compétences mobilisées :** Web enum, OSINT léger (page source), encodage, command injection filter bypass, navigation filesystem Linux, privilege escalation (sudo)

---

## 🎯 Objectif

Trouver les 3 ingrédients cachés sur la machine pour aider Rick à préparer sa potion et reprendre forme humaine.

---

## 🔍 Reconnaissance

- Inspection de la page d'accueil via **view-source** → découverte d'un **username**
- Scan de répertoires avec **Gobuster** → découverte de `robots.txt` contenant un mot, qui s'est révélé être le **password**
- Tentatives infructueuses avant la bonne piste : connexion SSH directe, interception Burp Suite (rien d'exploitable côté requête), recherche de scripts JS (sans résultat exploitable)

**Erreur de méthode initiale :** Gobuster lancé sans préciser d'extensions, ce qui a empêché de trouver la page de login du premier coup. Il a fallu relancer avec une extension type `.php` pour la repérer.

---

## 🔑 Accès initial

- Connexion sur la page de login trouvée avec **username (view-source) + password (robots.txt)**
- Accès à un **command panel** web restreint

---

## 🧩 Ingrédient 1 — Contournement du filtre de commande

Le panel bloquait le mot `cat`. Contournement réalisé avec la commande **`tac`** (cat à l'envers), qui n'était pas filtrée.

➡️ Premier ingrédient trouvé directement dans le répertoire courant (`/var/www/html`).

---

## 🧩 Ingrédient 2 — Recherche filesystem + fichier avec espace dans le nom

- Indice trouvé : *"Look around the file system for the other ingredient."*
- `pwd` confirmait être dans `/var/www/html`
- **Piège rencontré :** `cd ..` semblait fonctionner, mais la commande suivante repartait toujours de `/var/www/html`. Le panel exécute chaque commande indépendamment, sans conserver de session shell — donc `cd` seul ne "colle" pas entre deux commandes séparées.
- **Solution :** utiliser des chemins **absolus** directement dans la commande, ou chaîner avec `;` (ex : `cd /home/rick ; tac "second ingredients"`)
- Exploration de `/home` → découverte de l'utilisateur **rick**
- Fichier trouvé : `second ingredients` (nom contenant un **espace**)
- **Piège :** `ls "/home/rick/second ingredients"` ne renvoyait rien d'utile alors que le fichier existait bel et bien (confirmé via `ls -la /home/rick`)
- Lecture réussie avec : `cd /home/rick ; tac "second ingredients"` (toujours en contournant le filtre `cat`)

**Point à éclaircir (comportement noté mais non expliqué) :** `tac "/home/rick/second ingredients"` en chemin absolu ne fonctionnait pas, alors que `cd /home/rick ; tac "second ingredients"` fonctionnait. Comportement à creuser plus tard sur la gestion des guillemets/espaces par le panel selon que le chemin soit absolu ou relatif.

➡️ Deuxième ingrédient trouvé.

---

## 🧩 Ingrédient 3 — Privilege escalation via sudo

- Vérification des privilèges de l'utilisateur courant (`www-data`) avec `sudo -l` :

```
User www-data may run the following commands on ip-10-128-183-10:
    (ALL) NOPASSWD: ALL
```

→ `www-data` peut exécuter **n'importe quelle commande en tant que n'importe quel utilisateur, sans mot de passe**, donc accès root possible.

- `sudo ls /root` → fichier `3rd.txt` repéré
- **Piège :** `sudo cd /root ; tac 3rd.txt` ne fonctionnait pas, même en mettant `sudo` devant `tac` également
- **Cause identifiée :** `cd` n'est pas un binaire exécutable mais une commande interne (builtin) du shell — `sudo cd` n'a donc pas de sens, sudo ne sait pas l'exécuter en tant que processus séparé
- **Solution :** `sudo tac /root/3rd.txt` directement avec le chemin absolu, sans passer par `cd`

➡️ Troisième et dernier ingrédient trouvé. Room terminée.

---

## 🔄 Erreurs de parcours

| Avant | Correction | Pourquoi |
|---|---|---|
| Gobuster lancé sans extension précisée | Relancer en spécifiant une extension type `.php` | Une page de login dynamique a généralement une extension de script ; sans elle, Gobuster ne teste que des noms de dossiers/fichiers statiques |
| Tentative de décoder une chaîne suspecte trouvée dans la page source et de l'exécuter comme commande dans le panel | Identifier qu'il s'agissait d'une donnée encodée à décoder (CyberChef), pas d'une commande — puis abandonner cette piste (rabbit hole) car non pertinente pour la suite de la room | Toute chaîne qui ressemble à un encodage (présence de `==`, alphabet restreint) doit être décodée à part, séparément de l'action à mener dans le panel ; ne pas confondre "donnée à interpréter" et "commande à exécuter" |
| Penser que `cd ..` ou un `cd` isolé change durablement le répertoire de travail dans le panel | Utiliser des chemins absolus, ou chaîner `cd` et la commande suivante avec `;` dans la même requête | Le panel web exécute chaque commande envoyée comme un processus indépendant, sans état de session persistant — donc un `cd` seul ne survit pas à la commande suivante |
| Mettre `sudo` devant `cd` pour accéder à `/root` puis lire un fichier | Utiliser directement `sudo tac /chemin/absolu/fichier`, sans passer par `cd` | `cd` est une commande interne du shell (builtin), pas un programme externe — `sudo` ne peut élever les privilèges que pour un binaire exécutable, pas pour une fonction interne du shell |

---

## 📝 Conclusion

Room complétée en solo, avec une bonne intuition générale sur la démarche (recoupement de sources, recherche systématique de contournement face aux filtres, réflexe `sudo -l` en fin de course). Les blocages rencontrés étaient surtout liés à des détails techniques de syntaxe shell (gestion des espaces dans les noms de fichiers, distinction builtin/binaire, absence de session persistante dans le panel) plutôt qu'à la logique d'attaque elle-même.

**Compétences à retenir pour la suite du parcours (Jr Pentester) :**
- Toujours vérifier `sudo -l` après un accès initial, même limité
- Connaître plusieurs alternatives à `cat` (`tac`, `more`, `less`, `head`, `tail`) pour contourner des filtres basiques de command injection
- Distinguer commandes internes du shell (builtins comme `cd`) et binaires externes lors de l'utilisation de `sudo`
