# SmartCampus — Étape 7 : protocole de benchmark

## 1. Objectif et règles de mesure

Ce livrable définit un protocole reproductible pour évaluer PostgreSQL et les services SmartCampus. Il couvre l’ingestion nominale et en rafale, les tableaux de bord, les historiques et le traitement des alertes.

**Aucun résultat de performance n’est présenté ici.** Aucun benchmark n’a été exécuté dans le cadre de ce livrable; les débits, latences, consommations de ressources et taux d’erreur sont donc à mesurer. Les nombres de charge ci-dessous sont des paramètres déduits du cas ou des profils de test explicitement définis, pas des résultats observés ni des SLA.

Le benchmark doit comparer une architecture et une configuration identifiables, en conservant les mêmes données, requêtes, durée, environnement et paramètres de charge d’un essai à l’autre. Tout changement de schéma, index, taille de lot, réglage PostgreSQL ou code applicatif crée une nouvelle variante de test.

## 2. Périmètre et hypothèses de charge

### 2.1 Référence métier

| Paramètre | Valeur de référence | Usage dans le protocole |
|---|---:|---|
| Bâtiments | 8 | Population référentielle |
| Zones | ~160 | Population référentielle |
| Équipements | 250 | Population et dimensionnement des alertes |
| Capteurs | 600 | Sources d’ingestion et de lecture |
| Périodicité nominale | 1 mesure / 30 s / capteur | 20 mesures/s en moyenne |
| Débit nominal par jour | 1 728 000 mesures/jour | Contrôle d’intégralité et dimensionnement |
| Mesures annuelles sur 365 jours | 630 720 000 | Niveau de données pleine rétention à tester si les ressources le permettent |
| Rétention brute | 12 mois | Partitionnement et fenêtres historiques |
| Utilisateurs actifs | ~50 en journée | Profil de lecture concurrent |
| Actualisation du tableau de bord | Toutes les 5 min | Cadence métier; pic synchronisé testé séparément |

La rafale synchronisée de référence est de **600 mesures simultanées toutes les 30 secondes** : elle garde le même débit moyen nominal, mais vérifie le comportement en pointe. Elle ne modélise pas à elle seule une reconnexion après une longue coupure, qui doit être testée avec un backlog paramétré.

### 2.2 Environnement d’essai à consigner

Ne pas comparer des résultats si les informations suivantes manquent. Remplir cette fiche avant chaque campagne :

| Élément | Valeur à renseigner |
|---|---|
| Identifiant de campagne / date / opérateur | `[à renseigner]` |
| Commit applicatif, version du générateur et scripts SQL | `[à renseigner]` |
| Version PostgreSQL, extensions et système d’exploitation | `[à renseigner]` |
| CPU alloués, mémoire, type/nombre de disques et système de fichiers | `[à renseigner]` |
| Latence et débit du stockage, paramètres I/O disponibles | `[à renseigner]` |
| Réseau entre générateur, ingestion, file éventuelle et base | `[à renseigner]` |
| Topologie : hôtes, conteneurs/VM, réplication, placement des services | `[à renseigner]` |
| Paramètres PostgreSQL significatifs (`shared_buffers`, WAL, checkpoints, connexions, etc.) | `[à renseigner]` |
| Schéma exact, index, contraintes, partitions existantes et indexation activée | `[à renseigner]` |
| Données de cache chaud/froid, état du cache OS et du pool de connexions | `[à renseigner]` |
| Outils de charge, versions et paramètres d’échantillonnage | `[à renseigner]` |
| Télémétrie activée et éventuel coût de son instrumentation | `[à renseigner]` |

Utiliser un environnement isolé et représentatif de la cible. Exécuter le générateur et les clients de charge depuis un hôte identifié; mesurer leur CPU, réseau et limites afin de vérifier qu’ils ne sont pas eux-mêmes le goulot d’étranglement. Ne pas mélanger le trafic du benchmark avec une charge de production.

