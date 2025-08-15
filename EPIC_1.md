# EPIC 1 — Ingestion & Schéma des données (ETL Polars)

**Objectif global :**  
Mettre en place une chaîne d’importation de données TFT fiable et reproductible, qui lit les fichiers bruts, vérifie leur structure, les normalise (patch, stage, round), puis les stocke dans un format optimisé pour les futures étapes de Machine Learning.

---

## 🎯 TFT-ING-1 — Définir schéma brut & contrat de données (SP: 3)

**Description :**  
- Analyser les fichiers bruts (JSON, CSV ou autre) provenant de l’API ou d’exports locaux.  
- Identifier tous les champs disponibles pour **les parties**, **les rounds**, **les shops** et **les joueurs**.  
- Définir un **contrat de données** : types attendus, contraintes (ex. `match_id` unique, `round` ≥ 1), champs obligatoires vs optionnels.  
- Préparer un document de référence qui servira à toutes les étapes suivantes pour éviter les incohérences.  

**Pourquoi :**  
- Sans schéma clair, les étapes suivantes risquent de casser à cause de données manquantes ou mal typées.  
- Le contrat de données permettra aussi d’ajouter facilement de nouvelles sources à l’avenir.  

**Comment :**  
1. Lire un échantillon de fichiers bruts.  
2. Lister tous les champs et types détectés.  
3. Définir un dictionnaire `{champ: type, contraintes, description}`.  
4. Sauvegarder en Markdown (`schema.md`) et générer un exemple valide (`examples/sample.json`).  

**Critères d’acceptation :**  
- Document complet avec tous les champs, types, contraintes et explications.  
- Exemple de fichier valide fourni.  
- Tous les champs critiques marqués comme obligatoires.  

**Livrables :**  
- `docs/schema.md`  
- `examples/sample.json`  

**Dépendances :** —  

---

## 🎯 TFT-ING-2 — Lecteur Polars + validation de schéma (SP: 5)

**Description :**  
- Écrire un script Python utilisant **Polars** pour charger les données brutes.  
- Ajouter une validation automatique du schéma (ex. via **pydantic** ou **pandera**).  
- Les fichiers qui ne respectent pas le contrat doivent être rejetés avec un log clair expliquant le problème.  

**Pourquoi :**  
- Assurer que toutes les données utilisées plus tard soient propres et cohérentes.  
- Gagner du temps en détectant les anomalies dès l’import.  

**Comment :**  
1. Lire les fichiers avec `pl.read_json()` ou `pl.read_parquet()` selon le format.  
2. Passer chaque DataFrame dans un validateur qui vérifie types et contraintes.  
3. Écrire un log listant les fichiers invalides et la raison.  

**Critères d’acceptation :**  
- Un fichier valide passe sans erreur.  
- Un fichier invalide est rejeté avec un message explicite (ex. `"champ hp manquant"`).  
- Tests unitaires sur 3 cas : valide, champ manquant, type incorrect.  

**Livrables :**  
- `etl/load.py` (fonction `load_and_validate(path)` qui retourne un DataFrame Polars propre)  
- `tests/test_load.py`  

**Dépendances :** TFT-ING-1  

---

## 🎯 TFT-ING-3 — Normalisation par patch/stage/round (SP: 5)

**Description :**  
- Ajouter dans le DataFrame des colonnes dérivées normalisées :  
  - **`patch`** : version exacte (ex. `"14.15"`), extraite de la date ou du champ version.  
  - **`stage`** : ex. `"3-2"` (stage 3 round 2).  
  - **`round_idx`** : numéro séquentiel global du round (commence à 1 au début de la partie).  
- Gérer les fuseaux horaires si les timestamps sont utilisés.  

**Pourquoi :**  
- Le patch est nécessaire pour entraîner des modèles qui respectent la méta.  
- Les stages/rounds sont essentiels pour comprendre le tempo de la partie.  
- `round_idx` est utile pour traiter la partie comme une séquence.  

**Comment :**  
1. Écrire une fonction `normalize_round_info(df)` qui :  
   - lit les champs bruts (timestamp, round label, etc.)  
   - en déduit `patch`, `stage`, `round_idx`.  
2. Ajouter tests unitaires pour 3 exemples de parties.  

**Critères d’acceptation :**  
- Tous les rounds ont un `stage` et un `round_idx` corrects.  
- Aucun patch vide ou incorrect.  

**Livrables :**  
- `etl/normalize.py`  
- `tests/test_normalize.py`  

**Dépendances :** TFT-ING-2  

---

## 🎯 TFT-ING-4 — Stockage colonne (Parquet) + partitionnement (SP: 5)

**Description :**  
- Sauvegarder les données propres dans un format **Parquet** (stockage en colonnes).  
- Partitionner par `patch` et par `date` pour faciliter les filtrages ultérieurs.  
- Vérifier que la lecture partielle est rapide (< 1s pour un échantillon d’1M lignes).  

**Pourquoi :**  
- Le Parquet est optimisé pour l’analytique et réduit l’espace disque.  
- Le partitionnement permet de charger uniquement les données d’un patch sans tout lire.  

**Comment :**  
1. Utiliser `df.write_parquet("data/processed/", partition_by=["patch", "date"])`.  
2. Vérifier la taille du fichier et le temps de lecture avec `time.time()`.  
3. Écrire un test qui charge uniquement un patch donné et mesure le temps.  

**Critères d’acceptation :**  
- Lecture d’un seul patch en < 1s sur dataset d’1M lignes.  
- Données identiques avant/après sauvegarde (vérif hash).  

**Livrables :**  
- `etl/store.py`  
- `tests/test_store.py`  

**Dépendances :** TFT-ING-3  

---

## 📦 Résultat attendu de l'EPIC 1

- Un schéma de données clair et validé (`docs/schema.md`)  
- Un import fiable en **Polars** avec validation automatique  
- Des colonnes normalisées : `patch`, `stage`, `round_idx`  
- Un stockage rapide en Parquet, prêt pour la suite du projet
