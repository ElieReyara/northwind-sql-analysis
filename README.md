# Northwind Traders — Audit de la Performance Commerciale & Optimisation Supply Chain

## Contexte Business

Northwind Traders est une entreprise internationale de distribution et d'import-export de produits
alimentaires. Face à une concurrence accrue et à des marges de plus en plus serrées, la direction
générale exige un audit complet de ses opérations globales.

L'objectif de ce projet est de transformer les données brutes (transactions, stocks, clients,
employés) en leviers stratégiques actionnables pour :

- **Optimiser la Supply Chain** : délais de livraison, gestion des ruptures de stock.
- **Maximiser la Profitabilité** : analyse des remises, top produits, segmentation clients.
- **Piloter la Performance** : KPIs de vente par employé et par région.

## Stack Technique

- **Base de données :** PostgreSQL (modélisation relationnelle, requêtes complexes, CTEs, Window Functions)
- **Environnement :** Local / pgAdmin

---

## Key Findings

- **Beverages** est la catégorie n°1 avec **267 868 $ de CA net** sur 354 commandes — volume ET valeur au rendez-vous.
- **Meat/Poultry** génère **163 022 $** avec seulement 161 commandes, soit le panier moyen le plus élevé du catalogue — profil premium à protéger.
- **45 produits** sont en risque de rupture de stock dans les 30 prochains jours si la vélocité actuelle se maintient.
- **QUICK-Stop** est le client le plus précieux avec **110 277 $ de CA net** sur 28 commandes (panier moyen : 3 938 $).
- **Federal Shipping** est le transporteur le plus rapide avec un délai moyen de **7 jours**, contre 9 jours pour Speedy Express et United Package.
- **Meat/Poultry** subit le taux de remise le plus élevé (**9%**) — paradoxe à investiguer : la catégorie la plus premium est aussi la plus discountée.
- **Margaret Peacock** est la commerciale n°1 avec **232 891 $ de CA net** sur 156 commandes.
- **Avril 1998** est le meilleur mois enregistré avec **123 799 $ de CA net** sur 74 commandes.
- **2 clients** (Paris Spécialités, FISSA) n'ont passé aucune commande — à réactiver ou désactiver du CRM.

---

## Analyse de la Rentabilité et de la Gestion des Stocks

### Requête 1/10 — Chiffre d'Affaires Net par Catégorie

**Question business :** Quelles catégories de produits génèrent le plus de valeur réelle après
déduction des remises ? Comment distinguer les catégories à fort volume de celles à forte valeur
unitaire ?

**Pourquoi cette analyse ?** Présenter un CA brut à la direction, c'est mentir par omission. Les
remises réduisent directement la valeur encaissée. La formule retenue est
`(prix_unitaire × quantité) × (1 - remise)`, qui donne le montant réellement perçu par transaction.
On croise volume de commandes et valeur générée pour distinguer deux profils stratégiques opposés.

**Insight :** Beverages domine avec 267 868 $ sur 354 commandes — forte fréquence ET forte valeur.
Meat/Poultry, en revanche, génère 163 022 $ sur seulement 161 commandes : produits premium, achetés
moins souvent mais à panier moyen nettement supérieur. Ces deux catégories nécessitent des stratégies
commerciales radicalement différentes.

```sql
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
INNER JOIN products      p  ON p.category_id = c.category_id
INNER JOIN order_details od ON od.product_id = p.product_id
GROUP BY c.category_name
ORDER BY ca_net_par_categorie DESC;
```

---

### Requête 2/10 — Top 10 des Clients par Chiffre d'Affaires Net

**Question business :** Quels sont nos 10 clients les plus précieux, et comment se distinguent-ils
en termes de fréquence d'achat et de valeur moyenne par commande ?

**Pourquoi cette analyse ?** Un CA élevé peut masquer des réalités très différentes : un client peut
générer beaucoup de revenus parce qu'il commande souvent (fidélité), ou parce qu'il passe de très
grosses commandes ponctuelles (valeur unitaire élevée). Ces deux profils nécessitent des stratégies
de fidélisation distinctes. Le client est identifié par son `customer_id` unique pour éviter toute
ambiguïté sur les noms d'entreprises.

