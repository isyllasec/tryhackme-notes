# Networking Fundamentals — mes notes

## Ce que j'ai retenu

Le modèle qui m'a vraiment aidé à comprendre comment les données circulent, c'est de penser en couches : chaque couche ne se soucie que de son propre travail et fait confiance à celle d'en dessous.

Ce qui m'a clarifié les choses : faire le lien entre le **modèle OSI (7 couches, théorique)** et le **modèle TCP/IP (4 couches, ce qui tourne vraiment sur Internet)**. En pratique, sur le terrain, c'est surtout TCP/IP qu'on utilise pour raisonner.

## Commandes que j'utilise réellement

```bash
# Voir ma config réseau (Linux)
ip a

# Tester la connectivité
ping -c 4 <IP>

# Voir la table de routage
ip route

# Capturer le trafic (intro à Wireshark en CLI)
tcpdump -i eth0
```

## Un point où j'ai buté

Au début, je confondais **TCP** et **UDP** en pensant que TCP était "juste plus lent". En creusant, j'ai compris que la vraie différence c'est la fiabilité : TCP fait du contrôle de livraison (accusés de réception, retransmission), UDP n'en fait pas — donc UDP est utilisé quand la vitesse compte plus que la perte occasionnelle de paquets (streaming, DNS, jeux en ligne).

## Lien avec la suite

Ces bases réseau servent directement en pentest pour comprendre ce que montre un scan Nmap (ports, services, protocoles) plutôt que de juste lancer la commande sans savoir ce qu'elle fait.
