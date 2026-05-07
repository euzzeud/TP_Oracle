# TP SQL & PL/SQL Avancé – Oracle / FreeSQL

**Public :** Master 2 Informatique
**Plateforme d'exécution :** [https://freesql.com/](https://freesql.com/) — exclusivement
**Format :** 3 travaux pratiques progressifs, indépendants mais cohérents

---

## Vue d'ensemble

Cette série de trois TP couvre les compétences SQL et PL/SQL attendues d'un Master 2 Informatique sur une base Oracle, en s'appuyant uniquement sur la plateforme en ligne **FreeSQL**. Chaque TP exploite un **schéma différent** et illustre un **paradigme distinct** :

| TP | Schéma | Paradigme | Cas intégrateur | Difficulté |
|---|---|---|---|:---:|
| **[TP1](TP1/README.md)** | **HR** (Human Resources) | Relationnel normalisé, hiérarchies | Audit d'équité salariale | ★★★ |
| **[TP2](TP2/README.md)** | **CO** (Customer Orders) | Transactionnel, séries temporelles | Segmentation client RFM | ★★★★ |
| **[TP3](TP3/README.md)** | **SH** (Sales History) | Data Warehouse, schéma en étoile | Étude d'efficacité promotionnelle | ★★★★★ |

Les trois TP sont **indépendants** : un étudiant peut les traiter dans l'ordre ou en parallèle. Leur progression est néanmoins pensée pour bâtir une couverture cohérente.

---

## Compétences couvertes

### Transverses (présentes dans les trois TP)
- Requêtes SQL avancées : sous-requêtes, CTE, jointures complexes
- **Window functions** (`RANK`, `LAG`, `LEAD`, `NTILE`, agrégats `OVER`)
- Vues de reporting réutilisables
- PL/SQL : fonctions, procédures, **packages** (spec + body)
- Curseurs explicites paramétrés, gestion d'**exceptions** personnalisées
- Idempotence des scripts, modalités de rendu professionnelles

### Spécifiques à chaque TP

| Notion | TP1 | TP2 | TP3 |
|---|:---:|:---:|:---:|
| Hiérarchies (`CONNECT BY`, CTE récursive) | ●●● | ● | – |
| Cohortes & analyses temporelles | – | ●●● | ●● |
| Séries temporelles (`LAG` année N-1, running totals) | – | ●●● | ●● |
| Agrégations OLAP (`ROLLUP`, `CUBE`, `GROUPING SETS`) | – | – | ●●● |
| `PIVOT` / `UNPIVOT` natifs | – | ● | ●●● |
| Vues matérialisées (refresh strategies) | bonus | bonus | ●●● |
| **SCD type 2** avec `MERGE` | – | – | ●●● |
| SQL dynamique sécurisé (`EXECUTE IMMEDIATE` + `DBMS_ASSERT`) | – | – | ●●● |
| Détection d'anomalies (data quality) | ●● | ●● | ●●● |

---

## Détail des TP

### [TP1 — Schéma HR : Audit d'équité salariale](TP1/README.md)

**Domaine :** ressources humaines (employés, départements, postes, hiérarchies managériales).
**Tables principales :** `EMPLOYEES`, `DEPARTMENTS`, `JOBS`, `JOB_HISTORY`, `LOCATIONS`, `COUNTRIES`, `REGIONS`.
**Spécificité :** exploitation de la **hiérarchie manager → employé** via `CONNECT BY` et CTE récursive.
**Cas intégrateur :** détection d'iniquités salariales par rapport à la médiane du poste, rapport via `DBMS_OUTPUT`, recommandations RH.
**Package livré :** `pkg_hr_analytics`.
**Volume :** 23 questions sur 6 parties.

### [TP2 — Schéma CO : Segmentation RFM](TP2/README.md)

**Domaine :** e-commerce (clients, commandes, produits, lignes de commande).
**Tables principales :** `CUSTOMERS`, `ORDERS`, `ORDER_ITEMS`, `PRODUCTS`.
**Spécificité :** analyses **temporelles** (croissance N vs N-1, running totals, **cohortes mensuelles**) et **CTE récursive** pour générer un calendrier exhaustif.
**Cas intégrateur :** segmentation **RFM** complète (Recency, Frequency, Monetary), scoring `NTILE(5)`, segments métier (CHAMPIONS, A_RECONQUERIR, PERDUS…), recommandations marketing.
**Package livré :** `pkg_co_analytics`.
**Volume :** 23 questions sur 6 parties.

### [TP3 — Schéma SH : Étude d'efficacité promotionnelle](TP3/README.md)

**Domaine :** Data Warehouse / ventes multi-canal (schéma en étoile).
**Tables principales :** `SALES` (faits), `PRODUCTS`, `CUSTOMERS`, `TIMES`, `CHANNELS`, `PROMOTIONS`, `COUNTRIES`, `COSTS`.
**Spécificité :** **OLAP** (`ROLLUP`/`CUBE`/`GROUPING SETS`), **vues matérialisées** au cœur du TP, **SCD type 2** sur la dimension produit, **SQL dynamique sécurisé** contre l'injection.
**Cas intégrateur :** étude de **ROI promotionnel** (uplift, cannibalisation), rapport COMEX, recommandation budgétaire 2026.
**Package livré :** `pkg_sh_dw`.
**Volume :** 23 questions sur 6 parties — le plus dense des trois.

---

## Modalités générales

- **Plateforme d'exécution :** FreeSQL **uniquement** (aucune base locale).
- **Rendu :** un fichier `.sql` par TP, déposé dans ce repository (commit sur `etudiants/<nom>` ou Pull Request).
- **Script rejouable** de bout en bout (`CREATE OR REPLACE`, ordre cohérent).
- **En-tête de commentaire** obligatoire devant chaque réponse :

```sql
-- =====================================================
-- Q7 — Top 3 salaires par département (window function)
-- =====================================================
```

- **Note d'analyse** rédigée en commentaire SQL pour les cas intégrateurs (Partie 6 de chaque TP).
- **Chaque TP est noté sur 20 points** selon une grille publiée dans son README.

---

## Pré-requis communs

- SQL standard : `SELECT`, `JOIN`, `GROUP BY`, sous-requêtes
- Bases de PL/SQL : blocs anonymes, `DECLARE / BEGIN / END`
- Compte sur [https://freesql.com/](https://freesql.com/) avec accès au schéma cible
- Activation systématique de `SET SERVEROUTPUT ON SIZE UNLIMITED;` avant tout test PL/SQL

---

## Conseils de progression

- **Pour les étudiants confirmés en SQL :** TP1 sert de calibrage, TP2 et TP3 sont les vrais terrains M2.
- **Pour ceux qui découvrent les hiérarchies récursives :** commencer par le TP1.
- **Pour ceux qui découvrent l'OLAP / le DW :** ne pas commencer par le TP3 ; lire au moins la Partie 1 du TP2 d'abord.
- Les **cas intégrateurs (Partie 6)** sont le cœur de l'évaluation. Ne pas les bâcler en fin de session.

---

## Structure du repository

```
.
├── README.md           ← ce fichier
├── TP1/
│   └── README.md       ← énoncé TP1 (schéma HR)
├── TP2/
│   └── README.md       ← énoncé TP2 (schéma CO)
└── TP3/
    └── README.md       ← énoncé TP3 (schéma SH)
```

Les rendus étudiants viendront s'ajouter dans chaque sous-dossier sous la forme `TP<n>/nom_prenom_TP<n>.sql`.
