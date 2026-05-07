# TP3 SQL & PL/SQL Avancé – Schéma SH (Oracle / FreeSQL)

**Public :** Master 2 Informatique
**Durée indicative :** 8 à 10 heures (le plus dense des trois TP)
**Plateforme d'exécution :** [https://freesql.com/](https://freesql.com/) — schéma **SH (Sales History)** exclusivement
**Pré-requis :** TP1 (HR) et TP2 (CO) vivement recommandés

---

## Pourquoi le schéma SH ?

Le schéma **Sales History** d'Oracle est l'exemple canonique de **schéma en étoile** (star schema) : une table de faits centrale (`SALES`) entourée de dimensions (`PRODUCTS`, `CUSTOMERS`, `TIMES`, `CHANNELS`, `PROMOTIONS`, `COUNTRIES`).

C'est le bon support pour aborder, en complément des TP précédents, les notions M2 propres au **Data Warehousing** et à l'**OLAP** :

- **agrégations multidimensionnelles** : `ROLLUP`, `CUBE`, `GROUPING SETS`, `GROUPING_ID`
- **vues matérialisées** comme outil de production (cache, fast refresh, query rewrite)
- **pivot / unpivot** natifs Oracle
- **dimensions à variation lente (SCD type 2)** avec `MERGE`
- **pattern matching** (`MATCH_RECOGNIZE`, si supporté par FreeSQL)
- **indexation bitmap** et stratégie d'optimisation propre aux DW

> ⚠️ Si le schéma `SH` n'est pas disponible directement sur votre instance FreeSQL : signalez-le à l'enseignant. À défaut, le TP est **transposable au schéma `CO`** en traitant `ORDER_ITEMS` comme la table de faits — la grille d'évaluation et la logique restent valides.

---

## Objectifs pédagogiques

À l'issue de ce TP3, vous serez capables de :

- raisonner et requêter dans un **schéma en étoile**
- construire des **rapports multi-dimensionnels** avec `ROLLUP` / `CUBE` / `GROUPING SETS`
- concevoir et maintenir des **vues matérialisées** (refresh strategies, query rewrite)
- mettre en œuvre une **dimension à variation lente** avec `MERGE`
- modéliser une **stratégie d'indexation** adaptée à un DW (bitmap vs B-tree)
- conduire une étude **d'efficacité promotionnelle** (ROI promo) de bout en bout

---

## Contexte métier

Vous êtes **DW / BI analyst** dans une enseigne multi-canal (boutiques, web, partenaires). La direction commerciale veut :

1. Des **rapports cross-dimensionnels** (produit × canal × temps × promotion) prêts pour le COMEX
2. Un **socle de vues matérialisées** pour soulager la base transactionnelle et accélérer les dashboards
3. Une **étude d'efficacité des promotions** : quelles campagnes ont vraiment été rentables ?
4. Un **mécanisme propre de mise à jour** de la dimension produit lors des changements de catégorie / prix de référence (SCD)

---

## Schéma SH — vue d'ensemble

```
              ┌─────────────┐
              │   TIMES     │   (jour, semaine, mois, trimestre, année — fiscale & calendaire)
              └──────┬──────┘
                     │
┌─────────────┐   ┌──┴──────────┐   ┌─────────────┐
│  PRODUCTS   ├───┤    SALES    ├───┤ CHANNELS    │
│ (catégorie, │   │  (FAIT)     │   │ (web/store/ │
│  sous-cat,  │   │  quantity,  │   │  partner)   │
│  prix réf.) │   │  amount     │   └─────────────┘
└─────────────┘   └──┬───┬──────┘
                     │   │
              ┌──────┴┐  └────────┐
              │CUSTOM.│           │PROMOT.│
              └───────┘           └───────┘
                     │
              ┌──────┴──────┐
              │  COUNTRIES  │
              └─────────────┘
```

