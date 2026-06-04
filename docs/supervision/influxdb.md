# 👁️ Supervision, Sécurité et Métriques (Stack PLG & IDS)

La surveillance proactive des performances matérielles (CPU, RAM, Températures, Disques) et réseau de l'ensemble de l'infrastructure est centralisée sur une "Tour de Contrôle" dédiée. Cette architecture de monitoring a migré vers un modèle moderne de type "Pull", garantissant une visibilité totale tout en respectant une isolation stricte des environnements.

* **Machine Virtuelle :** `Superv` (Debian + Docker : Prometheus, Grafana)
* **Adresse IP :** `<IP_FIXE_SUPERV>`
* **Zone Réseau :** `DMZ_SUPERV` (<ZONE_DMZ_SUPERV>) - Sous-réseau strictement isolé de la Production et de la DMZ Publique.

## 1. Collecte des métriques (Node Exporter & Prometheus)
Le moteur d'ingestion historique (InfluxDB) a été remplacé par **Prometheus**, qui interroge régulièrement les cibles pour récupérer leurs constantes vitales.
* **Agents de collecte :** L'agent léger `Node Exporter` est installé directement sur les hyperviseurs physiques PVE1 (Production) et PVE2 (Labo isolé). Il lit et expose les métriques matérielles brutes du système hôte (sur le port `9100`).
* **Modèle Pull (Traction) :** Prometheus (sur la VM `Superv`) vient récupérer les données toutes les 15 secondes via des ouvertures chirurgicales et unidirectionnelles dans le pare-feu pfSense.
* **Architecture Hub & Spoke :** Le système est nativement conçu pour s'étendre aux infrastructures distantes (réseau `WG_SIO` en <ZONE_TRANSIT_WG>), permettant de monitorer les serveurs des collaborateurs connectés au tunnel WireGuard sans nécessiter d'ouverture de ports de leur côté.

## 2. Visualisation (Grafana)
Les données temporelles stockées dans Prometheus sont exploitées visuellement via Grafana.
* **Tableaux de bord :** Affichage optimisé se concentrant sur les métriques critiques de l'hyperviseur (Charge Système/Sys Load, utilisation de la RAM, activité du SWAP, et remplissage critique du stockage racine).
* **Filtrage réseau avancé :** Les requêtes PromQL sont adaptées pour ignorer dynamiquement les interfaces réseau virtuelles générées par Proxmox (tap, veth, fw), assurant une lecture claire du trafic réel sur les cartes physiques et le pont principal.
* **Sondes thermiques :** Intégration en temps réel des sondes physiques via `lm-sensors`, remontant automatiquement la température du composant le plus chaud (CPU ou NVMe) avec des jauges colorimétriques ajustées aux tolérances matérielles.

## 3. Centralisation des Journaux (Loki & Promtail)
Pour corréler les métriques matérielles avec les événements réseau, la stack intègre une collecte de logs centralisée.
 - **Ingestion Syslog :** Le conteneur Promtail écoute en UDP (port 1514) pour réceptionner le flux de logs natif (RFC 5424) en provenance du routeur frontal pfSense.
 - **Stockage et Requêtage :** Les journaux sont indexés par Loki et interrogés directement depuis l'interface Grafana via le langage LogQL, permettant un croisement immédiat entre une hausse de charge CPU et un événement réseau spécifique.

## 4. Détection d'Intrusion (IDS) et Réponse à Incident
L'infrastructure de transit (Hub & Spoke) est sécurisée par une approche de "Security by Design" pour anticiper le trafic des clients nomades.
 - **Suricata (pfSense) :** Un moteur de détection d'intrusion analyse en temps réel les paquets traversant l'interface WireGuard, en s'appuyant sur les signatures communautaires Emerging Threats. (Le "Hardware Offloading" des cartes virtuelles est désactivé pour garantir une inspection logicielle brute).
 - **Kill Switch Réseau :** En cas d'intrusion avérée, une règle de pare-feu préconfigurée (désactivée par défaut) sur pfSense permet de sectionner instantanément la liaison WireGuard d'un simple clic. Cela isole la menace externe sans perturber le fonctionnement interne de la Zone Serveurs (<ZONE_SERVEURS>) et de la Zone Clients (<ZONE_CLIENTS>).

## 5. Alerting & Notifications (Discord)
Le moteur d'alerte intégré de Grafana assure une veille automatique et silencieuse sur l'état de l'infrastructure.
* **Règles de criticité :**
  - Surchauffe CPU : Au dessus de 80°C
  - Hote physique hors-ligne : Extinction d'un infrastructure
  - Espace disque faible : Espace restant inférieur à 15G
  - Utilisation RAM : Utilisation supérieur à 90%
  - Charge CPU Elevée : Ratio de charge (Load Average) supérieur au nombre de cœurs physiques disponibles (Ratio > 2).
  - Activité Suspecte (IDS) : Déclenchement d'une signature Suricata dans le flux syslog.
* **Intégration Webhooks :** Notification push immédiate vers un salon technique Discord dédié. Les messages utilisent le système de labels de Prometheus (ex: `job: pve1`) pour identifier précisément et instantanément l'hyperviseur ou la VM en défaut.

## 6. Sécurisation des flux
La Tour de Contrôle est un outil d'administration critique qui n'a aucune existence publique (aucune règle de *Port Forward* sur le routeur WAN). L'accès à l'interface de gestion Grafana s'effectue exclusivement, et de manière chiffrée, depuis l'intérieur du réseau ou au travers des tunnels VPN **WireGuard**.
