# 🔓Crack the Hash

**Lien :** https://tryhackme.com/room/crackthehash
**Catégorie :** Password Cracking
**Date :** Juin 2026

## 🎯 Objectif

Identifier différents types de hashs et les craquer pour retrouver les mots de passe en clair. La room est divisée en deux parties : une série de hashs "faciles" (types courants), puis une série de hashs "difficiles" (formats moins évidents, salt inclus).

## 🛠️ Outils utilisés

- `hash-id.py` — identification rapide du type de hash (utilisé pour les premiers hashs)
- `name-the-hash` — outil plus puissant, utilisé pour la deuxième partie (hashs plus longs/ambigus)
- `hashcat` — cassage des hashs
- `awk` — filtrage de la wordlist par longueur de mot de passe
- `rockyou.txt` — wordlist de référence

## 📝 Étapes

### Partie 1 — Hashs faciles

Identification du type de hash avec `hash-id.py` pour la majorité des hashs : fonctionnel directement sur les premiers cas.

**Hash difficile à identifier (préfixe `$2$`)**

`hash-id.py` ne donnait pas de résultat exploitable pour ce hash précis. Recherche manuelle du préfixe `$2$` → correspond à **bcrypt**.

**Cassage bloqué pendant ~1h**

Plusieurs tentatives infructueuses :
- `hashcat` sans optimisation de wordlist
- CyberChef
- Plusieurs services de cracking en ligne

Aucun résultat. Observation clé : TryHackMe affiche le format de la réponse avec des étoiles à la place des caractères → cela révèle la **longueur exacte** du mot de passe attendu (ici : 4 caractères).

**Réduction de l'espace de recherche**

Construction d'une wordlist filtrée à partir de `rockyou.txt`, ne gardant que les mots de passe de 4 caractères :

```bash
awk 'length($0) == 4' rockyou.txt > rockyou_4.txt
```

Relance de `hashcat` avec `rockyou_4.txt` (espace de recherche bien plus petit) → cassage réussi.

### Partie 2 — Hashs difficiles

Hashs plus longs et plus ambigus dès le départ. Passage à `name-the-hash`, plus fiable que `hash-id.py` sur ce lot, utilisé pour identifier le type de chaque hash avant cassage avec `hashcat`.

**Dernière question — hash salé**

L'énoncé fournissait séparément le hash et le salt. Format attendu par `hashcat` pour ce type de hash salé :

```
<hash>:<salt>
```

Une fois le fichier construit dans ce format, `hashcat` a permis de retrouver le mot de passe.

## 🔄 Erreurs de parcours

| Avant | Correction | Pourquoi ça aide |
|---|---|---|
| Bruteforce direct sur `rockyou.txt` complet pour un mot de passe court | Filtrer la wordlist par longueur connue (`awk 'length($0) == N'`) avant de lancer `hashcat` | Réduit drastiquement l'espace de recherche quand on connaît une contrainte (ici la longueur via les étoiles de THM) — gain de temps énorme sur un cassage qui semblait bloqué |
| Utiliser uniquement `hash-id.py` pour identifier tous les hashs | Passer à `name-the-hash` sur les hashs plus longs/ambigus | Certains formats (ex. bcrypt `$2$`) ne sont pas toujours bien détectés par les outils basiques ; un outil d'identification plus robuste évite les recherches manuelles à chaque fois |
| Tenter de craquer un hash salé en ne fournissant que le hash | Construire le fichier au format `<hash>:<salt>` avant de lancer `hashcat` | `hashcat` a besoin du salt explicitement associé au hash pour les algorithmes salés — sans ça, le cassage échoue silencieusement ou ne trouve jamais le bon résultat |

## 💡 Points clés à retenir

- Les étoiles affichées par TryHackMe à la place de la réponse ne sont pas juste cosmétiques : elles donnent la longueur exacte du mot de passe, une info exploitable pour optimiser un bruteforce.
- Reconnaître un format de hash par son préfixe (`$2$`, `$2a$`, `$6$`, etc.) permet de gagner du temps quand les outils automatiques échouent.
- Pour les hashs salés, toujours vérifier le format d'entrée attendu par l'outil de cracking (`hash:salt` est un format très courant avec `hashcat`).
