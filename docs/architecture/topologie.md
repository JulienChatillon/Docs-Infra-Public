# 🗺️ Topologie Réseau & Proxmox

Cette page détaille la configuration logique des réseaux, la répartition des interfaces virtuelles (vmbr) sur l'hyperviseur Proxmox, ainsi que le plan d'adressage IP des deux environnements.

---

## 1. Nœud de Production & Front (PVE1 - `<IP_FIXE_INFRA>`)

Le premier nœud gère l'entrée du trafic depuis le routeur de l'opérateur (Livebox) et héberge les services exposés.

### Segmentation par Ponts Virtuels (Bridges)
Pour garantir l'étanchéité des zones, des interfaces virtuelles (vmbr) distinctes ont été configurées sur Proxmox :

* **`vmbr1` (Zone LAN) - `<ZONE_LAN>` :** Cœur du système d'information de PVE1.
* **`vmbr2` (Zone DMZ) - `<ZONE_DMZ>` :** Réseau contenant la VM Reverse Proxy pour hebergement du Portfolio.
* **`vmbr3` (Zone DMZ_SUPERV) - `<ZONE_DMZ_SUPERV>` :** Réseau contenant la VM SuperV pour Grafana et Supervision infra.

### Inventaire des Machines Virtuelles
| ID VM | Nom d'hôte | Rôle & Services | Interface | IP Statique / DHCP |
| :--- | :--- | :--- | :--- | :--- |
| **100** | `pfSense-Front` | Routeur frontal, pare-feu principal, point d'entrée WAN | Toutes | N/A |
| **102** | `SuperV-VM102` | Base de données temporelle pour le monitoring de l'infra | `vmbr3` | <IP_FIXE_INFLUXDB> |
| **103** | `Debian-VM103` | Reverse Proxy hébergeant `julien-chatillon.com` | `vmbr2` | <IP_FIXE_REVPROXY> |

---

## 2. Nœud de Laboratoire (PVE2 - `<IP_FIXE_PVE2>`)

Le PVE2 est un environnement de laboratoire strictement isolé de la production. Le routage interne est assuré de manière autonome par une instance pfSense dédiée.

### Segmentation par Ponts Virtuels (Bridges)
Idem PVE1, des interfaces virtuelles (vmbr) distinctes ont été configurées sur Proxmox :

* **`vmbr4` (Zone Serveurs) - `<ZONE_SERVEURS>` :** Cœur du système d'information du labo.
* **`vmbr5` (Zone Clients) - `<ZONE_CLIENTS>` :** Postes de travail et environnements utilisateurs.
* **`vmbr6` (Zone DMZ_LABO) - `<ZONE_DMZ_LABO>` :** Réseau en attente pour de futurs services exposés spécifiques au labo.

### Inventaire des Machines Virtuelles
| ID VM | Nom d'hôte | Rôle & Services | Interface | IP Statique / DHCP |
| :--- | :--- | :--- | :--- | :--- |
| **101** | `pfSenseLABO-VM101` | Routeur interne du labo, filtrage inter-VLANs | Toutes | N/A |
| **104** | `GLPI-VM104` | Inventaire de parc et synchronisation LDAP | `vmbr4` | <IP_FIXE_GLPI> |
| **105** | `BD-Python-VM105` | Serveur de base de données / scripts | `vmbr4` | DHCP |
| **200** | `WinServ25-VM200` | Contrôleur de Domaine (AD), DNS, DHCP principal | `vmbr4` | <IP_FIXE_SRV25> |
| **201** | `Win11-Client01-VM201`| Poste client Windows intégré au domaine | `vmbr5` | DHCP |
| **202** | `WinServSec-VM202` | Contrôleur de Domaine de secours (AD), DNS, DHCP | `vmbr4` | <IP_FIXE_SRV25SEC> |
| **300** | `TP-Linux-VM300` | Poste client / Machine de test Linux | `vmbr5` | DHCP |

---