**Tables de dimensions :**
- `PRODUCTS` — `prod_id`, `prod_name`, `prod_subcategory`, `prod_category`, `prod_list_price`, `prod_min_price`, `prod_status`...
- `CUSTOMERS` — `cust_id`, `cust_gender`, `cust_year_of_birth`, `cust_marital_status`, `cust_income_level`, `country_id`...
- `TIMES` — `time_id`, `calendar_year`, `calendar_quarter_number`, `calendar_month_number`, `day_name`...
- `CHANNELS` — `channel_id`, `channel_desc`, `channel_class`...
- `PROMOTIONS` — `promo_id`, `promo_name`, `promo_category`, `promo_cost`, `promo_begin_date`, `promo_end_date`...
- `COUNTRIES` — `country_id`, `country_name`, `country_region`...

**Tables de faits :**
- `SALES` — `prod_id`, `cust_id`, `time_id`, `channel_id`, `promo_id`, `quantity_sold`, `amount_sold`
- `COSTS` — `prod_id`, `time_id`, `promo_id`, `channel_id`, `unit_cost`, `unit_price`

> ⚠️ Les noms peuvent légèrement différer sur votre instance FreeSQL — **Q0 impose de tout vérifier avec `DESC`**.

---

## Modalités de rendu

- Plateforme d'exécution : **FreeSQL uniquement**
- Rendu : **un seul fichier** `nom_prenom_TP3.sql`, déposé dans ce repository (PR ou commit sur `etudiants/<nom>`)
- Script **rejouable de bout en bout** sur FreeSQL
- En-tête de commentaire obligatoire devant chaque réponse
- Pour les questions d'analyse (Q23) : commentaire SQL structuré

---

## Échauffement (non noté)

**Q0.** Inventaire du schéma SH :
1. `DESC` sur **chaque** table (faits + dimensions). Noter les colonnes clés en commentaire.
2. Compter les lignes de `SALES` et noter la **plage temporelle** couverte (`MIN`/`MAX(time_id)`).
3. Identifier la **cardinalité** de chaque dimension (`COUNT(DISTINCT ...)` sur la clé étrangère côté `SALES`).
4. Vérifier qu'il existe au moins une promotion non triviale (`promo_id != 999` ou équivalent — la promo « no promotion » par défaut).

---

# Partie 1 — OLAP : agrégations multidimensionnelles

> **Objectif :** maîtriser les extensions `GROUP BY` propres à l'OLAP — incontournables en DW.

### Q1. `ROLLUP` produit × année
Calculer les ventes (`SUM(amount_sold)`) ventilées par catégorie produit × année calendaire, **avec sous-totaux par catégorie et un total général**, en une seule requête (`GROUP BY ROLLUP(...)`). Utiliser `GROUPING(...)` pour étiqueter les lignes de sous-totaux (`'TOTAL CATÉGORIE'`, `'TOTAL GÉNÉRAL'`).

### Q2. `CUBE` canal × promotion
Calculer les ventes ventilées par canal × catégorie de promotion, avec **toutes les combinaisons de sous-totaux** (`GROUP BY CUBE(...)`). Utiliser `GROUPING_ID` pour générer un code identifiant chaque niveau d'agrégation et trier par ce code.

### Q3. `GROUPING SETS` ciblé
On veut **uniquement** les agrégations suivantes (pas les autres) :
- ventes par année
- ventes par catégorie produit
- ventes par couple (année, catégorie)
- total général

Écrire la requête en utilisant `GROUPING SETS` (et pas `CUBE`, qui produirait trop de combinaisons).

### Q4. Ranking dimensionnel
Pour chaque catégorie produit, retourner le **top 5 des sous-catégories** par CA, avec le **rang dense** et la **part de la sous-catégorie dans sa catégorie** (en %). Utiliser `DENSE_RANK()` et `RATIO_TO_REPORT(...)` ou un calcul manuel.

### Q5. Évolution YoY par catégorie
Pour chaque catégorie × année, afficher : CA, CA de l'année précédente (`LAG`), **croissance en %**, et **flag** `'EN_PROGRESSION'` / `'EN_DECLIN'` / `'STABLE'` (seuil ±5 %).

### Q6. Saisonnalité par jour de la semaine
Pour chaque catégorie, afficher la **répartition des ventes par jour de la semaine** (lun→dim) en **pourcentage** de la catégorie. Mettre en évidence le jour le plus fort (`FIRST_VALUE(... ORDER BY ... DESC)` ou flag).

---

