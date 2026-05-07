# TP2 SQL & PL/SQL Avancé – Schéma CO (Oracle / FreeSQL)

**Public :** Master 2 Informatique
**Durée indicative :** 6 à 8 heures
**Plateforme d'exécution :** [https://freesql.com/](https://freesql.com/) — schéma **CO** (exclusivement)
**Pré-requis :** TP1 (schéma HR) recommandé mais non obligatoire

---

## Objectifs pédagogiques

À l'issue de ce TP2, vous serez capables de :

- mener des **analyses commerciales avancées** (CA, panier moyen, top produits, saisonnalité)
- maîtriser les **analyses temporelles** : cohortes, retention, running totals, comparaisons N / N-1
- construire des **indicateurs e-commerce** prêts pour un dashboard
- développer du PL/SQL métier orienté **CRM / commerce** (loyauté, LTV, segmentation)
- conduire une étude **RFM** (Recency, Frequency, Monetary) de bout en bout

---

## Contexte métier

Vous êtes **data analyst e-commerce** dans une entreprise qui vend des produits en ligne à une base de clients internationale. La direction marketing vous mandate pour :

1. **comprendre la dynamique des ventes** (tendances, saisonnalité, top produits)
2. **segmenter les clients** afin de cibler les campagnes (RFM)
3. **détecter les anomalies** (commandes suspectes, clients dormants, produits en perte de vitesse)
4. **industrialiser** ces analyses dans des vues et un package PL/SQL réutilisables

---

## Schéma CO (Customer Orders)

Le schéma `CO` de FreeSQL contient (typiquement) les tables :

- `CUSTOMERS` — fiches clients
- `ORDERS` — en-têtes de commandes
- `ORDER_ITEMS` — lignes de commande (détail par produit)
- `PRODUCTS` — catalogue produit

> ⚠️ **Première étape obligatoire :** sur FreeSQL, exécuter `DESC <table>` pour **chaque table** afin d'identifier les noms exacts des colonnes (clés primaires/étrangères, types, dates). Adaptez vos requêtes en conséquence — les énoncés ci-dessous emploient les noms les plus courants (`customer_id`, `order_date`, `unit_price`, `quantity`...) mais **votre instance FreeSQL fait foi**.

---

## Modalités de rendu

- Plateforme d'exécution : **FreeSQL uniquement**
- Rendu : **un seul fichier** `nom_prenom_TP2.sql`, déposé dans ce repository (PR ou commit sur `etudiants/<nom>`)
- Script **rejouable de bout en bout** sur FreeSQL (`CREATE OR REPLACE`, ordre cohérent)
- Chaque réponse précédée d'un en-tête :

```sql
-- =====================================================
-- Q8 — Analyse de cohorte mensuelle (retention N+1, N+3, N+6)
-- =====================================================
```

- Capture(s) d'écran FreeSQL : optionnel, valorisé pour la Partie 6.

---

## Échauffement (non noté)

**Q0.** Pour chacune des 4 tables du schéma CO :
1. Exécuter `DESC <table>` et **noter les colonnes** dans un commentaire SQL en tête de fichier.
2. Compter les lignes (`SELECT COUNT(*) FROM ...`).
3. Identifier la **plage temporelle** des commandes (`MIN(order_date)`, `MAX(order_date)`).

> Cette étape conditionne la qualité de tout le reste du TP. Ne pas la sauter.

---

# Partie 1 — Analyses commerciales & temporelles

> **Objectif :** maîtriser le fenêtrage Oracle appliqué à des séries temporelles de ventes.

### Q1. Top 10 clients par chiffre d'affaires
Lister les 10 clients ayant généré le plus gros CA cumulé (somme `quantity × unit_price` sur toutes leurs commandes). Afficher : client, CA total, nombre de commandes, panier moyen.

### Q2. Évolution mensuelle du CA
Pour chaque mois (sur toute la période), afficher : mois (`TRUNC(order_date, 'MM')`), CA, nombre de commandes, nombre de clients distincts. Ordonner chronologiquement.

### Q3. Croissance N vs N-1
Pour chaque mois, afficher le CA du mois courant, le CA du **même mois l'année précédente** (`LAG(..., 12)`), et la **croissance en %**. Gérer les NULL pour la première année.

### Q4. Running total
Afficher le **CA cumulé jour par jour** depuis le premier jour d'activité (`SUM(...) OVER (ORDER BY ...)`). Calculer aussi la **moyenne glissante sur 7 jours**.

### Q5. Top produits par catégorie (ou par tranche de prix)
Pour chaque catégorie produit (ou, à défaut, chaque tranche de prix `< 50`, `50–200`, `> 200`), retourner les **3 produits les plus vendus** en quantité, avec leur rang. Utiliser `RANK()`/`DENSE_RANK()`.

