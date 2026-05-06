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

## 2. Interconnexion Site-to-Site (Partenaire distant)
Ce tunnel établit un pont réseau permanent avec le réseau local d'un partenaire (<PEER_S2S>). Il est configuré selon une approche "Zero Trust" avec un cloisonnement très strict.

* **Port d'écoute :** UDP <PORT_WG_S2S>
* **Réseau de Transit VPN :** `<ZONE_TRANSIT_WG>`
    * Passerelle Locale (PVE1) : `<IP_FIXE_LOCALE_S2S>`
    * Passerelle Distante (Partenaire) : `<IP_FIXE_DISTANT_S2S>`
* **Endpoint Distant :** `<IP_FIXE_PEER_S2S>`
* **Réseau Distant Routé :** `<ZONE_FRANCOIS>` (LAN <PEER_S2S>)
* **Sécurité :** Les règles appliquées sur l'interface dédiée `WG_FRANCOIS` limitent le trafic entrant. Le partenaire est **exclusivement** autorisé à communiquer avec la zone `SERVEURS_LABO` (`<ZONE_SERVEURS>`) du nœud PVE2. Tout accès à la production, aux clients ou à la DMZ est bloqué.

## 🔀 Spécificité de Routage (Cascade & No-NAT)
Afin de maintenir une connectivité de bout en bout (ping LAN à LAN) entre le réseau du partenaire et le Labo isolé, le routage s'appuie sur une configuration spécifique :
Une règle **Outbound "No-NAT"** est configurée sur le pfSense du Labo (PVE2). Ainsi, lorsque les serveurs du labo répondent au partenaire, leurs adresses IP sources d'origine (`<RESEAU_SERVEURS_LABO>.x`) sont conservées. Le pfSense Frontal (PVE1) peut alors identifier la source légitime, appliquer les règles de pare-feu correspondantes et router correctement les paquets de retour dans le tunnel Site-to-Site.
