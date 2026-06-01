# 🔗 Interconnexion WireGuard (Nomade & Site-to-Site)

Le routeur pfSense Frontal centralise toutes les connexions VPN entrantes via **WireGuard**. L'infrastructure exploite deux typologies de tunnels distinctes pour répondre à des besoins d'administration et de collaboration, tout en maintenant une ségrégation stricte des flux.

## 1. Accès Nomade (Administration Distante)
Ce tunnel est dédié à l'administration sécurisée de l'infrastructure. Il agit comme un accès privilégié contournant les restrictions de l'interface WAN publique.

* **Port d'écoute :** UDP <PORT_WG_NOMADE>
* **Réseau d'interconnexion (Tunnel) :** `<ZONE_WG>`
* **Peers autorisés :**
    * PC Linux Perso (`<IP_FIXE_LINUX>`)
    * PC Tour Windows (`<IP_FIXE_TOUR>`)
    * PC Batteur (`<IP_FIXE_BATTEUR>`)
* **Sécurité :** Les flux provenant de ce tunnel sont gérés via l'interface `OPT1WIREGUARD`. Ils bénéficient d'un accès total à l'ensemble des réseaux (LAN, DMZ, Labo) pour garantir l'administration à distance.

## 2. Réseau VPN Collaboratif Hub & Spoke (Interconnexion multi-sites)
Cette architecture remplace l'ancien tunnel Site-to-Site point-à-point. Le routeur frontal (pfSense-VM100) agit désormais comme un concentrateur central (Hub) permettant de relier les infrastructures de plusieurs camarades (Spokes). L'approche "Zero Trust" est maintenue avec un cloisonnement strict des flux.

* **Port d'écoute :** UDP <PORT_WG_SIO>
* **Réseau de Transit VPN :** `<ZONE_TRANSIT_WG>` (Sous-réseau élargi pour permettre jusqu'à 253 pairs)
    * Concentrateur Local (pfSense Frontal PVE1) : `<IP_FIXE_LOCALE_S2S>`
    * Pairs Distants : `<IP_FIXE_FRANCOIS_S2S>` (<PEER_S2S>), `<RESEAU_WG_SIO>.3`, etc.
* **Endpoint Distant :** Dynamique (Les pairs distants initient la connexion vers le Hub central)
* **Réseaux Distants Routés (via Allowed IPs) :** * Labo <PEER_S2S> : `<ZONE_FRANCOIS>` (Zone Serveurs) et `172.21.0.0/24` (Zone Clients)
    * *Règle d'architecture : Chaque pair doit déclarer des sous-réseaux uniques pour éviter tout conflit de routage (Overlapping IP).*
* **Sécurité :** Les règles appliquées de haut en bas sur l'interface dédiée `WG_SIO` limitent strictement le trafic entrant :
    * 🔴 **Bloqué :** Accès au réseau de Production (`<ZONE_LAN>`) et à la DMZ hébergeant les services exposés (`<ZONE_DMZ>`).
    * 🟢 **Autorisé :** Communication inter-VPN (`<ZONE_TRANSIT_WG>`) pour permettre aux camarades d'interagir entre eux.
    * 🟢 **Autorisé :** Accès exclusif aux zones de test du nœud PVE2 (`<ZONE_SERVEURS>` pour les serveurs et `<ZONE_CLIENTS>` pour les clients).
