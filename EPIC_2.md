# EPIC 2 — Feature Engineering (V1 tabulaire)

**Objectif global :**  
Créer des features tabulaires robustes, calculées à partir des données brutes nettoyées, pour représenter l’état du joueur et de son board à chaque round.  
Ces features serviront aux premiers modèles de prédiction et de ranking.

---

## 🎯 TFT-FEA-1 — Économie & tempo (SP: 5)

**Description :**  
- Calculer les variables économiques et de tempo de jeu par round :  
  - `gold` : or disponible à ce round  
  - `interest` : intérêt gagné (palier de 10 gold, max 5)  
  - `xp` : points d’expérience actuels  
  - `level` : niveau du joueur  
  - `cost_to_level` : or nécessaire pour passer au niveau suivant  
  - `roll_odds` : probabilités d’apparition des unités par coût, selon le niveau actuel  

**Pourquoi :**  
- L’économie est un des facteurs clés de la stratégie TFT.  
- Le niveau et les probabilités d’apparition influencent directement les décisions de roll et d’achat.  

**Comment :**  
1. Utiliser les données de `level` et `xp` pour calculer `cost_to_level` selon la table TFT.  
2. Ajouter les `roll_odds` en mappant le niveau à une table de probabilités (fixe pour le set).  
3. Stocker toutes ces valeurs dans un DataFrame Polars.  

**Critères d’acceptation :**  
- Les calculs correspondent aux règles TFT officielles.  
- Les `roll_odds` changent correctement selon le niveau.  
- Tests unitaires sur 3 exemples connus.  

**Livrables :**  
- `features/economy.py`  
- `tests/test_economy.py`  

**Dépendances :** EPIC 1  

---

## 🎯 TFT-FEA-2 — Board & synergies (SP: 8)

**Description :**  
- Extraire les informations de force du board actuel :  
  - Nombre total d’unités sur le board  
  - Nombre d’unités par niveau d’étoiles (1★, 2★, 3★)  
  - Coût moyen des unités  
  - Points de vie effectifs estimés (**EHP**)  
  - Dégâts par seconde estimés (**DPS**)  
  - Synergies actives (traits) encodées en **one-hot**, avec leur niveau (1, 2, 3, etc.)  

**Pourquoi :**  
- La composition et la qualité du board déterminent directement les chances de victoire aux rounds suivants.  
- Les synergies influencent le choix d’items, de champions et d’augmentations.  

**Comment :**  
1. Parcourir la liste des unités sur le board.  
2. Compter le nombre par étoiles et calculer le coût moyen.  
3. Approximer EHP et DPS à partir des stats de base et des items.  
4. Encoder chaque trait en one-hot avec le niveau atteint.  

**Critères d’acceptation :**  
- Les traits actifs correspondent à la réalité du board.  
- Les EHP/DPS sont cohérents avec la somme des unités + items.  
- Tests unitaires sur 3 boards différents.  

**Livrables :**  
- `features/board.py`  
- `tests/test_board.py`  

**Dépendances :** TFT-FEA-1  

---

## 🎯 TFT-FEA-3 — Upgrades possibles & odds (SP: 5)

**Description :**  
- Calculer la probabilité d’obtenir un upgrade d’unité (passer de 1★ à 2★ ou 2★ à 3★) dans les prochains shops :  
  - Basé sur le nombre de copies déjà possédées  
  - Nombre de copies vues dans le lobby  
  - Copies restantes dans la pioche  
  - Probabilités d’apparition par coût au niveau actuel  

**Pourquoi :**  
- Savoir si un upgrade est probable aide à décider quand roll ou conserver une unité sur le banc.  

**Comment :**  
1. Compter les copies possédées par unité.  
2. Estimer le nombre restant dans le pool (nombre total par coût – copies possédées par tous).  
3. Multiplier par les probabilités d’apparition (en fonction du niveau).  

**Critères d’acceptation :**  
- Les probabilités sont ≤ 1 et cohérentes avec les règles TFT.  
- Les tests sur des cas simples donnent les résultats attendus.  

**Livrables :**  
- `features/upgrades.py`  
- `tests/test_upgrades.py`  

**Dépendances :** TFT-FEA-2  

---

## 🎯 TFT-FEA-4 — Encodage shop/bench (SP: 3)

**Description :**  
- Représenter les unités disponibles à l’achat et sur le banc dans un format utilisable par un modèle de ranking :  
  - Encodage des unités (`champ_id`, `cost`, `traits`, `étoiles`)  
  - Groupes `shop_id` (identifiant unique par shop) pour le ranking listwise/pairwise  
  - Inclusion des features dérivées (compatibilité avec synergies actuelles, probabilité d’upgrade, coût)  

**Pourquoi :**  
- Le modèle de ranking d’achats doit avoir un format homogène pour scorer tous les candidats.  

**Comment :**  
1. Pour chaque round, lister les 5 unités du shop + celles du banc.  
2. Encoder les attributs dans un vecteur fixe (features numériques + one-hot traits).  
3. Associer un identifiant `shop_id` commun à tous les candidats du même shop.  

**Critères d’acceptation :**  
- Chaque shop a exactement 5 entrées en plus des éventuelles unités du banc.  
- Les encodages sont complets et sans valeurs manquantes.  

**Livrables :**  
- `features/shop.py`  
- `tests/test_shop.py`  

**Dépendances :** TFT-FEA-2
