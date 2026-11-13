# Organisation du projet SmartCampus

## 1. Objectif

Le dépôt sépare les sujets et documents de référence, les rendus des travaux dirigés et les documents du projet transverse. Cette organisation facilite la navigation et distingue clairement les consignes des productions.

**État appliqué :** les PDF de référence sont dans `sujets/`, les quatre rendus de TD dans `tds/`, et les huit étapes ainsi que la synthèse dans `projet/`. Le README et le présent document restent à la racine.

## 2. Arborescence appliquée

```text
SmartCampus/
├── README.md
├── Organisation-projet-SmartCampus.md
│
├── sujets/
│   ├── cas-pratique-smartcampus.pdf
│   └── tds/
│       ├── td1-du-cahier-des-charges-au-mcd.pdf
│       ├── TD 2 – Du MCD au MLD _ MPD SQL.docx.pdf
│       ├── TD 3 – Normaliser ou dénormaliser.pdf
│       └── TD 4 – Paradigmes NoSQL et benchmark.pdf
│
├── tds/
│   ├── td1-du-cahier-des-charges-au-mcd.md
│   ├── td2-du-mld-mpd-sql.md
│   ├── td3-arbitrage-normalisation-denormalisation.md
│   └── td4-nosql-paradigmes-dataset-benchmark.md
│
└── projet/
    ├── etapes/
    │   ├── 01-analyse-metier-et-dimensionnement.md
    │   ├── 02-mcd.md
    │   ├── 03-mld-mpd.md
    │   ├── 04-normalisation.md
    │   ├── 05-nosql.md
    │   ├── 06-architecture-polyglotte-et-resilience.md
    │   ├── 07-benchmark.md
    │   └── 08-recommandations.md
    ├── synthese-finale-smartcampus.md
    └── synthese-finale-smartcampus.pdf
```

## 3. Rôle des dossiers

- **`sujets/`** contient les copies des consignes pédagogiques et du cas pratique. Les originaux PDF sont conservés dans le dossier parent `Exo`.
- **`tds/`** regroupe les rendus finalisés des TD 1 à 4, à plat pour un accès direct.
- **`projet/`** rassemble les huit étapes de conception sous `etapes/` et la synthèse finale aux formats Markdown et PDF.
- **`README.md`** est le point d’entrée du dépôt : il présente le projet, les choix d’architecture et les liens vers les documents.
- **`Organisation-projet-SmartCampus.md`** décrit l’arborescence et les conventions de classement.

## 4. Conventions de maintenance

1. Conserver les PDF de consignes dans `sujets/` et les productions dans `tds/` ou `projet/`.
2. Mettre à jour le README et le présent document lors de l’ajout, du renommage ou du déplacement d’un fichier.
3. Employer des noms stables et explicites; préfixer les étapes par deux chiffres (`01` à `08`).
4. Signaler dans les documents les hypothèses non confirmées et les décisions restant à valider.
5. Ne publier des résultats de benchmark que s'ils ont réellement été mesurés et sont accompagnés de leur configuration et de leur protocole.
