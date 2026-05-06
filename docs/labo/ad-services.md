# ❤️ Cœur de réseau : Active Directory, DNS, DHCP, PKI & Fichiers (Windows Server)

La gestion des identités, de la résolution de noms, de l'attribution des adresses IP et de la sécurité cryptographique au sein du laboratoire est centralisée sur un serveur Microsoft. Cette machine est l'épine dorsale de l'environnement client/serveur et permet de simuler un réseau d'entreprise standard complet.

* **Machine Virtuelle :** `WinServ2025-VM200`
* **Zone Réseau :** `SERVEURS_LABO` (<ZONE_SERVEURS>)
* **Système d'exploitation :** Windows Server 2025

## 1. Active Directory Domain Services (AD DS)
Le serveur agit en tant que Contrôleur de Domaine (Domain Controller) pour la forêt du laboratoire.
* **Annuaire centralisé :** Authentification et gestion des comptes utilisateurs, des groupes de sécurité et des ordinateurs du parc (notamment les clients Windows 11).
* **Stratégies de Groupe (GPO) :** Déploiement automatisé de configurations, de politiques de mot de passe, de restrictions de sécurité et de lecteurs réseaux sur les postes de la zone Clients.

## 2. Serveur DNS (Domain Name System)
Pour assurer le bon fonctionnement de l'Active Directory et la navigation interne, cette machine est le serveur DNS primaire et exclusif du labo.
* **Résolution locale :** Traduction des noms de machines internes en adresses IP (indispensable pour l'intégration des clients au domaine et la communication inter-serveurs).
* **Redirecteurs (Forwarders) :** Les requêtes pour des noms de domaine externes (Internet) non reconnues par le domaine local sont transférées vers des serveurs DNS publics.

## 3. Serveur DHCP (Dynamic Host Configuration Protocol)
Le serveur a la charge de distribuer l'adressage IP dynamique, principalement pour les machines de la zone `CLIENTS` (`<ZONE_CLIENTS>`).
* **Intégration au routage :** N'étant pas sur le même sous-réseau que les clients, le DHCP s'appuie sur le *DHCP Relay* configuré sur le pfSense Labo. Le pare-feu écoute les requêtes des clients et les transfère directement à ce serveur Windows.
* **Avantage :** Cette centralisation permet d'avoir une vue unifiée sur les baux IP actifs et de coupler facilement l'attribution IP avec l'enregistrement automatique dans la zone DNS (Mises à jour dynamiques sécurisées).

## 4. Active Directory Certificate Services (AD CS)
Ce rôle met en place une Infrastructure à Clés Publiques (PKI) interne pour le laboratoire.
* **Autorité de Certification (CA) :** Permet d'émettre, de gérer et de révoquer des certificats numériques pour les utilisateurs, les ordinateurs et les services du domaine.
* **Sécurité renforcée :** Indispensable pour chiffrer les communications internes (mise en place du LDAPS, déploiement de certificats pour des serveurs Web internes en HTTPS, ou authentification forte).

## 5. Serveur Web (IIS)
Le service Internet Information Services (IIS) est actif sur ce serveur.
* **Support d'infrastructure :** Dans ce contexte de cœur de réseau, IIS héberge généralement les services d'inscription Web de l'Autorité de Certification (Certification Authority Web Enrollment), permettant de demander et de récupérer facilement des certificats via un navigateur.

## 6. Services de fichiers et de stockage
Ce rôle natif permet d'administrer le stockage et la distribution de fichiers sur le domaine.
* **Partages réseaux :** Création et gestion de dossiers partagés dont les accès sont sécurisés par les permissions NTFS couplées aux groupes de sécurité de l'Active Directory.
* **Intégration GPO :** Utilisé en conjonction avec les Stratégies de Groupe pour monter automatiquement des lecteurs réseaux sur les postes clients (comme `Win11-Client01-VM201`) ou gérer des profils utilisateurs centralisés.
