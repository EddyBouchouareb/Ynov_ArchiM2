# Changelog — TP architecture réseau « Campus NovaCorp »

> **Groupe :** BOUCHOUAREB · GIORDANO · GUILBAUD
> **Livrables :** dossier S1 (PDF), 5 fiches PDF, schéma draw.io, maquette `NovaCorp-Siege.pkt` (Cisco Packet Tracer 9.0.1)
> Ce fichier retrace le travail des trois séances, dans l'ordre chronologique.

---

## Séance 1 — Analyse du besoin et conception de l'architecture

### Analyse
- Lecture du cahier des charges et relevé des 6 contraintes :
  - séparer la voix et la data ;
  - isoler la salle serveur ;
  - prévoir un réseau de management dédié ;
  - aucune panne d'un seul équipement ou d'un seul lien ne doit couper le réseau ;
  - prévoir la croissance ;
  - préparer un plan IPv4/IPv6 dès la conception.
- Relevé de ce que le cahier des charges exclut du livrable S1 : la configuration détaillée, le choix HSRP/VRRP/GLBP (S5) et Spanning Tree (S3-S4).

### Conception
- **Architecture sur deux niveaux** (le cœur et la distribution sont réunis, *collapsed core*) :
  - 2 cœurs de niveau 3 (`SG-CORE-DC-01/02`) ;
  - 4 switches d'accès, 2 par étage ;
  - 2 switches pour la salle serveur.
- **Redondance** : chaque switch d'accès est relié aux **deux** cœurs, et les cœurs sont reliés entre eux par 2 liens.
- **VLANs** :

  | VLAN | Usage |
  |---|---|
  | 10 / 20 | Data étage 1 / étage 2 |
  | 110 / 120 | Voix étage 1 / étage 2 (règle : voix = 100 + data) |
  | 30 | Serveurs |
  | 90 | Management |
  | 99 | Natif des trunks |

- **IPv4** : `10.<site>.<vlan>.0/24`. Passerelle en `.254`, adresses `.252` et `.253` réservées aux cœurs.
- **IPv6** : `2001:db8:<site>:<vlan>::/64`, sur le même modèle que l'IPv4.
- **Nommage** : `<SITE>-<RÔLE>-<LOCALISATION>-<N°>`.

### Production
- `NovaCorp-Dossier-Architecture-Reseau-S1.pdf` (10 pages). Il contient :
  - le schéma ;
  - les tableaux VLAN, IPv4 et IPv6 ;
  - la convention de nommage ;
  - la conformité au cahier des charges, contrainte par contrainte ;
  - la liste des points renvoyés aux séances suivantes.
- Corrections de mise en page : titre coupé, texte qui débordait des boîtes, sommaire.

---

## Séance 2 — Mise au propre et découpage des livrables

- **Version draw.io** du schéma logique (`NovaCorp-Schema-Simple.drawio`), simple et modifiable.
- **5 fiches PDF d'une page**, une par livrable demandé :
  - `1-Schema-logique.pdf`
  - `2-Liste-VLANs.pdf`
  - `3-Plan-IPv4.pdf`
  - `4-Plan-IPv6.pdf`
  - `5-Convention-nommage.pdf`
- Vérification que les 5 fiches et le dossier complet disent la même chose.

---

## Séance 3 — Corrections, maquette Packet Tracer et routage

### 3.1 Corrections des livrables
- **Schéma logique refait** :
  - les liens suivent des angles droits au lieu de se croiser en diagonale ;
  - une couleur par zone (bleu = étage 1, vert = étage 2, orange = salle serveur) ;
  - une légende.
- **Retrait de l'hypothèse « ≈ 40 postes »** du schéma, remplacée par « postes de travail + téléphones IP », la formulation exacte du cahier des charges.

### 3.2 Maquette Packet Tracer : socle de commutation
Construction de `NovaCorp-Siege.pkt` à partir du plan S1.

- **Matériel** :
  - cœurs : 3560-24PS ;
  - accès et salle serveur : 2960-24TT ;
  - 4 PC, 4 téléphones IP 7960, 2 serveurs.
- **LACP (EtherChannel)** :
  - chaque switch d'accès et de salle serveur a **2 port-channels**, un vers chaque cœur, chacun de 2 liens (4 liens montants par switch) ;
  - les cœurs sont reliés par un port-channel de 2 liens Gigabit (`Po1`).
  - Au total : 13 port-channels et 26 liens physiques.