### Q6. Délai inter-commandes par client
Pour chaque client ayant passé au moins 2 commandes, calculer le **délai moyen entre deux commandes consécutives** (en jours). Utiliser `LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)`.

---

# Partie 2 — CTE, cohortes & analyses temporelles

> **Objectif :** structurer des analyses complexes en couches lisibles via `WITH`, et construire une analyse de cohorte.

### Q7. Première commande par client
Avec une CTE `first_order`, calculer pour chaque client la **date de sa première commande** (`MIN(order_date)`) et le **mois d'acquisition** (`TRUNC(..., 'MM')`).

### Q8. Cohortes mensuelles & retention
Construire une **table de retention par cohorte** :
- une ligne par mois d'acquisition × écart en mois (M+0, M+1, M+3, M+6, M+12)
- chaque cellule = nombre de clients de la cohorte ayant repassé commande dans le mois cible
- s'appuyer sur `MONTHS_BETWEEN` et plusieurs CTE chaînées

### Q9. Calendrier exhaustif (CTE récursive)
Générer une **table calendrier** contenant toutes les dates entre `MIN(order_date)` et `MAX(order_date)` du schéma, via une **CTE récursive** (`WITH cal (d) AS (... UNION ALL ...)`). L'utiliser pour :
- détecter les **jours sans aucune commande**
- calculer le CA jour par jour avec `0` les jours creux (LEFT JOIN sur le calendrier)

### Q10. Clients dormants
Lister tous les clients **sans commande depuis plus de 90 jours** par rapport à la dernière date connue du schéma. Afficher : client, dernière commande, jours d'inactivité, CA historique total. Trier par CA historique décroissant (cibles prioritaires de réactivation).

---

# Partie 3 — Vues décisionnelles

> **Objectif :** exposer des indicateurs prêts à être consommés par un outil BI.

### Q11. `V_CUSTOMER_360`
Vue à grain client contenant :
- identité (id, nom, email, pays)
- date de première commande, date de dernière commande
- nombre total de commandes
- CA total, panier moyen
- jours depuis la dernière commande
- statut (`'ACTIF'` < 90j, `'TIEDE'` 90–180j, `'DORMANT'` > 180j)

### Q12. `V_PRODUCT_PERFORMANCE`
Vue à grain produit contenant :
- identité produit
- quantité totale vendue
- CA total
- nombre de clients distincts ayant acheté
- **rang du produit** par CA dans sa catégorie (window function dans la vue)
- mois de la **dernière vente**

### Q13. `V_DAILY_SALES_DASHBOARD`
Vue à grain journalier (avec calendrier exhaustif de Q9) contenant :
- date
- CA du jour
- nombre de commandes
- nombre de clients distincts
- panier moyen
- CA cumulé sur 7 jours glissants

---

# Partie 4 — Fonctions PL/SQL métier

> **Objectif :** écrire des fonctions métier réutilisables avec gestion d'exceptions.

### Q14. `fn_customer_ltv`
Fonction `fn_customer_ltv(p_customer_id IN NUMBER) RETURN NUMBER` :
- retourne la **Lifetime Value** du client (CA cumulé historique)
- retourne `0` si le client existe mais n'a jamais commandé
- lève une exception personnalisée `e_customer_not_found` si le client n'existe pas

### Q15. `fn_loyalty_tier`
Fonction `fn_loyalty_tier(p_customer_id IN NUMBER) RETURN VARCHAR2` retournant le palier de fidélité :
- `'PLATINUM'` si LTV ≥ 10 000
- `'GOLD'` si 5 000 ≤ LTV < 10 000
- `'SILVER'` si 1 000 ≤ LTV < 5 000
- `'BRONZE'` sinon
- réutiliser `fn_customer_ltv`

### Q16. `fn_days_since_last_order`
Fonction `fn_days_since_last_order(p_customer_id IN NUMBER) RETURN NUMBER` :
- retourne le nombre de jours depuis la dernière commande, calculé par rapport à `MAX(order_date)` du schéma (et **non** `SYSDATE` — les données peuvent être anciennes)
- retourne `NULL` si le client n'a jamais commandé
- gérer `NO_DATA_FOUND` proprement

---

# Partie 5 — Procédures & Package

> **Objectif :** organiser le code en package, manipuler des curseurs explicites, formaliser une API métier.

