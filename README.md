# Spécification complète — Application de Suivi Financier de Projet
*(Généré le 05/10/2025)*

## 1. Contexte & objectifs
Application Web sécurisée pour le suivi d’avancement et de performance financière des projets, de la commande à la clôture, avec contrôle strict des accès (macro-droits, micro-droits par rôle), gestion d’une hiérarchie de données (branches), reporting mensuel, haute performance et conformité accessibilité (RGAA/WCAG).

## 2. Accès & sécurité
- Macro‑droits : l’absence renvoie 404 sur toute URL front/back (invisibilité de l’app).
- Micro‑droits (RBAC) par écran/action (C/R/U/D/A), avec périmètre de données par branche hiérarchique (héritage N → N+x).
- JWT d’accès courte durée + refresh, OIDC/OAuth2 PKCE, SSO possible, MFA recommandé.
- Chiffrement en transit/au repos, headers de sécurité (CSP, HSTS, etc.), anti-CSRF, rate limiting, audit.

## 3. Données (vue logique)
- **Branch**(id, name, level, parent_id, path_ltree, is_active)
- **Role**(id, name, code)
- **Screen**(id, name, code)
- **Permission**(role_id, screen_id, C,R,U,D,A)
- **User**(id, username, email, display_name, auth_source)
- **UserRole**(user_id, role_id)
- **UserBranchRole**(user_id, branch_id, role_id)
- **WelcomeMessage**, **Holiday**, **Tag**
- **Project**(name, key, start_date, end_date, archived, branch_id)
- **Order**, **ActivityType**, **ProjectPeriod**, **WorkloadCell**, **Absence**, **Consumption**, **BillingSchedule**, **FinancialKPI**, **Risk**, **RiskAction**, **Fees**

## 4. Écrans & fonctionnalités
- **Paramétrage (ADMIN)** : messages, rôles/écrans/droits, associations rôles↔utilisateurs, branches↔utilisateurs, jours chômés, imports/exports, audit.
- **Accueil** : bandeau d’accueil, arborescence des projets accessibles, cloche notifications.
- **Projets** : liste (CRUD + archivage), détail (commandes, équipe, activités, périodes, PDC, consommations, échéancier, financier, risques, frais, planning).
- **Synthèse Performance** : filtres (période, tags), cumul commandes/consommé/vendu/performance.

## 5. Performance & scalabilité
- ag‑Grid (virtualisation, server-side model), index & vues matérialisées, cache Redis, batchs asynchrones, autoscaling, tests de charge (≥ 500 utilisateurs), p95 < 500 ms sur lectures clés.

## 6. Accessibilité & i18n
- RGAA/WCAG AA : focus, contrastes, aria‑live, navigation clavier complète.
- i18n fr/en, formats dates/nombres/devises/fuseaux, jours chômés par pays.

## 7. API (brouillon)
- Endpoints admin (messages/roles/screens/permissions/user-roles/user-branches/holidays/import/export) et projet (orders/team/activity-types/periods/workload/absences/consumptions/billing/financial/risks/fees) + reporting (performance, billing-summary).

## 8. Audit & conformité
- AuditLog (who, when, what, before/after, ip, userAgent), export CSV, rétention configurable, RGPD.

---

# Backlog MVP — User Stories (INVEST + Gherkin)

## US-001 — Accès par macro-droits (404 si absent)

**En tant que** visiteur sans macro-droits,
**je veux** ne pas voir l’application ni ses endpoints,
**afin de** préserver la confidentialité et réduire la surface d’attaque.

**INVEST**: Indépendante, Négociable, Valeur, Estimable, Simple, Testable ✅

### Description (Gherkin)
Feature: Macro-access control returns 404
  Scenario: Anonymous or unauthorized user hits any app URL
    Given a user without macro-rights
    When the user requests "/" or any backend route
    Then the response status should be 404
    And the body should not reveal the existence of the application

