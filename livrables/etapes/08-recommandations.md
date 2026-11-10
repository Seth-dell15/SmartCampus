# SmartCampus — Étape 8 : recommandations d’architecture

## 1. Objet et synthèse exécutive

Ce livrable conclut le parcours d’architecture des données SmartCampus. Il consolide l’analyse métier, le MCD, le MLD/MPD PostgreSQL, la normalisation, l’évaluation NoSQL, l’architecture de collecte résiliente et le protocole de benchmark.

La recommandation centrale est de démarrer avec une architecture **relationnelle maîtrisée et mesurable**, plutôt qu’avec une accumulation de moteurs :

- PostgreSQL est l’unique source de vérité pour le référentiel, les mesures brutes, les associations historisées, les règles, alertes et interventions.
- Le référentiel reste normalisé; les associations de localisation et de capteurs sont datées et protégées contre les chevauchements.
- Les mesures, estimées à **20 messages/s en moyenne**, sont partitionnées mensuellement par horodatage d’événement et indexées selon les accès temporels mesurés.
- Les projections de lecture sont distinctes des données de référence, reconstruisibles et adoptées seulement si le benchmark prouve un bénéfice utile.
- Les passerelles et le dispositif d’ingestion tamponnent les interruptions; idempotence, rejeu et contrôle des retards protègent l’intégrité.
- Les objectifs de latence, disponibilité, RPO/RTO, rétention précise et croissance future restent à faire approuver. Aucun résultat de performance n’est présumé.

Le périmètre de référence compte 8 bâtiments, environ 160 zones, 250 équipements, 600 capteurs, 50 utilisateurs actifs en journée et un tableau de bord actualisé toutes les cinq minutes. La cadence nominale représente **1 728 000 mesures/jour** et **630 720 000 mesures/an** sur 365 jours, avant retransmissions. Ce volume exige un dimensionnement et des tests sérieux, mais ne justifie pas à lui seul un stockage distribué ou NoSQL.

## 2. Synthèse des choix retenus

| Domaine | Choix recommandé | Justification et cohérence |
|---|---|---|
| **Source de vérité** | PostgreSQL, sans moteur NoSQL complémentaire au démarrage. | Les relations, contraintes référentielles, transactions et historiques temporels sont au cœur des besoins. Un seul référentiel réduit les problèmes de synchronisation, de sauvegarde et d’exploitation. |
| **Modèle de référence** | MCD dérivé en tables normalisées; référentiel en 3NF/BCNF sous les dépendances métier retenues. | Évite les divergences entre campus, bâtiments, zones, équipements et capteurs; les tables de relation représentent explicitement les associations. |
| **Localisation / affectations** | Tables d’historique à intervalles semi-ouverts pour équipement-zone et capteur-équipement. | Permet de restituer le contexte valable à la date d’une mesure ou d’une intervention. Les contraintes temporelles excluent les chevauchements. |
| **Mesures IoT** | Table relationnelle partitionnée mensuellement par `horodatage_evenement`, stockage brut 12 mois selon politique validée. | Le partitionnement réduit le périmètre des lectures temporelles et facilite l’archivage/purge; index par capteur et temps pour les séries. |
| **Indexation** | Index ciblés sur clés de jointure, `(id_capteur, horodatage_evenement)` et files de traitement. | Accélère les parcours attendus sans multiplier les structures coûteuses à chaque insertion. Les index sont confirmés par `EXPLAIN` et benchmark. |
| **État courant** | Commencer par les index et, si nécessaire, une projection relationnelle explicite. | Pour 600 capteurs, la lecture de dernière valeur peut être évaluée avant l’ajout d’un service externe. Distinguer dernière mesure reçue et dernière valide. |
| **Analytique** | Agrégats ciblés, calculables et versionnés si les requêtes historiques l’exigent. | Réduit les scans répétés de mesures brutes tout en préservant la source détaillée. Les règles d’unité, qualité, attribution et corrections tardives doivent être explicites. |
| **Collecte / résilience** | Passerelles avec spool local, point d’entrée validant les messages et file durable si retenue par le design d’exploitation. | Absorbe coupures et rafales, découple les consommateurs et rend le rejeu possible. Une file mémoire seule ne permet pas de confirmer une écriture durable. |
| **Synchronisation** | Traitement au moins une fois avec consommateurs idempotents, inbox/outbox transactionnelles ou mécanisme durable équivalent. | Évite de supposer un exactement-une-fois de bout en bout; permet les redélivrances et la reconstruction des projections. |
| **NoSQL** | Aucun paradigme NoSQL actuellement nécessaire; clé-valeur seulement comme option conditionnelle pour la lecture d’état. | Le bénéfice doit être démontré face à PostgreSQL optimisé, en tenant compte du coût et de la cohérence supplémentaires. |
| **SLA / PRA** | Définir les seuils avec métier et exploitation avant la qualification. | RPO/RTO, disponibilité et latences n’ont pas été fournis; toute valeur provisoire reste une hypothèse tant qu’elle n’est pas approuvée et testée. |

