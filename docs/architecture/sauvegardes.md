# 💾 Stratégie de Sauvegarde et PRA (Plan de Reprise d'Activité)

La pérennité de l'infrastructure repose sur une stratégie de sauvegarde rigoureuse et un Plan de Reprise d'Activité (PRA) défini. L'objectif est de minimiser la perte de données (RPO) et le temps d'interruption des services (RTO) en cas de sinistre.

---

## 1. Politique de Sauvegarde (Proxmox VE)

La gestion des sauvegardes est centralisée directement via l'outil de backup natif de Proxmox (VZDump), qui permet de sauvegarder l'état complet des machines virtuelles (système, configuration, données).

### A. Philosophie 3-2-1 et État Actuel
La stratégie s'inspire de la règle standard de l'industrie (3-2-1), adaptée aux contraintes matérielles actuelles :
* **3 copies des données :** Les données en production et les archives de sauvegarde.
* **2 supports de stockage :** Les disques internes des machines (Production) et un disque dur externe dédié (Sauvegarde).
* **1 sauvegarde externalisée :** *[Axe d'amélioration]* Actuellement, toutes les sauvegardes sont physiquement au même endroit. L'objectif futur est de synchroniser ces archives vers un stockage Cloud chiffré ou le site distant (via le tunnel WireGuard) pour pallier un sinistre physique (vol, incendie).

### B. Planification et Rétention
Les sauvegardes sont automatisées pour l'ensemble des machines virtuelles :

| Environnement | Fréquence | Type | Rétention | Destination |
| :--- | :--- | :--- | :--- | :--- |
| **Production (PVE1)** | Hebdomadaire (Dimanche) | Snapshot (À chaud) | 5 dernières archives | Disque Dur Externe (Connecté en USB) |
| **Laboratoire (PVE2)** | Hebdomadaire (Dimanche) | Snapshot (À chaud) | 5 dernières archives | Partage Réseau (NAS hébergé sur le Disque Externe PVE1) |
| **Configurations critiques** | À chaque modification | Export manuel (.xml) | Illimité | Dépôt distant chiffré |

*Note : L'astuce architecturale réside dans le partage réseau (NAS) du disque dur externe connecté au PVE1, permettant au PVE2 d'y déposer ses propres sauvegardes de manière autonome.*

---

## 2. Plan de Reprise d'Activité (PRA)

Le PRA définit les procédures d'urgence pour restaurer l'infrastructure suite à un incident majeur.

### Scénario 1 : Crash logiciel ou corruption d'une Machine Virtuelle
* **Symptôme :** Une VM ne démarre plus ou un service est corrompu.
* **Procédure de restauration :**
  1. Connexion à l'interface Proxmox du nœud concerné.
  2. Sélection de la VM > Onglet *Backup*.
  3. Sélection de l'archive du dimanche précédent et exécution de la restauration.
* **RTO (Temps de reprise) estimé :** 5 à 15 minutes (selon la taille du disque).

### Scénario 2 : Crash matériel complet d'un Nœud Proxmox
* **Symptôme :** Panne matérielle empêchant le démarrage de l'hyperviseur.
* **Procédure de restauration :**
  1. Réinstallation d'un système Proxmox VE neuf sur un matériel fonctionnel.
  2. Reconnexion du Disque Dur Externe (en USB ou via montage réseau).
  3. Restauration globale des VMs depuis l'interface Proxmox.
  4. Vérification de la configuration réseau (Ponts `vmbr`).
* **RTO (Temps de reprise) estimé :** 2 à 4 heures.
