# SmartCampus — Architecture de données et IoT

## Présentation du projet

SmartCampus est un projet de conception d’une plateforme de supervision des infrastructures techniques et énergétiques d’un ou plusieurs campus. Le périmètre couvre les bâtiments et zones, les équipements, les capteurs et mesures IoT, les alertes, les interventions de maintenance et les techniciens.

Le dossier documente la démarche depuis l’analyse du besoin jusqu’aux recommandations d’architecture. Les premiers TD ont utilisé une hypothèse de 8 bâtiments, environ 160 zones, 250 équipements et 600 capteurs; le scénario étendu du TD 5 porte sur **20 bâtiments, 2 000 équipements et 5 000 capteurs**. À une mesure toutes les 30 secondes par capteur, ce dernier scénario représente environ 167 mesures/s en moyenne. PostgreSQL est retenu comme source de vérité; les projections et moteurs complémentaires restent conditionnels aux résultats de benchmark.

Les valeurs de dimensionnement sont des hypothèses de travail tirées du cas, et non des mesures de production. Les règles métier encore ouvertes et les décisions à valider sont signalées dans les livrables.

## Contenu du dossier

### Sujets et documents de référence

- [Cas pratique SmartCampus](./sujets/cas-pratique-smartcampus.pdf) — contexte fonctionnel et dimensionnement du cas.
- [Sujet du TD 1](./sujets/tds/td1-du-cahier-des-charges-au-mcd.pdf) — consignes pédagogiques de modélisation.
- [Sujet du TD 2](./sujets/tds/TD%202%20%E2%80%93%20Du%20MCD%20au%20MLD%20_%20MPD%20SQL.docx.pdf) — consignes de traduction du MCD vers le MLD et le MPD SQL.
- [Sujet du TD 3](./sujets/tds/TD%203%20%E2%80%93%20Normaliser%20ou%20d%C3%A9normaliser.pdf) — consignes d'arbitrage entre normalisation et dénormalisation.
- [Sujet du TD 4](./sujets/tds/TD%204%20%E2%80%93%20Paradigmes%20NoSQL%20et%20benchmark.pdf) — consignes d'étude des paradigmes NoSQL et du benchmark.
- [Sujet du TD 5](<./sujets/tds/TD 5 – Architecture polyglotte et défense.docx.pdf>) — architecture polyglotte, synchronisation, sécurité, sauvegarde et analytique.

Les originaux des PDF sont conservés dans le dossier parent `Exo`; les copies de référence du projet sont classées dans `sujets/`.

### Rendus des TD

- [TD 1 — Du cahier des charges au MCD](./tds/td1-du-cahier-des-charges-au-mcd.md) — entités métier, dictionnaire de données, cardinalités, historisation et hypothèses ouvertes.
- [TD 2 — Du MLD au MPD SQL](./tds/td2-du-mld-mpd-sql.md) — schéma relationnel, types PostgreSQL, contraintes, index, requêtes métier et tests d’intégrité.
- [TD 3 — Arbitrage entre normalisation et dénormalisation](./tds/td3-arbitrage-normalisation-denormalisation.md) — comparaison des trois scénarios, choix de modélisation, synchronisation et critères de réussite.
- [TD 4 — Paradigmes NoSQL, dataset et benchmark](./tds/td4-nosql-paradigmes-dataset-benchmark.md) — scénarios document, clé-valeur, wide-column et graphe, dataset de dimensionnement et protocole de benchmark.
- [TD 5 — Architecture polyglotte et défense](./tds/td5.md) — composants, responsabilités des bases, synchronisation, sécurité, sauvegardes, analytique et recommandation exécutive.

### Projet transverse

- [Dossier final de synthèse — architecture de données, modélisation et résilience IoT](./projet/synthese-finale-smartcampus.md) — consolidation des huit étapes, du contexte métier aux recommandations, sans résultats de benchmark inventés.
- [Dossier consolidé TD 1 à TD 5](./projet/smartcampus-projet-final.md) — synthèse unifiée de la modélisation, des choix SQL/NoSQL et de l’architecture polyglotte.
- [Compilation intégrale TD 1 à TD 5](./tds/smartcampus-projet-complet.md) — contenus des cinq rendus réunis bout à bout, sans résumé ni réécriture.
- [Version PDF du dossier final](./projet/synthese-finale-smartcampus.pdf)