### 2.1 Cohérence de bout en bout

Le modèle sépare les responsabilités sans créer plusieurs autorités :

1. Une mesure valide est authentifiée et acceptée durablement par le chemin d’ingestion.
2. PostgreSQL conserve la mesure brute et le référentiel qui permettent de l’interpréter.
3. Les associations temporelles relient la mesure à l’équipement et à la zone valables au moment de l’événement, et non à la localisation actuelle.
4. Les règles versionnées produisent des alertes qui gardent la trace de leur définition et de leur cible.
5. Les interventions sont transactionnelles, attribuables et peuvent mobiliser plusieurs techniciens.
6. Les tableaux de bord lisent des index, projections ou agrégats, sans faire de ceux-ci une vérité parallèle.

Ce parcours préserve l’auditabilité et permet de reconstruire les lectures dérivées à partir des données de référence encore retenues.

## 3. Alternatives écartées

| Alternative | Motif de l’écart | Condition éventuelle de réexamen |
|---|---|---|
| **Base wide-column comme stockage natif des mesures dès la première version** | Le débit nominal est d’environ 20 mesures/s; PostgreSQL dispose déjà d’un partitionnement temporel, d’index capteur-temps, de contraintes et de transactions. Migrer les séries vers un autre moteur introduirait double écriture, cohérence distribuée, nouveaux outils de sauvegarde et compétences sans gain mesuré. | Évaluer si le benchmark à pleine rétention, avec lectures mixtes et pics de reconnexion, met en évidence des limites durables de capacité, disponibilité ou coût après optimisation du schéma et des index. |
| **Base clé-valeur comme autorité pour les mesures ou le référentiel** | Les relations, historiques, agrégations et contraintes n’y sont pas naturellement exprimés; la reconstruction et les transactions multi-entités seraient plus complexes. | Aucune comme source de vérité dans l’architecture actuelle; seulement projection de lecture non autoritative si un besoin de latence est mesuré. |
| **Base documentaire pour les configurations et règles dès le départ** | Les objets métier actuels ont une structure connue, des versions et des liens référentiels. Le MPD relationnel les traite; un attribut imbriqué réellement variable peut être accueilli en `JSONB` si nécessaire sans second moteur. | Réexaminer si les profils deviennent très hétérogènes, principalement traités comme documents autonomes et nécessitent une évolution de schéma fréquente que PostgreSQL ne couvre pas simplement. |
| **Base graphe pour les dépendances** | Le parc compte quelques centaines d’entités et les parcours connus sont peu profonds et temporels; les jointures PostgreSQL gardent intégrité et historique. | Évaluer si des traversées multi-sauts dynamiques (impact en cascade, topologie évolutive) deviennent fréquentes, essentielles et limitantes dans les mesures SQL. |
| **Dénormalisation massive du référentiel** | Copier noms, codes, bâtiments ou zones dans les mesures et les entités enfants gonflerait fortement le stockage et créerait des divergences. La cardinalité du référentiel ne démontre pas de goulot de jointure. | Ne considérer qu’une projection de lecture ciblée, séparée et reconstruisible, après benchmark démontrant un avantage concret. |
| **Stockage de la seule localisation ou association courante** | Écraser le rattachement antérieur empêcherait de restituer la situation au moment d’une mesure, d’une alerte ou d’une intervention. | Non retenu; toute projection courante doit coexister avec l’historique de référence et ne jamais le remplacer. |
| **Absence d’historisation temporelle** | Une relation courante sans dates rendrait les analyses historiques fausses après déplacement d’équipement ou réaffectation de capteur. | Écartée par principe métier; seules les modalités d’audit des corrections rétroactives restent à finaliser. |
| **Réplication synchrone et RPO nul imposés sans SLA** | La réplication synchrone ajoute latence et peut affecter la disponibilité lors d’une panne de nœud ou de réseau. Le niveau de perte toléré et le coût n’ont pas été approuvés. | Décision HA à prendre après arbitrage RPO/RTO, topologie et tests de bascule; ne pas revendiquer RPO 0 sans mécanisme et exercices démontrés. |
| **Cache ou agrégats pré-calculés partout par anticipation** | Toute projection multiplie les chemins d’écriture, le coût de réconciliation et les cas de retard/panne; sans bénéfice mesuré, elle crée de la complexité inutile. | Ajouter au cas par cas en réponse à un SLO approuvé et à un écart de benchmark reproductible. |