## 3. Dataset reproductible

### 3.1 Population référentielle

Créer exactement les volumes de référence suivants, sauf pour les valeurs indiquées comme approximatives :

- 1 campus ou le nombre effectivement décidé dans le MCD, puis 8 bâtiments répartis entre les campus conformément à la configuration de test;
- environ 160 zones, réparties de manière déterministe entre les bâtiments;
- 250 équipements rattachés au référentiel et localisés à l’aide de périodes d’historisation;
- 600 capteurs avec identifiants externes déterministes et affectations temporelles aux équipements;
- règles, alertes, interventions, techniciens et utilisateurs nécessaires aux scénarios.

Documenter la distribution choisie (nombre de zones par bâtiment, équipements par zone, capteurs par équipement, règles par cible, états d’alertes/interventions). Ces distributions ne sont pas spécifiées dans le cas : utiliser une graine fixe et ne pas les présenter comme des statistiques réelles. Préserver les contraintes du MCD, notamment la non-superposition des périodes d’affectation.

### 3.2 Population de mesures et paliers

Utiliser un générateur déterministe, versionné et configuré par graine. Pour chaque mesure générée, renseigner au minimum `id_capteur`, `horodatage_evenement`, `horodatage_reception`, `valeur`, `unite` et `qualite`. Générer des séries temporelles continues, avec des variations plausibles, des données invalides contrôlées, des doublons de transport pour les tests d’idempotence et des messages tardifs pour les tests d’ordre. Les plages de valeurs sont définies par type de capteur dans un profil de données joint à la campagne, pas inventées comme plages métier validées.

Exécuter des paliers séparés, sans prétendre qu’un palier intermédiaire équivaut au plein historique :

| Palier | Nombre de mesures cible | Équivalent à 20 mesures/s (ordre de durée de production) | Objectif |
|---|---:|---:|---|
| Développement | 10 000 000 | ~5,8 jours | Vérifier les scripts, les requêtes et les premiers plans d’exécution. |
| Charge intermédiaire | 100 000 000 | ~57,9 jours | Observer la croissance des index, la distribution et l’effet des partitions. |
| Historique annuel | 630 720 000 | 365 jours de production nominale | Tester la capacité correspondant à 12 mois nominaux, si les ressources et le temps de préparation le permettent. |

Les équivalents de durée sont des conversions arithmétiques au débit nominal, pas des temps de génération ni des résultats de test. Un palier peut être réduit ou étendu, mais le nombre exact, la plage temporelle, le nombre de partitions et l’empreinte obtenue doivent être rapportés.

### 3.3 Génération et contrôle du dataset

1. Fixer la graine pseudo-aléatoire, les versions du générateur et les paramètres de distribution.
2. Créer le référentiel, les historiques et les partitions mensuelles avant le chargement. Vérifier la présence de chaque partition de la période couverte.
3. Charger les mesures par lots, en respectant le chemin d’écriture réellement utilisé (ingestion et validation si possible). Si un chargement accéléré en direct est aussi utilisé, le déclarer comme un scénario distinct non comparable à l’ingestion applicative.
4. Répartir les mesures sur les 600 capteurs et sur les horodatages choisis; générer les valeurs de manière déterministe par capteur et par temps afin qu’un rejeu donne le même contenu.
5. Mesurer séparément la durée de génération, la durée de chargement, l’espace avant/après et les coûts d’indexation; ne pas inclure ces temps dans une mesure de requête sans le préciser.
6. Après chargement, vérifier les comptes attendus par partition, capteur et période; contrôler les doublons, les trous introduits intentionnellement, les liens référentiels, les distributions de qualité et les contraintes temporelles.
7. Conserver un manifeste du dataset (graine, paramètres, compte attendu, empreinte/checksum par fichier ou lot, dates début/fin et nombre de lignes). Réinitialiser la base ou restaurer un snapshot immuable pour les essais comparatifs.

