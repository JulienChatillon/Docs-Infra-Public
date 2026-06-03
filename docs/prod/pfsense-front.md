# 🛡️ Routage et Pare-feu : pfSense Front (VM100)

Le routeur virtuel **pfSense Front** est le point d'entrée principal de l'infrastructure. Situé immédiatement derrière la box de l'opérateur (Livebox), il agit comme pare-feu frontal, gère la DMZ exposant les services web, et centralise les accès VPN distants.

---

## 1. Interfaces Réseau (Assignation)

La machine virtuelle possède plusieurs interfaces virtuelles (vmbr) pour segmenter physiquement les flux réseau.

| Interface | Nom pfSense | Réseau / CIDR | IP de l'interface (Passerelle) | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| **vmbr0** | `WAN` | `<ZONE_BOX>` | `<IP_FIXE_INFRA>` | Patte externe connectée à la Livebox. Reçoit le trafic public naté. |
| **vmbr1** | `LAN_PROD` | `<RESEAU_PVE1>.0/24` | `<IP_FIXE_PFSENSE>` | Réseau de production interne et de supervision. |
| **vmbr2** | `DMZ` | `10.10.19.0/24` | `10.10.19.254` | Zone démilitarisée isolée hébergeant le Reverse Proxy. |
| **vmbr3** | `DMZ_SUPERV` | `10.10.18.0/24` | `10.10.18.254` | Zone démilitarisée isolée hébergeant la supervision. |
| **WG0** | `VPN_NOMADE`| `<ZONE_WG>` | `<IP_FIXE_WG>` | Interface virtuelle du tunnel WireGuard (Administration). |

---

## 2. Gestion des Flux (NAT Entrant et Sortant)

Pour permettre la communication bidirectionnelle entre l'extérieur (WAN) et les services internes tout en garantissant la sécurité du réseau, des règles de traduction d'adresses (NAT) sont configurées sur l'interface WAN du pfSense.

### 2.1 NAT Entrant (Port Forwarding)

Ces règles gèrent les connexions initiées depuis l'extérieur vers les services internes, configurées en cascade depuis la box opérateur.

*Note technique : Les ports `<PORT_WG_NOMADE>` et `<PORT_WG_SIO>` (UDP) dédiés à WireGuard ne font pas l'objet d'une redirection (NAT) vers une autre machine derrière le routeur. Ils sont directement autorisés sur l'interface WAN du pfSense via une règle de pare-feu (Firewall Rule).*

| Port Externe | Protocole | Service cible | Destination Interne (Machine & IP) | Rôle dans l'infrastructure |
| :--- | :--- | :--- | :--- | :--- |
| **80 / 443** | TCP | Serveur Web (HTTP/HTTPS) | `Debian-VM103` (`10.10.19.251`) | Permet l'accès public sécurisé au site web (Reverse Proxy / Nginx). |
| **<PORT_SUPERVISION>** | TCP | Interface Supervision | `SuperV-VM102` (`10.10.18.253`) | Permet l'accès sécurisé (idéalement via VPN) au tableau de bord Grafana. |
| **<PORT_WG_NOMADE>** | UDP | Tunnel VPN (WireGuard) | `pfSense-Front` (VM100) | Permet la connexion distante chiffrée (Accès Nomade). |
| **<PORT_WG_SIO>** | UDP | Tunnel VPN (WireGuard) | `pfSense-Front` (VM100) & `pfSense-Labo` (VM101) | Permet la connexion distante chiffrée (Interco Site-to-Site). |

### 2.2 NAT Sortant (Outbound ou Masquerading)

Le routeur masque les adresses IP privées (RFC 1918) des machines internes en les remplaçant par sa propre adresse publique (WAN) lorsqu'elles tentent d'accéder à Internet. Sans cette traduction, le trafic sortant serait bloqué et détruit par la box opérateur. 

Le pfSense est configuré en mode **Hybrid Outbound NAT**, ce qui permet de conserver les règles automatiques tout en appliquant les règles manuelles suivantes pour nos différents sous-réseaux :

| Interface de sortie | Réseau Source | Adresse de traduction (NAT) | Description / Rôle |
| :--- | :--- | :--- | :--- |
| **WAN** | `DMZ_SUPERV subnets` | WAN address | NAT pour accès internet DMZ_SUPERV |
| **WAN** | `DMZ subnets` | WAN address | NAT pour accès internet DMZ |
| **WAN** | `<ZONE_WG>` | WAN address | NAT pour accès Proxmox depuis WireGuard (Tunnel Nomade) |
| **WAN** | `LAN subnets` | WAN address | NAT internet (Réseau de Prod principal) |
| **WAN** | `<ZONE_SERVEURS>` | WAN address | NAT pour accès internet LABO (Zone Serveurs) |

