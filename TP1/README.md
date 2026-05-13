# TP SQL & PL/SQL Avancé – Schéma HR (Oracle / FreeSQL)

**Public :** Master 2 Informatique
**Durée indicative :** 6 à 8 heures
**Plateforme d'exécution :** [https://freesql.com/](https://freesql.com/) — schéma **HR** (exclusivement)

---

## Objectifs pédagogiques

À l'issue de ce TP, vous serez capables de :

- formuler des requêtes analytiques de niveau professionnel (fonctions de fenêtrage, agrégats glissants, classements)
- structurer des requêtes complexes avec **CTE** et **requêtes récursives / hiérarchiques**
- concevoir des **vues de reporting** mobilisables par des outils décisionnels
- développer du **PL/SQL avancé** : fonctions déterministes, curseurs explicites, exceptions personnalisées, packages
- raisonner en **analyste de données RH** : formuler une question métier, la traduire en SQL, interpréter le résultat
- mener un **mini-projet intégrateur** sur la détection d'anomalies salariales

---

## Pré-requis

- Connaissance opérationnelle de SQL standard (`SELECT`, `JOIN`, `GROUP BY`, sous-requêtes)
- Notions de PL/SQL : blocs anonymes, `DECLARE / BEGIN / END`
- Compte sur [https://freesql.com/](https://freesql.com/) avec accès au schéma **HR**

> Les questions Q1 à Q3 servent de mise en jambes / vérification de l'environnement. Le cœur du TP commence en **Partie 1**.

---

## Contexte métier

Vous intervenez comme **data analyst RH** dans une multinationale. La direction RH vous mandate pour :

1. produire des **indicateurs de pilotage** (top performers, dispersion salariale, hiérarchies)
2. industrialiser ces indicateurs sous forme de **vues** et **packages PL/SQL** réutilisables
3. mener une **étude d'équité salariale** (anomalies, écarts intra-poste, biais)

Toutes les requêtes doivent être exécutables telles quelles sur FreeSQL, sur le schéma **HR** standard (tables `EMPLOYEES`, `DEPARTMENTS`, `JOBS`, `JOB_HISTORY`, `LOCATIONS`, `COUNTRIES`, `REGIONS`).

---

## Modalités de rendu

- Plateforme d'exécution : **FreeSQL uniquement** (aucune base locale)
- Rendu : **un seul fichier** `nom_prenom_TP.sql`, déposé dans ce repository (PR ou commit direct sur une branche `etudiants/<nom>`)
- Le fichier doit être **rejouable de bout en bout** sur FreeSQL (ordre des `CREATE OR REPLACE`, `DROP` éventuels en commentaire)
- Chaque réponse est précédée d'un en-tête de commentaire :

```sql
-- =====================================================
-- Q7 — Top 3 salaires par département (window function)
-- =====================================================
```

- Capture(s) d'écran FreeSQL des résultats : optionnel mais valorisé pour la Partie 6.

---

## Échauffement (non noté)

> Vérifiez votre accès à FreeSQL avant de commencer.

**Q0.** Lister les 5 premiers employés (`first_name`, `last_name`, `salary`) triés par salaire décroissant.

Requête :
```sql
-- =====================================================
-- SELECT first_name, last_name, salary FROM HR.EMPLOYEES ORDER BY salary DESC LIMIT 5;
-- =====================================================
```
    LIMIT n'existe pas en ORACLE SQL, la bonne requête est :
```sql
-- =====================================================
-- SELECT first_name, last_name, salary FROM HR.EMPLOYEES ORDER BY salary DESC FETCH FIRST 5 ROWS ONLY;
-- =====================================================
```

---

# Partie 1 — SQL analytique (fonctions de fenêtrage)

> **Objectif :** maîtriser `OVER (PARTITION BY ... ORDER BY ...)` et les fonctions de classement / décalage / agrégation glissante.

### Q1. Classement intra-département
Pour chaque employé, afficher : `last_name`, `department_id`, `salary`, et son **rang** (`RANK`) salarial **au sein de son département**. Trier par département puis rang.

### Q2. Top-N par groupe
Lister les **3 employés les mieux payés de chaque département**. Utiliser une fonction de fenêtrage et une CTE.

### Q3. Quartiles de salaire
Affecter chaque employé à un **quartile salarial global** (`NTILE(4)`). Afficher quartile, effectif, salaire min, salaire max par quartile.

### Q4. Écart à la moyenne du département
Pour chaque employé, afficher : salaire, **moyenne du département**, et **écart en %** entre les deux (positif = sur-payé). Utiliser `AVG() OVER (PARTITION BY ...)`.

### Q5. Cumul et part relative
Trier les départements par masse salariale décroissante, et afficher pour chaque département :
- masse salariale
- masse cumulée (`SUM() OVER (ORDER BY ...)`)
- part relative dans la masse totale (en %)

### Q6. Comparaison adjacente (LAG)
Au sein de chaque département, ordonner les employés par salaire et afficher pour chaque employé l'**écart avec le salaire de l'employé immédiatement mieux payé** (`LAG`).

---

# Partie 2 — CTE & requêtes hiérarchiques

> **Objectif :** structurer des requêtes complexes avec `WITH`, et exploiter la hiérarchie `manager_id → employee_id`.

### Q7. CTE simple
Avec une CTE `dept_stats` (effectif, salaire moyen par département), lister les départements dont l'effectif est supérieur à la moyenne globale d'effectif. Aucun calcul ne doit apparaître deux fois.

### Q8. Hiérarchie descendante (CONNECT BY)
À partir du **président** (employé sans manager), afficher l'arbre hiérarchique avec :
- niveau hiérarchique (`LEVEL`)
- chemin complet (`SYS_CONNECT_BY_PATH(last_name, ' / ')`)
- indentation visuelle (`LPAD(' ', 2*(LEVEL-1)) || last_name`)

### Q9. CTE récursive (variante portable)
Reproduire Q8 avec une **CTE récursive** (`WITH ... AS (... UNION ALL ...)`). Comparer le résultat avec Q8.

### Q10. Profondeur managériale
Pour chaque manager, calculer le **nombre total de subordonnés directs et indirects** (toute la sous-arborescence). Trier par nombre décroissant.

---

# Partie 3 — Vues de reporting

> **Objectif :** exposer des indicateurs prêts à consommer par un outil BI.

### Q11. `V_EMPLOYEE_360`
Vue à grain employé contenant : `employee_id`, nom complet, `email`, `job_title`, `department_name`, `city`, `country_name`, `manager_name`, `salary`, `hire_date`, `years_in_company`.

### Q12. `V_SALARY_BENCHMARK`
Vue à grain employé enrichie de :
- salaire moyen du **job** (toutes entreprises confondues, donc tous départements)
- salaire moyen du **département**
- flag `is_overpaid` (salaire > 1.2 × moyenne du job)
- flag `is_underpaid` (salaire < 0.8 × moyenne du job)

### Q13. `V_DEPT_DASHBOARD`
Vue à grain département : effectif, masse salariale, salaire médian (`MEDIAN`), salaire min/max, ratio max/min, ancienneté moyenne. Utilisable pour un dashboard manager.

---

# Partie 4 — Fonctions PL/SQL

> **Objectif :** écrire des fonctions **réutilisables**, **déterministes** quand c'est possible, avec **gestion d'exceptions**.

### Q14. `fn_get_bonus`
Fonction `fn_get_bonus(p_salary IN NUMBER) RETURN NUMBER DETERMINISTIC` :
- 12 % si salaire < 4 000
- 8 % si 4 000 ≤ salaire < 8 000
- 5 % sinon
- lever `VALUE_ERROR` si `p_salary` est NULL ou négatif

Tester avec un `SELECT last_name, salary, fn_get_bonus(salary) FROM employees`.

### Q15. `fn_dept_avg_salary`
Fonction `fn_dept_avg_salary(p_dept_id IN NUMBER) RETURN NUMBER` :
- retourne la moyenne salariale du département
- retourne `NULL` (et non une erreur) si le département n'existe pas ou n'a aucun employé
- gérer explicitement `NO_DATA_FOUND` et `OTHERS`

### Q16. `fn_seniority_label`
Fonction qui prend un `employee_id` et retourne une chaîne :
- `'Junior'` si ancienneté < 3 ans
- `'Confirmé'` si 3 ≤ ancienneté < 8
- `'Senior'` si ancienneté ≥ 8
- lever une exception personnalisée `e_employee_not_found` si l'employé n'existe pas

---

# Partie 5 — Procédures & Package

> **Objectif :** organiser le code PL/SQL en **package**, manipuler des **curseurs explicites**, et exposer une API métier propre.

### Q17. Procédure `pr_increase_salary`
`pr_increase_salary(p_emp_id IN NUMBER, p_percent IN NUMBER)` :
- augmente le salaire de l'employé
- valide que `p_percent` est entre 0 et 50 (sinon `RAISE_APPLICATION_ERROR(-20001, ...)`)
- vérifie l'existence de l'employé (sinon `-20002`)
- ne fait **pas** de `COMMIT` (laissé à l'appelant)

### Q18. Procédure `pr_show_employee_card`
`pr_show_employee_card(p_emp_id IN NUMBER)` qui affiche via `DBMS_OUTPUT` une fiche formatée :

```
┌────────────────────────────────────────┐
│  KING, Steven  (#100)
│  Job        : President
│  Department : Executive
│  Salary     : 24000  (Senior — +5% bonus = 1200)
│  Hired on   : 17-JUN-2003 (22 ans)
│  Manager    : —
└────────────────────────────────────────┘
```

(Réutiliser `fn_get_bonus` et `fn_seniority_label`.)

### Q19. Package `pkg_hr_analytics`
Spécification + corps d'un package exposant :

- `fn_get_bonus` (déplacée depuis Q14)
- `fn_dept_avg_salary` (déplacée depuis Q15)
- `fn_seniority_label` (déplacée depuis Q16)
- `pr_increase_salary` (déplacée depuis Q17)
- `pr_show_employee_card` (déplacée depuis Q18)
- une nouvelle procédure `pr_audit_department(p_dept_id IN NUMBER)` qui parcourt tous les employés du département via un **curseur explicite paramétré** et imprime un récapitulatif (effectif, masse, anomalies détectées par rapport à `V_SALARY_BENCHMARK`).

> Toutes les questions ultérieures appellent les routines via le package : `pkg_hr_analytics.fn_get_bonus(...)`.

---

# Partie 6 — Cas intégrateur : audit d'équité salariale

> **Objectif :** mobiliser tout ce qui précède sur une **étude métier réelle**.

**Contexte.** La direction RH soupçonne des **iniquités salariales** : à poste équivalent, certains employés seraient payés significativement moins (ou plus) que la médiane de leur poste. Vous devez livrer :

### Q20. Vue `V_SALARY_ANOMALIES`
Vue listant tous les employés dont le salaire **s'écarte de plus de 25 %** de la **médiane de leur `job_id`** (au-dessus ou en dessous). Colonnes : employé, job, salaire, médiane du job, écart absolu, écart en %, type (`'SOUS-PAYE'` / `'SUR-PAYE'`).

### Q21. Top 5 anomalies négatives par département
En s'appuyant sur `V_SALARY_ANOMALIES`, retourner pour chaque département les **5 employés les plus sous-payés** par rapport à la médiane de leur poste (window function).

### Q22. Procédure `pr_audit_equity`
Procédure qui prend un `p_dept_id` (ou `NULL` pour tous les départements) et imprime via `DBMS_OUTPUT` un rapport structuré :

```
=== AUDIT D'ÉQUITÉ — Département Sales (id=80) ===
Effectif analysé   : 34
Anomalies détectées: 7  (4 sous-payés, 3 sur-payés)
Écart moyen        : 31 %
--- Détail ---
  - RUSSEL John (#145)  Sales Manager   12000  (médiane 14500, -17%)
  - ...
```

### Q23. Note d'analyse (en commentaire SQL, ~15 lignes)
Dans le fichier de rendu, en commentaire structuré, rédiger :
- les **3 principales anomalies** détectées
- une **hypothèse explicative** par anomalie (ancienneté ? localisation ? historique de poste ?)
- une **recommandation actionnable** pour la DRH

---

# Critères d'évaluation (sur 20)

| Bloc | Critères | Points |
|---|---|:---:|
| **Partie 1** | Window functions correctes et idiomatiques | 3 |
| **Partie 2** | CTE, récursivité, hiérarchie | 3 |
| **Partie 3** | Vues — pertinence métier et factorisation | 2 |
| **Partie 4** | Fonctions PL/SQL — déterminisme, exceptions | 2 |
| **Partie 5** | Package — cohérence, encapsulation, curseurs | 3 |
| **Partie 6** | Cas intégrateur — qualité d'analyse | 4 |
| **Qualité** | Lisibilité, nommage, idempotence du script, commentaires | 3 |
| **Total** |  | **20** |

**Pénalités :**
- Code non rejouable sur FreeSQL : **−2 pts**
- Réponses sans en-tête / sans commentaire : **−1 pt**
- Recopie non créditée entre étudiants : **0**

---

# Conseils & spécificités FreeSQL

- Activer la sortie PL/SQL avant tout test de procédure :
  ```sql
  SET SERVEROUTPUT ON SIZE UNLIMITED;
  ```
- FreeSQL s'exécute sur Oracle : utilisez la **syntaxe Oracle** (`CONNECT BY`, `MEDIAN`, `NVL`, `TO_CHAR`, etc.) plutôt qu'ANSI/PostgreSQL.
- Pour rendre un script **rejouable**, préfixez les créations par `CREATE OR REPLACE` (vues, fonctions, procédures, packages). Pour les éventuels objets sans `OR REPLACE`, encadrer par un bloc `BEGIN EXECUTE IMMEDIATE 'DROP ...'; EXCEPTION WHEN OTHERS THEN NULL; END;`.
- Tester chaque requête **isolément** dans FreeSQL avant de l'intégrer au fichier final.
- Pour la Partie 6, **ne pas** modifier les données du schéma HR : le script doit être lisible sans pré-état particulier.

---

# Pour aller plus loin (hors barème)

- Réécrire `V_SALARY_ANOMALIES` avec une **vue matérialisée** (si autorisée sur FreeSQL) et discuter le compromis fraîcheur/performance.
- Proposer une **stratégie d'indexation** justifiée pour accélérer `V_SALARY_BENCHMARK` (en commentaire — pas d'exécution requise).
- Étendre `pkg_hr_analytics` avec une fonction **pipelined** retournant le top-N anomalies.

---

# Fin du TP
