# 🛡️ Routage interne et Pare-feu : pfSense Labo

Le cœur du réseau de l'environnement de laboratoire (PVE2) est géré par une machine virtuelle dédiée : **pfSenseLABO-VM101**. 
Son rôle est d'isoler complètement les expérimentations du réseau de production tout en assurant le routage inter-VLAN, la sécurité interne et la distribution IP.

## 1. Interfaces Réseau (Assignation)

| Interface | Nom pfSense | Réseau / CIDR | IP de l'interface (Passerelle) | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| **vmbr0** | `WAN` | `<ZONE_BOX>` | `<IP_FIXE_INFRA>` | Patte externe connectée à la Livebox. Reçoit le trafic public naté. |
| **vmbr4** | `SERVEURS` | `<ZONE_SERVEURS>` | `<GW_SERVEURS>` | Zone névralgique hébergeant l'infrastructure de base (AD, DNS, Monitoring Wazuh, GLPI). |
| **vmbr5** | `CLIENTS` | `<ZONE_CLIENTS>` | `<GW_CLIENTS>` | Réseau utilisateur simulant un parc informatique (Postes Windows 11, machines Linux). |
| **vmbr6** | `DMZ_LABO`| `<ZONE_DMZ_LABO>` | `<GW_DMZ_LABO>` | Zone isolée en attente pour l'exposition de futurs services de test. |

## 2. Stratégie de Routage et NAT
Pour permettre une communication bidirectionnelle avec le réseau distant du partenaire (Site-to-Site) tout en restant derrière le pfSense de production, une règle **Outbound "No-NAT"** est configurée sur le WAN du pfSense Labo. 
* **Trafic standard (Internet) :** NAT classique. Le pfSense masque les IP du labo derrière son IP WAN (`<IP_FIXE_PVE2>`).

## 3. Gestion DHCP (Relais et Redondance)
Afin de centraliser la gestion du parc informatique du labo au sein du contrôleur de domaine, le service DHCP n'est pas assuré par défaut par le pare-feu :
1. **DHCP Relay :** Le pfSense est configuré en relais DHCP. Il intercepte les requêtes des machines clientes (Zone Clients) et les transfère au serveur Windows Server (`WinServ2025-VM200`) situé dans la Zone Serveurs.
2. **Secours (PFSense) :** En cas d'extinction de la VM200 (maintenance ou économie de ressources), le serveur DHCP intégré au pfSense peut être activé manuellement pour reprendre le relais et maintenir la connectivité du labo.
3. **Secours (Windows) :** En cas de problèmes ou d'extinction de la VM200, le serveur Windows de secours (VM202) prend la main sur la gestion DHCP et DNS.

## 4. Règles de Pare-feu (Firewall Rules) - Matrice de Flux

Ce routeur segmente les différentes zones du laboratoire. La politique de sécurité applique le principe du moindre privilège (**Default Deny**). Le réseau Labo est par défaut totalement étanche vis-à-vis du réseau de production principal sur son interface WAN.

---

### 🌐 Onglet WAN (Lien vers LAN Prod)
Le port WAN de ce pfSense est connecté au réseau de production. Par défaut, tout trafic entrant est bloqué pour isoler le labo, à l'exception de règles spécifiques d'administration ou de routage.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | IPv4 | `WG_Classe` | `Zone_Labo` | Laisse entrer le trafic de l'alias "WG_Classe" vers les réseaux de l'alias "Zone_Labo". |
| ✅ Autoriser | IPv4 | `Admins_Reseau` | `*` (Any) | **Exception WAN** : Autorise l'accès total au labo, conditionné par l'appartenance à l'alias `Admins_Reseau` (utilisé par le VPN Nomade). |

---

### 🗄️ Onglet SERVEURS (Cœur de réseau - <ZONE_SERVEURS>)
Gère le trafic initié par tes serveurs d'infrastructure (AD, DNS, DHCP, GLPI...).

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | `*` | `*` | SERVEURS Address (22, 80, 443) | **Anti-Lockout Rule** : Empêche la perte d'accès à l'interface de gestion du pfSense depuis le réseau des serveurs. |
| ❌ Bloquer | IPv4 | SERVEURS subnets | DMZ_LABO subnets | Empêche un serveur compromis d'initier une attaque latérale vers la zone DMZ. |
| ❌ Bloquer | IPv4 | SERVEURS subnets | CLIENTS subnets | Empêche un serveur compromis d'initier une attaque latérale vers le parc informatique (Clients). |
| ✅ Autoriser | IPv4 / IPv6 | SERVEURS subnets | `*` (Any) | **Accès sortant** : Permet aux serveurs d'accéder à Internet (indispensable pour les mises à jour Windows, GLPI, Wazuh...). |

---

### 💻 Onglet CLIENTS (Postes de travail - <ZONE_CLIENTS>)
Gère le trafic initié par les machines utilisateurs (Windows 11, Linux Mint...).

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✋ Rejeter | TCP | CLIENTS subnets | `IP_ADMIN_ONLY` (443) | Interdit l'accès à l'interface d'administration web sécurisée (HTTPS) du pfSense depuis un poste client. |
| ✋ Rejeter | TCP | CLIENTS subnets | `IP_ADMIN_ONLY` (80) | Interdit l'accès à l'interface d'administration web (HTTP) du pfSense depuis un poste client. |
| ✅ Autoriser | IPv4 | CLIENTS subnets | SERVEURS subnets | Permet aux postes clients de consommer les services internes (Authentification AD, requêtes DNS, etc.). |
| ❌ Bloquer | IPv4 | CLIENTS subnets | DMZ_LABO subnets | Empêche les clients d'interagir directement avec les futurs services exposés de la DMZ Labo. |
| ✅ Autoriser | IPv4 | CLIENTS subnets | `*` (Any) | **Accès sortant** : Autorise la navigation standard vers Internet pour les machines clientes. |

---

### 🚧 Onglet DMZ_LABO (Zone exposée - <ZONE_DMZ_LABO>)
Gère le trafic des futurs services accessibles de l'extérieur. Cette zone est traitée comme non fiable.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | TCP/UDP | DMZ_LABO subnets | `<IP_FIXE_SRV25>` (53) | Autorise les machines de la DMZ à effectuer des requêtes de résolution DNS vers le serveur désigné (`<IP_FIXE_SRV25>`). |
| ❌ Bloquer | IPv4 | DMZ_LABO subnets | CLIENTS subnets | **Isolation stricte** : Bloque tout accès de la DMZ vers le réseau des utilisateurs. |
| ❌ Bloquer | IPv4 | DMZ_LABO subnets | SERVEURS subnets | **Isolation stricte** : Bloque tout accès de la DMZ vers le cœur de réseau (à l'exception du DNS au-dessus). |
| ✅ Autoriser | IPv4 | DMZ_LABO subnets | `*` (Any) | **Accès sortant** : Autorise l'accès à Internet pour la DMZ (mises à jour, requêtes sortantes des services). |
