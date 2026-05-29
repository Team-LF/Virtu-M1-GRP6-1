# Metalis — Compte rendu d'entretien & Analyse des besoins SI

---

## 1. Contexte

Metalis est une entreprise industrielle sur **site unique**, exploitant des machines à commande numérique (CNC). Pas de croissance prévue. 
L'organisation informatique est héritée, peu structurée, reposant sur des équipements vieillissants.

---

## 2. État de l'existant

### Réseau

| Élément               | Détail                       |
| --------------------- | ---------------------------- |
| Topologie             | Réseau triphasé (électrique) |
| Connectivité Internet | 2 arrivées ADSL              |
| VPN                   | Inexistant                   |
| Accès distant         | Aucun dispositif en place    |

### Serveurs & stockage

| Élément           | État                                                                    |
| ----------------- | ----------------------------------------------------------------------- |
| Serveur principal | Serveur Windows physique sur site, ~20 ans d'ancienneté                 |
| NAS               | Présent (usage et capacité non précisés)                                |
| Maintenance       | Prestataire externe, interventions ponctuelles, sans SLA contractualisé |
| Cloud             | Aucune solution en place ; pas de préférence exprimée                   |

### Postes de travail

| Catégorie                  | Détail                                                                |
| -------------------------- | --------------------------------------------------------------------- |
| Postes de production (CNC) | Windows 7 — **hors réseau** — contrôle des machines CNC               |
| Postes commerciaux         | Doivent pouvoir accéder aux machines de prod à distance (télétravail) |

---

## 3. Analyse des risques

| Domaine           | Risque identifié                                                               | Niveau       |
| ----------------- | ------------------------------------------------------------------------------ | ------------ |
| Disponibilité     | Serveur physique ~20 ans sans SLA — risque de panne totale                     | 🔴 Critique |
| Sécurité          | Windows 7 hors support (fin de vie depuis 2020) — vulnérabilités non corrigées | 🔴 Critique |
| Données sensibles | Brevets et procédés de fabrication sans protection formalisée                  | 🟠 Élevé    |
| Accès distant     | Absence de VPN — télétravail commercial impossible ou non sécurisé             | 🟠 Élevé    |
| Maintenance       | Prestataire ponctuel sans contrat — délais d'intervention non maîtrisés        | 🟠 Élevé    |
| Production CNC    | Postes isolés du réseau — accès distant bloqué, risque d'arrêt de production   | 🟠 Élevé    |
| Connectivité      | ADSL uniquement — faible bande passante, pas de lien professionnel garanti     | 🟡 Modéré   |

---

## 4. Feuille de route 

| Priorité | Action                                   | Bénéfice attendu                           | Horizon   |
| -------- | ---------------------------------------- | ------------------------------------------ | --------- |
| 🔴 P1   | Remplacement serveur + virtualisation    | Élimination du point de défaillance unique | 0–3 mois  |
| 🔴 P1   | Déploiement VPN + accès distant sécurisé | Télétravail commercial opérationnel        | 0–3 mois  |
| 🔴 P1   | Sauvegarde & PRA données sensibles       | Protection brevets & procédés              | 0–3 mois  |
| 🟠 P2   | Migration ADSL → Fibre professionnelle   | Bande passante & SLA réseau garantis       | 3–6 mois  |
| 🟠 P2   | Remplacement postes Windows 7            | Correction des vulnérabilités critiques    | 3–6 mois  |
| 🟠 P2   | Contractualisation maintenance (MSP/SLA) | Délais d'intervention maîtrisés            | 3–6 mois  |
| 🟡 P3   | Audit sécurité global                    | Vision complète des risques résiduels      | 6–12 mois |
| 🟡 P3   | Évaluation stratégie cloud               | Scalabilité & résilience long terme        | 6–12 mois |

---