**Insight :** QUICK-Stop mène avec 110 277 $ sur 28 commandes (panier moyen : 3 938 $). Ernst Handel
suit avec 104 874 $ mais sur 30 commandes — plus fidèle, légèrement moins rentable par commande.
Save-a-lot Markets se distingue avec le plus grand nombre de commandes (31) pour 104 361 $ de CA net :
profil volume par excellence.

```sql
-- ======================================
-- REQUÊTE 2 : Top 10 clients par CA net
-- Business context : identifier les clients
-- à fort enjeu commercial et comprendre
-- leur profil d'achat (fréquence vs valeur)
-- ======================================

SELECT
    cu.customer_id,
    cu.company_name,
    COUNT(DISTINCT o.order_id)                                   AS nb_commandes,
    SUM((od.quantity * od.unit_price) * (1 - od.discount))      AS ca_net_par_client,
    SUM((od.quantity * od.unit_price) * (1 - od.discount))
        / COUNT(DISTINCT o.order_id)                             AS panier_moyen
FROM customers cu
INNER JOIN orders        o  ON o.customer_id = cu.customer_id
INNER JOIN order_details od ON od.order_id   = o.order_id
GROUP BY cu.customer_id, cu.company_name
ORDER BY ca_net_par_client DESC
LIMIT 10;
```

---

### Requête 3/10 — Analyse des Produits Fantômes

**Question business :** Quels produits sont référencés au catalogue mais n'ont jamais généré une
seule commande, représentant ainsi un coût de stockage sans retour sur investissement ?

**Pourquoi cette analyse ?** Le Responsable Logistique suspectait une surcharge du catalogue. On
utilise un **anti-join** (LEFT JOIN + WHERE IS NULL) : on joint tous les produits avec les lignes de
commandes, et on filtre ceux pour lesquels aucune correspondance n'existe dans `order_details`. On
affiche également le statut `discontinued` pour distinguer les produits à archiver définitivement de
ceux encore actifs mais jamais achetés — deux situations qui appellent deux décisions différentes.

**Résultat :** Aucun produit fantôme détecté. Tous les articles du catalogue ont été commandés au
moins une fois. L'intuition du Responsable Logistique était infondée sur ce dataset. À noter : des
produits `discontinued` vendus par le passé n'apparaissent pas ici car ils ont bien des lignes dans
`order_details`. Une analyse complémentaire sur le stock résiduel de ces produits discontinués
serait pertinente.

```sql
-- ======================================
-- REQUÊTE 3 : Produits jamais commandés
-- Business context : identifier le stock mort
-- et les produits à retirer du catalogue
-- ======================================

SELECT
    p.product_name,
    p.unit_price,
    p.units_in_stock,
    p.discontinued
FROM products p
LEFT JOIN order_details od ON p.product_id = od.product_id
WHERE od.product_id IS NULL;
```

---

### Requête 4/10 — Analyse des Délais de Livraison

**Question business :** Quel transporteur offre les délais de livraison les plus fiables ? Existe-t-il
des disparités par pays qui justifieraient d'adapter notre choix de transporteur selon la destination ?

**Pourquoi cette analyse ?** Le délai de livraison est un levier direct de satisfaction client. On
mesure l'écart en jours entre `order_date` et `shipped_date`. Les commandes non encore livrées
(`shipped_date IS NULL`) sont exclues pour ne pas fausser la moyenne. Deux granularités sont
analysées : la performance globale par transporteur (vision stratégique) et la performance par pays
(vision opérationnelle). Un transporteur globalement plus rapide peut être moins performant sur
certaines destinations spécifiques.

**Insight :** Federal Shipping est le plus rapide avec 7 jours en moyenne sur 249 commandes. Speedy
Express et United Package affichent tous deux 9 jours de délai moyen — mais United Package traite
315 commandes, soit le volume le plus important. Par pays, United Package atteint 14 jours en Suisse
et Speedy Express 14 jours en Belgique : des anomalies qui justifient une renégociation contractuelle
sur ces destinations.