Les données synthétiques servent à comparer les comportements techniques. Elles ne démontrent pas la représentativité de la distribution réelle, qui devra être réévaluée à partir d’échantillons anonymisés ou de statistiques métier validées si disponibles.

## 4. Scénarios de charge

Pour chaque scénario, enregistrer les paramètres exacts : durée de préparation, durée mesurée, nombre de clients, débit planifié, taille de lots, délai entre requêtes, politique de reconnexion et éventuel cache. Ne pas modifier plusieurs paramètres à la fois lors d’une comparaison.

### S1 — Ingestion nominale continue

- Émettre une mesure toutes les 30 secondes pour chacun des 600 capteurs, en distribuant les émissions au cours des 30 secondes ou en les synchronisant selon deux variantes distinctes.
- Débit moyen attendu par définition de charge : **20 messages/s**; cette valeur est une consigne d’entrée, non le débit effectivement accepté.
- Inclure validation, déduplication, insertion dans la partition mensuelle, mécanismes de file et projections réellement prévues en production.
- Durée de mesure suffisante pour inclure des checkpoints et plusieurs rotations de lots; consigner sa valeur exacte plutôt que supposer qu’une courte exécution représente une charge durable.
- Vérifier l’écart entre messages émis, durablement acceptés, insérés, rejetés et en attente.

### S2 — Rafale synchronisée

- Émettre **600 messages à un instant de cycle**, puis répéter selon le cycle nominal de 30 secondes.
- Comparer cette variante avec l’émission étalée de S1, sans changer le contenu ou la configuration de base.
- Mesurer la latence de vidage de la rafale, la profondeur maximale de file, les erreurs, les attentes et le retour à un backlog nul.
- Rapporter le débit instantané observé et la fenêtre de calcul; ne pas appeler « débit de pointe » un simple débit moyen de cycle.

### S3 — Reconnexion avec backlog

- Injecter une coupure simulée de durée paramétrée, puis rejouer les mesures tamponnées par une ou plusieurs passerelles.
- Les durées de coupure et facteurs d’accélération de rattrapage ne sont pas donnés par le besoin : définir plusieurs profils d’essai, les documenter et les faire valider par l’exploitation.
- Vérifier l’absence de perte silencieuse, l’idempotence des répétitions et le traitement des mesures retardées sans régression de l’état courant.
- Mesurer le temps de rattrapage, la latence des messages en cours et l’effet du rattrapage sur les lectures concurrentes.

### S4 — Tableaux de bord, 50 utilisateurs

- Simuler 50 utilisateurs virtuels, chacun exécutant le parcours représentatif de supervision : état des capteurs, équipements, alertes/interventions visibles et indicateurs.
- Respecter une actualisation périodique de cinq minutes dans le scénario nominal réparti; exécuter à part un scénario de rafraîchissement synchronisé par 50 clients pour tester le pic.
- Décrire le nombre de requêtes par tableau de bord, les filtres, la taille de réponse, le comportement de pagination, les droits et le cache. Ces paramètres manquent au besoin initial et doivent être fixés pour chaque campagne.
- Faire tourner la lecture seule et la lecture concurrente à l’ingestion; isoler les résultats de chaque mode.
- Mesurer la fraîcheur des données servies : date d’événement et date de réception affichées par rapport à la source.

### S5 — Dernière valeur et état courant

- Parcourir la dernière valeur d’un capteur, puis l’état de l’ensemble des 600 capteurs, avec et sans projection SQL optionnelle.
- Distinguer dernière mesure reçue, dernière mesure valide et état calculé; appliquer la même règle de qualité dans chaque variante.
- Comparer les temps de réponse sous charge seule puis sous ingestion concurrente, y compris au moment d’une rafale.
- Si un cache clé-valeur est évalué, ajouter une variante séparée avec cache froid, chaud, retardé et indisponible avec repli PostgreSQL. Il n’est pas inclus dans le scénario de référence actuellement retenu.

