# Guide d'utilisation — Notebooks ML Chapitre 4

**Répertoire de travail** : `/home/digitheque/Documents/telechargement/`  
**Dossier de données** : `data/` (relatif au répertoire de travail)  
**Environnement Python** : `venv/` (activer avant tout run)

---

## Vue d'ensemble — Trois notebooks, deux paradigmes

| # | Notebook | Rôle | AUC attendu |
|---|---|---|---|
| 1 | `01_pipeline_etl.ipynb` | Collecte NVD + CISA KEV → CSV 14 colonnes | — (ETL) |
| 2 | `pipeline_etl_additions.ipynb` | CVSS + EPSS (injection temps réel) → **Paradigme 2** | 0.9936 |
| 3 | `02_modelisation_ml_benchmark.ipynb` | Benchmark approfondi CVSS seul → **Paradigme 1** | 0.7038 |

**Ordre d'exécution recommandé** : Notebook 1 → Notebook 2 → Notebook 3  
(Notebook 2 et 3 peuvent être exécutés indépendamment si le CSV existe déjà)

---

## Prérequis

```bash
# Activer l'environnement (à faire avant de lancer VSCode ou Jupyter)
source /home/digitheque/Documents/telechargement/venv/bin/activate

# Vérifier que le CSV de base existe
python3 -c "import pandas as pd; df=pd.read_csv('data/api_vulnerabilities_processed.csv'); print(df.shape)"
# Attendu : (10066, 14)
```

**Connexion Internet requise** pour les Notebooks 1 et 2 (APIs NVD et EPSS).

---

## Notebook 1 — `01_pipeline_etl.ipynb` (ETL)

**Objectif** : collecter les CVEs de NVD filtrées sur les 7 CWE Auth/Authz, identifier les exploitées via CISA KEV, produire le CSV de base.

### Cellules à exécuter (dans l'ordre)

| Cellule | Contenu | Durée estimée |
|---|---|---|
| 1 | Imports + configuration (DATA_DIR, clé NVD) | < 1 s |
| 2 | Collecte NVD via API (checkpoint JSON) | 5–20 min (API limitée) |
| 3 | Filtrage sur 7 CWE Auth/Authz | < 5 s |
| 4 | Fusion CISA KEV (is_exploited) | < 5 s |
| 5 | Nettoyage + encodage CVSS | < 5 s |
| 6 | Calcul cve_age_days (date de référence : 2026-06-15) | < 5 s |
| 7 | Sauvegarde finale → `data/api_vulnerabilities_processed.csv` | < 5 s |

### Résultat attendu

```
data/api_vulnerabilities_processed.csv     → (10,066 × 14 colonnes)
data/nvd_api_raw_vulnerabilities.json      → checkpoint réutilisable (évite re-collecte)
```

**14 colonnes** : `cve_id`, `cwe_id`, `published_date`, `base_score`, `attack_vector`, `attack_complexity`, `privileges_required`, `user_interaction`, `scope`, `confidentiality_impact`, `integrity_impact`, `availability_impact`, `is_exploited`, `cve_age_days`

**Note clé** : si `nvd_api_raw_vulnerabilities.json` existe déjà, la cellule 2 charge le checkpoint sans réappeler l'API. La ré-exécution prend alors < 30 s.

**Note clé 2** : les scores EPSS ne sont PAS dans ce CSV. Ils sont injectés en temps réel par le Notebook 2.

### Vérification avant de passer au Notebook 2

```python
import pandas as pd
df = pd.read_csv('data/api_vulnerabilities_processed.csv')
assert df.shape == (10066, 14), f"CSV dégradé : {df.shape}"
assert df['is_exploited'].sum() == 84, "Nombre d'exploitées inattendu"
print("CSV OK")
```

---

## Notebook 2 — `pipeline_etl_additions.ipynb` (Paradigme 2 — PRINCIPAL)

**Objectif** : entraîner LR et RF sur CVSS + EPSS injecté en temps réel. Produire les résultats **Paradigme 2** avec AUC ≈ 0.99.

**Ce notebook télécharge automatiquement le snapshot EPSS journalier** (Cyentia Institute / FIRST) et fusionne les scores avec le CSV au moment de l'exécution. Si l'API EPSS est indisponible, le notebook bascule en Paradigme 1 et l'affiche clairement.

### Cellules à exécuter (dans l'ordre)

| N° | Identifiant | Phase | Ce qui se passe |
|---|---|---|---|
| 1 | — | Imports | pandas, sklearn, requests, gzip, io |
| 2 | `add0018ac2` | **Chargement + injection EPSS** | Charge CSV, télécharge EPSS, fusionne, construit X (11 features), split train/test |
| 3 | — | Markdown Phase 5 | Titre de section |
| 4 | `add003293f` | **Phase 5 — Régression logistique** | LR baseline + LR balanced, AUC, seuil Youden, matrice de confusion |
| 5 | — | Markdown Phase 6 | Titre de section |
| 6 | `add005591b` | **Phase 6 — Random Forest** | RF balanced_subsample, AUC, importances variables |
| 7 | — | Markdown Phase 7 | Titre de section |
| 8 | `add007ed27` | **Phase 7 — Benchmark vs EPSS seul** | Télécharge EPSS sur test set, calcule AUC EPSS standalone |
| 9 | — | Markdown figures | Titre de section |
| 10 | `add00956ca` | **Figures ROC** | Sauvegarde `data/comparaison_complete_modeles.pdf` |
| 11 | — | Markdown récap | Titre de section |
| 12 | `add0115779` | **Tableau récapitulatif final** | Tous les chiffres à reporter dans le mémoire |

