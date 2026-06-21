# OWASP Top 10 (2025) — mes notes

## Ce que j'ai retenu

Le Top 10 OWASP n'est pas une liste figée de "bugs" — c'est une liste de **catégories de défauts de conception/implémentation** qui reviennent sans cesse dans les applications web. Comprendre la logique derrière chaque catégorie compte plus que mémoriser les noms.

## Les vulnérabilités que j'ai pratiquées concrètement

### IDOR (Insecure Direct Object Reference)
Le principe : une appli expose un identifiant (ex. `/api/user/482`) et fait confiance à cet ID sans vérifier que l'utilisateur connecté a bien le droit d'accéder à *cette* ressource précise. En changeant juste l'ID dans l'URL ou la requête, on peut accéder aux données d'un autre utilisateur.

Outils utilisés : Burp Suite (interception et modification de requêtes), `curl` pour rejouer des requêtes modifiées rapidement en CLI.

### Divulgation d'informations via stack trace (Flask)
Une appli Flask en mode debug peut afficher la stack trace complète en cas d'erreur — ça révèle la structure du code, parfois des chemins de fichiers, des variables d'environnement. J'ai appris à repérer ce genre de fuite simplement en provoquant des erreurs (input invalide, paramètre manquant) et en lisant la réponse.

### Insecure Design
Ce qui m'a marqué : ce n'est pas un "bug" au sens classique, c'est une faille dans la conception même du système (ex. logique métier qui ne vérifie pas un état avant une action). Pas de patch simple — il faut repenser le flux.

## Un point où j'ai buté

Au début j'avais tendance à chercher des vulnérabilités "techniques" complexes alors que beaucoup de failles réelles viennent de logique métier mal pensée (ex. pas de vérification d'autorisation). Le réflexe à avoir : toujours se demander *"qu'est-ce que cette appli suppose être vrai, et est-ce que je peux casser cette supposition ?"*

## Outils clés associés

- Burp Suite + FoxyProxy
- CyberChef (déchiffrement AES-ECB rencontré dans une room)
- `curl` pour tester rapidement sans interface graphique