```sql
-- ======================================
-- REQUÊTE 4A : Délais moyens par transporteur (global)
-- Business context : évaluer la fiabilité
-- globale de chaque partenaire logistique
-- ======================================

SELECT
    sh.company_name,
    COUNT(DISTINCT o.order_id)                             AS nb_commandes,
    ROUND(AVG(o.shipped_date::date - o.order_date::date)) AS delai_moyen_livraison
FROM orders o
INNER JOIN shippers sh ON sh.shipper_id = o.ship_via
WHERE o.shipped_date IS NOT NULL
GROUP BY sh.company_name
ORDER BY delai_moyen_livraison DESC;

-- ======================================
-- REQUÊTE 4B : Délais moyens par pays
-- Business context : affiner le choix du
-- transporteur selon la destination
-- ======================================

SELECT
    o.ship_country,
    sh.company_name,
    ROUND(AVG(o.shipped_date::date - o.order_date::date)) AS delai_moyen_livraison
FROM orders o
INNER JOIN shippers sh ON sh.shipper_id = o.ship_via
WHERE o.shipped_date IS NOT NULL
GROUP BY sh.company_name, o.ship_country
ORDER BY delai_moyen_livraison DESC;
```

---

### Requête 5/10 — Taux de Réapprovisionnement Critique

**Question business :** Quels produits risquent une rupture de stock dans les 30 prochains jours si
la vélocité de vente actuelle se maintient ?

**Pourquoi cette analyse ?** On calcule une **vélocité de vente** : le nombre d'unités vendues par
jour sur les 90 derniers jours. On compare ce rythme au stock disponible pour projeter dans combien
de jours le stock sera épuisé. La condition de risque est `units_in_stock < velocite_journaliere × 30`.
Une CTE est nécessaire car le calcul de vélocité (agrégation) doit être réalisé avant d'être comparé
au stock dans la requête principale.

**Note méthodologique :** Le filtre utilise `order_date` pour mesurer la demande réelle. La date de
référence est fixée à `1998-05-06` (date maximale du dataset) pour éviter toute circularité. La
fonction `NULLIF` protège contre une division par zéro si la vélocité est nulle.

**Insight :** 45 produits sont en risque de rupture. Les plus critiques affichent un stock à 0 unité
malgré une vélocité positive — Chef Anton's Gumbo Mix (1,33 unités/jour), Alice Mutton (1,88/jour),
Thüringer Rostbratwurst (3,33/jour). Ces produits génèrent de la demande sans pouvoir être livrés :
perte de CA directe et risque de désatisfaction client.

```sql
-- ======================================
-- REQUÊTE 5 : Risque de rupture de stock
-- Business context : anticiper les ruptures
-- et déclencher le réapprovisionnement
-- ======================================

-- CTE : vélocité de vente par produit sur 90 jours
WITH ventes_recentes AS (
    SELECT
        p.product_id,
        p.product_name,
        SUM(od.quantity) / 90.0 AS velocity_by_product
    FROM order_details od
    INNER JOIN products p ON p.product_id = od.product_id
    INNER JOIN orders   o ON o.order_id   = od.order_id
    WHERE o.order_date >= '1998-05-06'::date - INTERVAL '90 days'
    GROUP BY p.product_id, p.product_name
)

-- Requête principale : comparaison stock vs projection 30 jours
SELECT
    p.product_name,
    p.units_in_stock,
    ROUND(vr.velocity_by_product::numeric, 2)                    AS velocite_journaliere,
    ROUND(p.units_in_stock / NULLIF(vr.velocity_by_product, 0)) AS jours_restants
FROM products p
LEFT JOIN ventes_recentes vr ON p.product_id = vr.product_id
WHERE p.units_in_stock < vr.velocity_by_product * 30
ORDER BY jours_restants ASC;
```

---

### Requête 6/10 — Performance des Commerciaux

**Question business :** Quel commercial génère le plus de chiffre d'affaires net ? Comment se
distribuent volume, valeur et panier moyen entre les membres de l'équipe de vente ?

**Pourquoi cette analyse ?** Un classement par nombre de commandes seul peut être trompeur : un
commercial qui gère peu de commandes à très haute valeur est souvent plus stratégique qu'un autre
qui traite un grand volume de petites commandes. On croise trois métriques — volume, CA net, panier
moyen — pour avoir un portrait complet de chaque commercial.

**Insight :** Margaret Peacock domine avec 232 891 $ sur 156 commandes. Mais Andrew Fuller, avec
seulement 96 commandes, affiche le panier moyen le plus élevé (1 735 $) après Anne Dodsworth
(1 798 $) — des profils orientés grands comptes à valoriser différemment des commerciaux volume.