## 4. Risques techniques et opérationnels

| Risque | Effet possible | Prévention / surveillance | Décision ou preuve attendue |
|---|---|---|---|
| **Goulot d’étranglement PostgreSQL** (I/O, WAL, index, checkpoints, connexions, CPU, stockage) | Hausse des latences, retards d’ingestion, file qui croît ou indisponibilité. | Benchmark à paliers jusqu’au volume représentatif; réduire les index non utiles; mesurer WAL, I/O, checkpoints, vacuum, partitions, réplication et requêtes concurrentes. | Matériel cible et critères p95/p99, capacité, taux d’erreur et backlog approuvés avant production. |
| **Rafales de reconnexion** | La reprise produit une charge instantanée supérieure à la moyenne et pénalise les lectures. | Spool de passerelle, file durable, limitation de débit/back-pressure, consommateurs parallèles selon clé capteur et essais de backlog. | Durée de coupure à absorber, capacité de tampon et délai de rattrapage cibles. |
| **Dérive ou désordre des horloges** | Mesures dans la mauvaise période/partition, état courant qui régresse, alertes ou agrégats mal attribués. | Séparer événement/réception, surveiller leur écart, contrôler les dates hors plage, traiter l’état par horodatage d’événement et conserver les retards valides. | Tolérance d’horloge, mécanisme de synchronisation terrain et politique d’alerte sur données tardives. |
| **Déduplication insuffisamment spécifiée** | Mesures ou alertes dupliquées après perte d’accusé et rejeu; ou suppression indue de deux observations distinctes. | Définir identifiant source stable et portée d’unicité; inbox unique; consommateurs idempotents; corpus de test avec doublons et collisions. | Contrat de message passerelle et contrainte d’unicité finalisés. |
| **Purge des mesures référencées par les alertes** | Une FK vers une mesure déclencheuse peut empêcher détachement/suppression de partition après 12 mois; l’alerte peut perdre sa preuve si le lien est supprimé sans stratégie. | Définir les durées séparées mesures/alertes/audit; décider si l’alerte archive un instantané probant ou si la mesure reste conservée; tester la procédure de purge sur partitions. | Politique de rétention et de valeur probante validée avant activation de purge automatique. |
| **Maintenance des partitions** | Absence de partition future, échec d’insertion à la frontière de mois, croissance excessive, opérations de purge ou de sauvegarde imprévues. | Automatiser création anticipée, vérification de couverture, alerte de partition manquante, contrôle taille/index, archivage et suppression; tester les opérations. | Responsable, calendrier, horizon d’anticipation, procédure de secours et preuve de restauration. |
| **Projection ou cache en retard** | Dashboard affiche une valeur qui semble courante mais est périmée; divergences entre projection et table source. | Indiquer la fraîcheur, consommer via outbox, comparer timestamps d’événement, retry, réconciliation et fallback PostgreSQL. | Tolérance à la cohérence éventuelle et seuil de fraîcheur définis par le métier. |
| **RPO/RTO non établis** | La réplication, les sauvegardes et les procédures peuvent ne pas répondre aux attentes après perte de nœud ou de site. | Arbitrer SLA, sécuriser sauvegardes/WAL hors site, définir HA/DR et faire des exercices de restauration/failover. | Valeurs officielles et scénarios de reprise validés; résultats des exercices consignés. |
| **Estimation de capacité fondée sur une taille logique** | Taille réelle supérieure avec tuple, index, WAL, réplication, espace libre et sauvegardes. | Mesurer taille des lignes, partitions et index sur données représentatives; surveiller croissance et marge de capacité. | Budget disque fondé sur mesure et croissance observée, non sur une estimation brute seule. |
| **Goulot d’étranglement analytique** | Requêtes longues sur plusieurs capteurs et mois perturbent les tableaux de bord et l’ingestion. | Limiter fenêtres, utiliser plans d’exécution, agrégats ciblés, isolation de charge et quotas; tester lecture mixte. | SLO de lecture et jeu de rapports validés par les utilisateurs. |
| **Dépendance à la file et aux outils d’exploitation** | Le système devient plus complexe à opérer; backlog ou quarantaine peut saturer. | Choisir le mécanisme de file avec critères de durabilité et compétence; superviser lag, capacité, DLQ, partitions et rejeu; pratiquer la reprise. | Modèle opérationnel, rétention de file, procédures d’escalade et owners définis. |
| **Sécurité des accès IoT et services** | Message falsifié, exposition de données ou action non autorisée sur le référentiel/intervention. | Identités distinctes par passerelle, authentification forte, rotation de secrets, TLS, moindre privilège, audit et segmentation. | Politique d’identité, certificats, révocation, accès opérateurs et tests de sécurité validés. |

