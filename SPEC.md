# 漢字の庭 — Spécification technique complète

## Vue d'ensemble

Application web locale, fichier unique `index.html`, zéro dépendance sauf D3.js via CDN.
Ouvre directement dans Chrome avec double-clic. Fonctionne 100% offline sauf génération IA.

---

## Stack

- HTML5 + CSS3 + JavaScript ES6 vanilla
- D3.js v7 via `https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js`
- Stockage : `localStorage` clé `"kanjiNiwa_v1"` pour les fiches + `"kanjiNiwa_joyo"` pour le répertoire Jōyō
- Pas de framework, pas de build tool, pas de bundler

---

## Configuration

```js
const CONFIG = {
  apiKey: "", // L'utilisateur colle sa clé Anthropic ici
  model: "claude-sonnet-4-6",
  apiUrl: "https://api.anthropic.com/v1/messages",
};
```

Si `CONFIG.apiKey` est vide :

- Bouton "Générer" disabled avec tooltip "Ajoutez votre clé API dans ⚙️"
- Tout le reste fonctionne normalement

---

## Deux niveaux de données

### Niveau 1 — Répertoire Jōyō (localStorage "kanjiNiwa_joyo")

2136 entrées légères chargées une seule fois depuis JOYO_DATA (variable JS hardcodée) :

```js
{
  numero: number,       // ex: 47 (numéro Jōyō officiel)
  caractere: string,    // ex: "遅"
  grade: number,        // 1-6 = école primaire, 7 = collège/lycée
  on: string[],         // lectures on basiques
  kun: string[],        // lectures kun basiques
  sens_en: string       // sens en anglais (source kanjidic2)
}
```

### Niveau 2 — Fiches riches (localStorage "kanjiNiwa_v1")

Tes fiches personnelles complètes :

```js
{
  radicaux: Radical[],
  kanji: Kanji[]
}
```

#### Radical

```js
{
  id: string,              // ex: "rad_shinnyou"
  caractere: string,       // ex: "辶"
  nom_japonais: string,    // ex: "しんにょう"
  nom_français: string,    // ex: "Mouvement"
  sens: string,
  kanji_ids: string[]
}
```

#### Kanji (fiche riche)

```js
{
  id: string,              // ex: "k_osoi"
  caractere: string,       // ex: "遅"
  signification: string,
  on: string[],
  kun: string[],
  radicaux_ids: string[],
  etymologie: string,
  mnemo: string,
  theme: string,           // "mouvement"|"corps"|"nature"|"famille"|"langage"|"vie"|"divers"
  srs: string,             // "要復習"|"学習中"|"習得"
  mots: MotInline[],
  phrases: Phrase[],
  date_ajout: string
}
```

#### MotInline

```js
{
  japonais: string,
  lecture: string,
  sens: string,
  composants: [
    { c: string, id: string|null }
  ]
}
```

#### Phrase

```js
{
  japonais: string,    // japonais pur
  hiragana: string     // même phrase entièrement en hiragana
}
```

---

## Moteur de liens dynamiques (CRITIQUE)

Fonction pure appelée à chaque render :

```js
function resolveComposant(composant) {
  if (!composant.id) return { ...composant, cliquable: false };
  const kanji = db.kanji.find((k) => k.id === composant.id);
  return { ...composant, cliquable: !!kanji };
}
```

- id présent + kanji trouvé → noir, cliquable
- id présent + kanji absent → gris `#B4B2A9`, tooltip "Pas encore étudié"
- id null → neutre, non cliquable

---

## Moteur Pokédex (CRITIQUE)

Fonction qui détermine l'état d'une case de la grille :

```js
function getKanjiState(joyo_entry) {
  const fiche = db.kanji.find((k) => k.caractere === joyo_entry.caractere);
  if (!fiche) return { state: "locked", fiche: null, joyo: joyo_entry };
  return {
    state: fiche.srs === "習得" ? "mastered" : "studying",
    fiche,
    joyo: joyo_entry,
  };
}
```

**3 états :**

- `"locked"` → kanji non encore étudié → silhouette + numéro Jōyō
- `"studying"` → fiche existe, SRS = 要復習 ou 学習中 → kanji coloré, pastille orange/rouge
- `"mastered"` → fiche existe, SRS = 習得 → kanji coloré, pastille verte

**Déblocage automatique** : dès qu'une fiche est ajoutée en base, la case correspondante dans la grille s'allume sans aucune action supplémentaire. C'est le même appel à `db.kanji.find()` qui gère tout.

