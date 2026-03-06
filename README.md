# Anatomie d'un marché : Airbnb à Chicago

Analyse exploratoire avancée du marché Airbnb de Chicago à partir des données publiques Inside Airbnb (snapshot septembre 2025).

**Question centrale** : Le statut Superhost a-t-il un effet causal sur le prix d'un logement ?

**Réponse** : Non. Après PSM sur 3 332 paires appariées, l'ATT est de −$0.63 [IC 95 % : −$10.86 ; +$9.94] — aucun effet causal mesurable.

---

## Données

Source : [Inside Airbnb — Chicago](http://insideairbnb.com/get-the-data/)

| Fichier | Dimensions | Description |
|---|---|---|
| `listings.csv` | 8 660 × 79 | Listings avec prix, scores, caractéristiques |
| `reviews.csv` | 492 465 × 6 | Avis de 2009 à 2025 |
| `neighbourhoods.geojson` | 77 quartiers | Géométries pour l'analyse spatiale |

Après nettoyage : **7 746 listings**, 82 colonnes, seed = 2077.

---

## Phases d'analyse

### Phase 1 — Causalité et biais
- Identification de 3 biais structurels (sélection, survivorship, mesure)
- DAG causal à 9 nœuds avec justification des rôles
- Détection du paradoxe de Simpson sur `number_of_reviews ~ price` stratifié par `room_type`
- Propensity Score Matching avec Love plot et vérification des SMD

### Phase 2 — Visualisation haute dimension
- ACP sur 17 variables (7 composantes pour 80 % de variance)
- UMAP comparé à l'ACP — révèle deux clusters nets (Entire homes vs chambres)
- Ridgeline plot par quartier + scatterplot matrix coloré par statut Superhost

### Phase 3 — Relations non-linéaires
- Comparaison de 4 métriques de dépendance : Pearson, Spearman, Distance Correlation, MIC
- Corrélations partielles en contrôlant `room_type` et `neighbourhood`
- GAM (R² = 0.44) avec courbes de dépendance partielle — effet de seuil détecté à score > 4.7/5

### Phase 4 — Temporel et spatial
- Décomposition STL sur 188 mois (2010–2025) — pic saisonnier juillet–août–juin
- Tests ADF et KPSS — série non stationnaire, stationnaire après différenciation d=1
- Moran's I = 0.358 (p = 0.001) — clustering spatial significatif
- Classification LISA : 11 zones HH (Nord/Centre), 7 zones LL (Sud/Ouest)
- Calendar heatmap sur les 3 dernières années

---

## Principaux résultats

| Résultat | Valeur |
|---|---|
| ATT Superhost → prix (PSM) | −$0.63 [−$10.86 ; +$9.94] |
| Biais de survivorship | +32.6 % (listings inactifs vs actifs) |
| Biais de sélection (Kruskal-Wallis) | H = 88.5, p = 6.2e−20 |
| GAM R² pseudo | 0.44 |
| Moran's I global | 0.358 (p = 0.001) |
| Seuil effet score sur prix | > 4.7 / 5 |

---

## Installation

```bash
pip install numpy pandas matplotlib seaborn geopandas networkx scipy \
            scikit-learn umap-learn pygam statsmodels ruptures \
            libpysal esda folium plotly dcor
```

> La version `dcor 0.7` utilise `distance_correlation()` — le notebook détecte automatiquement la bonne fonction selon la version installée.

---

## Reproductibilité

Seed fixé à **2077** dans toutes les cellules aléatoires (PSM, bootstrap, UMAP, train/test split). Les chemins sont relatifs, placer les fichiers de données à la racine du projet.
