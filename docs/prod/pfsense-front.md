# 🛡️ Routage et Pare-feu : pfSense Front (VM100)

Le routeur virtuel **pfSense Front** est le point d'entrée principal de l'infrastructure. Situé immédiatement derrière la box de l'opérateur (Livebox), il agit comme pare-feu frontal, gère la DMZ exposant les services web, et centralise les accès VPN distants.

---

## 1. Interfaces Réseau (Assignation)

La machine virtuelle possède plusieurs interfaces virtuelles (vmbr) pour segmenter physiquement les flux réseau.

| Interface | Nom pfSense | Réseau / CIDR | IP de l'interface (Passerelle) | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| **vmbr0** | `WAN` | `<ZONE_BOX>` | `<IP_FIXE_INFRA>` | Patte externe connectée à la Livebox. Reçoit le trafic public naté. |
| **vmbr1** | `LAN_PROD` | `<ZONE_LAN>` | `<IP_FIXE_PVE1>` | Réseau de production interne et de supervision. |
| **vmbr2** | `DMZ` | `<ZONE_DMZ>` | `<GW_DMZ>` | Zone démilitarisée isolée hébergeant le Reverse Proxy. |
| **WG0** | `VPN_NOMADE`| `<ZONE_WG>` | `<IP_FIXE_WG>` | Interface virtuelle du tunnel WireGuard (Administration). |

---

## 2. Gestion des Flux et Redirections de Ports (NAT / Port Forwarding)

Pour permettre la communication entre l'extérieur (WAN) et les services internes tout en garantissant la sécurité du réseau, des règles de redirection (NAT) sont configurées en cascade depuis la box opérateur vers l'interface WAN du pfSense.

Note technique : Les ports `<PORT_WG_NOMADE>` et `<PORT_WG_S2S>` (UDP) dédiés à WireGuard ne font pas l'objet d'une redirection (NAT) vers une autre machine derrière le routeur. Ils sont directement autorisés sur l'interface WAN du pfSense via une règle de pare-feu (Firewall Rule).*

| Port Externe | Protocole | Service cible | Destination Interne (Machine & IP) | Rôle dans l'infrastructure |
| :--- | :--- | :--- | :--- | :--- |
| **80 / 443** | TCP | Serveur Web (HTTP/HTTPS) | `Debian-VM103` (`<IP_FIXE_REVPROXY>`) | Permet l'accès public sécurisé au site web (Reverse Proxy / Nginx). |
| **<PORT_SUPERVISION>** | TCP | API / Supervision | `InfluxDB-VM102` (`<IP_FIXE_INFLUXDB>`) | Permet la collecte et l'interrogation des métriques de supervision. |
| **<PORT_WG_NOMADE>** | UDP | Tunnel VPN (WireGuard) | `pfSense-Front` (VM100) | Permet la connexion distante chiffrée (Accès Nomade). |
| **<PORT_WG_S2S>** | UDP | Tunnel VPN (WireGuard) | `pfSense-Front` (VM100) & `pfSense-Labo` (VM101) | Permet la connexion distante chiffrée (Interco Site-to-Site). |

---

## 3. Routage Statique : La Spécificité en Cascade

En raison de l'isolation du laboratoire (PVE2) derrière un second routeur (`pfSenseLABO-VM101`), une problématique de routage asymétrique a dû être résolue pour permettre le ping inter-sites (LAN-to-LAN vers le réseau distant "<PEER_S2S>" en `<ZONE_FRANCOIS>`).

**Configuration mise en place :**
Une route statique est déclarée sur le **pfSense Front** :
* **Réseau de destination :** `<ZONE_FRANCOIS>`
* **Passerelle (Gateway) :** IP de l'interface WAN du `pfSenseLABO-VM101` (située dans le LAN Prod en `<IP_FIXE_PVE2>`).

Cette route permet au pfSense Front de savoir qu'il ne doit pas renvoyer le trafic destiné à ce réseau vers Internet (Livebox), mais le transférer au routeur du Labo qui gère le tunnel Site-to-Site.

---

## 4. Règles de Pare-feu (Firewall Rules) - Matrice de Flux

La politique de sécurité de l'infrastructure applique strictement le principe du moindre privilège. Le trafic n'est autorisé que s'il correspond à l'une des règles explicites ci-dessous. Le filtrage est appliqué à l'entrée de chaque interface.

