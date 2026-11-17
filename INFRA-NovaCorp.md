# Infrastructure réseau — NovaCorp

> **Groupe :** BOUCHOUAREB · GIORDANO · GUILBAUD
> **Sources :** plan d'architecture S1 et maquette `NovaCorp-Siege.pkt` (Cisco Packet Tracer 9.0.1, version v3).
> Les valeurs de ce document ont été **relevées directement dans les configurations de la maquette**.
> Quand la maquette s'écarte du plan papier, l'écart est signalé (voir la section 15).

---

## 1. Périmètre

| Zone | Contenu | Dans le cahier des charges ? |
|---|---|---|
| **Siège : 2 étages** | Postes de travail et téléphones IP | ✅ Oui |
| **Siège : salle serveur** | Cœurs, serveurs, routeurs de la dorsale | ✅ Oui |
| **Bâtiment B** (OSPF area 1) | 1 routeur, 1 switch, 1 PC | ➕ Extension de la maquette |
| **Bâtiment C** (OSPF area 2, *stub*) | 1 routeur, 1 PC | ➕ Extension de la maquette |
| **Sites distants 2 et 3** | 1 routeur et 1 PC chacun, reliés en BGP | ➕ Annoncés dans le CDG pour le bloc 3, simulés en avance |
| **Opérateur (ISP)** | 1 routeur, avec « Internet » simulé par une loopback | ➕ Simulation |

Les contraintes du cahier des charges portent sur le **siège**. Les bâtiments B et C et le WAN sont des extensions ajoutées pour travailler OSPF multi-aires et BGP.

| # | Contrainte du CDG | Où elle est traitée |
|---|---|---|
| C1 | Voix séparée de la data | VLANs voix dédiés (§5) |
| C2 | Salle serveur isolée | VLANs 30/31 sur des switches dédiés (§5, §10) |
| C3 | Management dédié | VLAN 90 (§12) |
| C4 | Aucune panne simple ne coupe le réseau | LACP, Rapid PVST+, HSRP, OSPF (§13) |
| C5 | Croissance | Marges en ports, adresses et VLANs (§14) |
| C6 | Plan IPv4 + IPv6 dès la conception | §6 et §7 |

---

## 2. Architecture générale

L'infrastructure est organisée en **trois couches** :

1. **Commutation du siège**, sur deux niveaux (*collapsed core*) :
   - 2 cœurs multicouches qui font le routage entre VLANs et portent les passerelles HSRP ;
   - en dessous, les switches d'accès (niveau 2).
2. **Dorsale routée OSPF**, en losange :
   - les deux cœurs sont reliés à deux ABR dédiés et au routeur de bordure ;
   - les bâtiments B et C sont raccordés derrière les ABR.
3. **WAN BGP** : le routeur de bordure du siège échange en BGP avec l'opérateur, qui relie aussi les sites distants.

```
        SG-RTR-DC-01 (bordure, AS 65001) ── eBGP ── ISP-RTR-WAN-01 (AS 65000) ──┬── S2-RTR-DC-01 ── PC-S2
          (1 lien /30 vers CHAQUE cœur)          « Internet » 198.51.100.1      └── S3-RTR-DC-01 ── PC-S3
                     │
               SG-CORE-DC-01                                   ◄── OSPF AREA 0 : dorsale en losange
              ╱      ║      ╲
   SG-RTR-DC-02     Po1      SG-RTR-DC-03
   (ABR Ouest)  (+VLAN 98)   (ABR Est)
       │      ╲      ║      ╱      │
       │       SG-CORE-DC-02       │
       │             │             │
  SG-RTR-BB-01       │        SG-RTR-BC-01
  SG-ACC-BB-01       │           PC-BC
     PC-BB           │        AREA 2 stub (bât. C)
  AREA 1 (bât. B)    │
                     │
   SG-ACC-E1-01/02 · SG-ACC-E2-01/02 · SG-SRV-DC-01/02
   chaque switch : 1 port-channel LACP vers CORE-01 + 1 vers CORE-02
   étages : VLAN 10/20 + 110/120 (PC + téléphones IP) · serveurs : VLAN 30 + 31 (2 cartes)
```

---

## 3. Équipements

### 3.1 Équipements réseau (18)

