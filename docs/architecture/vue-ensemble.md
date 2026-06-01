# 🌐 Vue d'ensemble de l'infrastructure

## 1. Contexte du projet
Cette infrastructure a été développée dans le cadre d'une reconversion en **BTS SIO (option SISR)**. Elle répond à un besoin double :
* **Production & Services personnels :** Un environnement stable pour l'hébergement de services (site web, monitoring).
* **Laboratoire d'expérimentation (Sandbox) :** Un environnement isolé pour simuler des infrastructures d'entreprise, tester des déploiements Active Directory, et s'entraîner aux problématiques de sécurité.

---

## 2. Architecture Matérielle (Hardware)
L'infrastructure repose sur deux environnements distincts permettant de séparer les flux critiques du bac à sable :

### A. Mes equipements clients :

| Composant | Modèle / Spécifications | Système d'exploitation |
| :--- | :--- | :--- |
| **PC Portable** | Lenovo IDEAPAD Slim 3 | Linux Mint |
| **PC Fixe** | PC Gamer | Windows 11 |

### B. Mes equipements infra :

| Composant | Modèle / Spécifications | Système d'exploitation |
| :--- | :--- | :--- |
| **PC PVE1** | HP i5 - 16G RAM | Proxmox VE |
| **PC PVE2** | Lenovo i5 - 32G RAM | Proxmox VE |

Pve2 est directement branché sur Pve1 via cable RJ45

---

## 3. Architecture Logique (Hyperviseurs)

L'ensemble de l'infrastructure est virtualisé sous **Proxmox VE**, réparti sur deux nœuds logiques pour isoler les environnements :

### A. Nœud de Production (PVE1)
Ce nœud est exposé à Internet (via une redirection Livebox) et gère les services accessibles de l'extérieur.
* **Réseau :** LAN <ZONE_LAN>.
* **Services clés :** pfSense (Frontal), Reverse Proxy (Nginx Proxy Manager), Supervision (InfluxDB).
* **Domaine public :** `julien-chatillon.com` (Hébergement statique).

### B. Nœud Labo Isolé (PVE2)
Ce nœud est strictement dédié aux travaux pratiques et aux simulations système. Il est isolé de la production pour éviter toute corruption des services critiques.
* **Réseau :** Zones étanches (Serveurs <ZONE_SERVEURS>, Clients <ZONE_CLIENTS> et DMZ_LABO <ZONE_DMZ_LABO>).
* **Services clés :** Windows Server 2025 (AD/DNS/DHCP), GLPI (Inventaire).

---

![Topologie de mon infrastructure](fossflow-export.svg)

---

## 4. Connectivité & Accès Distants
La connection exterieur se fait sur la Livebox via l'IP : <IP_FIXE_BOX>
L'infrastructure dispose ensuite d'une addresse LAN fixe : <IP_FIXE_INFRA>

La sécurité des accès extérieurs est assurée par des tunnels **WireGuard** :
1.  **Accès Nomade :** VPN pour l'administration distante via l'IP <IP_FIXE_WG>.
2.  **Hub & Spoke :** Hebergement d'une connexion sécurisée entre mon laboratoire et plusieurs infras distantes partenaires (SIO) via un transit dédié (<ZONE_TRANSIT_WG>).

---

## 5. Objectifs de cette documentation
Ce Wiki a pour but de recenser :
- Les procédures d'installation "from scratch".
- Les schémas d'adressage IP.
- Les configurations spécifiques (NAT, GPO, filtrage).
- Le journal des modifications techniques.

---

## 6. Zone de Test

Ce premier paragraphe est une introduction standard. Il sera parfaitement visible sur le dépôt public contrairement au 2eme paragraphe qui sera privé.



Ce 3eme et dernier paragraphe se trouve après les balises de masquage. Il sera donc conservé et affiché normalement à la suite de l'introduction sur la version publique, comme si la section privée n'avait jamais existé !