### S6 — Consultations historiques

Pour des capteurs tirés de manière reproductible, exécuter des plages de **5 minutes, 1 heure, 24 heures, 30 jours et 12 mois**. Ce sont des fenêtres de test couvrant plusieurs échelles; elles ne constituent pas une affirmation de fréquence métier.

- Essayer une série d’un capteur, puis un groupe de capteurs et une agrégation, en conservant les mêmes paramètres entre variantes.
- Mesurer le temps de réponse, les lignes retournées et examinées si disponibles, les partitions parcourues, les lectures disque/cache et la consommation de ressources.
- Tester des plages alignées et non alignées sur les frontières de partition mensuelles; noter le nombre de partitions concernées.
- Séparer les requêtes brutes des requêtes d’agrégats préparés. Documenter les unités, filtres qualité et règles d’attribution équipement/zone utilisées.

### S7 — Traitement des alertes

- Définir un jeu versionné de règles synthétiques et un corpus de mesures déclenchant et ne déclenchant pas ces règles.
- Fixer explicitement le taux de déclenchement dans la charge générée; ne pas déduire un taux réel d’alertes du cas, qui ne fournit ni seuils ni fréquence d’événements.
- Mesurer la latence mesure acceptée → règle évaluée → alerte durablement créée → alerte visible via API.
- Inclure plusieurs règles par flux si ce profil correspond à la configuration visée; noter le nombre de règles réellement exécutées.
- Réinjecter des doublons et des mesures tardives; vérifier qu’un rejeu ne crée pas d’alertes en double et observer la politique appliquée aux événements hors ordre.
- Rapporter erreurs, file d’attente du consommateur, temps de traitement, nombre d’alertes créées et nombre attendu selon le corpus de référence.

### S8 — Dégradation et reprise (optionnel)

- Tester l’indisponibilité contrôlée d’un consommateur, de la file ou de PostgreSQL dans un environnement isolé.
- Vérifier back-pressure, absence de faux accusé, tamponnement, rattrapage, intégrité des transactions et reconstruction des projections.
- Ces essais valident le comportement de reprise; ils ne remplacent pas un exercice PRA complet avec objectifs RPO/RTO approuvés.

## 5. Métriques et définitions

| Métrique | Définition opérationnelle | Source / précautions |
|---|---|---|
| **Débit offert** | Nombre de messages que le générateur tente d’émettre par seconde, avec détail par fenêtre et par source. | Journal du générateur; mesurer son CPU et sa capacité réseau. |
| **Débit durable accepté** | Messages confirmés après persistance durable selon le contrat d’accusé. | Accusés de l’API/file; distinguer de la seule réception réseau. |
| **Débit inséré** | Mesures validées et commités dans PostgreSQL par seconde. | Compteurs applicatifs et base; exclure les rejeux dédupliqués ou les rapporter séparément. |
| **Latence ingestion** | `horodatage_reception` ou horodatage d’acceptation − `horodatage_evenement`, avec distribution de retard. | Horloges synchronisées; distinguer délai réseau, file et traitement. |
| **Latence bout en bout** | Instant où la mesure est observable par le client/API − instant d’événement de la mesure. | Instrumenter événement, persistance, règle éventuelle et lecture/affichage; définir si « affichage » signifie réponse API ou rendu navigateur. |
| **Latence de dernière valeur** | Durée serveur/client de la requête définie au scénario S5, de l’envoi à la réponse complète. | Rapporter percentiles et erreurs, pas uniquement la moyenne. |
| **Latence historique** | Durée de requête et de transfert pour une fenêtre, un nombre de capteurs et un volume de résultat fixés. | Rapporter séparément temps serveur, sérialisation et transfert si instrumentés. |
| **Latence d’alerte** | Événement → décision de règle → commit alerte → première lecture visible. | Mesurer chaque étape et bout en bout; compter le backlog du consommateur. |
| **Taux d’erreur** | Rejets fonctionnels, erreurs applicatives, erreurs SQL, timeouts, retries épuisés et pertes constatées / messages offerts. | Distinguer catégories et codes; rapporter pertes et doublons séparément. |
| **Ressources** | CPU, mémoire RSS, I/O, latence stockage, espace tables/index/WAL, réseau et connexions. | Échantillonnage et intervalle constants; collecteur lui-même compris dans l’environnement. |
| **Backlog / reprise** | Nombre et âge des messages en attente; durée jusqu’au rattrapage complet. | Mesurer pic, évolution et temps de retour au niveau initial. |
| **Fraîcheur d’état** | Âge de l’horodatage événementiel de la dernière mesure servie par capteur/projection. | Distinguer absence de mesure, mesure invalide et projection périmée. |

