# SmartCampus — Architecture de données et IoT

## Présentation du projet

SmartCampus est un projet de conception d’une plateforme de supervision des infrastructures techniques et énergétiques d’un ou plusieurs campus. Le périmètre couvre les bâtiments et zones, les équipements, les capteurs et mesures IoT, les alertes, les interventions de maintenance et les techniciens.

Le dossier documente la démarche depuis l’analyse du besoin jusqu’aux recommandations d’architecture. Le dimensionnement de référence est de 8 bâtiments, environ 160 zones, 250 équipements et 600 capteurs. À une mesure toutes les 30 secondes par capteur, le flux nominal représente environ 20 mesures/s. PostgreSQL est retenu comme source de vérité; les projections et moteurs complémentaires restent conditionnels aux résultats de benchmark.

Les valeurs de dimensionnement sont des hypothèses de travail tirées du cas, et non des mesures de production. Les règles métier encore ouvertes et les décisions à valider sont signalées dans les livrables.

## Contenu du dossier

### Sources du projet

- [Cas pratique SmartCampus](./sources/cas-pratique-smartcampus.pdf) — contexte fonctionnel et dimensionnement du cas.
- [Sujet du TD 1](./sources/td/td1-du-cahier-des-charges-au-mcd.pdf) — consignes pédagogiques de modélisation.
- [Sujet du TD 2](./sources/td/TD%202%20%E2%80%93%20Du%20MCD%20au%20MLD%20_%20MPD%20SQL.docx.pdf) — consignes de traduction du MCD vers le MLD et le MPD SQL.
- [Sujet du TD 3](./sources/td/TD%203%20%E2%80%93%20Normaliser%20ou%20d%C3%A9normaliser.pdf) — consignes d'arbitrage entre normalisation et dénormalisation.
- [Sujet du TD 4](./sources/td/TD%204%20%E2%80%93%20Paradigmes%20NoSQL%20et%20benchmark.pdf) — consignes d'étude des paradigmes NoSQL et du benchmark.

Les originaux des PDF sont conservés dans le dossier parent `Exo`; les copies organisées du projet sont dans `sources/`.

### Livrables

#### Dossier final

- [Dossier final de synthèse — architecture de données, modélisation et résilience IoT](./livrables/synthese-finale-smartcampus.md) — consolidation des huit étapes, du contexte métier aux recommandations, sans résultats de benchmark inventés.
- [TD 2 — Du MCD au MLD / MPD SQL](./livrables/td2/td2-du-mcd-au-mld-mpd-sql.md) — schéma relationnel, types PostgreSQL, contraintes, index, requêtes métier et tests d’intégrité.
- [TD 3 — Arbitrage entre normalisation et dénormalisation](./livrables/td3/td3-arbitrage-normalisation-denormalisation.md) — comparaison des trois scénarios, choix de modélisation, synchronisation et critères de réussite.
- [TD 4 — Paradigmes NoSQL, dataset et benchmark](./livrables/td4/td4-nosql-paradigmes-dataset-benchmark.md) — scénarios document, clé-valeur, wide-column et graphe, dataset de dimensionnement et protocole de benchmark.

| Ordre | Livrable | Contenu |
|---:|---|---|
| TD 1 | [TD 1 — Du cahier des charges au MCD](./livrables/td1/td1-du-cahier-des-charges-au-mcd.md) | Entités métier, dictionnaire de données, cardinalités justifiées, historisation et hypothèses ouvertes. |
| TD 2 | [TD 2 — Du MCD au MLD / MPD SQL](./livrables/td2/td2-du-mcd-au-mld-mpd-sql.md) | Schéma relationnel, types PostgreSQL, contraintes, index et requêtes métier. |
| TD 3 | [TD 3 — Arbitrage normalisation / dénormalisation](./livrables/td3/td3-arbitrage-normalisation-denormalisation.md) | Tableau d'arbitrage des trois scénarios et mécanismes de synchronisation. |
| TD 4 | [TD 4 — Paradigmes NoSQL, dataset et benchmark](./livrables/td4/td4-nosql-paradigmes-dataset-benchmark.md) | Modèles candidats, scénarios d'usage, volumétrie et protocole de comparaison reproductible. |
| 1 | [Étape 1 — Analyse métier et dimensionnement](./livrables/etapes/01-analyse-metier-et-dimensionnement.md) | Acteurs, besoins, règles de gestion, exigences non fonctionnelles, volumes et scénarios d’accès. |
| 2 | [Étape 2 — MCD](./livrables/etapes/02-mcd.md) | Entités, identifiants, associations, cardinalités et dictionnaire de données. |
| 3 | [Étape 3 — MLD / MPD](./livrables/etapes/03-mld-mpd.md) | Schéma relationnel, types PostgreSQL, contraintes, index et stratégie de partitionnement. |
| 4 | [Étape 4 — Normalisation](./livrables/etapes/04-normalisation.md) | Analyse 3NF/BCNF, redondances et projections de lecture contrôlées. |
| 5 | [Étape 5 — NoSQL](./livrables/etapes/05-nosql.md) | Comparaison des quatre paradigmes et justification du maintien de PostgreSQL seul au démarrage. |
| 6 | [Étape 6 — Architecture polyglotte et résilience](./livrables/etapes/06-architecture-polyglotte-et-resilience.md) | Composants, flux, synchronisation, idempotence, indisponibilités et RPO/RTO à valider. |
| 7 | [Étape 7 — Benchmark](./livrables/etapes/07-benchmark.md) | Protocole reproductible, dataset, scénarios, métriques, répétitions et limites; aucun résultat fictif. |
| 8 | [Étape 8 — Recommandations](./livrables/etapes/08-recommandations.md) | Synthèse des choix, alternatives écartées, risques et pistes d’évolution. |

### Organisation du dépôt

L’arborescence et le plan de classement appliqué sont détaillés dans [Organisation-projet-SmartCampus.md](./Organisation-projet-SmartCampus.md). Les sources sont classées dans `sources/`, les livrables dans `livrables/` et les notes, brouillons et ressources de benchmark dans `travail/`.

## Choix d’architecture documentés

- **Source de vérité :** PostgreSQL relationnel.
- **Modèle :** référentiel normalisé et relations temporelles pour les localisations et affectations de capteurs.
- **Mesures :** historique IoT partitionné mensuellement dans le modèle physique, avec conservation brute cible de 12 mois.
- **Résilience :** tampons durables, rejeu et consommateurs idempotents; les modalités précises dépendent des objectifs d’exploitation retenus.
- **Dénormalisation :** projections d’état courant et agrégats uniquement lorsqu’un besoin mesuré le justifie.
- **NoSQL :** aucun moteur supplémentaire n’est requis par le scénario nominal; un cache clé-valeur reste une option conditionnelle aux benchmarks.
- **Performance et continuité :** les seuils de latence, disponibilité, RPO/RTO et critères de réussite du benchmark restent à faire approuver. Aucun résultat de performance n’est revendiqué.

## Conventions de maintenance documentaire

1. Conserver les PDF sources séparés des documents de remise et des notes de travail.
2. Mettre à jour ce README lorsqu’un livrable est ajouté, renommé ou déplacé.
3. Signaler dans le livrable toute hypothèse non confirmée; ne pas transformer une proposition en règle validée sans arbitrage.
4. Versionner ensemble les schémas, scripts et documents qui décrivent une même décision d’architecture.
5. Ne publier des mesures de benchmark qu’avec l’environnement, le dataset, les paramètres de charge et le protocole de répétition associés.