---

### 🌐 Onglet WAN (Internet)
Gère le trafic provenant d'Internet et entrant sur le routeur frontal.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | UDP (<PORT_WG_S2S>) | `*` (Any) | WAN Address | Autorise les requêtes externes pour établir le tunnel VPN Site-to-Site (<PEER_S2S>). |
| ✅ Autoriser | UDP (<PORT_WG_NOMADE>) | `*` (Any) | WAN Address | Autorise les requêtes externes pour établir le tunnel VPN Nomade (Julien). |
| ✅ Autoriser | TCP (80) | `*` (Any) | `<IP_FIXE_REVPROXY>` | Redirection (NAT) du trafic web HTTP vers le Nginx Proxy Manager en DMZ. |
| ✅ Autoriser | TCP (443) | `*` (Any) | `<IP_FIXE_REVPROXY>` | Redirection (NAT) du trafic web HTTPS sécurisé vers le Nginx Proxy Manager en DMZ. |

---

### 🏠 Onglet LAN (Réseau Local de Confiance)
Gère le trafic sortant de ton réseau local principal (le LAN de Prod).

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | `*` | `*` | LAN Address (80/443) | **Anti-Lockout Rule** : Règle système pfSense empêchant de se bloquer soi-même l'accès à l'interface d'administration. |
| ✅ Autoriser | IPv4 | `<ZONE_SERVEURS>` | `<ZONE_FRANCOIS>` | Autorise la zone **SERVEURS** du labo à initier des connexions vers le réseau distant de <PEER_S2S> (Routage spécifique lié au S2S). |
| ✅ Autoriser | IPv4 | `<ZONE_CLIENTS>` | `<ZONE_FRANCOIS>` | Autorise la zone **CLIENTS** du labo à initier des connexions vers le réseau distant de <PEER_S2S>. |
| ✅ Autoriser | IPv4 / IPv6 | LAN subnets | `*` (Any) | **Règle par défaut** (Allow LAN to any) : Autorise les machines de ce réseau de confiance à sortir librement vers Internet ou la DMZ. |

---

### 🚧 Onglet DMZ (Zone Démilitarisée)
Gère les services exposés (comme le Reverse Proxy). Cet environnement est considéré comme hostile/vulnérable.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | TCP/UDP | DMZ subnets | DMZ address (53) | Permet aux machines de la DMZ d'interroger le pfSense pour la résolution DNS. |
| ✅ Autoriser | IPv4 | DMZ subnets | `! Reseaux_Prives` | **Isolation de la DMZ** : L'utilisation de l'inversion (`!`) autorise la DMZ à sortir vers Internet, mais **bloque** formellement tout accès vers les réseaux locaux de l'infrastructure (RFC1918). |

---

### 🌍 Onglet WireGuard (Groupe Global)
Cet onglet est un groupe d'interfaces. Les règles ici s'appliquent à tous les tunnels VPN WireGuard confondus avant le filtrage spécifique par tunnel.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | IPv4 | `*` | `*` | Autorise le trafic à traverser le service WireGuard de manière globale. *(Note : La sécurité fine est ensuite déléguée aux onglets spécifiques de chaque tunnel).* |

---

### 👤 Onglet OPT1WIREGUARD (Tunnel Nomade)
Gère le trafic provenant de l'appareil distant connecté en nomade (Julien).

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | IPv4 | OPT1WIREGUARD subnets | `*` (Any) | **Accès Admin Total** : Le profil nomade n'a aucune restriction et peut joindre l'intégralité de l'infrastructure (LAN, Serveurs, DMZ, Internet). |

---

### 🤝 Onglet WG_FRANCOIS (Tunnel Site-to-Site)
Gère le trafic provenant du routeur distant partenaire (Réseau de <PEER_S2S>).

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | IPv4 | `<ZONE_FRANCOIS>` | `<ZONE_SERVEURS>` | Autorise les machines de <PEER_S2S> à accéder à la zone **SERVEURS** de ton Labo. |
| ✅ Autoriser | IPv4 | `<ZONE_FRANCOIS>` | `<ZONE_CLIENTS>` | Autorise les machines de <PEER_S2S> à accéder à la zone **CLIENTS** de ton Labo. *(Tout autre trafic vers ton LAN Prod ou la DMZ est implicitement bloqué).* |