| Nom | Modèle (maquette) | Rôle | Router-ID / IP d'admin |
|---|---|---|---|
| `SG-CORE-DC-01` | 3560-24PS | Cœur n° 1 : niveau 3, HSRP actif, racine STP, DHCP | Lo0 `10.1.255.1` · VLAN 90 `10.1.90.11` |
| `SG-CORE-DC-02` | 3560-24PS | Cœur n° 2 : HSRP en attente, racine STP de secours | Lo0 `10.1.255.2` · VLAN 90 `10.1.90.12` |
| `SG-ACC-E1-01` | 2960-24TT | Accès, étage 1 | `10.1.90.21` |
| `SG-ACC-E1-02` | 2960-24TT | Accès, étage 1 | `10.1.90.22` |
| `SG-ACC-E2-01` | 2960-24TT | Accès, étage 2 | `10.1.90.23` |
| `SG-ACC-E2-02` | 2960-24TT | Accès, étage 2 | `10.1.90.24` |
| `SG-SRV-DC-01` | 2960-24TT | Accès serveurs | `10.1.90.31` |
| `SG-SRV-DC-02` | 2960-24TT | Accès serveurs | `10.1.90.32` |
| `SG-RTR-DC-01` | 2911 | Routeur de bordure : OSPF area 0 + BGP AS 65001 | `10.1.255.3` |
| `SG-RTR-DC-02` | 2911 | ABR Ouest (area 0 ↔ area 1) | `10.1.255.4` |
| `SG-RTR-DC-03` | 2911 | ABR Est (area 0 ↔ area 2 *totally stubby*) | `10.1.255.5` |
| `SG-RTR-BB-01` | 2911 | Routeur du bâtiment B (area 1) | `10.1.255.6` |
| `SG-RTR-BC-01` | 2911 | Routeur du bâtiment C (area 2) | `10.1.255.7` |
| `SG-ACC-BB-01` | 2960-24TT | Accès, bâtiment B (VLAN 40) | `10.1.40.21` |
| `S2-RTR-DC-01` | 2911 | Routeur du site distant 2 (AS 65002) | `10.2.255.1` |
| `S3-RTR-DC-01` | 2911 | Routeur du site distant 3 (AS 65003) | `10.3.255.1` |
| `ISP-RTR-WAN-01` | 2911 | Opérateur simulé (AS 65000) | `198.51.100.1` |

> **Matériel cible ou matériel de maquette ?** Le plan S1 prévoit des switches d'accès **48 ports PoE**, pour alimenter les téléphones et laisser de la marge. Dans Packet Tracer, on utilise des 2960-24TT : 24 ports, dont 20 pour les utilisateurs (Fa0/1-20) et 4 pour les liens montants (Fa0/21-24). La logique est la même, seule la densité de ports change.

### 3.2 Terminaux

| Hôte | Emplacement | Adressage |
|---|---|---|
| PC0, PC1 | Étage 1 (E1-01, E1-02) | DHCP, VLAN 10 · IPv6 autoconfiguré (SLAAC) |
| PC2, PC3 | Étage 2 (E2-01, E2-02) | DHCP, VLAN 20 |
| IP Phone0 à 3 (7960) | Un par switch d'accès, port Fa0/2 | DHCP, VLANs 110 / 120 |
| Server0 | Fa0 → SRV-DC-01, Fa1 → SRV-DC-02 | `10.1.30.1` + `10.1.31.1`, passerelle `10.1.31.254` |
| Server1 | Fa0 → SRV-DC-02, Fa1 → SRV-DC-01 | `10.1.30.2` + `10.1.31.2`, passerelle `10.1.31.254` |
| PC-BB | Bâtiment B | `10.1.40.1/24`, passerelle `.254` |
| PC-BC | Bâtiment C | `10.1.50.1/24`, passerelle `.254` |
| PC-S2 / PC-S3 | Sites distants | `10.2.10.1` / `10.3.10.1`, passerelle `.254` |

---

## 4. Liens

### 4.1 Liens de niveau 2 : EtherChannel LACP (26 liens physiques, 13 port-channels)

| Port-channel | Entre | Ports physiques | VLANs autorisés |
|---|---|---|---|
| Po1 (inter-cœurs) | CORE-01 ↔ CORE-02 | Gi0/1 + Gi0/2 | 10, 20, 30, 31, 90, 98, 99, 110, 120 |
| Po11 / Po12 | CORE-01 et CORE-02 ↔ ACC-E1-01 / E1-02 | 2 × Fa par cœur | 10, 20, 30, 90, 99, 110, 120 |
| Po13 / Po14 | CORE-01 et CORE-02 ↔ ACC-E2-01 / E2-02 | 2 × Fa par cœur | 10, 20, 30, 90, 99, 110, 120 |
| Po15 / Po16 | CORE-01 et CORE-02 ↔ SRV-DC-01 / DC-02 | 2 × Fa par cœur | 10, 20, 30, **31**, 90, 99, 110, 120 |

- **Câblage côté switch d'accès** :
  - `Po1` vers CORE-01 : Fa0/21-22 ;
  - `Po2` vers CORE-02 : Fa0/23-24.
