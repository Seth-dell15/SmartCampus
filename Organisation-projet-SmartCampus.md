# Organisation du projet SmartCampus

## 1. Objectif

Cette organisation sépare les documents de référence, les livrables de remise et les matériaux de travail, tout en conservant une numérotation claire des étapes.

**État appliqué :** les PDF du projet sont classés dans `sources/`; le TD 1 et les étapes 1 à 8 sont dans `livrables/`; le `README.md` et le présent document restent à la racine. Les répertoires de suivi et de benchmark sont créés sous `travail/`. Les originaux des PDF sont conservés dans le dossier parent `Exo`.

## 2. Arborescence appliquée

```text
SmartCampus/
├── README.md
│   └── Présentation du projet, navigation, choix d’architecture et statut des documents
│
├── sources/
│   ├── cas-pratique-smartcampus.pdf
│   │   └── Cahier des charges et contexte fonctionnel de référence
│   └── td/
│       └── td1-du-cahier-des-charges-au-mcd.pdf
│           └── Sujet et consignes pédagogiques du TD 1
│
├── livrables/
│   ├── td1/
│   │   └── td1-du-cahier-des-charges-au-mcd.md
│   └── etapes/
│       ├── 01-analyse-metier-et-dimensionnement.md
│       ├── 02-mcd.md
│       ├── 03-mld-mpd.md
│       ├── 04-normalisation.md
│       ├── 05-nosql.md
│       ├── 06-architecture-polyglotte-et-resilience.md
│       ├── 07-benchmark.md
│       └── 08-recommandations.md
│
└── travail/
    ├── notes/
    │   └── décisions, questions ouvertes et comptes rendus de validation
    ├── brouillons/
    │   └── versions de travail non destinées à la remise
    └── benchmark/
        ├── scripts/
        ├── configurations/
        ├── datasets/
        └── resultats/
            └── mesures réellement exécutées et manifestes de campagne
```

### Pourquoi cette organisation

- **`sources/`** conserve les copies de référence du projet sans les mélanger avec les productions. Les originaux PDF restent dans le dossier parent `Exo`.
- **`livrables/`** rassemble les documents présentables et versionnés par ordre logique. Le TD 1 est séparé des étapes de conception qui constituent la suite du dossier.
- **`travail/`** accueille uniquement les notes, brouillons et ressources de test non finales. Les résultats de benchmark ne doivent contenir que des mesures exécutées, avec leurs configurations et manifestes.
- **`README.md`** constitue le point d’entrée et résume les documents, les décisions et les conventions.

Les dossiers `travail/benchmark/datasets/` et `resultats/` peuvent contenir des volumes importants ou des informations sensibles. Ne pas versionner de données personnelles, secrets, exports de production ou résultats sans contrôle d’accès. Ajouter au `.gitignore` les fichiers temporaires et jeux générés volumineux si le dossier est placé sous Git.

## 3. Déplacements et renommages appliqués

Les renommages utilisent des minuscules, des tirets et des noms sans espaces ou accents pour faciliter les chemins, commandes et liens Markdown. Les sources indiquent les noms qui se trouvaient à la racine du projet avant leur classement.

### 3.1 Sources PDF

Les fichiers PDF présents à la racine de `SmartCampus` ont été comparés aux documents du dossier parent par empreinte SHA-256. Ils étaient identiques. Les copies du projet ont été déplacées dans `sources/`; les originaux du dossier parent ont été conservés.

| Fichier avant classement | Destination appliquée | Action réalisée |
|---|---|---|
| `Cas Pratique - Smart Campus.docx.pdf` | `sources/cas-pratique-smartcampus.pdf` | Copie du projet déplacée et renommée; original parent conservé. |
| `TD 1 – Du cahier des charges au MCD.docx.pdf` | `sources/td/td1-du-cahier-des-charges-au-mcd.pdf` | Copie du projet déplacée et renommée; original parent conservé. |

### 3.2 Neuf livrables Markdown classés

| Nom avant déplacement (racine) | Destination appliquée |
|---|---|
| `TD1-Du-cahier-des-charges-au-MCD-SmartCampus.md` | `livrables/td1/td1-du-cahier-des-charges-au-mcd.md` |
| `Analyse-etape-1-SmartCampus.md` | `livrables/etapes/01-analyse-metier-et-dimensionnement.md` |
| `MCD-etape-2-SmartCampus.md` | `livrables/etapes/02-mcd.md` |
| `MLD-MPD-etape-3-SmartCampus.md` | `livrables/etapes/03-mld-mpd.md` |
| `Normalisation-etape-4-SmartCampus.md` | `livrables/etapes/04-normalisation.md` |
| `NoSQL-etape-5-SmartCampus.md` | `livrables/etapes/05-nosql.md` |
| `Architecture-polyglotte-etape-6-SmartCampus.md` | `livrables/etapes/06-architecture-polyglotte-et-resilience.md` |
| `Benchmark-etape-7-SmartCampus.md` | `livrables/etapes/07-benchmark.md` |
| `Recommandations-etape-8-SmartCampus.md` | `livrables/etapes/08-recommandations.md` |

### 3.3 Contrôles effectués et recommandations restantes

1. Les répertoires `sources/`, `livrables/` et `travail/` ainsi que leurs sous-dossiers ont été créés.
2. Les deux PDF ont été comparés aux originaux du dossier parent par SHA-256; les copies du projet ont été déplacées et renommées, sans supprimer les originaux parentaux.
3. Le TD 1 et les étapes 1 à 8 ont été déplacés et renommés conformément au tableau.
4. Les liens du README ont été mis à jour vers les nouveaux chemins et contrôlés.
5. Vérifier les liens externes ou favoris qui pourraient encore utiliser les anciens chemins.
6. Si le projet est placé sous Git, examiner les changements et valider le déplacement/renommage dans le dépôt selon le processus de versionnement du projet.

## 4. Contenu du README racine

Le [README.md](./README.md) présente le projet, les choix d’architecture et donne accès aux sources et livrables via leurs chemins désormais organisés. Les liens relatifs vers les sources et les neuf livrables ont été vérifiés après déplacement.

## 5. Règles de nommage et de version

- Noms de fichiers simples, stables, explicites, sans espaces ni accents; préfixer les étapes par deux chiffres (`01` à `08`).
- Employer le format Markdown pour les documents de projet et garder le format source original pour les PDF.
- Ne pas ajouter de numéro de version dans le nom des livrables si le suivi est assuré par Git; sinon, convenir d’une règle (`v1.0`, date ISO) avant de multiplier les copies.
- Distinguer visiblement brouillon et version de remise; ne pas publier les notes internes dans `livrables/`.
- Ne pas écrire « validé » dans un nom de fichier ou un statut sans validation explicite du contenu par les parties prenantes/enseignant.
- Conserver dans chaque livrable les hypothèses, limites et décisions ouvertes pertinentes, plutôt que de présenter des hypothèses comme exigences confirmées.
