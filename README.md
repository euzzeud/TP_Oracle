# 🧪 TP SQL & PL/SQL – Schéma HR (Oracle / FreeSQL)

## 🎯 Objectifs pédagogiques
À l’issue de ce TP, vous serez capables de :
- interroger une base relationnelle avec SQL (SELECT, JOIN, GROUP BY)
- créer des vues
- écrire des fonctions PL/SQL
- développer des procédures stockées
- manipuler des données RH dans un contexte réaliste

---

## 🧱 Contexte

Vous travaillez comme analyste SI au sein du service RH d’une entreprise.  
Vous devez produire des indicateurs, automatiser certains traitements et exposer des données utiles aux managers.

Vous utilisez le schéma **HR** fourni dans FreeSQL (https://freesql.com/).

---

## 📦 Données utilisées

Tables principales :
- EMPLOYEES
- DEPARTMENTS
- JOBS
- LOCATIONS

---

# 🔹 PARTIE 1 – Requêtes SQL (fondamentaux)

### Q1. Liste des employés
Afficher :
- prénom
- nom
- salaire

---

### Q2. Filtrage
Afficher les employés :
- ayant un salaire > 5000
- triés par salaire décroissant

---

### Q3. Jointure
Afficher :
- nom de l’employé
- nom du département

---

### Q4. Agrégation
Afficher :
- le salaire moyen par département

---

### Q5. Analyse métier
Afficher les départements dont :
- le salaire moyen > 8000

---

# 🔹 PARTIE 2 – Requêtes avancées

### Q6. Sous-requête
Afficher les employés :
- gagnant plus que le salaire moyen global

---

### Q7. Classement
Afficher les 5 employés :
- les mieux payés

---

### Q8. Hiérarchie
Afficher :
- employés + leur manager

---

# 🔹 PARTIE 3 – Création de vues

### Q9. Vue métier
Créer une vue `V_EMPLOYEES_INFO` contenant :
- nom complet
- salaire
- département
- job

---

### Q10. Vue analytique
Créer une vue `V_SALARY_STATS` contenant :
- département
- nombre d’employés
- salaire moyen
- salaire max

---

# 🔹 PARTIE 4 – Fonctions PL/SQL

### Q11. Fonction bonus salaire

Créer une fonction :
get_bonus(salary NUMBER) RETURN NUMBER

Règles :
- si salaire < 5000 → bonus = 10%
- sinon → bonus = 5%

---

### Q12. Utilisation de la fonction
Afficher :
- nom
- salaire
- bonus calculé

---

# 🔹 PARTIE 5 – Procédures stockées

### Q13. Augmentation de salaire

Créer une procédure :
increase_salary(p_emp_id NUMBER, p_percent NUMBER)

Elle doit :
- augmenter le salaire d’un employé
- en fonction d’un pourcentage

---

### Q14. Test
Tester la procédure sur un employé existant.

---

### Q15. Procédure métier

Créer une procédure :
show_employee_info(p_emp_id NUMBER)

Elle doit afficher :
- nom
- salaire
- département

(utiliser DBMS_OUTPUT)

---

# 🔹 PARTIE 6 – Bonus (niveau avancé)

### Q16. Fonction analytique
Créer une fonction :
get_department_avg(p_dept_id NUMBER)

Retourne :
- salaire moyen du département

---

### Q17. Détection anomalie
Afficher les employés :
- dont le salaire est supérieur à la moyenne de leur département

---

### Q18. Vue décisionnelle
Créer une vue `V_HIGH_EARNERS` :
- employés au-dessus de la moyenne de leur département

---

# 🧪 Modalités

- Environnement : FreeSQL (Oracle)
- Travail individuel
- Rendu :
  - fichier SQL
  - captures d’écran (optionnel)

---

# 📊 Critères d’évaluation

| Critère | Points |
|--------|------|
| Requêtes SQL correctes | 6 |
| Vues | 4 |
| Fonctions | 4 |
| Procédures | 4 |
| Bonus / qualité | 2 |
| **Total** | **20** |

---

# 🚀 Conseils

- tester chaque requête progressivement
- utiliser `DESC table_name` pour explorer les tables
- activer l’affichage PL/SQL :
SET SERVEROUTPUT ON;

---

# 🔚 Fin du TP
