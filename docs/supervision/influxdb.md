# 🖥️ Monitoring et métriques (InfluxDB)

La surveillance proactive des performances et de la santé (ressources matérielles et réseau) de l'ensemble de l'infrastructure est centralisée sur une machine dédiée. Ce socle permet d'anticiper les goulots d'étranglement et de conserver un historique de la charge.

* **Machine Virtuelle :** `InfluxDB-VM102` (avec Grafana)
* **Zone Réseau :** `LAN_PROD` (<ZONE_LAN>)
* **Hébergement :** Nœud 1 (PVE1)

## 1. Base de données Time-Series (InfluxDB)
Le moteur InfluxDB est spécifiquement optimisé pour ingérer et stocker des données horodatées en continu.
* **Collecte native Proxmox :** Les hyperviseurs PVE1 et PVE2 sont configurés pour pousser nativement leurs métriques (consommation CPU, utilisation RAM, I/O disques, trafic des interfaces réseau) vers cette base de données (port `<PORT_SUPERVISION_OLD>`).
* **Centralisation multi-environnements :** Permet d'agréger les données de l'environnement de production et de l'environnement de laboratoire isolé en un seul point de collecte.

## 2. Visualisation (Grafana)
Les données brutes stockées dans InfluxDB sont exploitées visuellement via l'outil Grafana.
* **Tableaux de bord (Dashboards) :** Interfaces graphiques sur mesure offrant une vue d'ensemble instantanée (et historique) de l'état de santé des nœuds et des machines virtuelles.
* **Alerting :** Possibilité de configurer des seuils d'alerte critiques (ex: RAM > 90%) pour une supervision active.

## 3. Sécurisation des flux
Conformément à la politique de sécurité globale (Zero Trust sur le WAN), les interfaces de la base de données et des tableaux de bord ne sont **pas** exposées sur Internet. La consultation des métriques à distance par l'administrateur s'effectue exclusivement et de manière chiffrée au travers du tunnel VPN **WireGuard Nomade**.
