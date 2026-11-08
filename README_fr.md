# 🫁 ChestX AI — `web_ui`

L'application Streamlit du projet : un espace de travail de type clinique qui exécute le **ResNet-50** entraîné sur une radiographie thoracique, affiche la prédiction avec son explication **Grad-CAM**, et aide à rédiger un compte-rendu radiologique préliminaire.

> ⚠️ **Aide à la décision uniquement.** Les résultats doivent être relus et signés par un radiologue agréé. Il s'agit d'un projet académique, pas d'un dispositif médical.

🇬🇧 English version: [`README.md`](README.md)

---

## Sommaire

- [Fonctionnalités](#fonctionnalités)
- [Démo](#démo)
- [Structure du projet](#structure-du-projet)
- [Prérequis](#prérequis)
- [Installation (une seule fois)](#installation-une-seule-fois)
- [Lancement](#lancement)
- [Configuration](#configuration)
- [Utilisation de l'application](#utilisation-de-lapplication)
- [Fonctionnement](#fonctionnement)
- [Dépannage](#dépannage)
- [Notes pour les développeurs](#notes-pour-les-développeurs)
- [Personnalisation et évolutions](#personnalisation-et-évolutions)

---

## Fonctionnalités

- **Prédiction en quatre classes** : COVID-19, Normal, Pneumonie, Tuberculose, avec un score de confiance
- **Probabilités différentielles** : les quatre classes sont affichées, triées, pour rendre l'incertitude visible
- **Carte de chaleur Grad-CAM** superposée à la radiographie, côte à côte avec l'image originale
- **Bandeau de findings** : indicateurs lisibles (condensation, verre dépoli, cavitation/TB, champs pulmonaires clairs) dérivés des probabilités de classe
- **Niveaux de gravité** (`ok` / `warn` / `alert`) qui pilotent les couleurs affichées
- **Carte du modèle** intégrée au flux de travail : architecture, taille d'entrée, normalisation, couche Grad-CAM, appareil (GPU/CPU), temps d'inférence
- **Formulaire patient et examen** : identifiant, numéro d'accession, nom, âge, sexe, modalité/incidence, priorité, historique clinique, facteurs de risque
- **Actions avec l'humain dans la boucle** : *Générer le compte-rendu*, *Confirmer*, *Signaler & orienter* — rien n'est publié automatiquement
- **Repli automatique sur le CPU** si aucun GPU n'est disponible

## Démo

Une vidéo de démonstration est disponible dans le dossier [`video_demo/`](../video_demo/) à la racine du projet.
Des radiographies d'exemple pour essayer l'application sont dans [`Image_test/`](../Image_test/).



## Structure du projet

L'application attend cette arborescence (le checkpoint est recherché en remontant d'un dossier depuis `web_ui/` puis en regardant dans `model/`) :

```
<racine-du-projet>/
├── Image_test/                     ← radiographies d'exemple pour essayer l'application
├── video_demo/                     ← vidéo de démonstration
├── model/
│   └── best_model_final.pth        ← checkpoint entraîné
├── web_guide/                      ← design de référence (inchangé)
└── web_ui/                         ← CE dossier
    ├── app.py                      Application Streamlit (interface)
    ├── inference.py                modèle + transformations + Grad-CAM
    ├── requirements.txt
    └── README.md
```

Pour utiliser un autre emplacement de checkpoint, définissez la variable d'environnement `CHESTX_WEIGHTS` (voir [Configuration](#configuration)).

## Prérequis

- Python **3.10+**
- Le checkpoint entraîné `best_model_final.pth`
- Paquets (voir `requirements.txt`) : `streamlit`, `torch`, `torchvision`, `Pillow`, `opencv-python`, `numpy`
- Un GPU est optionnel : l'application bascule automatiquement sur le CPU

## Installation (une seule fois)

Ouvrez un terminal dans `<racine-du-projet>` (le dossier qui contient `model/` et `web_ui/`).

**Windows (PowerShell)**

```powershell
# 1. Créer un environnement virtuel (recommandé)
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2. Installer les dépendances
pip install -r web_ui\requirements.txt
```

**Linux / macOS**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r web_ui/requirements.txt
```

> Si `pip install torch` est lent, ou si vous voulez une version CUDA précise, utilisez le sélecteur de <https://pytorch.org/get-started/locally/>.
> Exemple pour CUDA 12.1 :
> `pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121`

> Si PowerShell bloque le script d'activation, exécutez une fois :
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

## Lancement

**Windows (PowerShell)**

```powershell
streamlit run web_ui\app.py
```

**Linux / macOS**

```bash
streamlit run web_ui/app.py
```

Le navigateur s'ouvre sur <http://localhost:8501>. La première inférence prend quelques secondes (chargement du modèle + préchauffage de Grad-CAM) ; les suivantes sont rapides.

Pour utiliser un autre port :

```bash
streamlit run web_ui/app.py --server.port 8502
```

## Configuration

| Variable | Rôle | Valeur par défaut |
|---|---|---|
| `CHESTX_WEIGHTS` | Chemin complet vers le checkpoint `.pth` | `<racine-du-projet>/model/best_model_final.pth` |

**Windows (PowerShell)**

```powershell
$env:CHESTX_WEIGHTS = "C:\chemin\complet\best_model_final.pth"
streamlit run web_ui\app.py
```

**Linux / macOS**

```bash
export CHESTX_WEIGHTS=/chemin/complet/best_model_final.pth
streamlit run web_ui/app.py
```

## Utilisation de l'application

1. **Patient & examen** (panneau de gauche) : saisissez l'identifiant patient, le numéro d'accession, le nom, l'âge, le sexe, la modalité/incidence et la priorité.
2. **Historique clinique** : ajoutez l'indication (symptômes, durée, contexte) et sélectionnez les facteurs de risque éventuels (ex. fumeur, voyage récent).
3. **Image** : chargez une radiographie thoracique (PNG, JPG ou DICOM).
4. Cliquez sur **Run analysis**.
5. Lisez les résultats :
   - **Centre** : radiographie originale et superposition Grad-CAM, avec le bandeau de findings au-dessus
   - **Panneau de droite (analyse IA)** : prédiction principale, confiance, code ICD-10, probabilités différentielles et carte du modèle
6. Utilisez les boutons d'action :
   - **Générer le compte-rendu (PDF)** : brouillon à relire et signer par un radiologue
   - **Confirmer** : valider la suggestion de l'IA
   - **Signaler & orienter** : envoyer le cas en revue

Un bandeau permanent rappelle que l'outil est uniquement consultatif.

## Fonctionnement

```
Image chargée
   │
   ▼
Prétraitement ── RGB → redimensionnement 224×224 → normalisation ImageNet (moyenne/écart-type)
   │
   ▼
ResNet-50 (backbone ImageNet + tête personnalisée)
   │      Linear(2048→512) → BatchNorm → ReLU → Dropout(0,5) → Linear(512→4)
   ▼
Softmax ── probabilités sur [COVID, Normal, Pneumonia, Tuberculosis]
   │
   ├──► Grad-CAM sur layer4[-1].conv3 ── carte redimensionnée en 224×224 et superposée
   │
   ▼
Objet Prediction ── classe principale, confiance, code ICD-10, gravité,
                    probabilités différentielles, findings, carte de chaleur
   │
   ▼
Interface Streamlit ── carte de diagnostic, probabilités, bandeau de findings, carte du modèle
```

1. **Charge** `model/best_model_final.pth` dans un ResNet-50 avec la tête personnalisée du notebook d'entraînement.
2. **Prédit** les probabilités softmax sur `[COVID, Normal, Pneumonia, Tuberculosis]` — l'ordre exact produit par `ImageFolder` (alphabétique), celui sur lequel le checkpoint a été entraîné.
3. **Génère une carte Grad-CAM** à partir de `layer4[-1].conv3` (la même couche cible que dans le notebook) et la superpose à la radiographie originale.
4. **Affiche** le résultat dans l'espace de travail clinique : carte de diagnostic, probabilités différentielles, bandeau de findings et carte du modèle.

### Performances de référence (jeu de test, 1057 images)

| Exactitude | F1 macro | ROC-AUC macro |
|---|---|---|
| 98,01 % | 0,978 | 0,9992 |

Ces chiffres proviennent du jeu de test du projet ; ils ne garantissent pas les mêmes performances sur des images issues d'autres hôpitaux ou d'autres appareils.

## Dépannage

**Bandeau « Model weights not loaded » en haut de la page**
- L'application n'a pas trouvé ou n'a pas pu charger `model/best_model_final.pth`. Vérifiez le chemin affiché dans le bandeau. Si votre `.pth` est ailleurs, définissez `CHESTX_WEIGHTS` avant `streamlit run` (voir [Configuration](#configuration)).

**`Missing key(s) in state_dict` / `Unexpected key(s)`**
- L'architecture de `inference.py:build_model()` ne correspond pas à celle de l'entraînement. La version actuelle correspond exactement au notebook. Si vous modifiez la tête plus tard, changez-la aux deux endroits (notebook et `inference.py`).

**`Error(s) in loading state_dict ... size mismatch for fc.4`**
- Le nombre de classes a changé. `inference.py` définit `NUM_CLASSES = 4` pour correspondre à `["COVID", "Normal", "Pneumonia", "Tuberculosis"]`. Mettez à jour les deux si vous réentraînez avec d'autres classes.

**CUDA out of memory / pas de GPU**
- L'application bascule automatiquement sur le CPU. L'inférence prend ~1 à 3 s par image sur CPU contre ~50 ms sur GPU. Aucune modification de code n'est nécessaire.

**Les prédictions semblent fausses / inversées**
- L'ordre des classes est probablement incorrect. Vérifiez avec une image de chaque classe. Le notebook a utilisé `datasets.ImageFolder`, qui trie par ordre alphabétique → `['COVID', 'Normal', 'Pneumonia', 'Tuberculosis']`. Si vous avez entraîné avec un autre ordre, modifiez `CLASSES` dans `inference.py`.

**`streamlit` n'est pas reconnu comme commande**
- L'environnement virtuel n'est pas activé, ou les dépendances ne sont pas installées. Activez le venv et relancez `pip install -r web_ui/requirements.txt`. Sinon : `python -m streamlit run web_ui/app.py`.

**Port 8501 déjà utilisé**
- Une autre instance de Streamlit tourne. Fermez-la ou lancez l'application sur un autre port avec `--server.port 8502`.

**Un fichier DICOM ne se charge pas**
- Exportez d'abord l'image en PNG/JPG. Si vous avez besoin du support DICOM natif, installez `pydicom` et ajoutez-le à `requirements.txt`.

**Première analyse lente**
- C'est normal : le modèle est chargé et Grad-CAM est préchauffé lors de la première exécution. Les analyses suivantes les réutilisent.

## Notes pour les développeurs

- **`inference.py`** est le seul fichier qui touche à PyTorch. L'interface n'importe jamais `torch` directement, sauf pour l'étiquette GPU/CPU de la carte du modèle.
- **La dataclass `Prediction`** est le contrat entre l'inférence et l'interface. Ajoutez-y des champs pour afficher de nouvelles informations dans l'espace de travail.
- **Le prétraitement doit correspondre à l'entraînement** : même redimensionnement 224×224, conversion RGB et normalisation ImageNet. Toute modification des transformations d'entraînement doit être reportée dans `inference.py`.
- **La génération du compte-rendu PDF** est actuellement un placeholder. Ajoutez `reportlab` à `requirements.txt` et générez les vrais octets dans l'appel à `st.download_button`.

## Personnalisation et évolutions

| Je veux… | À modifier |
|---|---|
| Utiliser un autre checkpoint | Définir `CHESTX_WEIGHTS`, ou remplacer `model/best_model_final.pth` |
| Changer les classes | `CLASSES` et `NUM_CLASSES` dans `inference.py`, puis réentraîner en gardant le même ordre |
| Changer la tête de classification | `build_model()` dans `inference.py` **et** le notebook d'entraînement |
| Afficher une nouvelle information dans l'interface | Ajouter un champ à la dataclass `Prediction`, le remplir dans `inference.py`, l'afficher dans `app.py` |
| Générer un vrai compte-rendu PDF | Ajouter `reportlab` et construire le PDF dans `app.py` (`st.download_button`) |
| Déployer sur le Cloud | Héberger l'application (ex. Streamlit Community Cloud, Docker) et fournir le checkpoint via `CHESTX_WEIGHTS` ou une étape de téléchargement |

## Avertissement

ChestX AI est un projet académique. Ses sorties sont des suggestions préliminaires à relire par un radiologue qualifié et ne doivent jamais remplacer le jugement clinique.