### Critères d’acceptation
- 404 sur toutes les routes front/back si macro-droit manquant.
- Aucune bannière, aucun stacktrace, aucun header révélateur.
- Journalisation minimale côté reverse proxy (niveau info).

### Règles de gestion
- Le contrôle s’exécute avant l’auth et RBAC.
- La liste blanche d’URLs publiques (assets statiques/health interne) ne divulgue rien.

### Interface (Design système)
- N/A côté UI (page 404 générique DS, contraste AA, lien retour optionnel).


---

## US-002 — Authentification & session (OIDC/JWT)

**En tant que** utilisateur autorisé (macro-droits),
**je veux** m’authentifier via OIDC et disposer d’une session sécurisée,
**afin de** naviguer dans l’application selon mes rôles et branches.

**INVEST** ✅

### Description (Gherkin)
Feature: OIDC login and JWT session
  Scenario: Successful login via provider
    Given a user with macro-rights in IdP
    When the user completes OIDC login
    Then an access token (short-lived) and refresh token are issued
    And the user profile endpoint "/me" returns roles and branches

### Critères d’acceptation
- Login OIDC PKCE, refresh silencieux, logout IdP.
- Token d’accès ≤ 15 min, refresh ≤ 7 jours.
- Cookies httpOnly, SameSite Lax/Strict.
- "/me" renvoie id, displayName, roles, branches.

### Règles de gestion
- MFA si activé côté IdP.
- Expiration stricte : refus d’accès si token expiré.

### Interface
- Écran de redirection IdP + loader DS.
- Header avec avatar, menu compte, bouton logout.


---

## US-003 — RBAC par écran & action

**En tant que** administrateur,
**je veux** définir pour chaque rôle les droits C/R/U/D/A par écran,
**afin de** contrôler finement les capacités de chaque profil.

**INVEST** ✅

### Description (Gherkin)
Feature: Role-based permissions per screen
  Scenario: Deny update on a screen
    Given role "VIEWER" has Read only on "PROJECTS"
    When a user with role "VIEWER" tries to Update a project
    Then the API returns 404

### Critères d’acceptation
- Matrice Permissions: role × screen × {C,R,U,D,A}.
- Contrôle back **et** front (masquage UI).

### Règles de gestion
- 404 si RBAC échoue.
- Permissions versionnées et auditables.

### Interface
- Paramétrage > Droits: ag‑Grid {Role, Screen, C,R,U,D,A} toggles.
- Actions groupées (cocher colonne).


---

## US-004 — Affectation des rôles aux utilisateurs

**En tant que** administrateur,
**je veux** associer des rôles aux utilisateurs (CRUD + import/export CSV),
**afin de** gérer rapidement les habilitations.

**INVEST** ✅

### Description (Gherkin)
Feature: Assign roles to users
  Scenario: Bulk import role assignments
    Given a CSV with rows {user_code, role_code}
    When I import the file in the admin screen
    Then valid rows are created and invalid rows are reported

### Critères d’acceptation
- CRUD unitaire + import CSV (dry‑run + rapport).
- Export CSV filtrable.

### Règles de gestion
- Unicité (user, role).
- Audit sur création/suppression.

### Interface
- Paramétrage > Rôles/Utilisateurs: ag‑Grid, boutons Import/Export.
- Modale d’upload + aperçu erreurs.


---

## US-005 — Affectation des branches (périmètre hiérarchique)

**En tant que** administrateur,
**je veux** assigner des branches aux utilisateurs,
**afin de** limiter l’accès aux données d’une sous‑arborescence.

**INVEST** ✅

### Description (Gherkin)
Feature: Branch-based access inheritance
  Scenario: User sees only projects inside assigned branch subtree
    Given user is assigned to branch B at level N
    And there are projects under descendants of B
    When the user opens the home tree
    Then only those projects are listed

### Critères d’acceptation
- Arbre filtré côté back via path/ltree.
- 404 sur toute ressource hors sous‑arbre.

### Règles de gestion
- Héritage descendant N → N+x.
- Multi-branches permis (union des sous‑arbres).

