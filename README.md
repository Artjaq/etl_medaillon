# Architecture Médaillon ETL — Apache Hop + PostgreSQL

Pipeline ETL complet implémentant l'architecture Médaillon (Bronze → Silver → Gold) avec Apache Hop et PostgreSQL.

---

## Prérequis

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installé et en cours d'exécution
- [Apache Hop](https://hop.apache.org/download/) (version compatible avec les fichiers du repo)
- Git

---

## 1. Démarrer PostgreSQL avec Docker

Lancer un conteneur PostgreSQL :

```bash
docker run -d \
  --name postgres-medaillon \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=admin \
  -e POSTGRES_DB=hop_db \
  -p 5432:5432 \
  postgres:latest
```

Vérifier que le conteneur tourne :

```bash
docker ps
```

---

## 2. Cloner le projet

```bash
git clone https://github.com/<votre-username>/Medaillon.git
cd Medaillon
```

---

## 3. Lancer Apache Hop

### Option A — Interface graphique (recommandé)

```bash
cd hop-2
./hop-gui.sh        # macOS / Linux
hop-gui.bat         # Windows
```

### Option B — Ligne de commande

```bash
cd hop-2
./hop-run.sh        # macOS / Linux
hop-run.bat         # Windows
```

---

## 4. Configurer la connexion PostgreSQL dans Hop

1. Dans Hop GUI, ouvrir **File → Metadata → Relational Database Connection**
2. Sélectionner la connexion **postgreSQL**
3. Vérifier / mettre à jour les paramètres :

| Paramètre | Valeur |
|-----------|--------|
| Host      | `localhost` |
| Port      | `5432` |
| Database  | `hop_db` |
| Username  | `admin` |
| Password  | `admin` |

4. Cliquer sur **Test** pour valider la connexion, puis **OK**

---

## 5. Exécuter le workflow principal

### Via l'interface graphique

1. Dans Hop GUI, ouvrir le fichier **`main_workflow.hwf`**
2. Cliquer sur le bouton **Run** (▶)
3. Sélectionner la configuration d'exécution **local**
4. Cliquer sur **Launch**

### Via la ligne de commande

```bash
cd hop-2
./hop-run.sh \
  -r local \
  -j ../main_workflow.hwf \
  -p PROJECT_HOME=..
```

---

## 6. Ordre d'exécution des pipelines

Le workflow `main_workflow.hwf` orchestre automatiquement les pipelines dans l'ordre suivant :

```
main_workflow.hwf
  ├── load_bronze.hpl   → Ingestion des CSV bruts dans le schéma Bronze
  ├── load_silver.hpl   → Nettoyage et standardisation dans le schéma Silver
  └── load_gold.hpl     → Modélisation en étoile dans le schéma Gold
```

Pour exécuter un pipeline individuellement :

```bash
./hop-run.sh -r local -f ../load_bronze.hpl -p PROJECT_HOME=..
./hop-run.sh -r local -f ../load_silver.hpl -p PROJECT_HOME=..
./hop-run.sh -r local -f ../load_gold.hpl   -p PROJECT_HOME=..
```

---

## 7. Données sources

Les fichiers CSV sources se trouvent dans :

```
datasets/
  source_crm/    # Données CRM (clients, produits, ventes)
  source_erp/    # Données ERP (clients, localisations, catégories)

datasets 2/
  source_crm/
  source_erp/
```

---

## 8. Vérifier les résultats dans PostgreSQL

Se connecter à la base et vérifier les schémas créés :

```bash
docker exec -it postgres-medaillon psql -U admin -d hop_db
```

```sql
-- Vérifier les schémas
\dn

-- Compter les lignes chargées
SELECT COUNT(*) FROM bronze.crm_sales_details;
SELECT COUNT(*) FROM silver.crm_cust_info;
SELECT COUNT(*) FROM gold.fact_sales;
```

---

## Structure du projet

```
Medaillon/
├── main_workflow.hwf     # Workflow principal
├── load_bronze.hpl       # Pipeline Bronze
├── load_silver.hpl       # Pipeline Silver
├── load_gold.hpl         # Pipeline Gold
├── metadata/             # Connexions et configurations Hop
├── datasets/             # Fichiers CSV sources
├── datasets 2/           # Fichiers CSV sources (jeu alternatif)
└── hop-2/                # Runtime Apache Hop
```