Rapporter au minimum le nombre de requêtes, médiane, p90, p95 et p99 lorsque l’échantillon est suffisant, minimum/maximum à titre informatif, taux d’erreur et intervalle temporel mesuré. Les percentiles élevés calculés sur trop peu d’échantillons ne sont pas interprétables; publier l’effectif. Synchroniser les horloges des hôtes et utiliser une source de temps vérifiée pour les métriques événement → affichage.

## 6. Répétitions, variabilité et comparaison

1. **Préparer et figer** : exporter les versions, configurations, manifeste dataset, requêtes et paramètres de charge; vérifier les comptes de lignes et la santé des partitions.
2. **Séparer les phases** : génération/chargement, préchauffage, mesure et nettoyage. Ne pas inclure les opérations de chargement dans les latences applicatives.
3. **Préchauffage** : exécuter un passage non comptabilisé par scénario jusqu’à stabilisation du comportement de cache et du pool; consigner la durée et les indicateurs utilisés pour juger la stabilisation.
4. **Répéter** : réaliser au moins cinq répétitions mesurées de chaque scénario sur le même état initial restauré. Si la durée ou le coût empêche cinq répétitions, consigner le nombre et qualifier la conclusion d’exploratoire.
5. **Comparer les modes** : exécuter lecture seule, écriture seule et charge mixte comme campagnes séparées; alterner l’ordre des variantes comparées lorsque possible pour limiter le biais de chauffe ou de dérive.
6. **État de cache** : rapporter séparément cache froid et cache chaud; ne pas mélanger leurs observations dans une seule distribution.
7. **Stabilité** : relever bruit de fond, tâches système, charge des générateurs, saturation, erreurs et événements d’infrastructure. Annuler et refaire une répétition si une perturbation indépendante invalide la mesure, en conservant la trace de l’essai écarté et son motif.
8. **Analyse** : publier chaque répétition, médiane et dispersion (par exemple min/max et intervalle interquartile); comparer les distributions et non le seul meilleur résultat. Signaler toute variation de configuration.
9. **Répétabilité** : un autre opérateur doit pouvoir restaurer le snapshot, exécuter les scripts avec les mêmes graines et obtenir le même nombre de messages et requêtes; les temps peuvent varier et doivent être rapportés, non supposés identiques.

### 6.1 Critères d’acceptation à définir avant la campagne

Le projet n’a pas encore fixé de SLA de latence ou d’objectifs quantitatifs d’erreur. Avant le benchmark de qualification, les parties prenantes doivent définir :

- débit minimal accepté et capacité de rattrapage attendue après une coupure;
- seuils de latence p95/p99 pour ingestion, tableaux de bord, dernier état et historiques;
- fraîcheur maximale admissible événement → affichage;
- taux d’erreur/rejet toléré et politique de données invalides;
- saturation CPU, mémoire, stockage et croissance maximale acceptables;
- concurrence simultanée réelle et profils de requêtes représentatifs;
- RPO/RTO et durée maximale de panne à absorber par les passerelles et la file.