### Interface
- Paramétrage > Branches/Utilisateurs: tree selector + ag‑Grid attributions.


---

## US-006 — Message d’accueil (bandeau)

**En tant que** admin,
**je veux** publier un message d’accueil avec période de validité et importance,
**afin de** diffuser des informations clés.

**INVEST** ✅

### Description (Gherkin)
Feature: Welcome banner
  Scenario: Active message displays on home
    Given a message with starts_at <= today <= ends_at
    When a user opens the home screen
    Then the banner shows content and importance icon

### Critères d’acceptation
- Un seul message actif affiché (le plus récent si chevauchement).
- Importance: Alerte/Info avec icône ARIA‑label.

### Règles de gestion
- Conflits: priorité à la date de création la plus récente.

### Interface
- Bandeau haut (Alert DS), markdown basique.


---

## US-007 — Liste des projets (CRUD + archivage)

**En tant que** manager,
**je veux** créer, rechercher, modifier et archiver des projets,
**afin de** gérer mon portefeuille.

**INVEST** ✅

### Description (Gherkin)
Feature: Project list management
  Scenario: Archive a project
    Given a project is active
    When I trigger Archive
    Then the project state becomes archived and it disappears from default views

### Critères d’acceptation
- Champs: name, key, tags, start/end, utilisateurs (read-only agrégé).
- Archive non destructive (reversible par admin).
- Filtres: tags, dates, texte.

### Règles de gestion
- Clé projet unique par branche.
- Impossible d’archiver si factures en cours (statut Émis non payé) — avertissement.

### Interface
- ag‑Grid server-side, colonnes configurables, chips de tags, action Archive.


---

## US-008 — Détail projet — Commandes (CRUD)

**En tant que** manager,
**je veux** gérer les commandes d’un projet,
**afin de** cadrer le périmètre budgétaire.

**INVEST** ✅

### Description (Gherkin)
Feature: Manage orders within a project
  Scenario: Create an order
    Given a project exists
    When I create an order with code, qty, unit, unit_price
    Then amount_ht is computed = qty * unit_price * (1 - discount_pct)

### Critères d’acceptation
- Saisie/édition en ligne, validations synchrones.
- Calcul automatique montant/discount.
- Statuts: Brouillon, Actif, Clos.

### Règles de gestion
- Unicité code commande par projet.
- Si risk_level_override ≠ niveau projet → montant_de_risque imputé à la provision du projet.

### Interface
- ag‑Grid avec colonnes figées, résumé en pied (Σ montants).


---

## US-009 — Périodes projet (CRUD)

**En tant que** chef de projet,
**je veux** définir des périodes internes (lots/mois),
**afin de** grouper le PDC et les rapports.

**INVEST** ✅

### Description (Gherkin)
Feature: Project periods
  Scenario: Create a period
    Given a project
    When I add a period with start and end dates
    Then it is selectable as a grouping in PDC

### Critères d’acceptation
- Non chevauchement des périodes d’un même projet.
- Bornes dans [start_date, end_date] du projet.

### Interface
- Formulaire + liste triée, validation chevauchements.


---

## US-010 — Plan de charge (saisie jour ouvré)

**En tant que** chef de projet,
**je veux** saisir les charges par jour ouvré (commande × collaborateur × activité),
**afin de** planifier la production.

**INVEST** ✅

### Description (Gherkin)
Feature: Workload grid entry
  Scenario: Enter workload within project bounds
    Given project dates from D1 to D2
    And business days defined by holidays calendar
    When I enter a value on a non-business day or outside [D1, D2]
    Then the cell is disabled and an accessible tooltip explains why

### Critères d’acceptation
- Colonnes jour ouvré, entête (commande, collaborateur, activité).
- Groupement par périodes projet ou par mois.
- Saisie numérique validée (>=0, pas de décimal si unité = jours entiers).

### Règles de gestion
- Écriture batch (upsert) avec verrouillage optimiste.
- Respect des jours chômés (table Holidays).