---

## Palette de couleurs

```css
:root {
  --bg: #fafaf8;
  --surface: #f2f0eb;
  --border: #e8e4dc;
  --text: #1a1a18;
  --muted: #888780;
  --hint: #b4b2a9;
  --accent: #c0392b;
  --link: #2c3e7a;

  --mouvement: #378add;
  --corps: #d4537e;
  --nature: #1d9e75;
  --famille: #ef9f27;
  --langage: #7f77dd;
  --vie: #d85a30;
  --divers: #888780;

  --srs-rouge: #e24b4a;
  --srs-orange: #ef9f27;
  --srs-vert: #639922;

  --locked-bg: #f0eee9;
  --locked-text: #c8c4bc;
}
```

---

## Navigation

```
[漢] 漢字の庭    [漢字] [部首] [語彙] [復習] [⚙️]
```

- Logo : carré rouge 32×32px, 漢 en blanc, border-radius 6px
- Onglet actif : souligné ligne rouge 2px
- Fiches kanji et mots s'ouvrent dans le même conteneur avec bouton ← 戻る

---

## Vue 1 — 漢字 (Grille Pokédex)

**C'est la vue principale. Une seule grille unifiée de 2136 cases dans l'ordre Jōyō.**
Pas de toggle, pas de deux modes — une seule grille où chaque case peut être débloquée ou non.

### Barre de filtres

- Recherche texte (caractère, signification, kun, on) — filtre sur les 2136 cases
- Filtre grade : Tous · Grade 1 · Grade 2 · Grade 3 · Grade 4 · Grade 5 · Grade 6 · Secondaire
- Filtre thème (sur les fiches riches uniquement) : Tous · Mouvement · Corps · Nature · Famille · Langage · Vie
- Filtre SRS : Tous · 要復習 · 学習中 · 習得 · Non étudié
- Bouton "+ 追加" à droite, fond rouge

### Grille

- `grid-template-columns: repeat(auto-fill, minmax(100px, 1fr))`
- `gap: 10px`

### Carte — état DÉBLOQUÉ (fiche riche existe)

```
┌─────────────────┐  ← barre colorée 3px (thème)
│ #47          🟡 │  ← numéro Jōyō + pastille SRS
│                 │
│       遅        │  ← 44px, couleur thème
│                 │
│  Lent · Retard  │  ← 10px, --muted
│    おそい       │  ← 9px, --hint
└─────────────────┘
```

- Fond blanc, border colorée selon thème (0.5px)
- Clic → ouvre la fiche riche complète

### Carte — état VERROUILLÉ (pas encore étudié)

```
┌─────────────────┐
│ #48             │  ← numéro Jōyō, --hint
│                 │
│       ▓▓        │  ← silhouette 44px, couleur --locked-text
│                 │
│    Grade 1      │  ← grade scolaire, 9px, --hint
│                 │
└─────────────────┘
```

- Fond --locked-bg, border --border
- Hover : légère surbrillance, cursor pointer
- Clic → mini-fiche Jōyō (pas la fiche riche)

### Mini-fiche Jōyō (pour les cases verrouillées)

S'ouvre au clic sur une case verrouillée :

```
┌─────────────────────────────┐
│  ← 戻る                    │
│                             │
│         遅  #47             │  ← grand kanji + numéro
│                             │
│  Grade 1 · Jōyō             │
│  On : チ                    │
│  Kun : おそい, おくれる     │
│  Sens : slow, late          │  ← sens anglais kanjidic2
│                             │
│  [✨ Générer une fiche]     │  ← grisé si pas de clé API
│                             │
│  Pas encore dans tes fiches │
└─────────────────────────────┘
```

---

## Vue 2 — Fiche kanji riche

Bouton `← 戻る` en haut à gauche.
Badge `#47 · Grade 1` discret en haut à droite (numéro Jōyō si disponible).

### En-tête

- Caractère 96px, couleur thème en fond très léger (opacity 0.06)
- Signification 22px
- On'yomi badges bleus · Kun'yomi badges verts

### Sections dans l'ordre