### Q17. `pr_apply_promotion`
`pr_apply_promotion(p_product_id IN NUMBER, p_percent IN NUMBER, p_min_price IN NUMBER DEFAULT 0)` :
- applique une remise en % sur `unit_price` du produit (mise à jour `PRODUCTS`)
- valide que `p_percent` est dans `]0, 80]` (sinon `RAISE_APPLICATION_ERROR(-20010, ...)`)
- vérifie que le **prix après remise reste ≥ `p_min_price`** (sinon `-20011`)
- vérifie l'existence du produit (sinon `-20012`)
- ne fait **pas** de `COMMIT`

### Q18. `pr_customer_summary`
`pr_customer_summary(p_customer_id IN NUMBER)` qui imprime via `DBMS_OUTPUT` une fiche structurée :

```
┌────────────────────────────────────────────┐
│  Client #1042 — DUPONT Marie  (FR)
│  Tier        : GOLD
│  LTV         : 6 420.50
│  Commandes   : 18  |  Panier moyen : 356.70
│  Première    : 14-MAR-2022
│  Dernière    : 02-NOV-2024  (62 jours)
│  Statut      : ACTIF
│  Top 3 prod. : Casque BT (×4) | Webcam HD (×2) | ...
└────────────────────────────────────────────┘
```

(Réutiliser `fn_customer_ltv`, `fn_loyalty_tier`, `fn_days_since_last_order`. Le top 3 produits via curseur explicite.)

### Q19. Package `pkg_co_analytics`
Spécification + corps du package exposant :

- `fn_customer_ltv` (Q14)
- `fn_loyalty_tier` (Q15)
- `fn_days_since_last_order` (Q16)
- `pr_apply_promotion` (Q17)
- `pr_customer_summary` (Q18)
- une nouvelle procédure `pr_alert_anomalous_orders(p_threshold IN NUMBER DEFAULT 5000)` qui parcourt via **curseur paramétré** les commandes dont le montant dépasse `p_threshold` **OU** dont le nombre de lignes est anormalement élevé (> 20), et imprime une alerte par commande.

> Toutes les routines suivantes doivent être appelées via le package : `pkg_co_analytics.fn_loyalty_tier(...)`.

---

# Partie 6 — Cas intégrateur : Segmentation RFM

> **Objectif :** mener une étude marketing complète mobilisant tout ce qui précède.

**Contexte.** Le marketing souhaite **segmenter la base clients** pour cibler les campagnes 2026. Vous devez livrer une segmentation **RFM** (Recency, Frequency, Monetary) avec recommandations actionnables.

### Q20. Calcul des composantes RFM
Avec une CTE `customer_rfm`, calculer pour chaque client :
- **R** = jours depuis la dernière commande (Recency, plus petit = mieux)
- **F** = nombre de commandes sur les 12 derniers mois (Frequency)
- **M** = CA sur les 12 derniers mois (Monetary)

> Attention : « 12 derniers mois » est calculé par rapport à `MAX(order_date)` du schéma, pas à `SYSDATE`.

### Q21. Vue `V_CUSTOMER_RFM`
Vue qui enrichit Q20 avec un **score 1 à 5** sur chaque axe via `NTILE(5)` :
- `R_score` : 5 = très récent, 1 = très ancien
- `F_score` : 5 = très fréquent, 1 = peu fréquent
- `M_score` : 5 = gros panier, 1 = petit
- `rfm_code` = concaténation (ex : `'555'`, `'413'`...)
- `segment` :
  - `'CHAMPIONS'` si R ≥ 4 et F ≥ 4 et M ≥ 4
  - `'LOYAUX'` si F ≥ 4 et M ≥ 3
  - `'A_RECONQUERIR'` si R ≤ 2 et F ≥ 3
  - `'NOUVEAUX'` si R ≥ 4 et F ≤ 2
  - `'A_RISQUE'` si R ≤ 2 et M ≥ 3
  - `'PERDUS'` si R ≤ 2 et F ≤ 2 et M ≤ 2
  - `'AUTRES'` sinon

### Q22. Procédure `pr_rfm_report`
Procédure qui prend un `p_segment IN VARCHAR2 DEFAULT NULL` (ou `NULL` = tous segments) et imprime via `DBMS_OUTPUT` un rapport :

```
=== SEGMENTATION RFM — Snapshot au <max_date> ===
Effectif total : 1 240 clients

Segment        | Effectif | % | CA total      | Panier moy. | Recency moy.
---------------+----------+---+---------------+-------------+-------------
CHAMPIONS      |      87  | 7%| 1 245 600     |   3 240     |   18 j
LOYAUX         |     210  |17%|   856 100     |   1 875     |   45 j
A_RECONQUERIR  |     142  |11%|   312 400     |   1 120     |  187 j
...

--- Détail segment CHAMPIONS (top 10 par CA) ---
  - DUPONT Marie  (#1042)  LTV 14 200  R=12j  F=22  M=14200
  - ...
```