### Interface
- ag‑Grid virtualisée, sticky headers, clavier complet, aria-live sur validations.


---

## US-011 — Suivi consommations (import manuel)

**En tant que** contrôleur,
**je veux** importer un CSV de consommations (Jira/Redmine/PDC),
**afin de** consolider le consommé au fil de l’eau.

**INVEST** ✅

### Description (Gherkin)
Feature: Manual consumption import
  Scenario: Dry-run then import
    Given a CSV with columns (date, order_code, activity_code, qty)
    When I launch a dry-run
    Then I see a report of valid and invalid lines
    And on confirm, valid lines are persisted

### Critères d’acceptation
- Taille max 20 Mo, encodage UTF‑8, séparateur configurable.
- Dry‑run obligatoire, rapport téléchargeable.

### Règles de gestion
- Mapping strict codes commande/activité.
- Idempotence par (date, order, activity, source, payload hash).

### Interface
- Import wizard en 3 étapes, barre de progression, carte résumé import.


---

## US-012 — Échéancier de facturation

**En tant que** contrôleur,
**je veux** maintenir l’échéancier de facturation par commande,
**afin de** suivre encours, émis, payé, prévision globale.

**INVEST** ✅

### Description (Gherkin)
Feature: Billing schedule
  Scenario: Update status to Paid
    Given a schedule line with status "Émis"
    When I set status to "Payé"
    Then the encours decreases accordingly and paid increases

### Critères d’acceptation
- Colonnes: date prévue, montant, statut {Prévu, Émis, Payé, Annulé}.
- Section haute: encours, facturé, prévision globale (mise à jour en temps réel).

### Règles de gestion
- Montant ≥ 0, date future autorisée.
- Somme des lignes peut dépasser montant commande (alerte visuelle).

### Interface
- Tableau éditable, barres de statut (chip), résumé chiffré en header.


---

## US-013 — Suivi des risques & plan d’actions

**En tant que** chef de projet,
**je veux** enregistrer risques et actions (mitigation/contingence),
**afin de** piloter la criticité et les responsabilités.

**INVEST** ✅

### Description (Gherkin)
Feature: Risks and action plan
  Scenario: Compute criticality
    Given a risk with probability 4 and impact 5
    When I save the risk
    Then criticality equals 20 and is highlighted as High

### Critères d’acceptation
- Champs: clé, description, proba (1..5), impact (1..5), conséquence, montant, catégorie.
- Actions: type, porteur, date prévue/réelle, commentaire.

### Règles de gestion
- criticality = proba × impact ; seuils: Low (1..5), Medium (6..12), High (13..25).

### Interface
- Table + panneau latéral d’édition, badges de criticité, tri par criticité.


---

## US-014 — Suivi des frais

**En tant que** contrôleur,
**je veux** saisir frais engagés et prévisionnels par période et par commande,
**afin de** rapprocher dépenses et revenus.

**INVEST** ✅

### Description (Gherkin)
Feature: Fees tracking
  Scenario: Enter forecast and actuals
    Given an analysis period is selected
    When I input actual and forecast amounts
    Then totals per order and per period are updated

### Critères d’acceptation
- Période d’analyse (mois), saisie par commande.
- Totaux par période, par commande et cumul projet.

### Règles de gestion
- Devise projet appliquée aux montants.
- Historisation des corrections (audit).

### Interface
- Tableau avec résumé, filtres période, export CSV.


---

## US-015 — Synthèse Performance (reporting mensuel)

**En tant que** direction,
**je veux** voir une synthèse par mois (commandes, consommé, vendu, performance),
**afin de** piloter la santé économique.

**INVEST** ✅

### Description (Gherkin)
Feature: Monthly performance report
  Scenario: Filter by date range and tags
    Given projects open in the selected period
    And having at least one selected tag
    When I run the report
    Then I see monthly aggregates and a performance = vendu - dépensé

### Critères d’acceptation
- Filtres: période (YYYY‑MM .. YYYY‑MM), tags multi‑sélection.
- Agrégats par mois et totaux.

