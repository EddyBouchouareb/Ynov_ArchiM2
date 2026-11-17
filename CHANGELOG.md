# Changelog — TP architecture réseau « Campus NovaCorp »

> Groupe : loka · Bureau d'études réseau · Livrable S1
> Historique du travail réalisé sur les trois séances.

---

## Séance 1 — Analyse du besoin et conception de l'architecture

- Lecture et analyse du cahier des charges : deux étages de bureaux avec postes + téléphonie IP, une salle serveur isolée, un management dédié, redondance obligatoire (aucune panne d'un seul équipement ou lien ne doit couper le réseau), croissance à prévoir, plan dual-stack IPv4/IPv6 dès la conception.
- Choix d'une architecture hiérarchique à **deux niveaux (« collapsed core »)** : 2 cœurs L3 redondants (routage inter-VLAN, passerelles) + un niveau d'accès L2. Un troisième niveau de distribution a été écarté : inutile à l'échelle d'un bâtiment de deux étages.
- Dimensionnement : 2 switches d'accès 48 ports PoE par étage, 2 switches dédiés en salle serveur, chaque switch d'accès raccordé aux **deux** cœurs (soit 8 commutateurs et 14 liens).
- Définition du **plan de VLANs** : 10/20 (data étages), 110/120 (voix étages, règle voix = 100 + data), 30 (serveurs), 90 (management), 99 (natif des trunks, sans IP) ; VLAN 1 jamais utilisé.
- Définition du **plan IPv4** : `10.<site>.<vlan>.0/24`, passerelle en `.254`, `.252/.253` réservées aux cœurs (future redondance de passerelle), hôtes en `.1–.250` ; `10.2.0.0/16` et `10.3.0.0/16` gelés pour les sites distants (bloc 3).
- Définition du **plan IPv6**, miroir exact de l'IPv4 : `2001:db8:<site>:<vlan>::/64`, passerelle en `::fe` (= 254 en hexa), `::fc/::fd` réservées aux cœurs.
- Définition de la **convention de nommage** `<SITE>-<RÔLE>-<LOCALISATION>-<N°>` (ex. `SG-ACC-E1-02`) + inventaire des 8 équipements avec leurs IP de management.
- Production du **dossier complet** (10 pages) : schéma logique, tableaux VLAN/IPv4/IPv6, nommage, tableau de conformité contrainte par contrainte, liste des points volontairement non traités (Spanning Tree → S3-S4, HSRP/VRRP/GLBP → S5, IPv6 → bloc 2, sites distants → bloc 3).

**Fichier produit :** `NovaCorp-Dossier-Architecture-Reseau-S1.pdf`

---

## Séance 2 — Mise au propre et découpage des livrables

- Création d'une **version draw.io** du schéma logique, simple et modifiable (même topologie que le dossier).
- Découpage du rendu en **5 fichiers séparés**, un par livrable demandé dans l'énoncé, chacun sur une page :
  1. `1-Schema-logique.pdf` — schéma de l'architecture (équipements, rôles, liens) ;
  2. `2-Liste-VLANs.pdf` — liste des VLANs avec leur usage ;
  3. `3-Plan-IPv4.pdf` — réseau, masque, passerelle, plage utilisable par VLAN ;
  4. `4-Plan-IPv6.pdf` — préfixe /64 et passerelle par VLAN ;
  5. `5-Convention-nommage.pdf` — format, règles et inventaire des équipements.
- Vérification de la cohérence entre les 5 fichiers et le dossier complet (mêmes VLANs, mêmes adresses, mêmes noms).

**Fichiers produits :** `NovaCorp-Schema-Simple.drawio`, `1-…` à `5-….pdf`

---

## Séance 3 — Corrections après relecture

- **Refonte du schéma logique** jugé trop brouillon : remplacement des diagonales qui se croisaient par un **tracé orthogonal** (angles droits, une « voie » par switch) avec **une couleur par zone** (bleu = étage 1, vert = étage 2, orange = salle serveur) et une légende. La double liaison de chaque switch vers les deux cœurs se lit maintenant d'un coup d'œil.
- **Retrait de l'hypothèse chiffrée** « ≈ 40 postes » du schéma : remplacée par « postes de travail + téléphones IP », la formulation exacte du cahier des charges (le chiffre reste une hypothèse de dimensionnement interne, pas une donnée du client).
- Re-vérification du rendu final et remplacement du fichier `1-Schema-logique.pdf`.

---

## État actuel du rendu

| Fichier | Contenu |
|---|---|
| `NovaCorp-Dossier-Architecture-Reseau-S1.pdf` | Dossier complet (10 pages) avec justifications |
| `1-Schema-logique.pdf` | Schéma logique v2 (orthogonal, couleurs par zone) |
| `2-Liste-VLANs.pdf` | Plan de VLANs |
| `3-Plan-IPv4.pdf` | Plan d'adressage IPv4 |
| `4-Plan-IPv6.pdf` | Plan d'adressage IPv6 |
| `5-Convention-nommage.pdf` | Convention de nommage + inventaire |
| `NovaCorp-Schema-Simple.drawio` | Schéma modifiable dans draw.io |

**Points ouverts** : le dossier complet et le fichier draw.io utilisent encore la première version du schéma, pas la version orthogonale. Le nom du binôme n'est pas encore sur les livrables, qui portent seulement « groupe loka ».

**À venir (prochaines séances, hors S1)** : configuration des VLANs/trunks/EtherChannel, Spanning Tree et répartition de charge (S3-S4), redondance de passerelle HSRP/VRRP/GLBP (S5), mise en œuvre IPv6 (bloc 2), sites distants (bloc 3).