- **Câblage côté cœur** :
  - Fa0/1-12 vers les 6 switches (2 ports par switch) ;
  - Gi0/1-2 vers l'autre cœur.
- Tous les port-channels sont en **LACP `mode active`**, en trunk 802.1Q, avec le **VLAN natif 99**.

> **Pourquoi deux port-channels par switch, et pas un seul ?** LACP ne peut agréger que des liens qui arrivent sur **le même équipement**. Les deux cœurs sont deux switches indépendants (pas de stack, pas de VSS), donc chaque switch d'accès a un port-channel vers chaque cœur.
> - **LACP** apporte la redondance et le débit *à l'intérieur* d'un port-channel : si un câble tombe, l'autre continue.
> - **Spanning Tree** gère la redondance *entre* les deux port-channels : l'un transmet, l'autre est bloqué en secours.

### 4.2 Liens routés internes : bloc `10.1.250.0/24` découpé en /30

| Sous-réseau | Extrémité A | Extrémité B | Aire |
|---|---|---|---|
| `10.1.250.0/30` | CORE-01 Fa0/13 `.1` | RTR-DC-01 Gi0/0 `.2` | 0 |
| `10.1.250.4/30` | CORE-02 Fa0/13 `.5` | RTR-DC-01 Gi0/1 `.6` | 0 |
| `10.1.250.8/30` | CORE-01 Fa0/14 `.9` | RTR-DC-02 Gi0/0 `.10` | 0 |
| `10.1.250.12/30` | CORE-02 Fa0/14 `.13` | RTR-DC-03 Gi0/1 `.14` | 0 |
| `10.1.250.16/30` | RTR-DC-02 Gi0/2 `.17` | RTR-BB-01 Gi0/0 `.18` | 1 |
| `10.1.250.20/30` | CORE-01 Fa0/15 `.21` | RTR-DC-03 Gi0/0 `.22` | 0 |
| `10.1.250.24/30` | CORE-02 Fa0/15 `.25` | RTR-DC-02 Gi0/1 `.26` | 0 |
| `10.1.250.28/30` | RTR-DC-03 Gi0/2 `.29` | RTR-BC-01 Gi0/0 `.30` | 2 |

S'y ajoute le **VLAN 98** (`10.1.98.0/24` : `.252` CORE-01, `.253` CORE-02), transporté dans Po1, qui sert de lien de transit OSPF entre les deux cœurs.

**Pourquoi des /30 ?** Un lien point à point n'a que 2 équipements, et un /30 donne exactement 2 adresses utilisables. Le bloc `10.1.250.0/24` peut contenir 64 liens de ce type.

### 4.3 Liens WAN

| Sous-réseau | Extrémité A | Extrémité B |
|---|---|---|
| `203.0.113.0/30` | ISP Gi0/0 `.1` | SG-RTR-DC-01 Gi0/2 `.2` |
| `203.0.113.4/30` | ISP Gi0/1 `.5` | S2-RTR-DC-01 Gi0/0 `.6` |
| `203.0.113.8/30` | ISP Gi0/2 `.9` | S3-RTR-DC-01 Gi0/0 `.10` |

`203.0.113.0/24` et `198.51.100.0/24` sont des plages **réservées à la documentation** (RFC 5737). On les utilise pour simuler des adresses publiques sans emprunter celles de quelqu'un d'autre.

---

## 5. VLANs

| ID | Nom | Usage | Où | Ajouté en |
|---|---|---|---|---|
| 10 | DATA-E1 | Postes de l'étage 1 | Accès E1 + cœurs | S1 |
| 20 | DATA-E2 | Postes de l'étage 2 | Accès E2 + cœurs | S1 |
| 30 | SRV | Serveurs, 1ʳᵉ carte | Switches serveurs + cœurs | S1 |
| 31 | SRV-B | Serveurs, 2ᵉ carte (double attachement) | Switches serveurs + cœurs | Maquette v1 |
| 90 | MGMT | Administration des switches | Tous les switches du siège | S1 |
| 98 | TRANSIT-L3 | Voisinage OSPF entre les cœurs | Po1 seulement | Maquette v1 |
| 99 | NATIF | VLAN natif des trunks : aucun port d'accès, aucune adresse IP | Trunks | S1 |
| 110 | VOIX-E1 | Téléphones IP de l'étage 1 | Accès E1 + cœurs | S1 |
| 120 | VOIX-E2 | Téléphones IP de l'étage 2 | Accès E2 + cœurs | S1 |
| 40 | DATA-BB | LAN du bâtiment B | `SG-ACC-BB-01` | Maquette v2/v3 |
| 50 | DATA-BC | LAN du bâtiment C | Interface Gi0/1 de `SG-RTR-BC-01` | Maquette v2/v3 |

