# 🎫 Gestion de parc et synchronisation LDAP (GLPI)

Le suivi des équipements matériels/logiciels et la gestion de l'assistance (Helpdesk) du laboratoire sont assurés par la solution open-source GLPI (Gestionnaire Libre de Parc Informatique). 

* **Machine Virtuelle :** `GLPI-VM104`
* **Zone Réseau :** `SERVEURS_LABO` (<ZONE_SERVEURS>)

## 1. Inventaire et Gestion de Parc (ITAM)
GLPI centralise la base de données de tous les actifs informatiques du labo.
* Recensement des machines virtuelles, des routeurs (pfSense), et des terminaux de la zone Clients.
* Cartographie et suivi de l'attribution des adresses IP et de la topologie réseau.

## 2. Centre d'Assistance (ITSM / Ticketing)
Le portail permet de simuler un environnement de support d'entreprise (Helpdesk).
* Création, affectation et suivi des tickets d'incidents ou de demandes (simulés depuis la zone `CLIENTS`).
* Gestion des bases de connaissances associées aux procédures de résolution du labo.

## 3. Synchronisation Annuaire (LDAP / Active Directory)
Afin de maintenir une cohérence et d'appliquer le principe d'authentification unique (SSO/Centralisée), GLPI est directement interfacé avec le contrôleur de domaine Windows Server (`WinServ2025-VM200`).
* **Connexion LDAP :** Le serveur GLPI interroge l'Active Directory de manière sécurisée pour valider les identifiants.
* **Import automatisé :** Synchronisation régulière pour importer automatiquement les nouveaux utilisateurs, leurs informations (email, service) et leurs groupes depuis l'AD vers la base de données GLPI, évitant ainsi la double saisie administrative.
