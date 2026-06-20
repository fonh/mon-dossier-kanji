# Instructions pour Claude Code

Mets à jour l'application 漢字の庭 existante (index.html) avec le système Pokédex.

## Fichiers à lire en premier (OBLIGATOIRE — dans cet ordre)

1. `SPEC.md` — architecture complète
2. `joyo2010.json` — les 2136 kanji Jōyō à intégrer

## Étape 1 — Transformer joyo2010.json en JOYO_DATA

Le fichier joyo2010.json a ce format :

```json
{
  "134047": {
    "joyo_kanji": "遅",
    "yomi": {
      "on_yomi": ["チ"],
      "kun_yomi": ["おそい", "おくれる"],
      "example_yomi": ["おそ-い"]
    },
    "raw_info": "遅\t\t5\t7S\t2010\tチ、おそ-い、おく-れる"
  }
}
```

Le champ `raw_info` contient le grade :

- `1S` à `6S` = grades 1 à 6 (primaire)
- `7S` = secondaire (grade 7 dans notre app)

Construis une variable JS `JOYO_DATA` au format :

```js
const JOYO_DATA = [
  {
    numero: 1,
    caractere: "亜",
    grade: 7,
    on: ["ア"],
    kun: [],
    sens_en: "Asia",
  },
  {
    numero: 2,
    caractere: "哀",
    grade: 7,
    on: ["アイ"],
    kun: ["あわれ"],
    sens_en: "pity",
  },
  // ... tous les 2136 kanji
];
```

Pour `numero` : index séquentiel 1 à 2136 dans l'ordre du fichier.
Pour `grade` : extraire depuis raw_info (1S→1, 2S→2, ..., 6S→6, 7S→7).
Pour `sens_en` : laisser vide `""` (pas disponible dans cette source).

## Étape 2 — Modifier index.html

### 2a. Ajouter JOYO_DATA en haut du JS

Coller la variable JOYO_DATA construite à l'étape 1.

### 2b. Initialisation au premier chargement

```js
function initDB() {
  if (!localStorage.getItem("kanjiNiwa_v1")) {
    localStorage.setItem("kanjiNiwa_v1", JSON.stringify(DATA_INITIALE));
  }
  if (!localStorage.getItem("kanjiNiwa_joyo")) {
    localStorage.setItem("kanjiNiwa_joyo", JSON.stringify(JOYO_DATA));
  }
}
```

### 2c. Moteur Pokédex (CRITIQUE)

```js
function getKanjiState(joyo_entry) {
  const db = getDB();
  const fiche = db.kanji.find((k) => k.caractere === joyo_entry.caractere);
  if (!fiche) return { state: "locked", fiche: null, joyo: joyo_entry };
  return {
    state: fiche.srs === "習得" ? "mastered" : "studying",
    fiche,
    joyo: joyo_entry,
  };
}
```

### 2d. Transformer la vue 漢字 en grille Pokédex

La grille affiche JOYO_DATA complet (2136 cases) dans l'ordre, pas seulement db.kanji.

**Carte débloquée** (state = "studying" ou "mastered") :

```
┌─────────────────┐  ← barre colorée 3px (couleur thème)
│ #47          🟡 │  ← numéro Jōyō + pastille SRS
│       遅        │  ← 44px, couleur thème
│  Lent · Retard  │  ← signification
│    おそい       │  ← kun principal
└─────────────────┘
```

- Fond blanc, border colorée selon thème
- Clic → fiche riche complète

**Carte verrouillée** (state = "locked") :

```
┌─────────────────┐
│ #48             │  ← numéro, couleur --hint
│       ▓▓        │  ← silhouette 44px, couleur --locked-text
│    Grade 1      │  ← grade scolaire
└─────────────────┘
```

- Fond --locked-bg `#F0EEE9`, border --border
- Clic → mini-fiche Jōyō

### 2e. Mini-fiche Jōyō (clic sur carte verrouillée)

```
← 戻る

        遅  #47
   Grade 1 · Jōyō
   On : チ
   Kun : おそい, おくれる

[✨ Générer une fiche complète]   ← grisé si pas de clé API

   Pas encore dans tes fiches
```

Clic "Générer" → appel API → crée la fiche riche → débloque la case → animation unlock.

### 2f. Animation de déblocage

```css
@keyframes unlock {
  0% {
    transform: scale(0.8);
    opacity: 0;
  }
  60% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}
```

Se joue sur la carte quand elle passe de locked → studying.

### 2g. Filtre grade dans la barre de filtres

Ajouter : Tous · Grade 1 · Grade 2 · Grade 3 · Grade 4 · Grade 5 · Grade 6 · Secondaire

### 2h. Badge Jōyō dans la fiche riche

En haut à droite de chaque fiche riche, ajouter :

```js
const joyoEntry = JOYO_DATA.find((j) => j.caractere === kanji.caractere);
// Si trouvé → afficher badge discret "#47 · Grade 1"
```

### 2i. Lookup automatique dans le modal d'ajout

Quand l'utilisateur tape un kanji dans le champ :

- Chercher dans JOYO_DATA
- Si trouvé → afficher "#47 · Grade 1" sous le champ, pré-remplir on/kun

### 2j. Statistiques Jōyō dans ⚙️

Ajouter en haut de la vue paramètres :

```
漢字 débloqués : 20 / 2136  (0.9%)

Par grade :
Grade 1  [████░░░░░░░░]  X / 80
Grade 2  [██░░░░░░░░░░]  X / 160
Grade 3  [░░░░░░░░░░░░]  X / 200
Grade 4  [░░░░░░░░░░░░]  X / 202
Grade 5  [░░░░░░░░░░░░]  X / 193
Grade 6  [░░░░░░░░░░░░]  X / 191
Secondaire [░░░░░░░░░░]  X / 1110
```

## Ce qu'il ne faut PAS toucher

- DATA_INITIALE (les 20 kanji) → ne pas modifier
- La fiche riche kanji → ne pas modifier la structure
- La vue 部首 → ne pas modifier
- La vue 語彙 → ne pas modifier
- La vue 復習 → ne pas modifier
- Le système export/import → ne pas modifier
- Le moteur de liens dynamiques → ne pas modifier

## Résultat attendu

Ouvrir index.html → grille de 2136 cases.
Les 20 cases des kanji étudiés sont colorées et cliquables.
Les 2116 autres sont des silhouettes grises numérotées.
Chaque nouveau kanji ajouté débloque sa case avec animation.
