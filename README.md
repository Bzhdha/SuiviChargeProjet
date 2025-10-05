# Spécification fonctionnelle & technique (v1.1 enrichie)

## 0) Contexte et objectifs
Application Web sécurisée pour le suivi d’avancement et de performance financière des projets, de la commande à la clôture, avec contrôle strict des accès (macro-droits, micro-droits par rôle), gestion d’une hiérarchie de données (branches), reporting mensuel, haute performance et conformité accessibilité (RGAA/WCAG).

---

## 1) Principes d’accès & sécurité
(... inchangé ...)

---

## 2) Modèle de données (vue logique)
(... inchangé ...)

---

## 3) Règles d’accès (pseudo‑code)
(... inchangé ...)

---

## 4) Écrans & fonctionnalités (exigences détaillées)

### 4.1 Écran Paramétrage (ADMIN uniquement)
(... inchangé ...)

### 4.2 Écran Accueil (Read)
(... inchangé ...)

### 4.3 Écran Liste_projets (CRUD + Archivage)
(... inchangé ...)

### 4.4 Écran Détail_Projet
(... inchangé jusqu’à Suivi_Risques ...)

- **Onglet Planning** :
  - Permet de poser les jalons principaux du projet.
  - Champs : libellé jalon, date prévue, date réelle, statut (prévu, atteint, en retard, annulé).
  - Possibilité de définir des dépendances simples (jalon dépend d’un autre jalon).
  - Vue calendrier et vue liste.
  - Option d’import/export des jalons (CSV, iCal).
  - Prévoir une future intégration avec outils externes (MS Project, Jira Roadmap).

- **Onglet Suivi_Frais** :
(... inchangé ...)

### 4.5 Écran Synthese_Performance
(... inchangé ...)

---

## 5) API (brouillon endpoints REST/GraphQL)
(... inchangé ...)

---

## 6) Performance & scalabilité
(... inchangé ...)
- **Tests de charge** : objectifs cibles à valider :
  - 500 utilisateurs simultanés en phase de clôture.
  - Latence p95 < 500 ms pour consultation de projets.
  - Capacité de traitement de 50k lignes ag-Grid en < 2s avec pagination.

---

## 7) Accessibilité (RGAA/WCAG)
(... inchangé ...)

---

## 8) Notifications & messagerie
- **Types de notifications** :
  - **Critiques (non désactivables)** : risques critiques, échéances de facturation échues, jalons non atteints.
  - **Optionnelles (configurables par l’utilisateur)** : création de projet, affectation à une équipe, mise à jour de frais.
- **Préférences utilisateur** : chaque utilisateur peut choisir les canaux (in‑app, e‑mail) pour les notifications optionnelles.

---

## 9) Import/Export
(... inchangé ...)
- **Taille maximale fichiers importés** : 20 Mo par défaut (configurable).
- **Mode de traitement** : en batch avec retour asynchrone + rapport d’erreurs détaillé.
- **Imports partiels** : en cas d’échec sur certaines lignes, possibilité de rejeter le lot entier ou d’accepter les lignes valides avec log des erreurs.

---

## 10) Intégrations externes
(... inchangé ...)

---

## 11) Traçabilité & conformité
(... inchangé ...)

---

## 12) UX & ergonomie (extraits)
(... inchangé ...)

---

## 13) Qualité & tests
(... inchangé ...)

---

## 14) Architecture technique (référence)
- **Front** : React + TypeScript, ag‑Grid Enterprise/Community, state mgmt (RTK Query/Zustand), i18n.
- **Back** : choix unique à trancher :
  - **Node.js (NestJS)** si priorité à la rapidité de développement, cohérence front/back JavaScript, écosystème léger.
  - **Java (Spring Boot)** si priorité à la robustesse, intégrations lourdes, standardisation SI existant.
- Décision à documenter avant MVP.
- **DB** : PostgreSQL (ltree)...
(... reste inchangé ...)

---

## 15) Algorithmes & calculs clés
(... inchangé ...)

---

## 16) Internationalisation & fuseaux
- i18n (fr/en) dès v1 ; architecture extensible à d’autres langues.
- Gestion via fichiers de langue (JSON/YAML) versionnés.
- Formats localisés : dates, nombres, devises, fuseaux horaires.
- Prévoir paramétrage multi‑devise par projet.

---

## 17) Roadmap (proposée)
(... inchangé ...)

---

## 18) Matrice RBAC initiale (exemple)
(... inchangé ...)

---

## 19) Validation & critères d’acceptation (extraits)
(... inchangé ...)

---

## 20) Annexes
(... inchangé ...)