### 4.1 Risques de rétention et de partitionnement à traiter en priorité

La rétention de 12 mois porte sur les mesures brutes, mais les autres objets (alertes, interventions, historiques et audit) peuvent avoir une durée différente. Une alerte liée par FK à une mesure située dans une partition ancienne peut rendre impossible la suppression de cette partition. Il faut donc trancher avant l’automatisation :

1. quelle durée de preuve est exigée pour une alerte et son intervention;
2. si l’alerte conserve la valeur, l’unité, la qualité et l’horodatage pertinents comme instantané d’audit;
3. si la mesure source est gardée plus longtemps, archivée dans un stockage consultable, ou si la référence devient un pointeur d’audit sans FK physique après archivage;
4. comment une suppression de partition est vérifiée et journalisée.

La création des partitions à l’avance, leur monitoring et la purge par détachement sont des opérations de production à part entière. Elles doivent être idempotentes, auditées et testées sur une copie restaurable; elles ne se limitent pas à un `DROP` mensuel planifié.

## 5. Évolutions possibles et déclencheurs

Les évolutions sont conditionnelles à un besoin validé, aux mesures de l’étape 7 et à un coût d’exploitation acceptable.

| Évolution | Déclencheur mesurable | Conception cible |
|---|---|---|
| **Projection PostgreSQL de dernière mesure** | Requête indexée de dernière valeur ou agrégation des 600 capteurs ne satisfait pas le SLO approuvé dans des essais reproductibles. | Table `etat_courant_capteur`, une ligne par capteur, mise à jour atomique à l’ingestion; séparer dernière reçue et dernière valide; reconstruire depuis les mesures conservées. |
| **Cache clé-valeur externe** | La projection SQL reste insuffisante ou la charge de lecture doit être isolée, et un benchmark comparatif prouve un gain net. | Cache de lecture dérivé alimenté par outbox, idempotent, borné et observable; repli PostgreSQL; pas d’écriture métier exclusive dans le cache. |
| **Agrégats horaires/journaliers** | Rapports énergétiques fréquents scannent trop de mesures et les plans/résultats dépassent les objectifs validés. | Tables agrégées par capteur/période, puis par bâtiment ou zone lorsque l’attribution temporelle et les unités sont stables; recalcul des fenêtres tardives. |
| **Moteur analytique dédié** | Volume historique, concurrence analytique, coût de scan ou rétention analytique dépassent durablement PostgreSQL après index/partitions/agrégats. | Alimentation asynchrone depuis outbox/ETL, schéma analytique distinct, fraîcheur et stratégie de correction documentées; PostgreSQL reste maître du référentiel transactionnel. |
| **Moteur wide-column pour les séries** | Test pleine échelle et charge cible montrent des limites durables de capacité/coût/disponibilité et un modèle de requête par clé temporelle bien établi. | Partitionner par capteur et fenêtre; plan de double lecture/migration, replay, rétention, cohérence et réconciliation; décision fondée sur un benchmark comparatif de coût complet. |
| **Base graphe** | Parcours multi-sauts complexes et dynamiques deviennent une fonctionnalité fréquente dont le coût relationnel mesuré est excessif. | Représentation de graphe alimentée depuis PostgreSQL, avec temporalité des relations, synchronisation et reconstruction documentées. |
| **Automatisation du cycle de vie des partitions** | Avant la mise en production à pleine rétention; requis pour éviter l’oubli de partitions et rendre la purge répétable. | Tâche supervisée qui crée les partitions futures, vérifie la couverture, mesure tailles, alerte sur anomalies et détache/archive selon une politique approuvée. |
| **Extension multi-campus** | Le périmètre s’étend à plusieurs campus avec besoins de séparation d’accès, administration ou déploiement. | Référentiel déjà modélisé avec `Campus`; préciser tenants, périmètres, unicité des codes, délégation d’administration, résidence et éventuel partitionnement logique. |
| **Haute disponibilité / reprise après sinistre** | RPO/RTO, SLA et budget ont été approuvés. | Choisir réplication synchrone/asynchrone, sauvegarde continue, site secondaire, procédures de fencing/failover et exercices; ne pas confondre réplication et sauvegarde. |

