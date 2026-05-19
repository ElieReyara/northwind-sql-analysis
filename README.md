#  Northwind Traders — Audit de la Performance Commerciale & Optimisation Supply Chain

##  Contexte Business
Northwind Traders est une entreprise internationale de distribution et d'import-export de produits alimentaires. Face à une concurrence accrue et à des marges de plus en plus serrées, la direction générale exige un audit complet de ses opérations globales.

L'objectif de ce projet est de transformer les données brutes (transactions, stocks, clients, employés) en leviers stratégiques actionnables pour :
  - **Optimiser la Supply Chain** (délais de livraison, gestion des ruptures de stock).
  - **Maximiser la Profitabilité** (analyse des remises, top produits, segmentation clients).
  - **Piloter la Performance** (KPIs de vente par employé et par région).

###  Stack Technique
- **Base de données :** PostgreSQL (Modélisation relationnelle, requêtes complexes, CTEs, Window Functions)
- **Environnement :** Local / pgAdmin

### Requete SQL 
- **Requête 1/10 — Le calcul du Chiffre d'Affaires net (avec remises)**
  Le but ici est d'analyser la santé financière de l'entreprise. Nous voulons connaitre son chiffre et pour etre plus précis, nous voulons son chiffre net deduit des remises.
  La formula mathematique suivante peut etre retenu : (prix_unitaire * quantite_vendue)(1-remise)
  Cela nous donnera :
    SELECT SUM((unit_price * quantity)*(1-discount)) AS ca_net
    FROM order_details
- **Requête 2/10 — Le Top 10 des clients par Chiffre d'Affaires Net**
  Toujours dans l'optique d'analyser la santé financière de l'entreprise, nous voulons connaitre cette fois si le top 10 de ces meilleurs clients. Il vaut mieux concentrer nos efforts sur la satisfaction de ceux qui font le plus de notre CA.
    SELECT  c.company_name, SUM((od.quantity * od.unit_price)*(1-od.discount)) as ca_net_par_company 
    FROM customers c 
    INNER JOIN orders o
    ON c.customer_id = o.customer_id
    INNER JOIN order_details od
    ON od.order_id = o.order_id
    GROUP BY c.company_name
    ORDER BY ca_net_par_company DESC
    LIMIT 10
  --Variante 2 CA par categorie de produit
    SELECT  c.category_name, SUM((od.quantity * od.unit_price)*(1-od.discount)) as ca_net_par_category 
    FROM categories c 
    INNER JOIN products p
    ON p.category_id = c.category_id
    INNER JOIN order_details od
    ON od.product_id = p.product_id
    GROUP BY c.category_name
    ORDER BY ca_net_par_category DESC
- **Requête 3/10 : L'analyse des produits fantômes.**
  Le Responsable Logistique a une intuition : il pense que le catalogue est surchargé de produits qui coûtent cher en stockage mais que personne n'achète. Il veut la liste des produits qui n'ont jamais été commandés.
    SELECT p.product_name, od.order_id, od.product_id 
    FROM products p
    LEFT JOIN order_details od
    ON p.product_id = od.product_id
    WHERE od.product_id IS NULL
  ps : Tout les produit ont ete vendus, donc l'intuition de notre responsable logistic etait donc fausse. Imaginons que la table products contienne des produits archivés ou discontinus (des produits qu'on ne vend plus). S'ils ont été vendus il y a 3 ans, ils   apparaissent dans ta jointure, donc pas de NULL. Pourtant, ils encombrent peut-être encore l'entrepôt aujourd'hui.
  

  
  