| Ordre | Livrable | Contenu |
|---:|---|---|
| TD 1 | [TD 1 — Du cahier des charges au MCD](./tds/td1-du-cahier-des-charges-au-mcd.md) | Entités métier, dictionnaire de données, cardinalités justifiées, historisation et hypothèses ouvertes. |
| TD 2 | [TD 2 — Du MLD au MPD SQL](./tds/td2-du-mld-mpd-sql.md) | Schéma relationnel, types PostgreSQL, contraintes, index et requêtes métier. |
| TD 3 | [TD 3 — Arbitrage normalisation / dénormalisation](./tds/td3-arbitrage-normalisation-denormalisation.md) | Tableau d'arbitrage des trois scénarios et mécanismes de synchronisation. |
| TD 4 | [TD 4 — Paradigmes NoSQL, dataset et benchmark](./tds/td4-nosql-paradigmes-dataset-benchmark.md) | Modèles candidats, scénarios d'usage, volumétrie et protocole de comparaison reproductible. |
| TD 5 | [TD 5 — Architecture polyglotte et défense](./tds/td5.md) | Couches d'architecture, cohérence, résilience, sécurité, analytique et recommandation exécutive. |
| 1 | [Étape 1 — Analyse métier et dimensionnement](./projet/etapes/01-analyse-metier-et-dimensionnement.md) | Acteurs, besoins, règles de gestion, exigences non fonctionnelles, volumes et scénarios d’accès. |
| 2 | [Étape 2 — MCD](./projet/etapes/02-mcd.md) | Entités, identifiants, associations, cardinalités et dictionnaire de données. |
| 3 | [Étape 3 — MLD / MPD](./projet/etapes/03-mld-mpd.md) | Schéma relationnel, types PostgreSQL, contraintes, index et stratégie de partitionnement. |
| 4 | [Étape 4 — Normalisation](./projet/etapes/04-normalisation.md) | Analyse 3NF/BCNF, redondances et projections de lecture contrôlées. |
| 5 | [Étape 5 — NoSQL](./projet/etapes/05-nosql.md) | Comparaison des quatre paradigmes et justification du maintien de PostgreSQL seul au démarrage. |
| 6 | [Étape 6 — Architecture polyglotte et résilience](./projet/etapes/06-architecture-polyglotte-et-resilience.md) | Composants, flux, synchronisation, idempotence, indisponibilités et RPO/RTO à valider. |
| 7 | [Étape 7 — Benchmark](./projet/etapes/07-benchmark.md) | Protocole reproductible, dataset, scénarios, métriques, répétitions et limites; aucun résultat fictif. |
| 8 | [Étape 8 — Recommandations](./projet/etapes/08-recommandations.md) | Synthèse des choix, alternatives écartées, risques et pistes d’évolution. |

### Organisation du dépôt

L’arborescence et le plan de classement appliqué sont détaillés dans [Organisation-projet-SmartCampus.md](./Organisation-projet-SmartCampus.md). Les sujets sont classés dans `sujets/`, les rendus des TD dans `tds/` et le projet transverse dans `projet/`.

## Choix d’architecture documentés

- **Source de vérité :** PostgreSQL relationnel.
- **Modèle :** référentiel normalisé et relations temporelles pour les localisations et affectations de capteurs.
- **Mesures :** historique IoT partitionné mensuellement dans le modèle physique, avec conservation brute cible de 12 mois.
- **Résilience :** tampons durables, rejeu et consommateurs idempotents; les modalités précises dépendent des objectifs d’exploitation retenus.
- **Dénormalisation :** projections d’état courant et agrégats uniquement lorsqu’un besoin mesuré le justifie.
- **NoSQL :** aucun moteur supplémentaire n’est requis par le scénario nominal; un cache clé-valeur reste une option conditionnelle aux benchmarks.
- **Performance et continuité :** les seuils de latence, disponibilité, RPO/RTO et critères de réussite du benchmark restent à faire approuver. Aucun résultat de performance n’est revendiqué.

## Conventions de maintenance documentaire

1. Conserver les PDF de consignes dans `sujets/`, séparés des rendus de TD et du projet transverse.
2. Mettre à jour ce README lorsqu’un livrable est ajouté, renommé ou déplacé.
3. Signaler dans le livrable toute hypothèse non confirmée; ne pas transformer une proposition en règle validée sans arbitrage.
4. Versionner ensemble les schémas, scripts et documents qui décrivent une même décision d’architecture.
5. Ne publier des mesures de benchmark qu’avec l’environnement, le dataset, les paramètres de charge et le protocole de répétition associés.