### Q23. Note d'analyse (commentaire SQL, ~20 lignes)
Dans le fichier de rendu, en commentaire structuré, rédiger :

1. **Distribution observée** : quel segment domine ? quelle dispersion ?
2. **3 actions marketing** recommandées, une par segment-clé (ex : campagne email pour `A_RECONQUERIR`, programme VIP pour `CHAMPIONS`, offre de bienvenue pour `NOUVEAUX`)
3. **Limites de la méthode** : qu'est-ce que RFM **ne capture pas** (saisonnalité produit, B2B vs B2C, cycle de vie produit) ?
4. **Une métrique complémentaire** que vous proposeriez d'ajouter (ex : diversité de catégories achetées, taux de retour, NPS si dispo).

---

# Critères d'évaluation (sur 20)

| Bloc | Critères | Points |
|---|---|:---:|
| **Q0** | Inventaire schéma rigoureux et exploité dans la suite | 1 |
| **Partie 1** | Window functions / temporel — idiomatiques | 3 |
| **Partie 2** | CTE, récursivité, cohortes | 3 |
| **Partie 3** | Vues — pertinence, factorisation | 2 |
| **Partie 4** | Fonctions — exceptions, robustesse | 2 |
| **Partie 5** | Package & curseurs — cohérence d'API | 3 |
| **Partie 6** | Étude RFM — qualité de l'analyse et des recommandations | 4 |
| **Qualité** | Lisibilité, nommage, idempotence, commentaires | 2 |
| **Total** |  | **20** |

**Pénalités :**
- Code non rejouable sur FreeSQL : **−2 pts**
- Échauffement Q0 absent ou bâclé : **−1 pt**
- Recopie non créditée entre étudiants : **0**

---

# Conseils & spécificités FreeSQL / Oracle

**Patterns Oracle utiles :**

```sql
-- Tronquer une date au mois / trimestre / année
TRUNC(order_date, 'MM')   -- 1er du mois
TRUNC(order_date, 'Q')    -- 1er du trimestre
TRUNC(order_date, 'YYYY') -- 1er janvier

-- Différence en mois
MONTHS_BETWEEN(date1, date2)

-- Différence en jours
date1 - date2

-- Sortie PL/SQL
SET SERVEROUTPUT ON SIZE UNLIMITED;

-- Concaténation d'agrégat
LISTAGG(product_name, ', ') WITHIN GROUP (ORDER BY quantity DESC)

-- Pivot
SELECT * FROM ( ... ) PIVOT (SUM(amount) FOR month IN ('JAN', 'FEB', ...))
```

**Pièges fréquents :**

- **Ne pas utiliser `SYSDATE`** pour les calculs de recency : les données du schéma CO peuvent dater de plusieurs années. Toujours s'aligner sur `MAX(order_date)` (à mettre dans une CTE ou variable locale).
- Vérifier la **gestion des NULL** dans les `LAG`/`LEAD` (premier élément d'une partition).
- `NTILE` peut produire des classes de tailles inégales — attendu.
- Préférer `JOIN` explicite à la jointure cartésienne avec `WHERE`.

---

# Pour aller plus loin (hors barème)

- Implémenter `V_DAILY_SALES_DASHBOARD` en **vue matérialisée** (si autorisée sur FreeSQL) avec stratégie de rafraîchissement (`ON DEMAND` vs `ON COMMIT`) — discuter en commentaire le compromis fraîcheur / coût.
- Proposer une **stratégie d'indexation** justifiée pour `ORDERS` et `ORDER_ITEMS` (en commentaire SQL, pas d'exécution).
- Étendre `pkg_co_analytics` avec une fonction **pipelined** qui retourne le top-N clients d'un segment RFM donné.
- Implémenter une **détection d'anomalies** plus sophistiquée que Q19 : commande dont le montant dépasse 3× l'écart-type au-dessus de la moyenne du client (z-score).

---

# Articulation TP1 ↔ TP2

| | **TP1 (HR)** | **TP2 (CO)** |
|---|---|---|
| Domaine | Ressources humaines | E-commerce / orders |
| Cas intégrateur | Audit d'équité salariale | Segmentation RFM |
| Spécificité technique | Hiérarchies (`CONNECT BY`) | Séries temporelles, cohortes |
| Package livré | `pkg_hr_analytics` | `pkg_co_analytics` |

Les deux TP sont **indépendants** mais partagent les mêmes fondamentaux (window functions, CTE, packages, exceptions).

---

# Fin du TP2