1. **部首** — chips radicaux cliquables
2. **語源** — étymologie détaillée, line-height 1.8
3. **覚え方** — mnémo, fond #FAEEDA, bordure gauche 3px #FAC775
4. **読み方** — tableau on/kun
5. **語彙** — mots courants avec composants dynamiques + toggle hiragana sur phrases
6. **フレーズ** — 3 phrases avec bouton 読み方を見る
7. **SRS** — 3 boutons 要復習/学習中/習得
8. **IA** — bouton Régénérer (grisé si pas de clé)

### Section 語彙 — format détaillé

```
遅い        おそい        lent, en retard
  └─ [遅✓]

遅刻        ちこく        être en retard   [熟語]
  └─ [遅✓] [刻✗]

時代遅れ    じだいおくれ  démodé           [熟語]
  └─ [時✗] [代✗] [遅✓]
```

- Chip ✓ vert + cliquable = kanji dans la base
- Chip ✗ gris = pas encore étudié
- Clic sur une ligne → fiche mot

---

## Vue 3 — 語彙 (Mots)

### Filtres

- Onglets : Tous · 単語 · 熟語
- Recherche texte
- Filtre par kanji

### Fiche mot

- Mot 48px + lecture + sens + badge type
- Composants avec liens dynamiques
- 3 phrases avec toggle hiragana

---

## Vue 4 — 部首 (Radicaux)

### Grille des radicaux

- Chips : caractère + nom japonais + sens français + badge nombre de kanji

### Graphe D3.js force-directed

```js
const simulation = d3
  .forceSimulation(nodes)
  .force(
    "link",
    d3
      .forceLink(links)
      .id((d) => d.id)
      .distance(80),
  )
  .force("charge", d3.forceManyBody().strength(-200))
  .force("center", d3.forceCenter(width / 2, height / 2))
  .force(
    "collision",
    d3.forceCollide().radius((d) => d.r + 5),
  );
```

- Nœuds radicaux : r=28, couleur thème dominant
- Nœuds kanji : r=18, couleur thème
- Interactions : drag, clic → fiche, hover → tooltip, zoom/pan

---

## Vue 5 — 復習 (Quiz SRS)

### Sélection

```js
function getCardsForSession() {
  return [
    ...db.kanji.filter((k) => k.srs === "要復習"),
    ...db.kanji.filter((k) => k.srs === "学習中"),
    ...db.kanji.filter((k) => k.srs === "習得"),
  ];
}
```

### Recto

- Kanji 120px centré
- Bouton [Révéler ▼]
- Barre de progression

### Verso

- Kanji + signification + lectures
- Mnémo en fond --surface
- 3 boutons : [もう一度 🔴] [なんとか 🟡] [完璧 🟢]

### Mode mots (toggle)

- Affiche un mot → révèle kanji composants

---

## Vue 6 — ⚙️ Paramètres

### Clé API

- Input type password + toggle visibilité
- Stockage dans `"kanjiNiwa_apiKey"` (clé séparée, hors export)

### Export / Import

- Export → `kanji_niwa_backup_YYYYMMDD.json`
- Import → merge par ID, pas d'écrasement

### Statistiques

```
漢字 débloqués : 20 / 2136  (0.9%)

Par grade :
Grade 1   [████░░░░░░░░]  12/80
Grade 2   [█░░░░░░░░░░░]   4/160
...

Par SRS :
要復習  14  ████████████
学習中   4  ████
習得     2  ██
```

### Réinitialisation

- Bouton rouge, confirmation en tapant "RESET"

---

## Modal d'ajout de kanji

```
┌─────────────────────────────┐
│  Ajouter un kanji           │
│                             │
│  Kanji : [遅            ]   │
│  → Détecté : #47 · Grade 1  │  ← lookup auto dans Jōyō
│                             │
│  [✨ Générer avec l'IA]     │
│                             │
│  — ou saisir manuellement —  │
│  Signification, On, Kun, Thème
│                             │
│  [Annuler]    [Ajouter]     │
└─────────────────────────────┘
```

Quand l'utilisateur tape un kanji, lookup immédiat dans JOYO_DATA → affiche numéro et grade automatiquement.

---

## Génération IA

### Prompt système

