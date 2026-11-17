# TP Architecture réseau — Campus NovaCorp

Ce projet est réalisé dans le cadre du TP d'architecture réseau. Le groupe joue le rôle d'un bureau d'études qui conçoit le réseau du siège de NovaCorp :
- deux étages de bureaux ;
- une salle serveur ;
- un réseau prêt à accueillir les sites distants.

## Équipe

| Nom | Prénom |
|---|---|
| BOUCHOUAREB | Eddy |
| GIORDANO | Yoann |
| GUILBAUD | Maxime |

## Contenu du dossier

| Fichier | Description |
|---|---|
| `NovaCorp-Siege.pkt` | Maquette Cisco Packet Tracer 9.0.1, version finale v3 |
| `INFRA-NovaCorp.md` | Description détaillée de l'infrastructure (plan et maquette) |
| `CHANGELOG.md` | Historique du travail sur les trois séances |
| `NovaCorp-Dossier-Architecture-Reseau-S1.pdf` | Dossier complet du livrable S1 |
| `1-Schema-logique.pdf` à `5-Convention-nommage.pdf` | Les 5 livrables S1, un fichier par livrable |
| `NovaCorp-Schema-Simple.drawio` | Schéma logique modifiable dans draw.io |

## En bref

- **Siège** :
  - 2 cœurs de niveau 3 (`SG-CORE-DC-01/02`) qui portent les passerelles HSRP ;
  - 2 switches d'accès par étage et 2 switches pour la salle serveur.
- **Redondance** :
  - chaque switch est relié aux deux cœurs par des liens agrégés en LACP ;
  - Rapid PVST+ gère les liens de secours ;
  - les serveurs ont deux cartes réseau, une sur chaque switch serveur.
- **VLANs** : data (10, 20), voix (110, 120), serveurs (30, 31), management (90), transit (98), natif (99).
- **Adressage** : `10.<site>.<vlan>.0/24` en IPv4, `2001:db8:<site>:<vlan>::/64` en IPv6.
- **Routage** :
  - OSPF multi-aires : une dorsale en losange en area 0, le bâtiment B en area 1, le bâtiment C en area 2 *stub* ;
  - BGP vers un opérateur simulé et les sites distants 2 et 3.

Les détails et les justifications sont dans [`INFRA-NovaCorp.md`](INFRA-NovaCorp.md).