```sql
-- ======================================
-- REQUÊTE 6 : Performance des commerciaux
-- Business context : identifier les top
-- performers et les profils de vente
-- ======================================

SELECT
    e.first_name,
    e.last_name,
    COUNT(DISTINCT o.order_id)                                          AS nb_commandes,
    ROUND(SUM((od.quantity * od.unit_price) * (1 - od.discount)))      AS ca_net_par_employe,
    ROUND(SUM((od.quantity * od.unit_price) * (1 - od.discount))
        / COUNT(DISTINCT o.order_id))                                   AS panier_moyen
FROM employees e
INNER JOIN orders        o  ON o.employee_id = e.employee_id
INNER JOIN order_details od ON od.order_id   = o.order_id
GROUP BY e.first_name, e.last_name
ORDER BY ca_net_par_employe DESC;
```

---

### Requête 7/10 — Fidélité et Rétention des Clients

**Question business :** Quels clients commandent régulièrement et lesquels ont commandé une seule
fois avant de disparaître ? Quelle est la distribution de la fidélité dans notre base client ?

**Pourquoi cette analyse ?** La fidélisation d'un client existant coûte en moyenne 5 à 7 fois moins
cher que l'acquisition d'un nouveau client. Identifier les clients inactifs permet de déclencher des
actions de réactivation ciblées. On utilise un LEFT JOIN pour conserver tous les clients, y compris
ceux sans aucune commande.

**Insight :** 2 clients — Paris Spécialités et FISSA Fabrica — affichent 0 commande : présents dans
le CRM, jamais convertis. 1 client (Centro Comercial Moctezuma) n'a commandé qu'une seule fois.
Ces 3 profils méritent une action commerciale prioritaire avant de les désactiver définitivement.

```sql
-- ======================================
-- REQUÊTE 7 : Fidélité clients
-- Business context : segmenter les clients
-- par fréquence pour cibler les actions
-- de rétention et de réactivation
-- ======================================

SELECT
    cu.customer_id,
    cu.company_name,
    COUNT(DISTINCT o.order_id) AS nb_commandes
FROM customers cu
LEFT JOIN orders o ON o.customer_id = cu.customer_id
GROUP BY cu.customer_id, cu.company_name
ORDER BY nb_commandes ASC;
```

---

### Requête 8/10 — Impact des Remises sur la Marge

**Question business :** Les remises accordées stimulent-elles réellement le volume de ventes, ou
détruisent-elles de la marge sans contrepartie suffisante ?

**Pourquoi cette analyse ?** On compare pour chaque catégorie le CA brut et le CA net afin de
quantifier le montant et le pourcentage de marge sacrifié. Si une catégorie affiche un fort taux de
remise sans volume de commandes anormalement élevé en contrepartie, la politique tarifaire mérite
d'être révisée.

**Insight :** Meat/Poultry cumule le paradoxe le plus fort : taux de remise le plus élevé (9%) sur
la catégorie au panier moyen le plus premium. 15 166 $ de marge sacrifiée sans que cela se traduise
par un volume exceptionnel (161 commandes seulement). La politique de remise sur cette catégorie
doit être auditée en priorité.

```sql
-- ======================================
-- REQUÊTE 8 : Impact des remises sur la marge
-- Business context : évaluer si la politique
-- de remises est justifiée par le volume généré
-- ======================================

SELECT
    c.category_name,
    ROUND(SUM(od.quantity * od.unit_price))                            AS ca_brut,
    ROUND(SUM((od.quantity * od.unit_price) * (1 - od.discount)))     AS ca_net,
    ROUND(SUM(od.quantity * od.unit_price)
        - SUM((od.quantity * od.unit_price) * (1 - od.discount)))     AS montant_remise,
    ROUND(
        (SUM(od.quantity * od.unit_price)
            - SUM((od.quantity * od.unit_price) * (1 - od.discount)))
        / SUM(od.quantity * od.unit_price) * 100
    , 2)                                                               AS pct_remise
FROM categories c
INNER JOIN products      p  ON p.category_id = c.category_id
INNER JOIN order_details od ON od.product_id = p.product_id
GROUP BY c.category_name
ORDER BY pct_remise DESC;
```

---