---

## 3. Règles de Pare-feu (Firewall Rules) - Matrice de Flux

La politique de sécurité de l'infrastructure applique strictement le principe du moindre privilège. Le trafic n'est autorisé que s'il correspond à l'une des règles explicites ci-dessous. Le filtrage est appliqué à l'entrée de chaque interface.

---

### 🌐 Onglet WAN (Internet)
Gère le trafic provenant d'Internet et entrant sur le routeur frontal.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | UDP (<PORT_WG_SIO>) | `*` (Any) | WAN Address | Autorise les requêtes externes pour établir le tunnel VPN Site-to-Site (SIO). |
| ✅ Autoriser | UDP (<PORT_WG_NOMADE>) | `*` (Any) | WAN Address | Autorise les requêtes externes pour établir le tunnel VPN Nomade (Julien). |
| ✅ Autoriser | TCP (80) | `*` (Any) | `10.10.19.251` | Redirection (NAT) du trafic web HTTP vers le Nginx Proxy Manager en DMZ. |
| ✅ Autoriser | TCP (443) | `*` (Any) | `10.10.19.251` | Redirection (NAT) du trafic web HTTPS sécurisé vers le Nginx Proxy Manager en DMZ. |

---

### 🏠 Onglet LAN (Réseau Local de Confiance)
Gère le trafic sortant de ton réseau local principal (le LAN de Prod).

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | `*` | `*` | LAN Address (80/443) | **Anti-Lockout Rule** : Règle système pfSense empêchant de se bloquer soi-même l'accès à l'interface d'administration. |
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

### 🤝 Onglet WG_SIO (Tunnel Hub & Spoke)
Gère le trafic provenant du routeur distant partenaire (Réseau SIO).

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ❌ Bloquer | IPv4 | `WG_SIO subnets` | `any` | Intérupteur pour couper la liaison |
| ❌ Bloquer | IPv4 | `WG_SIO subnets` | `<RESEAU_PVE1>.0/24` | Interdit l'accès au réseau de PRODUCTION (PVE1). |
| ❌ Bloquer | IPv4 | `WG_SIO subnets` | `10.10.19.0/24` | Interdit l'accès à la zone DMZ (Portfolio). |
| ✅ Autoriser | IPv4 | `WG_SIO subnets` | `<ZONE_SERVEURS>` | Autorise les machines des camarades à accéder à la zone SERVEURS du Labo (PVE2). |
| ✅ Autoriser | IPv4 | `WG_SIO subnets` | `<ZONE_CLIENTS>` | Autorise les machines des camarades à accéder à la zone CLIENTS du Labo (PVE2). |
| ✅ Autoriser | IPv4 | `* (Tout)` | `WG_SIO subnets` | Autorise les camarades connectés au VPN à communiquer entre eux. |

La règle de l'interupteur est désactivée et est la en cas de problème pour couper tout traffic entrant via WG_SIO

---

### 👁️ Onglet DMZ_SUPERV (Zone Supervision)
Gère les flux de la Tour de Contrôle (Prometheus/Grafana). Cet environnement est isolé de la production et du LAN pour garantir l'étanchéité, avec des ouvertures chirurgicales strictes pour la collecte de métriques.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | TCP | `DMZ_SUPERV subnets` | `<IP_FIXE_PVE1>` (Port 9100) | Permet à Prometheus de collecter les métriques (Node Exporter) de l'hyperviseur physique pve1. |
| ✅ Autoriser | TCP/UDP | `DMZ_SUPERV subnets` | `DMZ_SUPERV address` (Port 53) | Permet aux machines de la zone d'interroger le pfSense pour la résolution DNS. |
| ❌ Bloquer | IPv4 | `DMZ_SUPERV subnets` | `LAN subnets` | Interdit formellement l'accès au réseau local de l'infrastructure (Sécurité). |
| ❌ Bloquer | IPv4 | `DMZ_SUPERV subnets` | `DMZ subnets` | Interdit l'accès à la DMZ Web publique (Portfolio) pour éviter les rebonds. |
| ✅ Autoriser | IPv4 | `DMZ_SUPERV subnets` | `*` (Any) | Autorise la VM à sortir vers Internet pour les mises à jour (APT) et le téléchargement d'images Docker. |
