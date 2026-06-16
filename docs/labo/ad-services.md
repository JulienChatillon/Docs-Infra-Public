# ❤️ Cœur de réseau : Active Directory, DNS, DHCP, PKI & Fichiers (Windows Server)

La gestion des identités, de la résolution de noms, de l'attribution des adresses IP et de la sécurité cryptographique au sein du laboratoire est centralisée sur un environnement Microsoft. Cet ensemble est l'épine dorsale de l'architecture client/serveur et permet de simuler un réseau d'entreprise standard complet et redondant.

* **Machine Virtuelle Principale :** `WinServ2025-VM200`
* **Machine Virtuelle de Secours :** `WinServSec-VM202`
* **Zone Réseau :** `SERVEURS` (<ZONE_SERVEURS>)
* **Système d'exploitation :** Windows Server 2025

## 1. Active Directory Domain Services (AD DS)
Le serveur principal agit en tant que Contrôleur de Domaine (Domain Controller) pour la forêt du laboratoire.
* **Annuaire centralisé :** Authentification et gestion des comptes utilisateurs, des groupes de sécurité et des ordinateurs du parc (notamment les clients Windows 11).
* **Stratégies de Groupe (GPO) :** Déploiement automatisé de configurations, de politiques de mot de passe, de restrictions de sécurité et de lecteurs réseaux sur les postes de la zone Clients.

## 2. Serveur DNS (Domain Name System)
Pour assurer le bon fonctionnement de l'Active Directory et la navigation interne, cette machine héberge la zone DNS principale du labo.
* **Résolution locale :** Traduction des noms de machines internes en adresses IP (indispensable pour l'intégration des clients au domaine et la communication inter-serveurs).
* **Redirecteurs (Forwarders) :** Les requêtes pour des noms de domaine externes (Internet) non reconnues par le domaine local sont transférées vers des serveurs DNS publics.

## 3. Serveur DHCP (Dynamic Host Configuration Protocol)
Le serveur a la charge de distribuer l'adressage IP dynamique pour les machines du parc client (`<ZONE_CLIENTS>`).
* **Attribution centralisée :** Le service distribue les baux IP et gère les réservations en répondant aux requêtes relayées en amont par le pare-feu.
* **Intégration DNS :** L'avantage majeur de cette centralisation sous Windows Server est le couplage direct avec le DNS : chaque attribution d'IP génère un enregistrement automatique et sécurisé du nom de la machine dans la zone DNS locale.

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

## 7. Redondance et Tolérance aux pannes (Failover)
Pour garantir la haute disponibilité des services d'infrastructure vitaux, un second contrôleur de domaine (`WinServSec-VM202`) est déployé dans la même zone réseau Serveurs.
* **Réplication AD DS :** L'annuaire Active Directory est répliqué en continu entre VM200 et VM202 via une architecture multi-maître. Si le serveur principal tombe, les utilisateurs peuvent toujours s'authentifier sans interruption.
* **Réplication DNS :** Les zones DNS du laboratoire étant intégrées à l'Active Directory, elles sont automatiquement et nativement synchronisées sur le serveur de secours.
* **Secours DHCP :** En cas d'indisponibilité de la VM200, l'infrastructure de secours ou le pare-feu prennent le relais pour assurer la continuité de la distribution des adresses IP et éviter toute paralysie de la zone Clients.
