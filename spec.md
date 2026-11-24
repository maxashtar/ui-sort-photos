# Spécification — Photo Sorter

Application de bureau (Python / Tkinter) pour trier des photos : marquer chaque photo comme « à garder » (avec un tag) ou « à supprimer », puis déplacer automatiquement les fichiers vers des dossiers de destination.

---

## 1. Vue d'ensemble

| Élément | Détail |
|---|---|
| Type | Application GUI de bureau, fenêtre unique |
| Langage / UI | Python 3, Tkinter (`tk`, `ttk`, `simpledialog`, `messagebox`, `filedialog`) |
| Images | Pillow (`PIL.Image`, `PIL.ImageTk`) |
| Concurrence | `threading` + `concurrent.futures.ThreadPoolExecutor` |
| Fichiers | `os`, `shutil`, `json` |

### Configuration

| Constante | Valeur | Rôle |
|---|---|---|
| `TAG_ROOT` | `/media/Disk_2/PHOTOS_TEMP/photos_Max_Temp/__TRIAGE_FAIT` | Racine des dossiers de destination. Un sous-dossier par tag ; sous-dossier spécial `to_delete` pour les photos marquées « à supprimer ». |
| `CONFIG_FILE` | `.photo_sorter_config.json` à côté du script | Sauvegarde du dernier tag utilisé (`{"last_tag": "..."}`). |
| `ICON_DIR` | dossier `other/` à côté du script | Contient `keep_icon.png` et `delete_icon.png`. |
| `ICON_SIZE` | 40 px | Taille des icônes incrustées sur les vignettes. |

---

## 2. Interface utilisateur

### 2.1 Fenêtre principale

- Titre : `Photo Sorter`.
- Maximisée au démarrage (attribut `-zoomed`, silencieusement ignoré si non supporté).
- Deux zones : panneau principal (gauche, extensible) et panneau latéral (droite, largeur 250 px).

### 2.2 Panneau principal (gauche)

1. **Bande de vignettes** (hauteur 200 px) : moteur tkinter `Canvas` avec scroll horizontal. Une vignette carrée de 200 px par photo, disposées en ligne. La vignette de la photo courante est encadrée en bleu (épaisseur 3), les autres en gris fin.
   - Clic sur une vignette → affiche la photo correspondante comme image courante.
   - Vignettes avec incrustation : si un flag est posé, la vignette est remplacée par une version avec icône `keep` (👍) ou `delete` (🗑️) collée en haut à droite.
   - Si un fichier n'a pas pu produire de vignette, il n'apparaît pas dans la bande.

2. **Barre de boutons** (centrée, une rangée) :

   | Bouton | Action |
   |---|---|
   | 📂 Ouvrir Dossier | Ouvre un sélecteur de dossier et charge les photos |
   | ⏮️ Précédent | Photo précédente (boucle circulaire) |
   | ▶️ Suivant | Photo suivante (boucle circulaire) |
   | 👍 Référence | Affiche la photo courante dans le panneau « Référence » |
   | ✅ Garder | Marque la photo courante « keep » avec le tag sélectionné, puis avance |
   | 🗑️ Supprimer | Marque la photo courante « delete », puis avance |
   | 🚀 Finaliser | (fond orange) Déplace les fichiers selon les flags |

3. **Zone de comparaison en deux colonnes** :

   | | Colonne gauche | Colonne droite |
   |---|---|---|
   | Titre | `Référence` | `Courante` |
   | Image | `ref_canvas` (photo de référence) | `current_canvas` (photo courante) |
   | Libellé sous l'image | État du flag sur la **référence** : vide, `✅ Tag : <tag>` (vert) ou `🗑️ À supprimer` (rouge) | État du flag sur la **courante** : vide, `✅ À GARDER → <tag>` (vert), `🗑️ À SUPPRIMER` (rouge) — en gras, taille 16 |

   - La taille d'affichage des grandes images est calculée au démarrage : environ 45 % de la largeur d'écran × 60 % de la hauteur (les images sont réduites via `thumbnail`, ratio conservé).

### 2.3 Panneau latéral (droite)

- **Statistiques** (mises à jour après chaque flag) :
  - `À garder: <n>`
  - `À supprimer: <n>`
  - `À traiter: <n>` (total − marquées)
  - `Progression: <x.x>%` ((marquées / total) × 100, en gras)
