# Prédiction des propriétés chimiques du fumier par spectroscopie NIR

Pipeline complet de chimiométrie : analyse exploratoire → prétraitement spectral → modélisation PLS → API de prédiction → dashboard interactif.

---

## Description

Ce projet applique la spectroscopie proche infrarouge (NIR) à la prédiction de propriétés chimiques de fumiers bovins et volailles. À partir de spectres bruts (700 variables, 1100–2498 nm), des modèles PLS entraînés séparément pour chaque type de fumier permettent de prédire 7 composés chimiques clés en quelques secondes.

L'ensemble du pipeline est exposé via une API Flask et une interface Streamlit pensée pour un usage terrain.

---

## Variables prédites

| Variable | Description |
|---|---|
| NH₄ | Ammonium |
| DM | Matière sèche |
| N | Azote total |
| P₂O₅ | Pentoxyde de phosphore |
| K₂O | Oxyde de potassium |
| CaO | Oxyde de calcium |
| MgO | Oxyde de magnésium |

---

## Méthodologie

### 1. Analyse exploratoire

**Analyse univariée**
- Distribution des variables quantitatives (histogrammes, boxplots, asymétrie, kurtosis)
- Analyse des variables qualitatives (type de fumier, origine)
- Détection de valeurs aberrantes par inspection visuelle et règle IQR

**Analyse bivariée**
- Corrélations entre variables chimiques (matrice de Pearson/Spearman)
- Comparaison des distributions par type de fumier (bovin vs volaille)
- Visualisation des relations spectre–propriété chimique

**Tests statistiques**
- Tests de normalité (Shapiro-Wilk, Kolmogorov-Smirnov)
- Tests de significativité pour comparer les groupes (Mann-Whitney, t-test selon normalité)
- Vérification des hypothèses avant modélisation

---
### 2. Prétraitement spectral

- Application du filtre **Savitzky-Golay** (fenêtre = 11, polynôme d'ordre 1 et 2, dérivée seconde) pour supprimer la ligne de base et accentuer les bandes d'absorption

---

### 3. Analyse en composante principale (ACP)

- **Analyse en Composantes Principales (ACP)** pour réduire la dimensionnalité, visualiser la structure des données spectrales et identifier les sources de variance principales
- **Clustering naturel** : identification de groupes homogènes dans l'espace spectral par méthodes hiérarchiques (dendrogramme)
- **K-Means** : clustering non supervisé pour confirmer et affiner la segmentation naturelle des échantillons
- **Détection et filtrage des outliers** : identification des échantillons aberrants dans l'espace PCA (distance de Mahalanobis, score de Hotelling T²) et exclusion justifiée
- **Sélection des variables spectrales** : filtrage des longueurs d'onde les plus informatives pour réduire le bruit et améliorer la parcimonie des modèles
- **Séparation du jeu de données** : split entraînement / validation en préservant la représentativité des groupes (stratification par type de fumier)

---

### 4. Sélection de la méthode de modélisation

Évaluation comparative de plusieurs approches chimiométriques sur la base de métriques de performance (R², RMSE, MAE en validation croisée) :

| Méthode | Caractéristiques évaluées |
|---|---|
| PLS (Partial Least Squares) | Gestion de la colinéarité, robustesse sur données spectrales |
| Random forest | Réduit le surraprentissage, idéal pour les variables non linéaires et les interactions complexes. |
| Ridge / Lasso | Régularisation sur données à haute dimensionnalité |

La **régression PLS** a été retenue pour sa capacité à gérer la colinéarité des variables spectrales et ses performances supérieures sur ce type de données.

---

### 5. Développement et optimisation des modèles PLS

- Un modèle PLS entraîné **par variable cible et par type de fumier** — 14 modèles au total (7 composés × 2 types)
- Optimisation du nombre de composantes latentes par validation croisée leave-one-out
- Évaluation finale sur le jeu de validation indépendant :
  - **R²** (coefficient de détermination)
  - **RMSE** (erreur quadratique moyenne)
  - **MAE** (erreur absolue moyenne)
- **Courbes de régression** : visualisation réel vs prédit par composé et par type de fumier
- Analyse des résidus et des loadings pour interpréter la contribution des longueurs d'onde
- Sérialisation des modèles avec `joblib` pour intégration dans l'API

---

## Architecture
```
.
├── api/
│   └── app.py                  # API Flask — prétraitement + prédiction PLS
├── app/
│   └── dashboard.py            # Interface Streamlit multi-pages
├── modele/
│   ├── models_bovin/           # 7 modèles PLS bovins (.pkl)
│   └── models_volaille/        # 7 modèles PLS volailles (.pkl)
├── data/                       # Spectres NIR + mesures chimiques (publiques)
├── notebooks/                  # Analyses exploratoires et modélisation
└── README.md
```

---

## API Flask

| Route | Méthode | Description |
|---|---|---|
| `/` | GET | Retourne la configuration (nb variables spectrales, composés prédits) |
| `/predict` | POST | Reçoit un fichier CSV/TXT, applique le prétraitement SG, retourne les 7 prédictions |

**Format du fichier attendu**
- Séparateur de colonnes : `;`
- Séparateur décimal : `,` (format français)
- 700 colonnes spectrales par ligne, sans en-tête
- Extensions acceptées : `.csv` ou `.txt`

---

## Interface Streamlit

Le dashboard est organisé en une page :

- **Prédiction** — chargement d'un fichier spectral, sélection du type de fumier, lancement des prédictions, tableau des résultats, visualisations (barres groupées, Intervalle de confiance), export CSV/TXT


---

## Utilisation

### 1. lancer le Dashboard

L'API démarre sur `https://dashboard-855393546916.europe-west9.run.app/`.

### 2. Choix du type de fumier
Bovin ou volaille


### 3. Importer un fichier spectre depuis le repo
Dans le dossier "spectre_test" se trouve les spectres en fonction du type dans un format csv avec en-tête et colonne index

### 4. Côcher la colonne index et l'en-tête de colonne


### 5. Lancer la prédiction

### 6. Exporter les prédictions en format csv ou txt



---

## Stack technique

| Domaine | Outils |
|---|---|
| Chimiométrie | PLS, RF, Lasso, Ridge Savitzky-Golay (`scikit-learn, scipy`) |
| Analyse exploratoire | `pandas`, `numpy`, `scipy.stats`, `seaborn` |
| Réduction de dimension | ACP (`scikit-learn`) |
| Clustering | K-Means (`scikit-learn`, `scipy`) |
| Modélisation | `scikit-learn`, `joblib` |
| Validation des entrées | `pydantic` |
| API | `Flask` |
| Interface | `Streamlit`, `Plotly` |

---

## Données

Les données utilisées sont publiques et comprennent des spectres NIR (1100–2498 nm, 700 longueurs d'onde) associés à des mesures chimiques de référence pour des fumiers bovins et volailles. Disponibles dans le dossier `/data`.

> NB: ceci est une copie du Repo originel qui contient que les informations utiles à l'utilisation de ce l'app Flask
---

## Auteur

**Ekanza Gilles**
Ouvert aux échanges autour de la chimiométrie, de l'analyse spectrale et de la data science appliquée.