Sans ces critères, le benchmark fournit des mesures descriptives et comparatives, mais ne permet pas de déclarer la solution « conforme » ou « suffisamment performante ».

## 7. Restitution des résultats — modèle vierge

Aucune cellule ci-dessous ne doit être complétée par estimation. Reporter les mesures réellement observées, leur unité, leur période, leur environnement et le nombre d’échantillons.

| Campagne / scénario | Dataset et configuration | Charge offerte | Débit accepté / inséré | Latence p50 / p95 / p99 | Erreurs / doublons | CPU / mémoire / I/O | Répétitions et observations |
|---|---|---:|---:|---|---|---|---|
| `[à mesurer]` | `[à renseigner]` | `[à renseigner]` | `[à mesurer]` | `[à mesurer]` | `[à mesurer]` | `[à mesurer]` | `[à renseigner]` |

Pour les historiques, ajouter la fenêtre, le nombre de capteurs, le nombre de lignes retournées, les partitions touchées et la taille de réponse. Pour les alertes, ajouter le nombre de règles évaluées, le nombre d’événements déclencheurs attendus/observés et le backlog maximum.

## 8. Limites méthodologiques et interprétation

- Un dataset synthétique reproductible n’est pas nécessairement représentatif des corrélations, distributions de valeurs, pics, invalidités ou comportements réseau réels.
- Le débit nominal de 20 mesures/s ne représente pas seul les reprises massives, les déconnexions de passerelles, les retransmissions ni la synchronisation des horloges.
- Un test de plusieurs millions de lignes valide des plans et tendances, mais ne remplace pas un test à l’échelle de 12 mois (~631 millions de mesures), notamment pour l’espace, les index, le vacuum, les partitions, les sauvegardes et la maintenance.
- Les résultats dépendent fortement du matériel, du stockage, de la réplication, des réglages PostgreSQL, du cache, de la taille des lots, du pool et du coût des validations applicatives. Les résultats d’un environnement ne sont pas transposables directement à un autre.
- Un benchmark de la base seule ne qualifie pas le chemin de bout en bout (capteur → passerelle → réseau → file → ingestion → PostgreSQL → règles → API). Mesurer séparément chaque étape et l’expérience complète.
- Des tests à 50 utilisateurs ne sont pertinents que si le parcours, le nombre de requêtes par tableau de bord, la synchronisation des rafraîchissements et la taille des réponses sont précisés.
- Les alertes ne peuvent être comparées sans un taux de déclenchement synthétique défini et un corpus attendu permettant de contrôler la justesse fonctionnelle.
- Les temps d’écriture accélérée en bulk ne sont pas représentatifs de l’ingestion temps réel si les contraintes, index, outbox, déduplication ou validations sont contournés.
- Les mesures de latence API ne sont pas équivalentes à un temps de rendu navigateur; le point de mesure doit être explicite.
- Aucun seuil de réussite, SLO, résultat ou recommandation de capacité chiffrée ne peut être déduit avant l’exécution et validation de la campagne.

## 9. Checklist de campagne

- [ ] Environnement et versions consignés; cible de test isolée.
- [ ] Dataset, graine et manifeste archivés; comptes et partitions vérifiés.
- [ ] Schéma, index et paramètres identiques entre variantes comparées.
- [ ] Générateurs et collecteurs non limitants ou limites mesurées.
- [ ] Scénarios nominal, rafale, backlog, 50 utilisateurs, historique et alertes exécutés.
- [ ] Horloges synchronisées et points de mesure documentés.
- [ ] Préchauffage séparé; répétitions et caches qualifiés.
- [ ] Débits offert/accepté/inséré et pertes réconciliés.
- [ ] Latences avec percentiles, effectifs et erreurs rapportées.
- [ ] Ressources, backlog, croissance disque et métriques PostgreSQL collectés.
- [ ] Résultats bruts et écarts de répétition archivés.
- [ ] Critères de réussite approuvés avant toute conclusion de capacité.

