# Estimer le prix d'un logement à partir des données DVF 2025

Projet Kit Data Science

**Membres du groupe :**
**1 :** NJIMOH PASSO Berthoni
**2 :** TCHATCHUENG DJOUM Gilles Boris
**3 :** Welehela Patricia

## Question business

**Peut-on estimer le prix d'une maison ou d'un appartement à partir de ses caractéristiques et de sa localisation, mieux qu'avec une règle simple comme « prix médian au m² de la commune × surface » ?**

## Données

- **Source :** [Demandes de valeurs foncières (DVF)](https://www.data.gouv.fr/fr/datasets/demandes-de-valeurs-foncieres/), publiées par la DGFiP sur data.gouv.fr.
- **Millésime :** 2025 (ventes du 1er janvier au 31 décembre 2025).
- **Fichier brut :** 3 793 113 lignes et 43 colonnes, au format texte avec le séparateur `|`.
- **Couverture :** la France entière, sauf l'Alsace-Moselle (Bas-Rhin, Haut-Rhin, Moselle) et Mayotte, qui ne figurent pas dans DVF.

Le fichier n'est pas propre : une vente occupe plusieurs lignes, certaines lignes sont répétées, des codes ont perdu leurs zéros initiaux, et les montants utilisent une virgule décimale. Ce travail de préparation est au cœur du projet.

## Organisation du dépôt

```
.
├── README.md
├── requirements.txt
├── clean_dvf_2025_ventes.parquet       # données préparées, produites par data_mining.ipynb
├── notice-descriptive-du-fichier-dvf-20221017.pdf
├── data/                               # données brutes (ignorées par Git)
│   └── ValeursFoncieres-2025.txt       # créée par get_data.ipynb
└── notebooks/
    ├── get_data.ipynb                 # télécharge et extrait le fichier DVF
    ├── data_mining.ipynb               # explore, nettoie et prépare les données
    └── train_models.ipynb              # entraîne et compare les modèles
```

## Installation et exécution

```bash
# Depuis la racine du dépôt, installer les dépendances
pip install -r requirements.txt
```

Ouvrir et exécuter les notebooks dans cet ordre, en gardant la racine du dépôt comme
répertoire de travail :

1. `notebooks/get_data.ipynb` télécharge et extrait le fichier brut dans `data/`.
2. `notebooks/data_mining.ipynb` lit ce fichier et produit
   `clean_dvf_2025_ventes.parquet` à la racine.
3. `notebooks/train_models.ipynb` charge ce parquet et entraîne les modèles.

Le dossier `data/` est ignoré par Git (voir `.gitignore`) car il contient le fichier
brut, volumineux. Le parquet préparé est un résultat intermédiaire nécessaire au
notebook de modélisation.

## Démarche

Chaque décision du notebook d'exploration suit le même schéma : **sortie de code → constat chiffré → décision**.

### 1. Lecture et correction des formats

- Le fichier est lu avec `dtype=str` : sans cela, pandas convertirait les codes en nombres et supprimerait leurs zéros initiaux.
- **Codes postaux :** 285 246 codes n'avaient que 4 caractères. 285 102 d'entre eux (99,95 %) appartiennent aux départements 01 à 09 : le zéro initial manque dans le fichier source. Ils ont été complétés avec `str.zfill(5)`. Les 144 restants concernent 5 communes frontalières dont le code postal appartient au département voisin ; ils sont corrects et ont été conservés.
- **Code commune INSEE :** le code commune avait aussi perdu ses zéros. Le code INSEE (5 caractères) a été reconstruit à partir du département et de la commune, en tenant compte de l'outre-mer (3 caractères de département). Il a été vérifié sur des communes connues (Farges → `01158`, Le Moule → `97117`). Il y a 32 712 communes distinctes.
- **Montants et surfaces :** la virgule décimale a été remplacée par un point avant la conversion en nombre. Le nombre de valeurs manquantes est identique avant et après chaque conversion : aucune valeur n'a été perdue.
- **Dates :** converties avec le format `%d/%m/%Y`, pour éviter que `07/01/2025` soit lu comme le 1er juillet.

### 2. Valeurs manquantes

- 19 colonnes vides à plus de 95 % ont été supprimées, dont 8 entièrement vides.
- Les colonnes d'adresse (455 906 noms de voie distincts) ont été supprimées : une adresse exacte ne se généralise pas, et la localisation est portée par le code INSEE.
- Les valeurs manquantes restantes ont surtout un sens « non applicable », vérifié dans les données :
  - 92 % des lignes sans type de local sont des terrains sans bâtiment ;
  - 93 % des lignes sans type de local ni terrain sont des ventes en l'état futur d'achèvement (VEFA) : le bien n'est pas encore construit ;
  - 98 % des lignes sans surface de terrain ont au moins un lot : ce sont des biens en copropriété, sans terrain privatif.
- Les 50 484 lignes sans valeur foncière (1,3 %) ont été retirées : sans prix, elles sont inutilisables pour l'entraînement.

### 3. Une ligne par vente

- Le fichier ne contient pas d'identifiant de vente. Un identifiant a été construit à partir de la date, du numéro de disposition, de la valeur foncière et du code INSEE : 3 742 629 lignes correspondent à 1 347 969 ventes.
- Une vente occupe souvent plusieurs lignes (logement, dépendances, parcelles), et la valeur foncière est répétée sur chacune. De plus, 19,2 % des lignes de locaux et 22 % des lignes de terrain sont des doublons (un même local répété pour chaque type de terrain de la parcelle). Les lignes ont été dédoublonnées avant d'additionner les surfaces, puis regroupées en une ligne par vente.

### 4. Périmètre

- **Natures de mutation :** seules les ventes classiques et les VEFA sont gardées. Les échanges, adjudications, expropriations et ventes de terrains à bâtir ne reflètent pas un prix de marché d'un logement (24 958 ventes retirées, 1,9 %).
- **Type de vente :** seules les ventes d'une seule maison ou d'un seul appartement sont gardées, avec ou sans dépendances. Pour les ventes de plusieurs logements ou mixtes, le prix total couvre des biens différents, et leur prix au m² est beaucoup plus dispersé. Les 8 850 ventes ambiguës (logements identiques sans numéro de lot) sont aussi retirées. Il reste 823 258 ventes.

### 5. Valeurs aberrantes

- Le diagnostic montre des prix absurdes (de 0,15 € à 257,5 M€) et une ligne de ventes à 1 €, probablement des cessions symboliques.
- Le prix médian au m² est 11,4 fois plus élevé à Paris (9 692 €/m²) que dans la Creuse (851 €/m²) : un seuil national n'aurait pas de sens.
- **Filtres retenus :** surface habitable entre 9 m² (surface minimale d'un logement décent) et 1 000 m², et prix au m² entre le 1er et le 99e percentile de son département et de son type de bien.
- 17 074 ventes ont été retirées (2,1 %). **Il reste 806 184 ventes.**

### 6. Choix de la cible

Trois cibles ont été comparées avec leur asymétrie (*skew*) :

| Cible | Skew |
|---|---|
| Valeur foncière | 8,25 |
| Prix au m² | 2,29 |
| **Logarithme de la valeur foncière** | **−0,22** |

La cible retenue est le **logarithme de la valeur foncière**, la plus symétrique. La valeur foncière et le prix au m² sont exclus des features, car ils servent à calculer la cible (data leakage). Les prédictions sont ramenées en euros avec `np.exp` pour calculer des erreurs compréhensibles.

### 7. Feature engineering

| Feature | Construction |
|---|---|
| `log_surface_bati` | logarithme de la surface habitable |
| `log_prix_med_commune` | logarithme du prix médian au m² de la commune (voir ci-dessous) |
| `log_surface_terrain` | `log(1 + surface du terrain)`, car les appartements ont un terrain de 0 m² |
| `nb_pieces` | nombre de pièces principales |
| `est_vefa` | 1 si vente en l'état futur d'achèvement (0,9 % des ventes) |
| `a_dependance` | 1 si la vente comprend une cave, un garage ou un parking (56 % des ventes) |
| `mois_sin`, `mois_cos` | mois de la vente placé sur un cercle, pour que décembre soit voisin de janvier |
| `type_vente` | maison ou appartement (one-hot) |
| `code_departement` | département (one-hot) |

**Prix médian de la commune.** Un one-hot sur 32 712 communes est inutilisable. La commune est donc remplacée par son prix médian au m², **calculé sur le jeu d'entraînement uniquement** pour éviter le data leakage. Les communes de moins de 10 ventes dans le train reçoivent la médiane de leur département. Seules 9 011 communes sur 29 237 ont au moins 10 ventes, mais elles regroupent 86,7 % des ventes du test.

**Pourquoi des logarithmes ?** Le prix est le produit d'un prix au m² et d'une surface. Le logarithme transforme ce produit en somme : log(prix) = log(prix au m²) + log(surface), ce qu'une régression linéaire peut apprendre.

### 8. Modélisation

- **Séparation :** 75 % des ventes pour l'entraînement (604 638) et 25 % pour le test (201 546), avec `random_state=41`.
- **Préprocessing :** un `Pipeline` avec un `ColumnTransformer` (imputation et standardisation pour les colonnes numériques, imputation et one-hot encoding pour les colonnes qualitatives).
- **Métriques :** MAE et RMSE en euros, R² sur le logarithme. Les scores du train et du test sont comparés pour repérer l'overfitting.
- **Modèles, dans l'ordre :**
  1. deux baselines : la moyenne, et la règle « prix médian au m² de la commune × surface » ;
  2. une régression linéaire avec les variables brutes, puis avec les variables en logarithme ;
  3. Ridge et Lasso, avec `alpha` choisi par `GridSearchCV` ;
  4. un arbre de décision, avec `max_depth` et `min_samples_leaf` choisis par `GridSearchCV`.

## Résultats

Résultats sur le jeu de test (201 546 ventes) :

| Modèle | MAE (€) | RMSE (€) | R² (log) | Écart R² train − test |
|---|---|---|---|---|
| **Arbre de décision** (profondeur 15, feuilles ≥ 50 ventes) | **60 664** | 120 722 | **0,707** | 0,025 |
| Lasso | 62 987 | 126 427 | 0,691 | 0,007 |
| Ridge | 63 030 | 126 703 | 0,692 | 0,006 |
| Régression linéaire (logarithmes) | 63 031 | 126 727 | 0,692 | 0,006 |
| Baseline : médiane de la commune × surface | 63 177 | **119 928** | 0,671 | — |
| Régression linéaire (variables brutes) | 77 667 | 608 730 | 0,633 | 0,000 |
| Baseline : moyenne | 123 417 | 237 398 | 0,000 | — |

**Ce qu'on retient :**

- **La règle simple est une référence difficile à battre.** « Prix médian au m² de la commune × surface » explique déjà 67 % de la variance.
- **La forme des variables compte plus que la régularisation.** Avec les variables brutes, la régression linéaire fait moins bien que la règle et produit des prédictions aberrantes (jusqu'à 157,5 M€, pour un appartement de 547 m² dans une commune à 13 107 €/m²). Avec les logarithmes, elle égale la règle. Ridge et Lasso n'apportent presque rien, car la régression n'overfit pas.
- **L'arbre de décision est le meilleur modèle** : −4 % d'erreur moyenne par rapport à la règle. Il overfit légèrement plus que les modèles linéaires. Les arbres trop profonds avec de petites feuilles (10 ventes) ont les moins bons scores en validation croisée.
- **La localisation et la surface** sont, de loin, les variables les plus importantes. Le mois de la vente ne pèse presque rien : le prix médian au m² varie d'environ 6 % d'un mois à l'autre.
- **Aucun modèle ne réduit les plus grosses erreurs :** la règle de la baseline garde la plus petite RMSE.



**Réponse à la question business :** oui, un modèle de machine learning estime le prix mieux que la règle simple, mais le gain reste modeste (−4 % d'erreur). Une erreur moyenne d'environ 60 000 €, pour un prix médian de 188 000 €, reste trop élevée pour fixer un prix de vente : le modèle donne un ordre de grandeur, pas un prix précis.

## Limites

- Les seuils de nettoyage sont des choix : 95 % de valeurs manquantes pour supprimer une colonne, 9 et 1 000 m² pour la surface, 1er et 99e percentiles pour le prix au m², 10 ventes minimum pour la médiane d'une commune.
- L'identifiant de vente peut fusionner deux ventes différentes conclues le même jour, dans la même commune et au même prix.
- Dans le train, chaque vente compte dans le prix médian de sa propre commune : les scores du train sont un peu optimistes.
- Les VEFA ne représentent que 0,9 % des ventes : le modèle a peu d'exemples pour apprendre leur prix.
- DVF ne décrit ni l'état du logement, ni l'étage, ni l'exposition : une partie du prix ne peut pas être expliquée avec ces données.
- La Random Forest n'a pas été entraînée, à cause du temps de calcul sur 600 000 ventes.

## Pistes

- Entraîner une Random Forest.
- Ajouter une localisation plus fine (latitude et longitude) ou des données externes (transports, DPE) etc.
- Étudier les ventes sur lesquelles le modèle se trompe le plus.
