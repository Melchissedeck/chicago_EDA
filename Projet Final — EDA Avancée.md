---
updated: 2026-02-25T15:06:07.211+01:00
edited_seconds: 269
---
## "Anatomie d'un marché : Airbnb au microscope"

**Type : Projet individuel**
## Contexte
Inside Airbnb publie des snapshots publics de la plateforme dans les grandes villes mondiales. Ces données semblent simples en surface, mais on y retrouve  de multiples biais, documentés, ou non, de relations non-linéaires, de structures temporelles et spatiales fortes.

Votre mission : conduire une analyse exploratoire **rigoureuse** pour répondre à la question centrale:

> **"Le statut Superhost d'un hôte a-t-il un effet causal sur le prix de son logement dans la ville sélectionnée ?"**

Cette question apparemment simple va vous forcer à mobiliser l'ensemble des outils du module : raisonnement causal, visualisation haute dimension, détection de relations non-linéaires, analyse temporelle et spatiale.

---
## Dataset
**Source** : [Inside Airbnb](http://insideairbnb.com/get-the-data/) (snapshot le plus récent)
Télécharger les trois fichiers suivants :
- `listings.csv.gz` 
- `reviews.csv.gz` 
- `neighbourhoods.geojson`

**Caractéristiques du dataset** :
- Variables mixtes 
- Missing values non aléatoires
- Biais de sélection documenté (les listings inactifs restent dans le dataset)
- Relations fortement non-linéaires 
- Dimension temporelle 
- Dimension spatiale

---
## Structure du projet — 4 phases

---
### Phase 1 — Causalité et biais

_Séance 1 : Modèle de Rubin, DAGs, Simpson, Bootstrapping, PSM_

**Contexte** : Avant toute analyse, il faut comprendre ce que le dataset représente vraiment — et ce qu'il ne représente pas.

#### **1.1 Identification et quantification des biais**
Identifier et documenter au minimum **3 biais structurels** dans le dataset. Pour chacun, donner une estimation chiffrée de son ampleur :
- **Biais de sélection** : comparer les distributions de prix entre listings avec 0 avis, 1-5 avis, et 5+ avis. Ces trois groupes représentent-ils la même population ? Justifier avec des statistiques descriptives et un test approprié.
- **Biais de survivorship** : les listings qui existent encore aujourd'hui ne sont pas représentatifs de tous les listings qui ont jamais existé. Comment cela affecte-t-il les conclusions sur le "prix moyen" du marché ?
- **Biais de mesure** : le nombre d'avis est-il un proxy fiable de l'activité réelle ? Argumenter en comparant les distributions et en calculant des intervalles de confiance par bootstrapping (B=2000) sur la corrélation avis/prix.

#### **1.2 Construction du DAG causal**
Construire un DAG pour la question centrale : _l'effet du statut Superhost sur le prix_.
Le DAG doit contenir **au minimum 8 nœuds** parmi : `superhost_status`, `price`, `location` (arrondissement), `room_type`, `host_experience` (ancienneté), `number_of_reviews`, `review_score`, `availability`, `minimum_nights`.

Pour chaque nœud, justifier son rôle : confounder, médiateur, collider, ou variable instrumentale potentielle. Identifier clairement les backdoor paths qui biaiseraient une comparaison naïve Superhost vs non-Superhost.

#### **1.3 Détection d'un paradoxe de Simpson**
Calculer la corrélation globale entre `number_of_reviews` et `price`. Puis la recalculer **séparément** pour chaque `room_type` (Entire home, Private room, Shared room).

Si un renversement de tendance est observé : l'expliquer causalement à partir du DAG construit en 1.2. Quelle variable est le confounder responsable ?

#### **1.4 Propensity Score Matching**
Estimer l'effet causal du statut Superhost sur le prix en utilisant le PSM.
- Estimer le score de propension par régression logistique sur les confounders identifiés dans le DAG (arrondissement, room_type, ancienneté de l'hôte, review_score)
- Apparier avec la méthode du plus proche voisin (caliper = 0.05)
- Valider le matching avec un Love plot et vérifier SMD < 0.1 pour toutes les covariables
- Interpréter l'ATT (Average Treatment Effect on the Treated) : quel est l'effet estimé du statut Superhost sur le prix, une fois les confounders contrôlés ?

---
### Phase 2 — Visualisation haute dimension

_Séance 2 : Réduction dimensionnelle, perception visuelle, visualisations spécialisées_

**Contexte** : Le dataset contient 75 colonnes. L'exploration une-à-une est impossible. Il faut des outils pour voir la structure globale et détecter des groupes.

#### **2.1 Préparation et sélection de features**
Sélectionner manuellement 15 à 20 variables numériques pertinentes (prix, scores, ancienneté, disponibilité, nombre d'avis, etc.). Justifier les exclusions. Normaliser.

#### **2.2 PCA exploratoire**
- Tracer le scree plot : combien de composantes expliquer 80% de la variance ?
- Produire un biplot PC1/PC2 coloré par `room_type`
- Analyser les loadings : quelles variables contribuent le plus à PC1 ? à PC2 ? Que représentent ces axes ?

#### **2.3 UMAP pour découverte de clusters**
- Appliquer UMAP sur les mêmes features
- Colorer successivement par : `room_type`, `superhost_status`, `arrondissement`, `price` (discrétisé en quartiles)
- Comparer avec la projection PCA : quelles structures UMAP révèle-t-il que PCA manque ?

#### **2.4 Visualisations spécialisées**
Produire **deux** des trois visualisations suivantes (au choix, en justifiant) :
- _Parallel coordinates_ sur un sous-échantillon de 2000 listings, coloré par `room_type`, avec réordonnancement des axes pour faire apparaître les patterns les plus lisibles
- _Ridgeline plot_ de la distribution de `log(price)` par arrondissement, ordonné par médiane croissante
- _Scatterplot matrix_ (pairplot) sur 6 variables clés, coloré par `superhost_status`, avec courbe de régression locale (LOESS) sur chaque scatter

---
### Phase 3 — Relations non-linéaires et analyses multivariées

_Séance 3 : Distance correlation, MIC, partial correlations, GAMs, interactions_

**Contexte** : Les relations dans ce dataset sont rarement linéaires. Pearson seul va vous tromper.

#### **3.1 Comparaison de métriques de dépendance**

Sur les 5 paires de variables suivantes, calculer **Pearson, Spearman, distance correlation et MIC** :

| Paire                                        | Hypothèse sur la relation               |
| -------------------------------------------- | --------------------------------------- |
| `price` ~ `number_of_reviews`                | Non-linéaire, décroissante puis plateau |
| `price` ~ `review_scores_rating`             | Non-linéaire, seuil d'effet             |
| `price` ~ `availability_365`                 | Complexe, bimodale possible             |
| `price` ~ `host_listings_count`              | Non-linéaire, économies d'échelle       |
| `number_of_reviews` ~ `review_scores_rating` | Faible globalement, forte localement    |

Pour chaque paire : visualiser le scatter plot, comparer les 4 métriques, et commenter les divergences. Quelles paires Pearson sous-estime-t-il le plus fortement ?

#### **3.2 Partial correlations**

La corrélation brute entre `number_of_reviews` et `price` est influencée par plusieurs confounders. Calculer la corrélation partielle entre ces deux variables en contrôlant successivement :

- uniquement `room_type`
- uniquement `arrondissement` (encodé numériquement ou via dummy)
- les deux simultanément

Que reste-t-il de la relation une fois les deux confounders contrôlés ? Interpréter en lien avec le DAG construit en Phase 1.

#### **3.3 GAM — Modélisation non-linéaire**

Fitter un GAM pour modéliser `log(price)` en fonction de :

- `s(distance_to_center)` — calculer la distance en km depuis le centre de Paris (48.8566°N, 2.3522°E)
- `s(number_of_reviews)`
- `s(review_scores_rating)`
- `room_type` (terme linéaire)

Pour chaque terme lissé, tracer la courbe de dépendance partielle avec son intervalle de confiance. Identifier les non-linéarités majeures : à quelle distance le prix commence-t-il à chuter ? Quel est le volume d'avis optimal ?


---
### Phase 4 — Temporel et spatial
_Séance 4 : STL, changepoints, Moran's I, LISA, visualisations spatiales_

**Contexte** : Les prix Airbnb ne sont pas constants dans le temps ni uniformes dans l'espace. Ces deux dimensions recèlent des patterns que les analyses cross-sectionnelles manquent entièrement.

#### Tâches

**4.1 Analyse temporelle de l'activité de la plateforme**

À partir du fichier `reviews.csv`, construire une série temporelle mensuelle du nombre d'avis (proxy de l'activité de réservation).
- Appliquer une décomposition STL (période = 12 mois)
- Visualiser les trois composantes séparément (tendance, saisonnalité, résidus)
- Interpréter la saisonnalité : quels mois concentrent l'activité ? Est-ce stable d'une année à l'autre ?
- Appliquer PELT pour détecter les changepoints. Combien en trouvez-vous ? Les dater et les expliquer avec des événements réels (COVID, réglementation Airbnb Paris, etc.)

**4.2 Autocorrélation temporelle**
Sur la même série temporelle mensuelle :
- Tracer ACF et PACF jusqu'au lag 24
- La série est-elle stationnaire ? Appliquer les tests ADF et KPSS et interpréter conjointement
- Si non stationnaire : différencier et re-tester

**4.3 Autocorrélation spatiale des prix**
Agréger les listings par arrondissement (médiane de `log(price)`).
- Construire la matrice de poids spatiaux W par contiguïté (arrondissements qui partagent une frontière) en utilisant le fichier GeoJSON
- Calculer le **Moran's I** global et interpréter : les prix similaires ont-ils tendance à se regrouper géographiquement ?
- Calculer les **indicateurs LISA** (Local Moran's I) pour chaque arrondissement
- Produire une carte choroplèthe interactive (Plotly ou Folium) avec deux couches : prix médian par arrondissement, et classification LISA (HH, LL, HL, LH)

**4.4 Visualisation combinée**
Produire un **calendar heatmap** du nombre d'avis par jour sur les 3 dernières années disponibles. Identifier visuellement les ruptures et les cycles saisonniers.

---
## Livrables

### 1. Notebook analytique (obligatoire)
Un seul fichier `.ipynb`, organisé en 4 sections correspondant aux 4 phases. Chaque cellule de code doit être précédée d'une cellule markdown expliquant **ce qu'on cherche à montrer** et suivie d'une cellule markdown **interprétant le résultat**.

Contraintes minimales :
- Seed fixé pour reproductibilité
- 20+ visualisations (dont au minimum : 1 DAG, 1 Love plot, 1 biplot PCA, 1 projection UMAP, 1 graphique GAM, 1 décomposition STL, 1 carte choroplèthe)
- Chaque analyse doit conclure explicitement sur la question centrale (effet du statut Superhost) ou sur un biais identifié

### 2. Executive summary (obligatoire)
Un PDF d'**une page exactement** structuré ainsi :
- **5 insights** (une phrase chacun, actionnables, non triviaux)
- **1 visualisation "hero"** : la plus impactante du projet, légendée
- **2 limitations** : ce que l'analyse ne permet pas de conclure, et pourquoi

---
## Critères d'évaluation — 20 points
### Rigueur causale — 6 pts

| Critère                                                      | Points |
| ------------------------------------------------------------ | ------ |
| DAG complet et justifié (rôles des nœuds explicités)         | 2      |
| PSM correctement implémenté, Love plot produit, SMD vérifiés | 2      |
| Interprétation appropriée (causalité vs corrélation)         | 1      |
| Simpson's paradox détecté et expliqué causalement            | 1      |

### Analyses multivariées — 5 pts

|Critère|Points|
|---|---|
|Comparaison des 4 métriques de dépendance commentée|2|
|Partial correlations calculées et interprétées|1|
|GAM fitté, courbes de dépendance tracées et commentées|2|

### Visualisation — 5 pts

| Critère                                                           | Points |
| ----------------------------------------------------------------- | ------ |
| PCA biplot informatif avec interprétation des axes                | 1      |
| UMAP pertinent, comparaison avec PCA argumentée                   | 1      |
| Visualisations spécialisées (phase 2.4) bien choisies et lisibles | 1      |
| Carte choroplèthe avec LISA produite                              | 1      |
| Cohérence visuelle globale (titres, légendes, palettes adaptées)  | 1      |

### Temporel et spatial — 3 pts

|Critère|Points|
|---|---|
|STL décomposée, changepoints datés et expliqués|1|
|Moran's I calculé et interprété|1|
|Calendar heatmap produit|1|

### Communication — 1 pt

| Critère                                                                  | Points |
| ------------------------------------------------------------------------ | ------ |
| Executive summary : insights non triviaux, visualisation hero impactante | 1      |
### Malus
- Code non reproductible (seed manquant, chemins absolus) : **-2 pts**
- Conclusions causales sans justification (PSM absent, DAG ignoré) : **-2 pts**
- Visualisations sans titre, légende ou unités : **-1 pt**
- Visualisations générées out-of-the-box: **-5 pts**

### Bonus — jusqu'à +2 pts
- Interaction SHAP implementée et commentée (Phase 3.4) : **+1 pt**
- L'executive summary raconte une histoire cohérente qui relie les 4 phases : **+1 pt**


> [!WARNING] Attention
> Toute production manifestement générée par LLM sera pénalisée; la note reflétera alors uniquement la partie du travail réellement réalisée par l'étudiant. Je me réserve le droit de pondérer votre note à l'utilisation de LLM présente dans votre rendu.


---
## Conseils méthodologiques

**Sur le DAG** : ne pas chercher à inclure toutes les variables — un DAG trop dense est illisible et peu utile. Se concentrer sur les variables qui ont une relation causale plausible avec soit le traitement (superhost), soit l'outcome (prix).

**Sur le PSM** : si le support commun est faible (peu de non-Superhosts avec un score proche des Superhosts), le dire explicitement. Un PSM mal équilibré dont on reconnaît les limites vaut mieux qu'un PSM qu'on fait semblant de valider.

**Sur UMAP** : ne pas chercher à "nommer" les axes. Se concentrer sur les groupes qui émergent et sur ce qui les caractérise (profil de variables).

**Sur le GAM** : si `pygam` ne converge pas, réduire le nombre de knots ou augmenter la régularisation. Un GAM simple qui converge est plus utile qu'un GAM complexe qui ne converge pas.

**Sur le Moran's I** : 20 arrondissements est un petit n pour l'autocorrélation spatiale — les résultats seront moins stables que sur un découpage plus fin. Le mentionner dans les limitations.

**Règle d'or** : mieux vaut une analyse incomplète bien interprétée qu'une analyse complète sans interprétation. Chaque résultat doit être suivi d'une phrase de conclusion.