- **Règles de numérotation** :
  - la dizaine indique la fonction ;
  - voix = 100 + data du même étage ;
  - le 3ᵉ octet de l'adresse IPv4 reprend le numéro du VLAN.
- **VLAN 1** : jamais utilisé (`interface Vlan1` désactivée).
- **VLAN natif 99** : ne porte aucun trafic utile.
- **VLANs 40 et 50** : ce sont surtout des **numéros de LAN** qui gardent la règle d'adressage. Ils ne passent pas dans les trunks du siège, car chaque bâtiment est derrière son propre routeur.
- **Numéros encore libres** : 41-80 (sauf 50), 130-190.

---

## 6. Adressage IPv4

### 6.1 Règle

**`10.<site>.<vlan>.0/24`** pour les LAN. Le siège et ses bâtiments sont le site 1, les sites distants 2 et 3.

| Plage | Usage |
|---|---|
| `10.1.<vlan>.0/24` | LAN du siège et des bâtiments B et C |
| `10.1.250.0/24` | Liens routés internes en /30 |
| `10.1.255.0/24` | Loopbacks / router-ID en /32 (`.1` à `.7`) |
| `10.2.0.0/16`, `10.3.0.0/16` | Sites distants (LAN `10.x.10.0/24`, loopback `10.x.255.1`) |
| `203.0.113.0/24` | Liens WAN (simulation) |
| `198.51.100.1/32` | « Internet » simulé |

### 6.2 LAN du siège

| VLAN | Réseau | Passerelle (adresse virtuelle HSRP) | CORE-01 | CORE-02 | Hôtes |
|---|---|---|---|---|---|
| 10 | `10.1.10.0/24` | `.254` | `.252` | `.253` | DHCP `.1-.250` |
| 20 | `10.1.20.0/24` | `.254` | `.252` | `.253` | DHCP `.1-.250` |
| 30 | `10.1.30.0/24` | `.254` | `.252` | `.253` | Fixes (`.1`, `.2`) |
| 31 | `10.1.31.0/24` | `.254` | `.252` | `.253` | Fixes (`.1`, `.2`) |
| 90 | `10.1.90.0/24` | `.254` | `.11` | `.12` | Fixes, switches `.21-.32` |
| 98 | `10.1.98.0/24` | pas de HSRP (transit) | `.252` | `.253` | |
| 110 | `10.1.110.0/24` | `.254` | `.252` | `.253` | DHCP `.1-.250` |
| 120 | `10.1.120.0/24` | `.254` | `.252` | `.253` | DHCP `.1-.250` |

Le VLAN 90 fait exception : les cœurs y utilisent leur adresse de management (`.11` et `.12`, règle « 1x = cœurs ») au lieu de `.252` et `.253`.

### 6.3 DHCP

Le DHCP est assuré par **`SG-CORE-DC-01`** :
- pools DATA-E1, DATA-E2, VOIX-E1 et VOIX-E2 ;
- adresses `.251` à `.254` exclues de chaque pool ;
- passerelle par défaut annoncée : `.254` ;
- DNS : `10.1.30.1` (Server0).

Le serveur DHCP étant **sur le cœur**, aucun relais (`ip helper-address`) n'est nécessaire : les SVI sont directement dans chaque VLAN.

> ⚠️ **Limite** : seul CORE-01 distribue des adresses. Si CORE-01 tombe, les postes qui ont déjà un bail continuent de fonctionner, mais aucun nouveau poste n'obtient d'adresse. Correction prévue : des pools sur CORE-02 avec des plages séparées, par exemple `.1-.125` sur CORE-01 et `.126-.250` sur CORE-02.

---

## 7. Adressage IPv6

| VLAN | Préfixe | CORE-01 | CORE-02 | Passerelle des hôtes |
|---|---|---|---|---|
| 10 | `2001:db8:1:10::/64` | `::fc` | `::fd` | Adresse link-local virtuelle HSRP |
| 20 | `2001:db8:1:20::/64` | `::fc` | `::fd` | Idem |
| 30 | `2001:db8:1:30::/64` | `::fc` | `::fd` | Idem |
| 31 | `2001:db8:1:31::/64` | `::fc` | `::fd` | Idem |
| 90 | `2001:db8:1:90::/64` | `::fc` | `::fd` | Idem |
| 110 | `2001:db8:1:110::/64` | `::fc` | `::fd` | Idem |
| 120 | `2001:db8:1:120::/64` | `::fc` | `::fd` | Idem |

