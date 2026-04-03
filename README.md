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
├── modèle/
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

**Exemple de réponse**
```json
{
  "statut": "succès",
  "type_fumier": "bovin",
  "predictions": {
    "NH4": 1.243,
    "DM": 28.17,
    "N": 3.85,
    "P2O5": 2.14,
    "K2O": 3.61,
    "CaO": 4.02,
    "MgO": 0.98
  }
}
```

---

## Interface Streamlit

Le dashboard est organisé en trois pages :

- **Prédiction** — chargement d'un fichier spectral, sélection du type de fumier, lancement des prédictions, tableau des résultats, visualisations (barres groupées, radar par échantillon), export CSV/TXT
- **Spectres** — exploration des spectres de référence : spectre moyen avec enveloppe ±1σ, spectres individuels superposés, carte de chaleur des absorbances
- **Données brutes** — statistiques descriptives et aperçu des fichiers de référence chargés

---

## Lancement

### 1. Installer les dépendances
```bash
pip install -r requirements.txt
```

### 2. Lancer l'API
```bash
python api/app.py
```

L'API démarre sur `http://127.0.0.1:8000`.

### 3. Lancer le dashboard
```bash
streamlit run app/dashboard.py
```

> L'API doit être active avant de lancer le dashboard — l'indicateur de connexion en bas de la barre latérale confirme l'état.

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

---

## Auteur

**Ekanza Gilles**
Ouvert aux échanges autour de la chimiométrie, de l'analyse spectrale et de la data science appliquée.
