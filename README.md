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
  (Claude)
  -- ======================================
  -- REQUÊTE 1 : CA net par catégorie
  -- Business context : identifier les catégories
  -- qui génèrent le plus de valeur réelle
  -- (après remises) vs volume brut
  -- ======================================
  
  SELECT
      c.category_name,
      COUNT(DISTINCT od.order_id) AS nb_commandes,
      SUM(od.quantity)            AS quantite_totale,
      SUM((od.quantity * od.unit_price) * (1 - od.discount)) AS ca_net_par_categorie
  FROM categories c
  INNER JOIN products p  ON p.category_id = c.category_id
  INNER JOIN order_details od ON od.product_id = p.product_id
  GROUP BY c.category_name
  ORDER BY ca_net_par_categorie DESC;

  Meat/Poultry a un panier moyen élevé avec peu de volume — ça signifie des produits premium, achetés moins souvent mais à haute valeur unitaire. En business, ça change complètement la stratégie commerciale vs une catégorie haute fréquence / faible valeur.(a rajouter correctement)
- **Requête 2/10 — Le Top 10 des clients par Chiffre d'Affaires Net**
  Toujours dans l'optique d'analyser la santé financière de l'entreprise, nous voulons connaitre cette fois si le top 10 de ces meilleurs clients. Il vaut mieux concentrer nos efforts sur la satisfaction de ceux qui font le plus de notre CA.
  Claude me pousse beaucoup plus loin en allant me denmander comment on identie vraiment une campagnie de maniere unique, aller chercher ds metris pertinente plutot que de se limiter a celle qu'on a la.
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
    ou encoe
    SELECT  
    	cu.customer_id,
    	cu.company_name,
    	COUNT(o.order_id) as nb_commandes,
    	SUM((od.quantity * od.unit_price)*(1-od.discount)) as ca_net_par_company, 
    	SUM((od.quantity * od.unit_price) * (1 - od.discount)) 
    	/ COUNT(DISTINCT o.order_id) AS panier_moyen
    FROM customers cu 
    INNER JOIN orders o
    ON cu.customer_id = o.customer_id
    INNER JOIN order_details od
    ON od.order_id = o.order_id
    GROUP BY cu.customer_id, cu.company_name
    ORDER BY ca_net_par_company DESC
    LIMIT 10
- **Requête 3/10 : L'analyse des produits fantômes.**
  Le Responsable Logistique a une intuition : il pense que le catalogue est surchargé de produits qui coûtent cher en stockage mais que personne n'achète. Il veut la liste des produits qui n'ont jamais été commandés.
    SELECT 
    	p.product_name, 
    	p.unit_price, 
    	p.units_in_stock, 
    	p.discontinued
    FROM products p
    LEFT JOIN order_details od
    ON p.product_id = od.product_id
    WHERE od.product_id IS NULL
  ps : Tout les produit ont ete vendus, donc l'intuition de notre responsable logistic etait donc fausse. Imaginons que la table products contienne des produits archivés ou discontinus (des produits qu'on ne vend plus). S'ils ont été vendus il y a 3 ans, ils   apparaissent dans ta jointure, donc pas de NULL. Pourtant, ils encombrent peut-être encore l'entrepôt aujourd'hui.
- **Requête 4/10 : L'analyse des délais de livraison.**
  Le Directeur des Opérations veut auditer la Supply Chain. Il veut connaître le délai moyen en jours entre la date de commande et la date de livraison moyen, par compagnie mais aussi par pays de livraison. Cela nous permet d'apprecier les livreurs les plus rapides.
Enfaite, il peut arriver qu'une entrepise soir plus rapide sur un pays qu'un autre meme si son concurrent est globalement plus rapide, ca nous sert aaffiner nos choix. Donc la plus pertinente pour la question businnes est la 1ere, mais la seconde est utile aussi.
  --Par trasnporteur global
  SELECT
  	sh.company_name,
  	COUNT(DISTINCT o.order_id) AS nb_commandes,
  	ROUND(AVG( shipped_date::date - order_date::date)) as delai_moyen_livraison
  FROM orders o
  INNER JOIN shippers sh
  ON sh.shipper_id = o.ship_via
  WHERE o.shipped_date IS NOT NULL
  GROUP BY sh.company_name
  ORDER BY delai_moyen_livraison DESC 
  
  --Par pays
  SELECT
  	o.ship_country,
  	sh.company_name,
  	ROUND(AVG( shipped_date::date - order_date::date)) as delai_moyen_livraison
  FROM orders o
  INNER JOIN shippers sh
  ON sh.shipper_id = o.ship_via
  WHERE o.shipped_date IS NOT NULL
  GROUP BY sh.company_name, o.ship_country
  ORDER BY delai_moyen_livraison DESC 
- **Requête 5/10 — Taux de réapprovisionnement critique.**
Question business : Quels produits risquent une rupture de stock dans les 30 prochains jours si la vélocité de vente actuelle continue ?**
Le stock actuel : Tu l'as dans la table products (colonne units_in_stock).
La vitesse de vente (Vélocité) : C'est le nombre d'unités vendues par jour. Si tu constates que tu as vendu 300 unités d'un produit sur les 30 derniers jours, cela signifie que ta vélocité est de $300 / 30 = 10 unités/jour.
La projection : Si tu vends 10 unités par jour et qu'il te reste 50 unités en stock, tu as du stock pour $50 / 10 = 5 \text{ jours}$. Tu seras donc en rupture bien avant les 30 prochains jours.
- **Requête 5/10 :  — Taux de réapprovisionnement critique..**
  Question business : "Quels produits risquent une rupture de stock dans les 30 prochains jours si la vélocité de vente actuelle continue ?"
Le stock actuel : Tu l'as dans la table products (colonne units_in_stock).
La vitesse de vente (Vélocité) : 
C'est le nombre d'unités vendues par jour. Si tu constates que tu as vendu 300 unités d'un 
produit sur les 30 derniers jours, cela signifie que ta vélocité est de $300 / 30 = 10 unités/jour$.
La projection : Si tu vends 10 unités par jour et qu'il te reste 50 unités en stock, tu as du stock pour $50 / 10 = 5\jours$. 
Tu seras donc en rupture bien avant les 30 prochains jours.'

  -- CTE 1 : calculer les unités vendues 
  --         par produit sur les 90 derniers jours
  
  WITH ventes_recentes AS (
     SELECT
     	  p.product_id,
  	  p.product_name,
  	  (SUM(od.quantity)   /
  	  90) as velocity_by_product
    FROM order_details od 
    INNER JOIN products p  
    ON p.product_id = od.product_id
    INNER JOIN orders o
    ON o.order_id = od.order_id
    WHERE o.order_date >= '1998-05-06'::date - INTERVAL '90 days'
    GROUP BY p.product_id, p.product_name
  )
  --Nous pourrions aussi bien utiliser o.shipped_date car la la commande quitte effectivement le stcok, il faut documenter le choix.
  
  -- Requête principale : joindre avec products
  -- et appliquer la condition de risque
  
  SELECT 
  	p.product_name, 
  	p.units_in_stock,
  	vr.velocity_by_product,
  	ROUND(p.units_in_stock / NULLIF(vr.velocity_by_product, 0)) AS jours_restants
  FROM products p
  LEFT JOIN ventes_recentes vr 
  ON p.product_id = vr.product_id
  WHERE p.units_in_stock < vr.velocity_by_product * 30

  
  
