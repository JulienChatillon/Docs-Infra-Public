# 👁️ Supervision et Métriques (Prometheus & Grafana)

La surveillance proactive des performances matérielles (CPU, RAM, Températures, Disques) et réseau de l'ensemble de l'infrastructure est centralisée sur une "Tour de Contrôle" dédiée. Cette architecture de monitoring a migré vers un modèle moderne de type "Pull", garantissant une visibilité totale tout en respectant une isolation stricte des environnements.

* **Machine Virtuelle :** `Superv` (Debian + Docker : Prometheus, Grafana)
* **Zone Réseau :** `DMZ_SUPERV` - Sous-réseau strictement isolé de la Production et de la DMZ Publique.

## 1. Collecte des métriques (Node Exporter & Prometheus)
Le moteur d'ingestion historique (InfluxDB) a été remplacé par **Prometheus**, qui interroge régulièrement les cibles pour récupérer leurs constantes vitales.
* **Agents de collecte :** L'agent léger `Node Exporter` est installé directement sur les hyperviseurs physiques PVE1 (Production) et PVE2 (Labo isolé). Il lit et expose les métriques matérielles brutes du système hôte (sur le port `9100`).
* **Modèle Pull (Traction) :** Prometheus (sur la VM `Superv`) vient récupérer les données toutes les 15 secondes via des ouvertures chirurgicales et unidirectionnelles dans le pare-feu pfSense.
* **Architecture Hub & Spoke :** Le système est nativement conçu pour s'étendre aux infrastructures distantes (réseau `WG_SIO`), permettant de monitorer les serveurs des collaborateurs connectés au tunnel WireGuard sans nécessiter d'ouverture de ports de leur côté.

## 2. Visualisation (Grafana)
Les données temporelles stockées dans Prometheus sont exploitées visuellement via Grafana.
* **Tableaux de bord :** Affichage optimisé se concentrant sur les métriques critiques de l'hyperviseur (Charge Système/Sys Load, utilisation de la RAM, activité du SWAP, et remplissage critique du stockage racine).
* **Filtrage réseau avancé :** Les requêtes PromQL sont adaptées pour ignorer dynamiquement les interfaces réseau virtuelles générées par Proxmox (tap, veth, fw), assurant une lecture claire du trafic réel sur les cartes physiques et le pont principal.
* **Sondes thermiques :** Intégration en temps réel des sondes physiques via `lm-sensors`, remontant automatiquement la température du composant le plus chaud (CPU ou NVMe) avec des jauges colorimétriques ajustées aux tolérances matérielles.

## 3. Alerting & Notifications (Discord)
Le moteur d'alerte intégré de Grafana assure une veille automatique et silencieuse sur l'état de l'infrastructure.
* **Règles de criticité :** Les alertes sont conditionnées à des seuils d'activation persistants (ex: Température de la sonde principale > 80°C pendant plus de 5 minutes continues) pour éviter les faux positifs lors de pics de charge normaux.
* **Intégration Webhooks :** Notification push immédiate vers un salon technique Discord dédié. Les messages utilisent le système de labels de Prometheus (ex: `job: pve1`) pour identifier précisément et instantanément l'hyperviseur ou la VM en défaut.

## 4. Sécurisation des flux
La Tour de Contrôle est un outil d'administration critique qui n'a aucune existence publique (aucune règle de *Port Forward* sur le routeur WAN). L'accès à l'interface de gestion Grafana s'effectue exclusivement, et de manière chiffrée, depuis l'intérieur du réseau ou au travers des tunnels VPN **WireGuard**.