### Résultats confirmés (run du 2026-06-15, N=10,066, seed=42)

```
DATASET
  N total           : 10,066
  N train           : 7,549
  N test            : 2,517
  Y=1 (exploitées)  : 84 total / 21 dans test (0.83%)
  Nb features X     : 11 (8 OHE CVSS + cve_age_days + epss_score + percentile)

PARADIGME 2 — CVSS + EPSS
  LR sans balance   : AUC = 0.9936  ← meilleur modèle
  LR balanced       : AUC = 0.9891
  Random Forest     : AUC = 0.9908

SEUILS DE YOUDEN
  LR balanced  τ*   : 0.3418  (J = 0.9243)
  Random Forest τ*  : 0.1657  (J = 0.9531)

MATRICE DE CONFUSION — Random Forest (AUC = 0.9908, τ* = 0.1657)
  TN = 2379  FP = 117
  FN =    0  TP =  21
  Rappel   = 100.00%   (zéro CVE exploitée manquée)
  Précision =  15.22%

MATRICE DE CONFUSION — LR balanced (AUC = 0.9891, τ* = 0.3418)
  TN = 2307  FP = 189
  FN =    0  TP =  21
  Rappel   = 100.00%
  Précision =  10.00%

RÉFÉRENCE — EPSS SEUL (Jacobs et al., 2021)
  AUC EPSS standalone : 0.9910
  Δ RF vs EPSS        : −0.0002  (RF légèrement en dessous)

SIGNAL EPSS (avant modélisation)
  EPSS moyen exploitées (Y=1) : 0.6937
  EPSS moyen corpus           : 0.0305
  Ratio                       : 23×

IMPORTANCE DES VARIABLES — Random Forest
  epss_score             : 43.69%
  percentile             : 43.56%
  cve_age_days           : 7.54%
  privileges_required_NONE : 1.85%
  privileges_required_LOW  : 1.51%
  (autres CVSS < 1% chacun)
```

### Fichiers générés

```
data/importance_variables_rf.pdf        → graphique importances (RF)
data/comparaison_complete_modeles.pdf   → courbes ROC toutes variantes
```

---

## Notebook 3 — `02_modelisation_ml_benchmark.ipynb` (Paradigme 1 — Benchmark approfondi)

**Objectif** : benchmark exhaustif CVSS seul — 4 familles de modèles, SMOTE, LightGBM, optimisation bayésienne Optuna. Quantifier le plafond informationnel du CVSS.

**Ce notebook utilise uniquement le CSV (14 colonnes)**, sans injection EPSS. Il produit les résultats **Paradigme 1** de référence.

### Phases et résultats

| Phase | Modèle / Stratégie | AUC |
|---|---|---|
| Phase 5 | LR sans correction (baseline) | 0.6738 |
| Phase 5 | LR `class_weight='balanced'` | 0.6802 |
| Phase 6 | **Random Forest `balanced_subsample`** | **0.7038** ← meilleur |
| Phase 6b | LR + SMOTE | 0.6721 |
| Phase 6c | LightGBM `scale_pos_weight` | ≈ 0.67 (run précédent) |
| Phase 6d | LR Optuna bayésien (50 essais) | ≈ 0.698 (run précédent) |

**Constat** : aucun modèle ne franchit AUC = 0.71. Le plafond est informationnel, pas algorithmique.

### Journalisation

Chaque run ajoute une ligne à `data/experiments_log.csv` avec : timestamp, modèle, AUC, seuil Youden, paramètres. Permet de comparer les itérations sans recopie manuelle.

### Note sur les phases LightGBM et Optuna

Les Phases 6c et 6d font appel à `lightgbm` et `optuna` (packages optionnels). Si non installés :
```bash
pip install lightgbm optuna
```

---

## Résumé des Δ AUC pour le mémoire

| Paradigme | Meilleur AUC | Δ vs P1 |
|---|---|---|
| P1 — CVSS seul | 0.7038 (RF) | — |
| P2 — CVSS + EPSS | 0.9936 (LR) | **+0.2898** |
| EPSS seul (référence état de l'art) | 0.9910 | — |
| P2 vs EPSS seul | — | −0.0002 (RF) / +0.0026 (LR) |

**Gain total EPSS : +0.2898 AUC** (0.7038 → 0.9936)  
→ Quantifie la valeur informationnelle de la Threat Intelligence exogène absente du CVSS.

---

## Dépannage

**CSV dégradé (shape ≠ 10066×14)** : ré-exécuter le Notebook 1 en entier.

**EPSS indisponible** (Notebook 2 affiche `[!] EPSS indisponible`) : vérifier la connexion Internet. Le notebook continue en mode Paradigme 1 automatiquement.

**NameError en Phase 6c (LightGBM)** : la Phase 6c a son propre scaler (`scaler_lgb`). Si l'erreur persiste, vérifier que la cellule de configuration a bien été exécutée.

**Kernel mort ou résultats incohérents** : redémarrer le kernel Python et ré-exécuter toutes les cellules depuis le début.