# Partie 2 — Pivot, unpivot & analyses transverses

> **Objectif :** restructurer les données pour le reporting — pivot natif Oracle.

### Q7. Pivot canal × année
Construire un tableau croisé : **une ligne par catégorie produit**, **une colonne par canal** (`'STORE'`, `'INTERNET'`, `'PARTNERS'`, etc.), valeurs = CA. Utiliser la clause native `PIVOT (...)`.

### Q8. Pivot mois × année
Tableau croisé **mois (1–12) × année** sur la masse `amount_sold`. Une ligne par mois, une colonne par année. Identifier visuellement (en commentaire SQL) le **mois le plus saisonnier** et l'**année la plus dynamique**.

### Q9. Unpivot
On dispose virtuellement (issue de Q7) d'un tableau large catégorie × canaux. Écrire une requête `UNPIVOT` qui le ramène en format long (`category, channel, amount`). Justifier en commentaire **quand ce reformat est utile** (alimentation d'un BI, normalisation pour ML).

### Q10. Cohorte client par âge
Bucketer les clients par **tranche d'âge** (`< 30`, `30-45`, `45-60`, `60+`) calculée à partir de `cust_year_of_birth`, et croiser avec la **catégorie de produit** : matrice tranche × catégorie, valeurs = CA et nombre de clients distincts. Utiliser `CASE` + agrégation conditionnelle ou `PIVOT`.

---

# Partie 3 — Vues matérialisées & stratégie de cache

> **Objectif :** comprendre quand, pourquoi et comment matérialiser un agrégat.

> Sur FreeSQL, vérifier la disponibilité de `CREATE MATERIALIZED VIEW`. À défaut, créer une **table de cache** rafraîchie via une procédure (`pr_refresh_cache_*`) et le mentionner explicitement en commentaire.

### Q11. `MV_SALES_BY_MONTH_CATEGORY`
Vue matérialisée à grain **(mois calendaire, catégorie produit)** contenant : nombre de transactions, quantité totale, CA total, panier moyen.
- Stratégie de refresh : `REFRESH COMPLETE ON DEMAND` — justifier ce choix en commentaire (vs `FAST` / `ON COMMIT`).
- Ajouter une **procédure** `pr_refresh_mv_sales_month` qui rafraîchit la vue (`DBMS_MVIEW.REFRESH(...)`).

### Q12. `MV_TOP_PRODUCTS_YOY`
Vue matérialisée à grain produit contenant : CA année N, CA année N-1, croissance %, rang dans la catégorie sur N. Utiliser des window functions à l'intérieur de la définition.

### Q13. `MV_PROMO_ROI`
Vue matérialisée centrale du TP, à grain **(promo_id, prod_category, channel_id)** :
- nombre de transactions promo
- CA promo
- coût de la promotion (`PROMOTIONS.promo_cost`, distribué au prorata des ventes)
- coût marchandise (`COSTS.unit_cost × quantity_sold`)
- **ROI** = (CA − coût promo − coût marchandise) / coût promo
- comparaison vs ventes hors promo sur le même couple (catégorie, canal) sur la même fenêtre temporelle

Justifier en commentaire le **grain choisi** et pourquoi cette MV mérite d'être matérialisée plutôt que recalculée.

---

# Partie 4 — Fonctions PL/SQL DW

> **Objectif :** encapsuler la logique métier DW dans des fonctions réutilisables.

### Q14. `fn_sales_for_period`
`fn_sales_for_period(p_start IN DATE, p_end IN DATE, p_channel_id IN NUMBER DEFAULT NULL) RETURN NUMBER` :
- retourne le CA total sur la fenêtre `[p_start, p_end]`
- filtre optionnellement sur un canal
- lève `VALUE_ERROR` si `p_start > p_end`
- gère `NO_DATA_FOUND` en retournant `0`

### Q15. `fn_promo_roi`
`fn_promo_roi(p_promo_id IN NUMBER) RETURN NUMBER` :
- retourne le ROI d'une promotion (cf. formule Q13)
- retourne `NULL` si la promo n'a généré aucune vente
- lève `e_promo_not_found` si l'ID n'existe pas

