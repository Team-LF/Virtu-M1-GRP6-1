# METALIS — Objectifs pédagogiques

> Ces 8 objectifs couvrent les compétences attendues par le module Virtualisation, appliquées au cas METALIS.

---

## Objectif 1 — Proxmox VE : hyperviseur de type 1

**Contexte METALIS :** Le serveur Windows Server physique (~20 ans) est en fin de vie. Il est remplacé par un hôte Proxmox VE installé en salle serveur du site principal.

**Ce que nous devons expliquer :**
- Proxmox VE est un hyperviseur de **type 1** (bare-metal) basé sur KVM + LXC, qui s'installe directement sur le matériel sans OS hôte intermédiaire.
- Il permet de faire tourner plusieurs VMs isolées sur un seul serveur physique : AD, Odoo, NAS, VPN, supervision — chacune dans son propre environnement.
- L'interface web Proxmox centralise la gestion des VMs, des snapshots, des ressources et des datastores.
- **Pourquoi Proxmox plutôt que XCP-ng ici :** budget limité, communauté active, intégration native avec Proxmox Backup Server (PBS), pas de licence commerciale requise.

---

## Objectif 2 — Ressources & sécurité : dimensionnement, isolation, accès

**Contexte METALIS :** 40 utilisateurs, postes Windows 7 en atelier, données sensibles (brevets, procédés), accès prestataires à cadrer.

**Ce que nous devons expliquer :**
- **Dimensionnement :** chaque VM se voit allouer des ressources (vCPU, RAM, stockage) proportionnelles à son usage — ex. `vm-odoo` reçoit 4 vCPU / 8 Go RAM pour absorber les pics de charge ERP.
- **Isolation :** segmentation en VLANs (PROD / BUREAU / MGMT / VPN / CAM) pour qu'un incident sur un segment ne propage pas au reste du réseau. Les postes Windows 7 CNC restent sur VLAN 10, sans accès internet direct.
- **Accès :** Active Directory centralise l'authentification. GPO par profil (commerciaux, atelier, direction, prestataires). Comptes prestataires temporaires et révocables.
- **Tiers de confiance :** les données sensibles (brevets) sont stockées sur un partage dédié TrueNAS, accessible uniquement à la direction et au bureau d'études, avec audit des accès fichiers.

---

## Objectif 3 — Architecture hybride on-premise / cloud

**Contexte METALIS :** Client favorable à OVH (site déjà hébergé sur VPS OVH). Budget limité, pas de croissance prévue.

**Ce que nous devons expliquer :**
- **On-premise (site principal) :** héberge les services critiques internes — AD, Odoo, NAS, VPN, PBS. Ces services ne doivent pas dépendre d'une connexion internet pour fonctionner en production.
- **Cloud OVHcloud :** héberge les services exposés — WordPress, WooCommerce, sauvegardes offsite. OVH est retenu car le client y fait déjà confiance et c'est un hébergeur souverain français.
- **Hybride :** les deux couches sont complémentaires. En cas de panne internet, les services internes (AD, Odoo, fichiers) continuent de fonctionner. En cas de panne du site principal, le site e-commerce reste en ligne.
- **Justification du choix hybride vs full cloud :** les données de brevets et procédés ne doivent pas transiter sur un cloud public sans contrôle fort — elles restent on-premise.

---

## Objectif 4 — Supervision : suivi des VMs et services critiques

**Contexte METALIS :** Pas d'IT interne, prestataire ponctuel — la supervision doit être automatisée et alerter proactivement.

**Ce que nous devons expliquer :**
- **Outil retenu : Zabbix** (open source, déployé sur `vm-supervision`).
- Éléments surveillés : état des VMs Proxmox (CPU, RAM, disque, réseau), services critiques (AD, Odoo, NAS, VPN, PBS), liens réseau (fibre + 4G failover).
- **Alertes :** email et/ou SMS au prestataire MSP en cas d'incident (seuils configurables : CPU > 90 %, disque > 80 %, service down).
- **Tableaux de bord :** visualisation en temps réel de la santé de l'infra — utilisable par le prestataire MSP lors de ses interventions.
- Sans supervision, une panne sur `vm-odoo` ou `vm-nas` pourrait passer inaperçue pendant des heures, ce qui est incompatible avec le SLA élevé exigé.

---

## Objectif 5 — Sauvegardes & PRA : stratégie et tests de restauration

**Contexte METALIS :** Panne NAS l'an dernier avec restauration partielle. Pas de politique de sauvegarde formalisée. Données de brevets à protéger impérativement.

**Ce que nous devons expliquer :**
- **Règle 3-2-1 appliquée :**
  - Copie 1 : sauvegarde nightly des VMs via Proxmox Backup Server (`vm-backup`)
  - Copie 2 : snapshot hebdomadaire vers datastore secondaire (site principal)
  - Copie 3 : export hebdomadaire chiffré (AES-256) vers OVH Object Storage (offsite)