- **Trunks 802.1Q** : VLAN natif 99, et seuls les VLANs nécessaires sont autorisés sur chaque trunk.
- **Rapid PVST+** :
  - `SG-CORE-DC-01` est pont racine (priorité 24576) ;
  - `SG-CORE-DC-02` est racine de secours (priorité 28672).
- **Ports d'accès** :
  - VLAN data + VLAN voix sur le même port ;
  - `portfast` et `bpduguard` activés ;
  - 20 ports utilisateurs par switch (Fa0/1-20).
- **HSRP version 2**, sur les SVI des cœurs, en IPv4 et en IPv6 :
  - `SG-CORE-DC-01` est actif (priorité 110, *preempt*) ;
  - adresse virtuelle IPv4 : `.254` ;
  - adresses réelles : `.252` et `.253`.
- **DHCP sur `SG-CORE-DC-01`** :
  - pools DATA-E1, DATA-E2, VOIX-E1 et VOIX-E2 ;
  - adresses `.251` à `.254` exclues ;
  - DNS : `10.1.30.1`.
- **Management** : une SVI VLAN 90 sur chaque switch, de `10.1.90.11` à `.32`, et `Vlan1` désactivée.

### 3.3 v1 — OSPF, BGP et double attachement des serveurs
- **OSPF area 0 au siège** :
  - ajout du routeur de bordure `SG-RTR-DC-01` (2911), avec un lien routé /30 vers chaque cœur ;
  - **VLAN 98 TRANSIT-L3** (`10.1.98.0/24`), qui passe dans le port-channel entre les cœurs, pour que les deux cœurs soient voisins OSPF ;
  - loopbacks utilisées comme router-ID, dans `10.1.255.0/24` ;
  - `passive-interface default` : OSPF n'échange que sur les liens de transit ;
  - route par défaut annoncée dans OSPF (`default-information originate`).
- **BGP (WAN simulé)** :

  | Routeur | AS | Rôle |
  |---|---|---|
  | `ISP-RTR-WAN-01` | 65000 | Opérateur ; loopback `198.51.100.1` = « Internet » |
  | `SG-RTR-DC-01` | 65001 | Siège ; annonce `10.1.0.0/16` |
  | `S2-RTR-DC-01` | 65002 | Site distant 2 ; LAN `10.2.10.0/24` |
  | `S3-RTR-DC-01` | 65003 | Site distant 3 ; LAN `10.3.10.0/24` |

  Les liens WAN utilisent `203.0.113.0/30`, `.4/30` et `.8/30`.
- **Double attachement des serveurs** :
  - ajout d'une 2ᵉ carte réseau sur `Server0` et `Server1`, reliée à l'autre switch serveur ;
  - création du **VLAN 31 SRV-B** (`10.1.31.0/24`), avec une passerelle HSRP en `10.1.31.254`.

### 3.4 v2 — OSPF multi-aires, première version (remplacée)
- Bâtiment B en **area 1** : 2 routeurs `SG-RTR-BB-01/02` en HSRP.
- Bâtiment C en **area 2** : `SG-RTR-BC-01`.
- Les cœurs servaient d'**ABR** (routeurs à la frontière entre aires).
- Version remplacée par la v3 pour suivre le modèle d'une dorsale avec des ABR dédiés.

### 3.5 v3 — Dorsale OSPF en losange avec ABR dédiés (état final)
- Ajout de 2 ABR dédiés, chacun relié **aux deux cœurs**. L'area 0 forme ainsi un losange redondant :
  - **`SG-RTR-DC-02`**, ABR Ouest, entre l'area 0 et l'area 1 ;
  - **`SG-RTR-DC-03`**, ABR Est, entre l'area 0 et l'area 2.
- Les cœurs redeviennent de **simples routeurs internes de l'area 0**.
- **Area 1, bâtiment B** :
  - `SG-RTR-BB-01` ;
  - switch `SG-ACC-BB-01` et `PC-BB` ;
  - LAN `10.1.40.0/24`.
- **Area 2, bâtiment C** :
  - aire *totally stubby* : `area 2 stub no-summary` sur l'ABR Est, `area 2 stub` sur le routeur du bâtiment ;
  - `SG-RTR-BC-01` et `PC-BC` ;
  - LAN `10.1.50.0/24`.
- Les déclarations OSPF génériques `network 10.1.0.0/16` sont remplacées par une déclaration par sous-réseau.
- Schéma Packet Tracer annoté : cadres « OSPF AREA 0 – DORSALE », « AREA 1 – BÂTIMENT B », « AREA 2 (STUB) – BÂTIMENT C » et bandeau « WAN BGP ».