- **Panneau Tags** :
  - En-tête « Tags » avec bouton `+ Nouveau tag` à droite.
  - Liste clicables des tags (un par ligne) ; le tag actif apparaît en rouge, gras, taille 13, avec bordure `solid`.
  - Bouton `+ Nouveau tag` : boîte de dialogue `simpledialog.askstring`. Le nom est nettoyé des espaces extrêmes ; un dossier `<TAG_ROOT>/<nom>` est créé (le créée aussi s'il existe déjà), la liste est rechargée et le nouveau tag est sélectionné. Erreur → `messagebox.showerror`.
  - Liste vide → message « Aucun tag. Créez-en un. ».

---

## 3. Comportements fonctionnels

### 3.1 Chargement d'un dossier (`load_images`)

1. Sélecteur de dossier natif ; annulation = aucun changement.
2. Liste les fichiers `.jpg`, `.jpeg`, `.png` (insensible à la casse), triés par nom. Les flags précédents sont réinitialisés, l'index courant repart à 0.
3. Une fenêtre de progression modale (Toplevel, titre `Chargement...`) s'affiche avec une `Progressbar` et un compteur `Chargement X / N`.
4. Les vignettes sont générées **en arrière-plan** via un `ThreadPoolExecutor` limité à `min(8, cpu_count())` threads :
   - ouverture en RGBA, réduction en vignette carrée 200×200 (fond transparent, image centrée) ;
   - les images PIL sont stockées temporairement, la progression est mise à jour sur le thread principal via `root.after`.
5. Conversion en `ImageTk.PhotoImage` côté thread principal (`finalize_thumbs`) :
   - génération des deux variantes d'incrustation (icône keep / delete) par vignette ;
   - fermeture de la fenêtre de progression ;
   - affichage des vignettes, de la photo courante et des statistiques.

### 3.2 Navigation

- Suivant / Précédent : navigation circulaire modulo le nombre de photos.
- Clic sur une vignette : saut direct (met à jour `current_index`, l'image courante, le libellé de flag et les cadres de vignettes).

### 3.3 Photo de référence (`set_reference`)

- La photo courante devient la référence (chemin complet mémorisé, grande image affichée à gauche).
- Le libellé sous la référence reflète son éventuel flag existant.

### 3.4 Marquage (`flag_image`)

- **Garder** : exige un tag sélectionné ; sinon `messagebox` d'avertissement « Sélectionnez un tag avant de marquer comme 'Garder'. » et aucun marquage. Le flag enregistré est `('keep', tag)`.
- **Supprimer** : flag `('delete', None)`.
- Après marquage : mise à jour du libellé d'état, remplacement de la vignette par la version incrustée (icône 40 px en haut à droite), mise à jour des statistiques, puis passage automatique à la photo suivante.
- Remarque : flaguer plusieurs fois la même photo écrase le flag précédent.

### 3.5 Finalisation (`finalize_sort`)

Conditions : un dossier chargé et au moins un flag. Sinon aucun effet.

1. Fenêtre de progression `Finalisation...` + barre de progression (mode déterminate), mise à jour sur le thread principal.
2. Pour chaque fichier flagué (worker en arrière-plan) :
   - `keep` → déplacement vers `<TAG_ROOT>/<tag>/` ;
   - `delete` → déplacement vers `<TAG_ROOT>/to_delete/` ;
   - dossier de destination créé si besoin (`os.makedirs(..., exist_ok=True)`) ;
   - fichier source introuvable → simplement ignoré, la progression avance quand même ;
   - échec de déplacement → message en console, le traitement continue.
3. À la fin (thread principal) : fermeture de la fenêtre, réinitialisation des flags, rechargement des tags et du panneau, **rechargement du dossier courant** pour rafraîchir la liste.

---

## 4. Modèle de données

### État de l'application

| Attribut | Type | Rôle |
|---|---|---|
| `photos` | `list[str]` | Noms de fichiers du dossier triés (ordre alphabétique) |
| `current_index` | `int` | Index de la photo courante |
| `ref_image_path` | `str \| None` | Chemin de la photo de référence |
| `folder` | `str \| None` | Dossier en cours de tri |
| `flags` | `dict[str, tuple]` | nom de fichier → `('keep', tag)` ou `('delete', None)` |
| `tags` | `list[str]` | Liste triée des tags (sous-dossiers de `TAG_ROOT`) |
| `tag_var` | `tk.StringVar` | Tag actif (persisté dans le fichier de config) |
| `thumbs` | `dict[str, Image.Image]` puis `dict[str, ImageTk.PhotoImage]` | Vignettes par nom de fichier |
| `thumbs_with_flag` | `dict[str, dict]` | nom de fichier → `{'keep': PhotoImage, 'delete': PhotoImage}` |
| `thumb_labels` | `dict[str, tk.Label]` | Libellés de vignettes (pour maj du cadre / incrustation) |

### Fichier de configuration

`.photo_sorter_config.json`

```json
{"last_tag": "nom_du_tag"}
```

- Lu au démarrage (erreurs ignorées silencieusement).
- Écrit à chaque sélection de tag.

---

## 5. Points d'attention / limites connues

- **Chemins codés en dur** : `TAG_ROOT` est une constante ; aucune option pour le changer dans l'UI.
- **Formats limités** : JPG/JPEG/PNG uniquement (pas de RAW, HEIC, GIF…).
- **Pas de raccourcis clavier** : tout se fait à la souris.
- **Pas d'annulation** : « Finaliser » déplace réellement les fichiers ; aucune fonctionnalité undo.
- **Doublons de noms** : un fichier déplacé vers un dossier qui contient déjà un fichier du même nom peut écraser le fichier existant (comportement de `shutil.move`).
- **Rechargement après finalisation** : rouvre le même dossier ; les fichiers déplacés disparaissent de la liste, les non-traités restent.
- **Fichiers en échec de vignette** : invisibles dans la bande, mais toujours comptés dans les statistiques et navigables via Suivant/Précédent.
- **Icônes optionnelles** : en cas d'échec de chargement de `other/keep_icon.png` ou `other/delete_icon.png`, les vignettes ne sont pas incrustées (les flags restent fonctionnels).

---

## 6. Dépendances et lancement

```bash
pip install pillow
python photo_sorter.py
```

Arborescence attendue à côté du script :

```
.
├── photo_sorter.py
├── .photo_sorter_config.json   (créé automatiquement)
└── other/
    ├── keep_icon.png
    └── delete_icon.png
```

Destination des fichiers :

```
/media/Disk_2/PHOTOS_TEMP/photos_Max_Temp/__TRIAGE_FAIT/
├── <tag1>/
├── <tag2>/
└── to_delete/
```