- **PRA NAS (prioritaire) :** migration des fichiers CAO vers TrueNAS Scale avec ZFS RAIDZ1 (tolérance panne disque) + snapshots quotidiens (rétention 30 jours).
- **RTO cible : < 4h** — en cas de panne totale, les VMs sont restaurées depuis PBS en moins d'une demi-journée.
- **RPO cible : < 24h** — perte de données maximale acceptable d'une journée de travail.
- **Tests de restauration :** mensuel sur une VM, trimestriel depuis OVH S3 — documentés et validés (la panne NAS passée a montré qu'une restauration non testée = restauration partielle).

---

## Objectif 6 — VDI & profils : accès distant et postes centralisés

**Contexte METALIS :** Les commerciaux en télétravail doivent accéder aux postes de production CNC à distance. Les postes Windows 7 CNC sont hors réseau général.

**Ce que nous devons expliquer :**
- **Solution retenue : accès RDP via jump host** plutôt qu'un VDI complet (trop coûteux pour une PME de 40 personnes sans IT interne).
- Les commerciaux se connectent via **WireGuard VPN** → VLAN 40 → jump host sécurisé → RDP vers les postes CNC sur VLAN 10.
- Les postes CNC ne sont jamais directement exposés : le jump host fait office de proxy d'accès contrôlé et journalisé.
- **Profils itinérants AD :** les commerciaux disposent d'un profil AD centralisé — leurs données sont stockées sur `vm-nas` et non sur le poste local, ce qui facilite le télétravail.
- **Pertinence du VDI complet :** non retenu ici (coût, complexité, effectif stable), mais documenté comme évolution possible si l'effectif devait croître.

---

## Objectif 7 — Hyper-V & résilience : lien avec l'atelier résilience Windows

**Contexte METALIS :** Le projet s'appuie sur Proxmox, mais l'atelier « résilience Windows » porte sur Hyper-V — il faut établir le lien entre les deux.

**Ce que nous devons expliquer :**
- **Hyper-V** est l'hyperviseur de type 1 de Microsoft, intégré à Windows Server. Il aurait pu être retenu pour METALIS (environnement déjà Windows Server existant), mais Proxmox a été préféré pour des raisons de coût de licence et de flexibilité.
- **Résilience Windows appliquée à METALIS :**
  - Le contrôleur de domaine (`vm-dc01` sous Windows Server 2022) héberge l'AD — un second DC en réplication pourrait être ajouté pour éviter le SPOF sur l'annuaire.
  - Les fonctions de **Cluster Failover** d'Hyper-V (redondance de VMs sur plusieurs hôtes physiques) sont analogues aux fonctions de **haute disponibilité Proxmox** (HA Proxmox Cluster) — même concept, implémentations différentes.
- **Lien atelier :** les mécanismes de bascule automatique (VM restart sur hôte sain en cas de panne d'un nœud) étudiés en atelier Hyper-V se retrouvent dans Proxmox HA — les deux répondent à la même exigence : *"Plus jamais de vendredi après-midi sans production"*.

---

## Objectif 8 — PRA / PCA : plan de continuité documenté

**Contexte METALIS :** Aucun PRA formalisé. La panne NAS l'an dernier a coûté plusieurs journées de production. La direction exige zéro panne.

**Ce que nous devons expliquer :**
- **PRA (Plan de Reprise d'Activité) :** procédure de redémarrage des services après incident majeur. Déclenché quand le service est interrompu.
- **PCA (Plan de Continuité d'Activité) :** procédure permettant de maintenir une activité minimale *pendant* l'incident. Plus exigeant que le PRA.
- **Pour METALIS, on cible un PRA avec des éléments de PCA :**
  - PCA : lien 4G failover maintient l'accès internet si la fibre tombe ; AD et Odoo continuent localement même sans internet.
  - PRA : en cas de panne serveur, restauration des VMs depuis PBS en < 4h (RTO).
- **Scénarios documentés :**

| Scénario | Impact | Action | RTO |
|----------|--------|--------|-----|
| Panne fibre principale | Perte accès distant + internet | Bascule automatique 4G failover | < 5 min |
| Panne NAS (disque) | Lenteurs / perte fichiers CAO | ZFS RAIDZ1 absorbe la panne disque, pas d'interruption | 0 |
| Panne VM Odoo | ERP inaccessible | Restauration depuis PBS | < 2h |
| Panne hôte Proxmox | Toutes VMs down | Restauration depuis PBS sur nouveau matériel | < 4h |
| Ransomware | Données chiffrées | Restauration depuis snapshot PBS (avant infection) | < 4h |

- Le plan est testé trimestriellement et mis à jour après chaque entretien avec le prestataire MSP.