### Tests de validation réalisés pendant la séance
| Test | Résultat |
|---|---|
| `show ip ospf neighbor` sur chaque cœur | 4 voisins FULL : l'autre cœur, `SG-RTR-DC-01` et les 2 ABR |
| `show ip bgp summary` | 3 sessions eBGP établies, une par site ; routes 10.2, 10.3 et `198.51.100.1` reçues |
| `tracert` PC-BB → PC-BC | Chemin BB-01 → ABR Ouest → CORE-01 → ABR Est → BC-01 : traverse les 3 aires |
| Pings PC-BB → Server0, PC-S2, « Internet » | OK |
| Coupure du lien ABR Ouest ↔ CORE-01 | Le trafic repasse par CORE-02 sans perte de service |
| Coupure du lien CORE-01 → routeur de bordure | La route par défaut est réapprise via CORE-02 |
| `SG-SRV-DC-01` éteint | Les serveurs restent joignables via le second switch |

### Problèmes rencontrés
- Après un `no network 10.1.0.0 …`, Packet Tracer ne réévaluait pas les interfaces déjà actives. Il a fallu réappliquer les `network` de l'area 0, y compris après un `clear ip ospf process`.
- Un serveur Packet Tracer à 2 cartes n'utilise qu'**une seule passerelle par défaut**. Pendant la panne d'un switch serveur, il répond à ses voisins directs mais pas forcément aux autres VLANs. En production, ce cas est géré par le *NIC teaming*.

---

## État actuel du rendu

| Fichier | Contenu | À jour avec la maquette ? |
|---|---|---|
| `NovaCorp-Siege.pkt` | Maquette Packet Tracer, version v3 | ✅ Référence |
| `INFRA-NovaCorp.md` | Description détaillée de l'infrastructure (plan et maquette) | ✅ |
| `NovaCorp-Dossier-Architecture-Reseau-S1.pdf` | Dossier complet S1 | ⚠️ Plan S1 seulement |
| `1-Schema-logique.pdf` | Schéma logique v2 | ⚠️ Sans routeurs, OSPF ni WAN |
| `2-Liste-VLANs.pdf` | Plan de VLANs | ⚠️ Il manque 31, 98, 40 et 50 |
| `3-Plan-IPv4.pdf` | Plan IPv4 | ⚠️ Il manque les /30, les loopbacks et le WAN |
| `4-Plan-IPv6.pdf` | Plan IPv6 | ⚠️ Passerelle `::fe` dans le plan, mais adresse link-local virtuelle HSRP dans la maquette |
| `5-Convention-nommage.pdf` | Nommage et inventaire | ⚠️ Il manque les 7 routeurs et `SG-ACC-BB-01` |
| `NovaCorp-Schema-Simple.drawio` | Schéma modifiable | ⚠️ Ancienne version |

## À faire

### Mettre à jour les livrables
- [ ] Ajouter la maquette dans les fiches 1 à 5 : routeurs, VLANs 31/98/40/50, liens /30, loopbacks, WAN, passerelle IPv6.
- [ ] Ajouter les noms du groupe sur les PDF, qui portent encore « groupe loka ».

### Corriger la maquette (relevé en relisant les configs)
- [ ] Ajouter les VLANs 31 et 98 à `spanning-tree vlan … priority` sur les cœurs. Sinon, la racine du VLAN 31 est élue au hasard des adresses MAC et peut tomber sur un 2960.
- [ ] Dupliquer les pools DHCP sur `SG-CORE-DC-02` (avec des plages séparées). Aujourd'hui, si CORE-01 tombe, plus aucun bail n'est distribué.
- [ ] Sécuriser l'administration : mots de passe, `enable secret`, SSH, ACL sur les lignes VTY. Pour l'instant, VTY est en `login` sans mot de passe.
- [ ] Nettoyer les descriptions de la v2 restées sur `SG-ACC-BB-01` (« VERS-SG-RTR-BB-01-02 »).

### Pistes d'amélioration
- [ ] Doubler les bâtiments B et C : un 2ᵉ routeur avec HSRP, ou un 2ᵉ ABR.
- [ ] Ajouter un lien entre les deux ABR.
- [ ] Passer IPv6 en routage dynamique (OSPFv3, BGP IPv6).
- [ ] Doubler le routeur de bordure et l'accès WAN.

### Séances suivantes du cours
- [ ] Répartition de charge Spanning Tree (S3-S4).
- [ ] Comparaison HSRP / VRRP / GLBP (S5).
- [ ] IPv6 (bloc 2).
- [ ] Sites distants (bloc 3).