```
Tu es un expert en kanji japonais et en pédagogie.
Génère une fiche complète pour le kanji demandé, destinée à un apprenant francophone.
Réponds UNIQUEMENT avec du JSON valide, aucun texte avant ou après, aucun markdown.

Schéma exact :
{
  "id": "k_{romanisation}",
  "caractere": "...",
  "signification": "...",
  "on": ["..."],
  "kun": ["..."],
  "radicaux_ids": [],
  "etymologie": "Texte détaillé, 3-4 paragraphes. Évolution pictographique, composants, logique.",
  "mnemo": "Histoire narrative mémorable, 3-4 phrases visuelles et concrètes.",
  "theme": "mouvement|corps|nature|famille|langage|vie|divers",
  "srs": "要復習",
  "mots": [
    {
      "japonais": "...",
      "lecture": "...",
      "sens": "...",
      "composants": [{ "c": "...", "id": "k_id_ou_null" }]
    }
  ],
  "phrases": [
    { "japonais": "...", "hiragana": "..." },
    { "japonais": "...", "hiragana": "..." },
    { "japonais": "...", "hiragana": "..." }
  ],
  "date_ajout": "TODAY"
}

Règles :
- Minimum 7 mots courants
- 3 phrases à niveaux différents (familier, concret, formel/figuré)
- Hiragana = phrase ENTIÈRE transcrite, pas seulement les kanji
```

### Code appel API

```js
async function generateKanji(caractere) {
  const response = await fetch(CONFIG.apiUrl, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": CONFIG.apiKey,
      "anthropic-version": "2023-06-01",
      "anthropic-dangerous-direct-browser-access": "true",
    },
    body: JSON.stringify({
      model: CONFIG.model,
      max_tokens: 4000,
      messages: [
        { role: "user", content: `Génère la fiche pour : ${caractere}` },
      ],
      system: SYSTEM_PROMPT,
    }),
  });
  const data = await response.json();
  return JSON.parse(data.content[0].text);
}
```

---

## Données Jōyō (JOYO_DATA)

Variable JS hardcodée contenant les 2136 kanji Jōyō.
Format compact pour minimiser la taille :

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
  {
    numero: 3,
    caractere: "挨",
    grade: 7,
    on: ["アイ"],
    kun: [],
    sens_en: "push open",
  },
  // ... 2133 entrées supplémentaires
];
```

Au premier chargement, stocker dans `"kanjiNiwa_joyo"` si absent.

**Source des données :** utiliser la liste Jōyō officielle 2010 (常用漢字表). Les données doivent être complètes pour les 2136 kanji.

---

## Animations

```css
@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
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

.fiche-kanji {
  animation: fadeIn 150ms ease;
}
.hiragana-reveal {
  animation: slideDown 200ms ease;
}
.card-unlocked {
  animation: unlock 400ms ease;
}
```

L'animation `unlock` se joue sur une carte qui vient d'être débloquée (quand on ajoute un kanji et qu'on revient sur la grille).

---

## Typographie

```css
body {
  font-family:
    system-ui,
    -apple-system,
    "Hiragino Sans",
    "Yu Gothic",
    sans-serif;
  font-size: 15px;
  line-height: 1.6;
  color: var(--text);
  background: var(--bg);
}
.kanji-display {
  font-family: "Hiragino Mincho ProN", "Yu Mincho", "MS Mincho", serif;
}
```

---

## Initialisation

```js
function initDB() {
  // Fiches riches
  if (!localStorage.getItem("kanjiNiwa_v1")) {
    localStorage.setItem("kanjiNiwa_v1", JSON.stringify(DATA_INITIALE));
  }
  // Répertoire Jōyō
  if (!localStorage.getItem("kanjiNiwa_joyo")) {
    localStorage.setItem("kanjiNiwa_joyo", JSON.stringify(JOYO_DATA));
  }
}
```

---

## Points critiques

1. **Moteur Pokédex** : `getKanjiState()` appelé à chaque render de la grille, pas mis en cache
2. **Moteur liens** : `resolveComposant()` appelé à chaque render des composants, pas mis en cache
3. **Numéro Jōyō** : affiché sur chaque carte débloquée (coin haut gauche, petit, discret)
4. **Animation unlock** : se joue uniquement lors de l'ajout d'un nouveau kanji
5. **Graphe D3** : se redessine au retour sur la vue 部首
6. **Export JSON** : inclut uniquement `"kanjiNiwa_v1"` (fiches riches + radicaux), pas le répertoire Jōyō (trop lourd et reconstituable)
7. **Import JSON** : merge par ID sans écraser
8. **Clé API** : stockée dans `"kanjiNiwa_apiKey"`, hors export
9. **Position fixed interdite** : wrappers en flow normal pour les modaux
10. **JOYO_DATA doit être complète** : tous les 2136 kanji, pas d'entrées manquantes