### Règles de gestion
- Inclusion d’un projet si intersecte la période.
- Périmètre respectant RBAC/ABAC.

### Interface
- Carte filtres (Select + DateRange), tableau, graph barres (optionnel), export CSV.


---

## US-016 — Jours chômés (paramétrage)

**En tant que** admin,
**je veux** gérer les jours chômés par année,
**afin de** piloter la grille PDC et les validations.

**INVEST** ✅

### Description (Gherkin)
Feature: Holidays management
  Scenario: Add a holiday
    Given year 2026
    When I add a date with optional label
    Then PDC disables that date for entry

### Critères d’acceptation
- Vue par année, création/suppression rapide.

### Règles de gestion
- Unicité par date et pays (si multi‑pays ultérieur).

### Interface
- Liste annuelle + calendrier, actions inline.


---

## US-017 — Audit & export des journaux

**En tant que** admin,
**je veux** exporter l’audit (qui/quand/quoi),
**afin de** répondre aux exigences de conformité.

**INVEST** ✅

### Description (Gherkin)
Feature: Audit export
  Scenario: Export filtered audit
    Given filters by date range and user
    When I export
    Then I receive a CSV with events and before/after where applicable

### Critères d’acceptation
- Filtres par période, utilisateur, action.
- Export CSV horodaté, sécurisé.

### Règles de gestion
- Pas d’exposition de secrets/PII non nécessaires.

### Interface
- Tableau filtres + bouton Export, indicateur de volume.


---

## US-018 — Notifications (in‑app) MVP

**En tant que** utilisateur,
**je veux** visualiser une cloche avec compteur et une liste de notifications,
**afin de** être alerté des événements importants.

**INVEST** ✅

### Description (Gherkin)
Feature: In-app notifications
  Scenario: Unread counter
    Given I have unread critical notifications
    When I open the app
    Then the bell icon shows a bold counter
    And the list displays critical first

### Critères d’acceptation
- Types: critiques (non désactivables) et optionnelles (préférences).
- Marquer comme lu/non lu.

### Règles de gestion
- Retention configurable (ex: 90 jours).

### Interface
- Icône cloche (badge), panneau latéral list, filtres, lecteur clavier.


---

## US-019 — Import/Export CSV génériques

**En tant que** admin/manager,
**je veux** importer/exporter les entités clés,
**afin de** accélérer les mises à jour de masse.

**INVEST** ✅

### Description (Gherkin)
Feature: Generic CSV import/export
  Scenario: Import with partial errors
    Given a CSV file (≤ 20MB)
    When I run a dry-run
    Then I see errors per line
    And I can choose to reject all or accept valid lines only

### Critères d’acceptation
- Dry‑run obligatoire, rapport d’erreurs téléchargeable.
- Exports filtrables, respect RBAC/ABAC.

### Règles de gestion
- File queue + worker asynchrone, idempotence par empreinte fichier.

### Interface
- Wizard import, toasts de statut, page « Mes imports » avec historiques.


---

## US-020 — Accessibilité RGAA (socle)

**En tant que** utilisateur,
**je veux** une interface accessible au clavier, contrastée et annonçant les changements,
**afin de** pouvoir l’utiliser quel que soit mon profil.

**INVEST** ✅

### Description (Gherkin)
Feature: Accessibility baseline
  Scenario: Keyboard navigation
    Given I navigate only with the keyboard
    When I traverse grids and forms
    Then focus is always visible and order logical
    And dynamic feedback is announced via aria-live

### Critères d’acceptation
- Contraste AA, focus visibles, labels/erreurs associées, roles ARIA ag‑Grid.
- Tests automatiques (axe-core/pa11y) sans régressions critiques.

### Règles de gestion
- Aucune fonctionnalité MVP ne contourne les guidelines RGAA/WCAG AA.

### Interface
- Utilisation cohérente du design system (typography scale, spacing, buttons, alerts, badges).


---
