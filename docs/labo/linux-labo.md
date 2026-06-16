# 🐧 4. Environnement de lab LINUX

## 🛡️ Routage interne et Pare-feu : Routeur Linux

Le routage de cet environnement expérimental parallèle sur le PVE2 est assuré par une machine virtuelle Linux dédiée : **Routeur-Linux-VM302**. 
Son rôle est de segmenter les zones serveurs et clients de l'écosystème Linux, de gérer la translation d'adresses (NAT) via `nftables` et d'assurer le relais des requêtes DHCP vers le serveur d'infrastructure.

### 1. Interfaces Réseau (Assignation)

| Interface | Nom logique | Réseau / CIDR | IP de l'interface (Passerelle) | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| **Interface 1** | `WAN` | `<ZONE_LAN>` | `<IP_FIXE_ROUTEUR_LINUX>` | Patte de sortie connectée au réseau de production/front. Reçoit le trafic naté de la zone Linux. |
| **vmbr10** | `SERVEURS_2` | `<ZONE_SERVEURS_2>` | `<GW_SERVEURS_2>` | Zone névralgique hébergeant l'infrastructure Linux (Serveur DHCP/DNS via dnsmasq). |
| **vmbr11** | `CLIENTS_2`| `<ZONE_CLIENTS_2>` | `<GW_CLIENTS_2>` | Zone utilisateur simulant un parc informatique client (Postes de TP comme TP-Linux-VM300). |

### 2. Stratégie de Routage et NAT

Contrairement à l'interface graphique de pfSense, le routage et le NAT sont ici gérés nativement par le noyau Linux et le pare-feu **nftables**.
* **Trafic standard (Internet) :** Une règle de *Masquerading* (NAT) est appliquée en sortie sur l'interface WAN. Le routeur masque les IP privées des zones `SERVEURS_2` et `CLIENTS_2` derrière sa propre adresse IP WAN (`<IP_FIXE_ROUTEUR_LINUX>`) pour leur permettre d'accéder à internet ou au reste du réseau via le routeur de bordure principal.

### 3. Gestion DHCP et DNS (Agent Relais et dnsmasq)

Pour conserver une architecture centralisée et modulaire au sein de l'environnement Linux, la distribution des IP et la résolution de noms sont déportées :
1. **Agent Relais (DHCP Relay) :** Le routeur `Routeur-Linux-VM302` exécute un agent relais. Il écoute les requêtes de diffusion (broadcast) provenant des postes de la Zone Clients (`vmbr11`) et les relaie en unicast (transfert direct) au serveur d'infrastructure situé dans la Zone Serveurs.
2. **Serveur DHCP & DNS :** La machine **Lin-Serv-DHCP-VM303** assure ce rôle grâce au service `dnsmasq`. Elle distribue les baux IP et gère la résolution de noms pour le domaine local expérimental **`sky-linux.lan`**.

### 4. Règles de Pare-feu (Firewall Rules) - Matrice de Flux

La politique de sécurité appliquée dans le fichier de configuration `nftables` suit le principe du moindre privilège (**Default Deny** sur la chaîne *FORWARD*). Les réseaux Linux sont isolés et seules les communications strictement nécessaires sont autorisées par des règles explicites.

---

#### 🌐 Matrice de filtrage inter-VLAN et WAN
Le pare-feu inspecte et filtre le trafic transitant à travers le routeur entre les différents réseaux.

| Action | Protocole | Source | Destination | Explication du flux |
| :--- | :--- | :--- | :--- | :--- |
| ✅ Autoriser | `IPv4` | `<ZONE_SERVEURS_2>` & `<ZONE_CLIENTS_2>` | `WAN / Internet` | Laisse sortir le trafic des zones internes vers l'extérieur (couplé au *Masquerading* / suivi de connexion ESTABLISHED). |
| ✅ Autoriser | `UDP` | `vmbr11` (Clients) | `Lin-Serv-DHCP-VM303:67` | Autorise le transfert des trames DHCP par l'agent relais vers le serveur dnsmasq. |
| ✅ Autoriser | `TCP/UDP` | `vmbr11` (Clients) | `Lin-Serv-DHCP-VM303:53` | Autorise les requêtes DNS des clients vers le serveur d'infrastructure. |
| ❌ Bloquer | `IPv4` | `WAN` | `<A DEFINIR>` | Par défaut, rejette toute tentative de connexion initiée depuis l'extérieur vers le labo Linux. |
