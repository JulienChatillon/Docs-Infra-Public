# 🌍 Hébergement Web & DMZ (Reverse Proxy)

La zone DMZ (Demilitarized Zone) sur le réseau `<ZONE_DMZ>` a pour objectif d'héberger les services exposés publiquement. Elle est strictement isolée des réseaux internes (Production et Labo) pour éviter toute compromission latérale en cas d'attaque.

Le point d'entrée unique pour le trafic web est géré par un **Reverse Proxy**.

## 1. Composants de l'infrastructure Web
* **Hôte :** Machine Virtuelle Debian (`VM103` sur PVE1).
* **Moteur :** Docker.
* **Service Principal :** Nginx Proxy Manager (NPM).
* **IP DMZ :** `<IP_FIXE_REVPROXY>`.
* **Services hébergés :** * Site web statique principal (`julien-chatillon.com`).
  * Les fichiers HTML sont montés directement via un volume Docker depuis l'hôte (`~/portfolio/html/`).

## 2. Cycle de vie d'une requête Web (Flux de trafic)

Lorsqu'un visiteur distant souhaite accéder au site web, la requête suit ce cheminement :

1. **Internet (DNS) :** Le visiteur saisit le nom de domaine. La requête arrive sur l'IP publique de la box internet (Livebox).
2. **Passerelle FAI (Livebox) :** Une règle de redirection (Port Forwarding) transfère le trafic des ports HTTP (80) et HTTPS (443) vers l'interface WAN du pare-feu pfSense.
3. **Pare-feu (pfSense Principal) :** Une règle de NAT (Network Address Translation) intercepte les ports 80/443 et les redirige vers l'adresse IP de la machine NPM dans la DMZ (`<IP_FIXE_REVPROXY>`).
4. **Reverse Proxy (NPM) :** Nginx Proxy Manager réceptionne la requête, analyse le nom de domaine demandé (`julien-chatillon.com`), gère le certificat SSL pour sécuriser la connexion, et sert directement les fichiers statiques HTML correspondants.