### Q16. `fn_dimension_completeness`
Fonction de **diagnostic DW** : `fn_dimension_completeness(p_dimension_table IN VARCHAR2) RETURN NUMBER` :
- accepte `'PRODUCTS'`, `'CUSTOMERS'`, `'TIMES'`, `'CHANNELS'`, `'PROMOTIONS'`
- retourne le **pourcentage de clés** de la dimension qui apparaissent au moins une fois dans `SALES` (mesure du taux d'utilisation de la dimension)
- utiliser `EXECUTE IMMEDIATE` (SQL dynamique) — c'est l'occasion d'introduire ce pattern
- lever une exception si le nom de table fourni n'est pas dans la liste autorisée (sécurité : éviter une injection via `EXECUTE IMMEDIATE`)

---

# Partie 5 — Procédures, MERGE & SCD

> **Objectif :** gestion d'une dimension à variation lente (SCD type 2) et package final.

### Q17. `pr_refresh_all_mv`
Procédure orchestrant le refresh de toutes les MV créées en Partie 3, dans le bon ordre, avec **chronométrage** (`DBMS_OUTPUT` du temps écoulé par MV) et **gestion d'exceptions** ligne à ligne (l'échec d'une MV ne bloque pas les suivantes).

### Q18. `pr_apply_scd2_product` — **SCD type 2 sur PRODUCTS**

Pour gérer un **changement de catégorie ou de prix de référence** d'un produit, on **ne veut pas** écraser l'historique. Mettre en place une SCD type 2 :

1. Créer une table d'historique `PRODUCTS_HIST` avec : tous les attributs de `PRODUCTS`, plus `version_id`, `valid_from`, `valid_to`, `is_current` (`'Y'`/`'N'`).
2. Initialiser `PRODUCTS_HIST` avec une version courante de chaque produit actuel.
3. Écrire la procédure `pr_apply_scd2_product(p_prod_id NUMBER, p_new_category VARCHAR2, p_new_list_price NUMBER)` qui :
   - **clôture** la version courante (`valid_to = SYSDATE`, `is_current = 'N'`)
   - **crée une nouvelle version** courante avec les nouvelles valeurs
   - utilise une instruction **`MERGE`** pour la clôture
   - rejette le changement si aucun champ ne diffère (idempotence)
4. Démontrer le bon fonctionnement avec 2-3 mises à jour de test, et une requête finale qui affiche **l'historique complet** d'un produit modifié.

### Q19. Package `pkg_sh_dw`
Spécification + corps regroupant :

- `fn_sales_for_period`, `fn_promo_roi`, `fn_dimension_completeness`
- `pr_refresh_all_mv`
- `pr_apply_scd2_product`
- une nouvelle procédure `pr_data_quality_report` qui :
  - parcourt `SALES` via curseur
  - identifie les anomalies : montant ≤ 0, quantité ≤ 0, FK orphelines (clés qui ne joignent dans aucune dimension), commandes à des dates hors plage `TIMES`
  - imprime un rapport synthétique via `DBMS_OUTPUT`

---

# Partie 6 — Cas intégrateur : Étude d'efficacité promotionnelle

> **Objectif :** mobiliser tout ce qui précède pour **arbitrer** sur les promotions à reconduire en 2026.

**Contexte.** La direction commerciale a investi sur ~50 campagnes promotionnelles ces dernières années. Coût total cumulé : conséquent. Elle vous demande de **trier le bon grain de l'ivraie** :

> *« Quelles promotions ont réellement créé de la valeur ? Lesquelles ont seulement déplacé du CA déjà acquis ? Lesquelles devons-nous abandonner ? »*

### Q20. CTE consolidée `promo_kpi`
Une CTE retournant, pour chaque promotion :
- nom, catégorie, durée (jours)
- nombre de transactions, CA promo, quantité vendue
- `promo_cost` (de `PROMOTIONS`)
- coût marchandise (jointure `COSTS`)
- **ROI** (cf. Q13)
- **uplift estimé** : CA observé pendant la promo vs **CA estimé sans promo** (moyenne du même produit hors promo, sur fenêtre comparable de 30j avant/après)
- nombre de **clients uniques touchés** ; **% nouveaux clients** (1ère commande pendant la promo)

### Q21. Vue `V_PROMO_VERDICT`
S'appuyant sur `MV_PROMO_ROI` (Q13) et `promo_kpi` (Q20), produire une vue avec :
- toutes les KPI ci-dessus
- un **`verdict`** :
  - `'STAR'` : ROI ≥ 2.0 et uplift ≥ 30 %
  - `'A_RECONDUIRE'` : ROI ≥ 1.0 et uplift ≥ 10 %
  - `'NEUTRE'` : ROI entre 0.7 et 1.0 (cannibalisation possible)
  - `'A_ABANDONNER'` : ROI < 0.7 ou uplift négatif
- un score composite `verdict_score` (numérique, pour tri)

### Q22. Procédure `pr_promo_executive_report`
Procédure produisant via `DBMS_OUTPUT` un **rapport COMEX** :

```
============================================================
       RAPPORT D'EFFICACITÉ PROMOTIONNELLE — 2020-2024
============================================================
Périmètre analysé : 47 promotions / Coût cumulé 2.4 M€
CA promo cumulé   : 18.3 M€  |  ROI moyen pondéré : 1.42

Répartition des verdicts :
  STAR             :  8 promos  (17%)  ROI moy. 3.1
  A_RECONDUIRE     : 21 promos  (45%)  ROI moy. 1.5
  NEUTRE           : 11 promos  (23%)  ROI moy. 0.8
  A_ABANDONNER     :  7 promos  (15%)  ROI moy. 0.4

--- TOP 5 STARS ---
  - Spring Sale 2023 (Software/Other)  ROI 4.2  uplift +52%  150K€ marge
  - ...

--- A_ABANDONNER (par coût décroissant) ---
  - Winter Promo 2021 (Hardware)  ROI 0.3  uplift -8%  coût 80K€ → 56K€ perdus
  - ...
============================================================
```

### Q23. Note d'analyse stratégique (commentaire SQL, ~25 lignes)
Dans le fichier de rendu, en commentaire structuré :

1. **3 leçons** tirées de l'analyse (ex : « les promos catégorie Hardware ont un ROI 2× supérieur à Software », « les campagnes < 7 jours sous-performent »...)
2. **Hypothèse de cannibalisation** : les promos déclarées `'NEUTRE'` ont-elles vraiment apporté du CA neuf, ou simplement déplacé des ventes qui auraient eu lieu de toute façon ? Comment l'arbitrer rigoureusement ?
3. **Recommandation budgétaire 2026** : sur la base de l'analyse, comment réallouer les budgets ? Soyez concret (« renforcer X de +20 % ; supprimer Y ; tester Z »).
4. **Limites** : qu'est-ce que cette analyse **ne capture pas** ? (effet de marque long terme, effet promo concurrente, biais de sélection de la cible promo, etc.)

---

# Critères d'évaluation (sur 20)

| Bloc | Critères | Points |
|---|---|:---:|
| **Q0** | Inventaire schéma & vérification des cardinalités | 1 |
| **Partie 1** | OLAP — `ROLLUP` / `CUBE` / `GROUPING SETS` maîtrisés | 3 |
| **Partie 2** | Pivot / unpivot / cohortes | 2 |
| **Partie 3** | Vues matérialisées — choix de grain et stratégie justifiés | 3 |
| **Partie 4** | Fonctions PL/SQL DW + SQL dynamique sécurisé (Q16) | 2 |
| **Partie 5** | SCD type 2 + package + qualité du diagnostic Q19 | 3 |
| **Partie 6** | Étude promo — qualité analytique et stratégique | 4 |
| **Qualité** | Lisibilité, idempotence, cohérence d'API | 2 |
| **Total** |  | **20** |

**Pénalités :**
- Code non rejouable sur FreeSQL : **−2 pts**
- Q16 vulnérable à une injection SQL via `EXECUTE IMMEDIATE` : **−2 pts**
- Note d'analyse Q23 absente ou descriptive sans recommandation : **−2 pts**
- Recopie non créditée entre étudiants : **0**

---

# Conseils & spécificités FreeSQL / Oracle

**Patterns OLAP utiles :**

```sql
-- ROLLUP / CUBE / GROUPING SETS
GROUP BY ROLLUP(prod_category, calendar_year)
GROUP BY CUBE(channel_id, prod_category)
GROUP BY GROUPING SETS ((calendar_year), (prod_category), (calendar_year, prod_category), ())

-- Identifier les niveaux d'agrégation
GROUPING(col)        -- 1 si la colonne est agrégée à ce niveau, 0 sinon
GROUPING_ID(c1, c2)  -- code binaire combiné

-- Pivot natif
SELECT * FROM (
  SELECT prod_category, channel_id, amount_sold FROM sales s JOIN ...
)
PIVOT (SUM(amount_sold) FOR channel_id IN (2 AS store, 3 AS internet, 4 AS partners));

-- Vue matérialisée
CREATE MATERIALIZED VIEW mv_x
  REFRESH COMPLETE ON DEMAND
  AS SELECT ...;
EXEC DBMS_MVIEW.REFRESH('MV_X', 'C');

-- SQL dynamique sécurisé
EXECUTE IMMEDIATE 'SELECT COUNT(DISTINCT ' || DBMS_ASSERT.SIMPLE_SQL_NAME(p_col)
                 || ') FROM ' || DBMS_ASSERT.SIMPLE_SQL_NAME(p_table)
   INTO v_count;
```

**Pièges à éviter :**

- **`SYSDATE` proscrit** pour les calculs temporels — les données SH peuvent dater. S'aligner sur `MAX(time_id)` du fait `SALES`.
- Sur FreeSQL, certaines fonctionnalités peuvent être **bridées** (vues matérialisées avec `FAST REFRESH`, `MATCH_RECOGNIZE`, `MODEL`). Si bloqué, **document the workaround** en commentaire — c'est valorisé, pas pénalisé.
- Une promo « no promotion » (souvent `promo_id = 999`) **biaise les agrégats** si vous ne l'excluez pas. Toujours filtrer ou la traiter explicitement.
- `GROUPING_ID` est plus fiable que de tester `GROUPING(col)` colonne par colonne pour identifier les niveaux d'un cube.
- Une MV avec `REFRESH FAST` exige des **MV logs** sur les tables sources — souvent indisponible sur les schémas en lecture comme `SH` sur FreeSQL.

---

# Pour aller plus loin (hors barème, valorisé)

- **`MATCH_RECOGNIZE`** : détecter les **séquences d'achat** (3 commandes croissantes en montant sur 6 mois → client en ascension). Implémenter une telle requête si la fonctionnalité est disponible.
- **Clause `MODEL`** : Oracle exclusif. Réécrire Q5 (croissance YoY) avec `MODEL` et discuter en commentaire **quand `MODEL` apporte vs window functions**.
- **Stratégie d'indexation** : pour `SALES`, justifier en commentaire (sans exécution) :
  - quels index **bitmap** créer sur les FK de dimensions
  - pourquoi B-tree serait pénalisant sur ces colonnes en DW
  - quel index composite pour accélérer `MV_PROMO_ROI`
- **Pipelined function** : `fn_top_promos_pipelined(p_year IN NUMBER, p_n IN NUMBER)` retournant un curseur du top-N promos d'une année.

---

# Articulation TP1 / TP2 / TP3

| | **TP1 (HR)** | **TP2 (CO)** | **TP3 (SH)** |
|---|---|---|---|
| Domaine | Ressources humaines | E-commerce / orders | Data Warehouse / OLAP |
| Modèle | Relationnel normalisé | Transactionnel | **Schéma en étoile** |
| Cas intégrateur | Audit d'équité salariale | Segmentation RFM | Efficacité promotionnelle |
| Spécificité technique | Hiérarchies (`CONNECT BY`) | Cohortes, séries temporelles | **OLAP, MV, SCD, pivot** |
| Package | `pkg_hr_analytics` | `pkg_co_analytics` | `pkg_sh_dw` |
| Difficulté relative | ★★★ | ★★★★ | **★★★★★** |

Les trois TP forment une progression cohérente couvrant l'essentiel de ce qu'on attend d'un M2 Informatique sur Oracle / SQL avancé.

---

# Fin du TP3