### 5.1 Feuille de route de mise en œuvre

1. **Valider les règles métier ouvertes** : durées de conservation, unité/qualité, événements tardifs, clé de déduplication, transitions d’alerte/intervention, campus et permissions.
2. **Finaliser l’exploitation PostgreSQL** : taille réelle, index, disponibilité des partitions futures, sauvegardes/WAL, supervision, contrôle des connexions et tests de restauration.
3. **Définir les contrats d’ingestion** : message source, accusé durable, inbox/outbox, back-pressure, quarantaine, capacité de spool et procédure de rattrapage.
4. **Réaliser les campagnes de benchmark** selon le protocole d’étape 7 sur plusieurs paliers et en charge mixte; faire approuver les critères d’acceptation avant le test de qualification.
5. **Activer uniquement les projections utiles** : commencer par une projection relationnelle d’état courant si elle apporte un bénéfice mesuré; envisager le cache ou l’analytique dédiés seulement ensuite.
6. **Automatiser et exercer** le cycle de partitionnement, la rétention, les reprises de file, les bascules et les restaurations.

## 6. Gouvernance des décisions

Pour préserver la cohérence du dossier et éviter que les choix conditionnels soient interprétés comme déjà déployés :

- tenir un registre des décisions d’architecture avec état `retenu`, `à valider`, `conditionnel` ou `écarté`;
- associer à chaque évolution un besoin, un indicateur déclencheur, un propriétaire, un coût et un plan de retour arrière;
- versionner les schémas, règles d’alerte, contrats d’événements, partitions et scripts d’exploitation;
- conserver le manifeste des benchmarks et relier toute conclusion de capacité au matériel, au schéma et au dataset testés;
- réexaminer les décisions après évolution substantielle des volumes, des SLA, des campus, des utilisateurs ou des requêtes;
- traiter une projection ou un moteur secondaire comme une représentation dérivée avec un contrat de cohérence et de reconstruction.

## 7. Conclusion

SmartCampus n’a pas besoin d’une architecture polyglotte complexe pour être évolutive. Il lui faut d’abord une source de vérité fiable, un modèle temporel juste, une ingestion qui tolère les interruptions et un système d’exploitation vérifiable. PostgreSQL fournit un socle cohérent au périmètre actuel; la normalisation préserve l’intégrité, le partitionnement et les index ciblent les séries temporelles, et les tampons durables réduisent la dépendance à la disponibilité instantanée du réseau et des consommateurs.

La prudence porte moins sur le choix d’un moteur supplémentaire que sur la maîtrise des garanties : mesure réelle des performances, identifiants idempotents, dérive d’horloge, rétention des données référencées, cycle de vie des partitions et objectifs de reprise explicitement approuvés. Les caches, agrégats, moteurs analytiques ou NoSQL ne sont pas écartés par principe; ils sont différés jusqu’à ce qu’un besoin métier et des mesures reproductibles en démontrent la valeur nette.

La trajectoire recommandée est donc progressive : **sécuriser les règles et l’exploitation, benchmarker le socle complet, automatiser la rétention et la reprise, puis n’ajouter une projection ou un moteur que pour résoudre une limite observée**. Cette approche limite le risque opérationnel tout en maintenant des voies d’évolution vers le multi-campus, des volumes plus élevés et des usages analytiques plus ambitieux.

