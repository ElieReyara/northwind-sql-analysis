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

- **Requête 1/10 — Le calcul du Chiffre d'Affaires net (avec remises)**
  Le but ici est d'analyser la santé financière de l'entreprise. Nous voulons connaitre son chiffre et pour etre plus précis, nous voulons son chiffre net deduit des remises.
  La formula mathematique suivante peut etre retenu : (prix_unitaire * quantite_vendue)(1-remise)
  Cela nous donnera :
    SELECT SUM((unit_price * quantity)*(1-discount)) AS ca_net
    FROM order_details
  
