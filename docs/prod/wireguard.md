# 🔗 Interconnexion WireGuard (Nomade & Site-to-Site)

Le routeur pfSense Frontal centralise toutes les connexions VPN entrantes via **WireGuard**. L'infrastructure exploite deux typologies de tunnels distinctes pour répondre à des besoins d'administration et de collaboration, tout en maintenant une ségrégation stricte des flux.

## 1. Accès Nomade (Administration Distante)
Ce tunnel est dédié à l'administration sécurisée de l'infrastructure. Il agit comme un accès privilégié contournant les restrictions de l'interface WAN publique.

* **Port d'écoute :** UDP 51820
* **Réseau d'interconnexion (Tunnel) :** `10.99.10.0/24`
* **Peers autorisés :**
    * PC Linux Perso (`10.99.10.2`)
    * PC Tour Windows (`10.99.10.3`)
    * PC Batteur (`10.99.10.4`)
* **Sécurité :** Les flux provenant de ce tunnel sont gérés via l'interface `OPT1WIREGUARD`. Ils bénéficient d'un accès total à l'ensemble des réseaux (LAN, DMZ, Labo) pour garantir l'administration à distance.

## 2. Réseau VPN Collaboratif Hub & Spoke (Interconnexion multi-sites)
Cette architecture remplace l'ancien tunnel Site-to-Site point-à-point. Le routeur frontal (pfSense-VM100) agit désormais comme un concentrateur central (Hub) permettant de relier les infrastructures de plusieurs camarades (Spokes). L'approche "Zero Trust" est maintenue avec un cloisonnement strict des flux.

* **Port d'écoute :** UDP 51821
* **Réseau de Transit VPN :** `10.250.0.0/24` (Sous-réseau élargi pour permettre jusqu'à 253 pairs)
    * Concentrateur Local (pfSense Frontal PVE1) : `10.250.0.1`
    * Pairs Distants : `10.250.0.2` (François), `10.250.0.3` (Léo), `10.250.0.4` (Marley), `10.250.0.10` (Batteuse), etc.
* **Endpoint Distant :** Dynamique (Les pairs distants initient la connexion vers le Hub central)
* **Réseaux Distants Routés (via Allowed IPs) :**
 - Labo François : `10.20.0.0/16` (Zone Serveurs) et `10.21.0.0/16` (Zone Clients)
 - Labo Batteuse : `10.0.0.0/16` à `10.9.0.0/16`
    * *Règle d'architecture : Chaque pair doit déclarer des sous-réseaux uniques pour éviter tout conflit de routage (Overlapping IP).*
* **Sécurité :** Les règles appliquées de haut en bas sur l'interface dédiée `WG_SIO` limitent strictement le trafic entrant :
    * 🔴 **Bloqué :** Accès au réseau de Production (`<ZONE_LAN_WG>`) et à la DMZ hébergeant les services exposés (`<ZONE_DMZ>`).
    * 🟢 **Autorisé :** Communication inter-VPN (`10.250.0.0/24`) pour permettre aux camarades d'interagir entre eux.
    * 🟢 **Autorisé :** Accès exclusif aux zones de test du nœud PVE2 (`<ZONE_SERVEURS>` pour les serveurs et `<ZONE_CLIENTS>` pour les clients).