- **Adresses link-local fixes** sur les cœurs : `FE80::FC` (CORE-01) et `FE80::FD` (CORE-02).
- **Passerelle** : HSRP en IPv6 (`standby <groupe> ipv6 autoconfig`) crée une **adresse link-local virtuelle** (de la forme `FE80::5:73FF:FEA0:xxxx`). Les postes en **SLAAC** la reçoivent comme passerelle.
  - C'est la pratique normale en IPv6 : la passerelle d'un hôte est toujours une adresse link-local.
  - Le plan papier prévoyait `::fe` comme passerelle, mais cette adresse n'est **pas utilisée** dans la maquette.
- **Portée** : IPv6 est déployé **uniquement à l'intérieur du siège**. Il n'y a ni OSPFv3, ni BGP IPv6, ni IPv6 sur les liens /30 et le WAN. C'est l'objet du bloc 2.
- **`2001:db8::/32`** est le préfixe réservé à la documentation (RFC 3849). En production, il serait remplacé par le préfixe /48 fourni par l'opérateur, en gardant la structure `<site>:<vlan>`.

---

## 8. Commutation : trunks, Spanning Tree et ports d'accès

- **Trunks** : 802.1Q, `switchport mode trunk` forcé, natif 99, VLANs autorisés limités à ce qui est utile. Par exemple, le VLAN 31 ne passe que vers les switches serveurs, et le VLAN 98 que sur Po1.
- **Spanning Tree** :
  - mode **Rapid PVST+** sur tous les switches ;
  - priorités pour les VLANs 10, 20, 30, 90, 99, 110 et 120 :

    | Switch | Priorité | Rôle |
    |---|---|---|
    | `SG-CORE-DC-01` | **24576** | Pont racine |
    | `SG-CORE-DC-02` | **28672** | Racine de secours |
    | Switches d'accès | 32768 (défaut) | Jamais racine |

  - Résultat : sur chaque switch d'accès, **le port-channel vers CORE-01 transmet** et celui vers CORE-02 est **bloqué en secours**.
  - La racine STP et le HSRP actif sont **sur le même cœur**, donc le trafic ne fait pas de détour par le lien entre les cœurs.
  - Pas encore de **répartition de charge** : on pourrait mettre CORE-02 racine pour certains VLANs (par exemple ceux de l'étage 2), avec le HSRP actif aligné. C'est le sujet des séances S3-S4.
  - ⚠️ Les VLANs **31 et 98** ne figurent pas dans la commande de priorité : leur racine est élue par adresse MAC. À corriger.
- **Ports d'accès utilisateurs** (Fa0/1-20) :
  - `switchport access vlan 10|20` + `switchport voice vlan 110|120` : le PC et le téléphone partagent la prise ;
  - `spanning-tree portfast` + `bpduguard enable` : le port passe en transmission tout de suite, et se coupe si quelqu'un y branche un switch.

---

## 9. Redondance de passerelle : HSRP version 2

| Paramètre | Valeur |
|---|---|
| Version | HSRP v2, nécessaire pour IPv6 et pour les numéros de groupe au-delà de 255 |
| Groupes IPv4 | Numéro du groupe = numéro du VLAN (10, 20, 30, 31, 90, 110, 120) |
| Groupes IPv6 | 1000 + VLAN (1010, 1020, …, 1120), en `ipv6 autoconfig` |
| Routeur actif | `SG-CORE-DC-01` (priorité 110, `preempt`) |
| Routeur en attente | `SG-CORE-DC-02` (priorité 100 par défaut) |
| Adresse virtuelle IPv4 | `.254` dans chaque VLAN |

- **`preempt`** : quand CORE-01 revient après une panne, il **reprend** le rôle actif.
- **Choix du protocole** : HSRP est un protocole Cisco ; VRRP est son équivalent normalisé ; GLBP répartit la charge entre plusieurs passerelles. La comparaison détaillée est prévue en S5.

---

## 10. Routage OSPF (processus 1)

### 10.1 Aires et rôles

| Routeur | Router-ID | Aire(s) | Rôle OSPF |
|---|---|---|---|
| `SG-CORE-DC-01` | 10.1.255.1 | 0 | Interne à la dorsale ; annonce les LAN du siège |
| `SG-CORE-DC-02` | 10.1.255.2 | 0 | Interne à la dorsale ; annonce les LAN du siège |
| `SG-RTR-DC-01` | 10.1.255.3 | 0 | ASBR : annonce la route par défaut (`default-information originate`) |
| `SG-RTR-DC-02` | 10.1.255.4 | 0 + 1 | ABR Ouest |
| `SG-RTR-DC-03` | 10.1.255.5 | 0 + 2 | ABR Est ; `area 2 stub no-summary` |
| `SG-RTR-BB-01` | 10.1.255.6 | 1 | Interne à l'aire 1 ; annonce le LAN 10.1.40.0/24 |
| `SG-RTR-BC-01` | 10.1.255.7 | 2 | Interne à l'aire 2 ; `area 2 stub` |

### 10.2 Choix de configuration
- **`passive-interface default`** sur les cœurs et le routeur de bordure. OSPF n'envoie de *hello* que là où il y a un voisin :
  - cœurs : Fa0/13-15 et VLAN 98 ;
  - routeur de bordure : Gi0/0-1.

  Les LAN utilisateurs sont annoncés, mais aucun poste ne reçoit de paquets OSPF. Cela évite qu'un PC se fasse passer pour un routeur.
- **Une déclaration `network` par sous-réseau**, avec un masque inverse précis, au lieu d'un `10.1.0.0/16` générique. On contrôle ainsi exactement quelle interface tombe dans quelle aire.
- **Loopbacks en router-ID** : l'identifiant est stable et ne dépend pas de l'état d'une interface physique.
- **Dorsale en losange** :
  - chaque ABR a un lien vers chacun des deux cœurs ;
  - le routeur de bordure a aussi un lien vers chaque cœur ;
  - les deux cœurs sont reliés par le VLAN 98.

  La perte d'un cœur ou d'un lien ne coupe donc pas l'area 0. Sur chaque cœur, on compte **4 voisins FULL**.
- **Area 2 *totally stubby*** : le bâtiment C ne reçoit **qu'une route par défaut** de l'ABR Est, ni routes inter-aires ni routes externes. Sa table de routage reste minimale. C'est adapté à un site qui n'a qu'une seule sortie.
- **Area 1 standard** : le bâtiment B reçoit les routes inter-aires (`O IA`) et la route par défaut externe (`O*E2`).

### 10.3 Route par défaut
`SG-RTR-DC-01` a une route statique `0.0.0.0/0` vers l'ISP (`203.0.113.1`) et la **propage dans OSPF**. Tous les routeurs internes envoient donc vers le routeur de bordure tout ce qu'ils ne connaissent pas : les sites distants et « Internet ».

---

## 11. WAN et BGP

| AS | Routeur | Voisin | Réseaux annoncés |
|---|---|---|---|
| 65001 (siège) | `SG-RTR-DC-01` | 203.0.113.1 (AS 65000) | `10.1.0.0/16` (agrégat) |
| 65000 (opérateur) | `ISP-RTR-WAN-01` | les 3 sites | `198.51.100.1/32` et les 3 liens WAN |
| 65002 (site 2) | `S2-RTR-DC-01` | 203.0.113.5 | `10.2.10.0/24` |
| 65003 (site 3) | `S3-RTR-DC-01` | 203.0.113.9 | `10.3.10.0/24` |

- **3 sessions eBGP** au total, une entre chaque site et l'opérateur. Les numéros d'AS 64512 à 65534 sont **privés**.
- **Annonce agrégée** : le siège annonce un seul préfixe, `10.1.0.0/16`, au lieu de chaque /24.
  - BGP n'annonce un réseau que s'il est présent tel quel dans la table de routage. D'où la route **`ip route 10.1.0.0 255.255.0.0 Null0`**.
  - Cette route jette aussi le trafic vers une adresse `10.1.x.x` qui n'existe pas, ce qui évite les boucles.
- **OSPF et BGP ne sont pas redistribués l'un dans l'autre** :
  - à l'intérieur du siège, les routeurs n'apprennent pas les routes BGP une par une, ils utilisent la route par défaut ;
  - c'est le routeur de bordure qui connaît les préfixes des sites 2 et 3.

> ⚠️ **Simplification de la maquette** : les sites échangent des adresses **privées (10.x)** à travers « l'opérateur ». Un vrai fournisseur d'accès ne routerait jamais du RFC 1918. En production, on relierait les sites par un **tunnel VPN** (IPsec, GRE sur IPsec, DMVPN) ou par un **VPN MPLS** de l'opérateur.

---

## 12. Management

- **Switches** : une SVI dans le **VLAN 90** pour chacun. Le dernier octet suit la règle **1x = cœurs, 2x = accès, 3x = serveurs**. Passerelle `ip default-gateway 10.1.90.254` (adresse virtuelle HSRP).
- **Routeurs** : administrés par leur **loopback** (`10.1.255.x`), joignable par OSPF tant qu'un chemin existe.
- **Bâtiment B** : `SG-ACC-BB-01` est administré en `10.1.40.21`, dans le LAN des utilisateurs. Le bâtiment n'a pas de VLAN 90, puisque son routeur n'a qu'une seule interface LAN. C'est un écart à la règle (voir §15).
- Aucun poste utilisateur n'est dans le VLAN 90 (contrainte C3).

---

## 13. Redondance : que se passe-t-il en cas de panne ?

| Panne | Mécanisme qui prend le relais | Résultat |
|---|---|---|
| Un câble d'un port-channel | **LACP** : le second câble du même Po continue | ✅ Transparent |
| Un port-channel complet entre un switch d'accès et CORE-01 | **Rapid PVST+** débloque le Po vers CORE-02 | ✅ Bascule en quelques secondes |
| Un câble entre les cœurs | **LACP** sur Po1 | ✅ Transparent |
| `SG-CORE-DC-01` entier | **HSRP** : CORE-02 devient actif · **RPVST+** : CORE-02 devient racine · **OSPF** reconverge | ✅ Le réseau tient · ⚠️ plus de nouveaux baux DHCP |
| Un lien entre un cœur et un ABR, ou le routeur de bordure | **OSPF** passe par l'autre cœur (testé : `.250.25 → .250.14`) | ✅ Testé |
| `SG-SRV-DC-01` | Serveurs joignables par leur 2ᵉ carte (VLAN 31) | ✅ Testé · ⚠️ limite Packet Tracer, voir §16 |
| Un switch d'accès d'étage | Seuls les postes branchés dessus sont coupés ; l'autre switch de l'étage fonctionne | ✅ Le réseau tient |
| `SG-RTR-DC-01` (bordure) | Aucun : **un seul routeur et un seul lien WAN** | ❌ Plus d'accès aux sites distants ni à « Internet » |
| `SG-RTR-DC-02` ou `SG-RTR-DC-03` (ABR) | Aucun : un seul ABR par bâtiment | ❌ Bâtiment B ou C isolé |
| `SG-RTR-BB-01` ou `SG-RTR-BC-01` | Aucun : un seul routeur par bâtiment | ❌ Bâtiment isolé |

> **À savoir défendre** : la contrainte C4 du cahier des charges porte sur le **siège**, et le siège la respecte : tout y est doublé. Les points de panne uniques restants (bordure, ABR, routeurs des bâtiments) concernent des **extensions ajoutées** dans la maquette. Ils sont identifiés et ont une piste de correction (§16). Une première version (v2) doublait le bâtiment B avec HSRP ; la v3 a privilégié une dorsale avec des ABR dédiés.

---

## 14. Croissance

| Ressource | Capacité | Marge |
|---|---|---|
| Ports utilisateurs | Plan cible : 2 × 48 ports PoE par étage (maquette : 2 × 20) | Plusieurs dizaines de postes par étage |
| Adresses | 250 hôtes par VLAN | Très au-delà du besoin |
| Ports des cœurs | Fa0/16-24 libres sur chaque cœur | Un switch ou un routeur de plus par cœur |
| Liens routés | 8 /30 utilisés sur 64 dans `10.1.250.0/24` | 56 liens de réserve |
| Loopbacks | 7 sur 254 dans `10.1.255.0/24` | Large |
| Numéros de VLAN | 41-80 (sauf 50), 130-190 | Wi-Fi, invités, imprimantes, nouveaux bâtiments |
| OSPF | Ajouter une aire = ajouter un ABR relié aux deux cœurs | Modèle déjà en place |
| Sites | `10.2/16` et `10.3/16` en service, `10.4/16` et plus libres | Un nouveau site = un nouvel AS privé |

---

## 15. Écarts entre le plan papier (S1) et la maquette

| Sujet | Plan S1 | Maquette | Commentaire |
|---|---|---|---|
| Switches d'accès | 48 ports PoE | 2960-24TT, 20 ports utilisateurs | Limite de Packet Tracer, logique identique |
| Liens montants | 1 lien vers chaque cœur | **2 liens en LACP** vers chaque cœur | Amélioration : redondance de câble + débit |
| Passerelle IPv6 | `::fe` | Link-local virtuelle HSRP | La maquette suit la bonne pratique ; corriger la fiche 4 |
| Adresses des cœurs dans le VLAN 90 | `.252` / `.253` | `.11` / `.12` | La règle de management prime |
| VLANs | 7 | 7 + **31, 98, 40, 50** | Ajouts liés au routage et au double attachement |
| Routage | Inter-VLAN sur les cœurs | + OSPF multi-aires + BGP | Extension |
| Équipements | 8 switches | 9 switches + 7 routeurs | Routeurs : bordure, ABR, bâtiments, WAN |
| Management du bâtiment B | — | Dans le VLAN 40 (data) | Pas de VLAN 90 hors du siège |

---

## 16. Limites connues et corrections à faire

**Corrections simples, relevées dans les configurations :**
1. Ajouter `31,98` à `spanning-tree vlan … priority` sur les deux cœurs.
2. Créer des pools DHCP de secours sur CORE-02.
3. Sécuriser l'administration : `enable secret`, comptes locaux, **SSH v2**, `transport input ssh`, ACL sur VTY. Aujourd'hui les lignes VTY sont en `login` sans mot de passe, donc inaccessibles, mais aussi non sécurisées.
4. Désactiver les ports d'accès inutilisés, ou les placer dans un VLAN poubelle.
5. Supprimer les descriptions restées de la v2 sur `SG-ACC-BB-01` (Fa0/1-2, « VERS-SG-RTR-BB-01-02 »).

**Limites de Packet Tracer :**
- Un serveur à 2 cartes n'a **qu'une seule passerelle par défaut** (ici `10.1.31.254`). Pendant la panne d'un switch serveur, il répond à ses voisins directs, mais pas toujours aux autres VLANs. En production, c'est le **NIC teaming** (agrégation côté serveur) qui gère ce cas.
- Les téléphones obtiennent une adresse, mais aucun serveur de téléphonie (CME / CUCM) n'est simulé. Il n'y a pas d'option DHCP 150.

**Pistes d'évolution :**
- Doubler la bordure WAN : 2 routeurs, 2 accès opérateur, éventuellement 2 opérateurs.
- Doubler les bâtiments B et C : 2ᵉ routeur avec HSRP, ou 2ᵉ ABR.
- Ajouter un lien entre les deux ABR (dorsale en anneau complet).
- Router IPv6 de bout en bout : OSPFv3, BGP IPv6.
- Répartir la charge STP et HSRP par VLAN entre les deux cœurs (S3-S5).
- Filtrer les flux entre VLANs (ACL) : utilisateurs → serveurs, accès au VLAN 90.

---

## 17. Convention de nommage

Format : **`<SITE>-<RÔLE>-<LOCALISATION>-<N°>`**, en majuscules, avec des tirets, sans accents, 15 caractères au maximum.

| Champ | Valeurs utilisées |
|---|---|
| SITE | `SG` = siège ; `S2`, `S3` = sites distants ; `ISP` = opérateur (exception : équipement qui n'appartient pas à NovaCorp) |
| RÔLE | `CORE` = cœur, `ACC` = accès utilisateurs, `SRV` = accès serveurs, `RTR` = routeur. Réservés : `FW`, `AP` |
| LOCALISATION | `E1`, `E2` = étages ; `DC` = salle serveur ; `BB`, `BC` = bâtiments B et C ; `WAN` = côté opérateur |
| N° | `01` à `99` |

Exemples :
- `SG-RTR-DC-02` : routeur du siège installé en salle serveur, n° 02 (l'ABR Ouest) ;
- `SG-ACC-BB-01` : switch d'accès du bâtiment B ;
- `S2-RTR-DC-01` : routeur du site distant 2.

---

## 18. Glossaire

| Terme | Définition |
|---|---|
| **Collapsed core** | Cœur et distribution réunis dans les mêmes équipements |
| **SVI** | Interface VLAN d'un switch multicouche (`interface VlanX`), qui sert de passerelle |
| **LACP / EtherChannel** | Agrégation de plusieurs câbles en un seul lien logique (*port-channel*), normalisée IEEE 802.3ad |
| **Trunk 802.1Q / VLAN natif** | Lien qui transporte plusieurs VLANs étiquetés ; le VLAN natif est celui qui passe sans étiquette |
| **Rapid PVST+** | Spanning Tree rapide avec une instance par VLAN ; bloque les boucles et réactive un lien de secours |
| **PortFast / BPDU Guard** | Port d'accès qui transmet tout de suite / qui se coupe s'il reçoit des messages d'un switch |
| **HSRP** | Protocole Cisco de redondance de passerelle : deux routeurs partagent une adresse virtuelle |
| **OSPF / aire / dorsale** | Routage interne à état de liens, découpé en aires ; l'aire 0 (dorsale) relie toutes les autres |
| **ABR / ASBR** | Routeur à la frontière entre deux aires / routeur qui injecte des routes externes (ici la route par défaut) |
| **Stub / totally stubby** | Aire qui ne reçoit pas les routes externes / qui ne reçoit qu'une route par défaut |
| **Router-ID / loopback** | Identifiant unique d'un routeur OSPF ou BGP, porté par une interface virtuelle toujours active |
| **BGP / AS / eBGP** | Routage entre systèmes autonomes (AS) ; eBGP = session entre deux AS différents |
| **Null0** | Interface « poubelle » : tout paquet routé vers elle est jeté |
| **SLAAC** | Autoconfiguration d'une adresse IPv6 par l'hôte à partir des annonces du routeur |