### Requête 9/10 — Saisonnalité des Ventes

**Question business :** Quels mois génèrent le plus de chiffre d'affaires net ? Y a-t-il un pattern
saisonnier récurrent exploitable pour planifier les stocks et les campagnes commerciales ?

**Pourquoi cette analyse ?** Comprendre la saisonnalité permet d'anticiper les pics de demande,
d'optimiser les niveaux de stock et de concentrer les efforts marketing sur les périodes à fort
potentiel. On extrait l'année et le mois séparément pour éviter la fusion de périodes identiques sur
des années différentes.

**Insight :** Avril 1998 est le meilleur mois avec 123 799 $ de CA net sur 74 commandes. Le début
d'année 1998 (janvier à avril) affiche une croissance continue exceptionnelle. En 1997, le pattern
est plus stable avec des pics en juillet (51 021 $) et octobre (66 749 $) — un cycle saisonnier
T3/T4 classique en distribution alimentaire.

```sql
-- ======================================
-- REQUÊTE 9 : Saisonnalité des ventes
-- Business context : identifier les pics
-- d'activité pour optimiser stocks et
-- planification commerciale
-- ======================================

SELECT
    EXTRACT(YEAR  FROM o.order_date) AS annee,
    EXTRACT(MONTH FROM o.order_date) AS mois,
    COUNT(DISTINCT o.order_id)       AS nb_commandes,
    ROUND(SUM((od.quantity * od.unit_price) * (1 - od.discount))) AS ca_net_par_mois
FROM orders o
INNER JOIN order_details od ON od.order_id = o.order_id
GROUP BY annee, mois
ORDER BY annee DESC, mois DESC;
```

---

### Requête 10/10 — Top Produit par Catégorie (Window Function)

**Question business :** Quel est le produit champion de chaque catégorie en termes de chiffre
d'affaires net ? Cette information guide les décisions de mise en avant commerciale et de gestion
des stocks prioritaires.

**Pourquoi cette analyse ?** On ne peut pas filtrer directement sur un rang calculé par une window
function : SQL évalue les window functions **après** le WHERE, donc le rang n'existe pas encore au
moment du filtrage. La solution est une double CTE : la première agrège les ventes par produit et
catégorie, la seconde applique `ROW_NUMBER()` avec `PARTITION BY category_id` pour attribuer un rang
au sein de chaque catégorie. On filtre ensuite sur `rang = 1`.

**Insight :** Côte de Blaye domine Beverages avec 141 397 $ de CA net — un produit unique qui
représente à lui seul plus de 50% du CA de sa catégorie. Raclette Courdavault mène Dairy Products
avec 71 156 $. Ces produits champions sont des actifs stratégiques : une rupture de stock sur l'un
d'eux impacte directement le CA de toute la catégorie.

```sql
-- ======================================
-- REQUÊTE 10 : Top produit par catégorie
-- Business context : identifier les produits
-- champions pour prioriser stocks et
-- actions commerciales par catégorie
-- ======================================

-- CTE 1 : agrégation CA net par produit et catégorie
WITH ca_par_produit AS (
    SELECT
        c.category_id,
        p.product_id,
        p.product_name,
        COUNT(DISTINCT od.order_id)                                    AS nb_commandes,
        SUM(od.quantity)                                               AS quantite_totale,
        ROUND(SUM((od.quantity * od.unit_price) * (1 - od.discount))) AS ca_net_par_produit
    FROM categories c
    INNER JOIN products      p  ON p.category_id = c.category_id
    INNER JOIN order_details od ON od.product_id = p.product_id
    GROUP BY c.category_id, p.product_id, p.product_name
),

-- CTE 2 : classement par catégorie avec window function
rang_produits AS (
    SELECT
        c.category_name,
        cpp.product_name,
        cpp.ca_net_par_produit,
        cpp.quantite_totale,
        ROW_NUMBER() OVER (
            PARTITION BY cpp.category_id
            ORDER BY cpp.ca_net_par_produit DESC
        ) AS rang
    FROM ca_par_produit cpp
    INNER JOIN categories c ON c.category_id = cpp.category_id
)

-- Sélection finale : uniquement le champion de chaque catégorie
SELECT *
FROM rang_produits
WHERE rang = 1
ORDER BY ca_net_par_produit DESC;
```
