# 📜 Documentation Infrastructure & Labo SIO

Rréférentiel de documentation de mon infrastructure informatique. Base de connaissances centralisée pour l'architecture, les configurations et les procédures liées à mon environnement de production (PVE1) et mon laboratoire isolé (PVE2).

---

## 📖 Sommaire

### 🌐 1. Architecture Globale
* [Vue d'ensemble de l'infrastructure](docs/architecture/vue-ensemble.md)
* [Topologie Réseau & Proxmox](docs/architecture/topologie.md)
* [Stratégie de sauvegarde et PRA (Plan de Reprise d'Activité)](docs/architecture/sauvegardes.md)

### 🌍 2. Environnement de Production & Front (PVE1)
* [Routage et Pare-feu : pfSense Front](docs/prod/pfsense-front.md)
* [Hébergement Web & DMZ (Nginx Proxy Manager)](docs/prod/reverse-proxy.md)
* [Interconnexion WireGuard (Nomade & Site-to-Site)](docs/prod/wireguard.md)

### 🧪 3. Environnement de Laboratoire (PVE2)
* [Routage interne et Pare-feu : pfSense Labo](docs/labo/pfsense-labo.md)
* [Cœur de réseau : Active Directory, DNS & DHCP (Windows Server)](docs/labo/ad-services.md)
* [Gestion de parc et synchronisation LDAP (GLPI)](docs/labo/glpi.md)

### 👁️ 4. Supervision & Sécurité
* [Monitoring et métriques (Prometheus)](docs/supervision/superv.md)
