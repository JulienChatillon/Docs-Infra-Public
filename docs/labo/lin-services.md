# ⚙️ Services d'infrastructure : DNS & DHCP (Debian / dnsmasq)

Contrairement à l'environnement Windows qui centralise un grand nombre de rôles lourds (Annuaire, PKI, Fichiers), le cœur de réseau de ce laboratoire Linux adopte une approche minimaliste et efficace. Une seule machine gère la résolution de noms et l'attribution des adresses IP, fournissant les services réseau vitaux pour simuler une infrastructure open-source fonctionnelle.

* **Machine Virtuelle :** `Lin-Serv-DHCP-VM303`
* **Zone Réseau :** `SERVEURS_2` (<ZONE_SERVEURS_2>)
* **Système d'exploitation :** Debian
* **Service principal :** `dnsmasq`

## 1. Serveur DNS (Domain Name System)
Le service `dnsmasq` agit comme un redirecteur et un serveur DNS local léger pour l'ensemble du laboratoire Linux.
* **Zone locale :** Il gère la résolution de noms pour le domaine expérimental **`sky-linux.lan`**. Cela permet aux différentes machines de se joindre par leur nom d'hôte plutôt que par leur adresse IP.
* **Mise en cache et Forwarding :** Les requêtes externes sont mises en cache pour accélérer la navigation, et celles qu'il ne connaît pas sont transférées vers des résolveurs publics ou le pare-feu en amont.

## 2. Serveur DHCP (Dynamic Host Configuration Protocol)
Toujours via `dnsmasq`, cette machine distribue les configurations réseau dynamiques, principalement à destination des postes de travail de la zone `CLIENTS_2` (`<ZONE_CLIENTS_2>`), comme la machine `TP-Linux-VM300`.
* **Fonctionnement avec Agent Relais :** Ce serveur n'étant pas physiquement raccordé à la patte réseau des clients, il s'appuie sur l'agent relais configuré sur le `Routeur-Linux-VM302`. Le routeur capte les requêtes DHCP Broadcast des clients et les achemine en Unicast directement vers ce serveur Debian.
* **Couplage DNS/DHCP :** L'avantage d'utiliser `dnsmasq` est que chaque machine qui reçoit un bail DHCP voit automatiquement son nom enregistré dans le DNS local (`sky-linux.lan`), rendant la gestion de l'infrastructure fluide et sans intervention manuelle.
