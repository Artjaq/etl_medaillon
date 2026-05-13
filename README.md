# 📝 Rapport de Projet ETL : Architecture Médaillon avec Apache Hop

## 1. Contexte et Objectifs du Projet
Ce projet consiste à concevoir et implémenter une chaîne d'intégration de données (ETL) *on-premise* en suivant l'approche **Médaillon**. L'objectif est d'extraire de manière intégrale (full extraction) les données brutes de deux systèmes sources simulés par des fichiers CSV — un CRM et un ERP — afin de les charger, de les nettoyer et de les modéliser dans un entrepôt de données décisionnel PostgreSQL.

L'entrepôt est structuré en trois schémas distincts reflétant la progression de la qualité de la donnée :
* **Bronze** : Stockage des données brutes.
* **Silver** : Données nettoyées, dédupliquées et standardisées.
* **Gold** : Données modélisées en étoile pour l'analyse BI.

## 2. Architecture de la Solution

### Couche Bronze (Ingestion)
Création de 6 tables d'accueil calquées sur la structure des fichiers sources :
* **CRM** : `crm_sales_details`, `crm_cust_info`, `crm_prd_info`.
* **ERP** : `erp_cust_az12`, `erp_loc_a101`, `erp_px_cat_g1v2`.

### Couche Silver (Qualité & Standardisation)
Dans cette couche, les tables conservent le même nom et intègrent systématiquement un champ technique `dwh_create_date` horodatant l'insertion. Les règles appliquées incluent :
* **`crm_cust_info`** : Suppression des doublons (conservation de la ligne la plus récente via `ROW_NUMBER()`), nettoyage des espaces (*trim*) sur les noms/prénoms, et harmonisation des statuts maritaux et des genres (`Single`, `Married`, `Male`, `Female`, `n/a`).
* **`crm_prd_info`** : Extraction du `cat_id` depuis la clé produit (remplacement du tiret par un underscore), correction de la clé `prd_key`, remplacement des coûts nuls par `0`, et décodage complet de la ligne de produit.
* **`crm_sales_details`** : Conversion des entiers au format de date standard SQL et recalculs de cohérence financière sur les montants de ventes et prix unitaires erronés ou nuls.
* **`erp_cust_az12`** : Nettoyage des dates de naissance futures et codification uniforme du genre.
* **`erp_loc_a101`** : Suppression des tirets sur l'identifiant client, *trim* et harmonisation des codes pays (`DE` ➔ `Germany`, `US`/`USA` ➔ `United States`).
* **`erp_px_cat_g1v2`** : Copie directe conforme sans transformation applicative.

### Couche Gold (Modélisation Décisionnelle)
Transformation de la structure opérationnelle en un **modèle en étoile** optimisé pour les requêtes BI :
* **`dim_customers`** : Consolidation des attributs clients CRM et ERP via des jointures externes sur les clés nettoyées.
* **`dim_products`** : Regroupement des informations produits du CRM et de leurs libellés de catégories issus de l'ERP.
* **`fact_sales`** : Table de faits centrale contenant les clés de substitution associées aux dimensions ainsi que les métriques quantitatives (quantités, prix, montants de ventes).

## 3. Journal des Incidents & Problèmes Corrigés (Bug Fixes)
* **Incident 1 : Authentification PostgreSQL** -> Résolu en configurant explicitement les credentials globaux dans les métadonnées de connexion Hop.
* **Incident 2 : Troncature sur les chaînes (Christopher)** -> Résolu en élargissant préventivement les VARCHAR à 50/100 dans le CSV Input et en appliquant un DROP/CREATE sur les tables Bronze.
* **Incident 3 : Requêtes UPDATE invalides de l'IHM** -> Résolu en nettoyant le bloc SQL d'Hop pour forcer un script DDL propre.
* **Incident 4 : Parsing de date dans les ventes (Erreur "MM")** -> Résolu en sécurisant l'extraction SQL : n'applique le TO_DATE que si la chaîne fait strictement 8 caractères (`LENGTH = 8`), sinon renvoie NULL.
* **Incident 5 : Référence circulaire DockerRun** -> Résolu en basculant la Run Configuration des actions du workflow sur le moteur 'local'.

## 4. Conclusion et Performance
Grâce à l'approche ELT combinant Apache Hop et la puissance de PostgreSQL, le workflow complet s'exécute de bout en bout en seulement **1,432 seconde** pour l'intégralité du volume de données.
