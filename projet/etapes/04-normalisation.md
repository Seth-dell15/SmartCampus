# SmartCampus — Étape 4 : normalisation et dénormalisations maîtrisées

## 1. Objet et principes

Cette étape évalue le schéma relationnel du MLD/MPD SmartCampus sous l’angle de la normalisation, identifie les coûts de lecture liés aux mesures IoT et aux jointures temporelles, puis propose des projections de lecture uniquement lorsqu’elles répondent à un besoin mesurable.

Le principe retenu est de conserver le **modèle relationnel normalisé comme source de vérité**. Une donnée calculée ou dupliquée pour accélérer une lecture doit être isolée, identifiable comme projection, reconstruisible depuis les données sources et mise à jour par un mécanisme explicite. Les tables de projection ne remplacent ni le référentiel normalisé ni l’historique.

Le dimensionnement de référence est d’environ 8 bâtiments, 160 zones, 250 équipements, 600 capteurs et 1 mesure toutes les 30 secondes par capteur, soit environ 1,728 million de mesures par jour. Les tableaux de bord sont actualisés toutes les 5 minutes. Ces volumes rendent l’accès aux séries temporelles et à l’état courant plus sensible que les jointures du référentiel, de faible cardinalité.

## 2. Évaluation de la normalisation

### 2.1 Référentiel spatial et parc technique

Le référentiel est décomposé selon les dépendances fonctionnelles métier :

- `campus` porte les attributs du campus; `id_campus` et `code_campus` identifient chacun une occurrence.
- `batiment` porte ses propres attributs et sa référence à un campus; `id_batiment` et `code_batiment` sont des identifiants candidats.
- `zone` porte les attributs de la zone et le rattachement au bâtiment; l’unicité métier de `code_zone` est limitée au bâtiment, d’où la clé candidate `(id_batiment, code_zone)`.
- `equipement` porte les propriétés propres de l’actif; ses rattachements spatiaux et aux capteurs sont séparés et temporalisés.
- `capteur` porte les propriétés stables d’identification et de configuration nominale; ses associations d’équipement et les mesures sont ailleurs.

Les clés primaires et clés candidates déterminent les attributs non-clés de ces tables. Les dépendances transitives sont évitées : par exemple, les informations du campus ne sont pas répétées dans `batiment` ou `zone`, et les informations du bâtiment ne sont pas recopiées dans `zone`. Sous réserve que les contraintes d’unicité déclarées correspondent bien aux règles métier (notamment le périmètre des codes), le référentiel satisfait la **3e forme normale (3NF)** et la **forme normale de Boyce-Codd (BCNF)**.

Cette conclusion repose sur les dépendances métier retenues, pas seulement sur les noms des tables. Si un attribut non clé est déterminé par un autre attribut non clé — par exemple un libellé de type déterminé par `type_equipement` — il devra être remplacé par une référence à un dictionnaire plutôt que recopié dans chaque équipement.

### 2.2 Associations et tables d’historisation

`historique_zone_equipement` et `affectation_capteur_equipement` modélisent des associations plusieurs-à-plusieurs dans le temps. Chaque occurrence est identifiée par son identifiant technique, référence une occurrence de chaque entité participante et porte ses propres dates d’effet et motif.

Ces tables séparées ne constituent pas une dénormalisation : elles évitent d’écraser une relation passée ou de répéter plusieurs valeurs dans une colonne de `equipement` ou `capteur`. Les contraintes d’exclusion temporelle interdisent les périodes qui se chevauchent pour un équipement ou un capteur, et les clés étrangères préservent l’intégrité du référentiel. La date d’événement d’une mesure permet de retrouver son équipement et sa zone historique sans figer un rattachement courant dans chaque mesure.

L’association `affectation_technicien_intervention` répond de même à la participation plusieurs-à-plusieurs; une ligne représente une période et un rôle éventuel. Elle évite une liste de techniciens sérialisée dans `intervention`.

### 2.3 Mesures et relations métier

`mesure` est une ligne atomique d’observation avec un seul capteur, un horodatage d’événement, un horodatage de réception, une valeur, une unité et une qualité. Ces champs décrivent une observation et ne sont pas des listes ni des groupes répétitifs. Les partitions mensuelles et index physiques ne modifient pas cette normalisation logique.

Le lien à `capteur` est explicite plutôt que de répéter le type, le libellé ou l’unité nominale du capteur dans chaque ligne de mesure. L’unité effective (`unite`) est néanmoins conservée sur la mesure : elle décrit la valeur réellement reçue et permet d’interpréter l’historique si la configuration nominale évolue.

`alerte` référence la version de règle appliquée et enregistre la sévérité constatée. Cette sévérité peut correspondre à la sévérité de la règle lors du déclenchement, mais elle est conservée comme **instantané métier** : une modification future de règle ne doit pas réécrire l’interprétation d’une alerte passée. C’est une redondance sémantique intentionnelle, pas une anomalie de normalisation.

### 2.4 Limite propre à la table partitionnée des mesures

Dans PostgreSQL, la clé primaire de `mesure` est `(horodatage_evenement, id_mesure)` afin d’inclure la colonne de partitionnement. Le MCD présente `id_mesure` comme identifiant conceptuel; dans le MPD, l’identité globale est générée par séquence, mais l’unicité déclarative est garantie par la PK composite.

Pour les références vers une mesure, le MPD transporte donc les deux attributs (`horodatage_evenement`, `id_mesure`). Toute extension qui dépend d’une unicité globale strictement contrôlée de `id_mesure` devra expliciter cette contrainte ou conserver la clé composite. Cette nuance physique ne justifie pas la duplication de données descriptives dans la mesure.

## 3. Redondances possibles et coûts de lecture

| Besoin / zone | Coût sans projection | Risque de redondance | Évaluation |
|---|---|---|---|
| Dernière mesure d’un capteur | Recherche dans les mesures récentes, filtrage capteur/qualité, classement temporel. | Copier valeur, date, qualité dans `capteur` crée deux vérités et augmente les écritures concurrentes. | Besoin concret pour l’état courant et les tableaux de bord; une projection dédiée peut être justifiée si la requête indexée ne tient pas le budget de latence. |
| État de santé d’un équipement | Combinaison de mesures récentes de ses capteurs, qualité/obsolescence, alertes ouvertes et éventuelles interventions. | Ajouter un statut de santé permanent à `equipement` confond état du référentiel (en service/retiré) et état opérationnel calculé. | Calcul potentiellement coûteux et règle métier non entièrement spécifiée; projection dérivée envisageable après définition de la règle et mesure des performances. |
| Emplacement courant | Recherche de la période ouverte dans `historique_zone_equipement`, puis jointure vers zone et bâtiment. | Ajouter `id_zone_courante` dans `equipement` duplique l’association historisée et peut diverger lors d’un déplacement. | Avec quelques centaines d’équipements, la jointure est modeste; ne pas dupliquer dans le modèle de référence. |
| Lien mesure–équipement–zone | Résolution des affectations temporelles, puis historique de zone à la date d’événement. | Copier équipement et zone dans chaque mesure crée jusqu’à des centaines de millions de références redondantes et peut figer un rattachement erroné. | Le coût peut être élevé pour des agrégations historiques; déplacer cette résolution dans des agrégats temporels contrôlés, pas dans chaque mesure brute par défaut. |
| Agrégats de consommation | Somme/moyenne/compte sur un grand volume de mesures avec conversions et qualité. | Agrégats recalculés ou modifiés sans gestion des arrivées tardives, corrections et changements d’unité. | Candidat fort à matérialisation pour les tableaux de bord et rapports, à condition de fixer métriques, granularité et règles de correction. |
| Détail référentiel dans les listes | Jointures bâtiment/zone/équipement/capteur selon l’écran. | Répéter noms et codes d’entités parentes dans chaque table ou mesure. | Le référentiel est petit; les index de clés étrangères et des requêtes ciblées suffisent normalement. Pas de dénormalisation requise. |
| Compteurs d’alertes ouvertes ou interventions actives | `COUNT` regroupé par cible/statut. | Compteur persisté invalidé par chaque transition ou suppression logique. | À envisager seulement si le volume et la fréquence de lecture le justifient; les index partiels peuvent suffire. |

Un coût de jointure n’est pas à lui seul un motif de dénormalisation. Les cardinalités de campus, bâtiments, zones, équipements et capteurs sont faibles à l’échelle de l’application; ces relations doivent rester propres et normalisées tant qu’une mesure de charge ne démontre pas un problème.

## 4. Dénormalisations contrôlées proposées

### 4.1 Projection de dernière mesure par capteur

**Proposition.** Ajouter une table de lecture `etat_courant_capteur`, distincte du référentiel `capteur`, contenant une ligne par capteur et les pointeurs vers la dernière mesure reçue et, séparément, la dernière mesure valide :

| Élément projeté | Utilité |
|---|---|
| `id_capteur` (PK/FK) | Une entrée de projection par capteur. |
| `id_mesure_derniere`, `horodatage_derniere` | Accès direct à la dernière observation reçue, même si sa qualité est invalide. |
| `id_mesure_valide`, `horodatage_valide` | Accès direct à la dernière observation considérée valide. |
| `valeur_valide`, `unite_valide`, `qualite_valide` | Restitution de l’état affichable sans reparcourir l’historique. |
| `date_calcul`, `version_regle_etat` | Traçabilité de l’actualisation et de la règle de validité appliquée. |

**Bénéfice attendu.** Les écrans d’état courant évitent une recherche par capteur dans les partitions historiques et peuvent charger les quelques centaines de capteurs par une lecture compacte. Ceci correspond au rafraîchissement des tableaux de bord toutes les 5 minutes.

**Coût et cohérence.** La projection est dérivée, jamais éditée directement par les utilisateurs. Le service d’ingestion met à jour la mesure brute et la projection dans la même transaction, ou écrit un événement durable consommé par un processeur idempotent avec reprise et réconciliation. Il compare les horodatages d’événement : une mesure arrivée en retard ne remplace pas la dernière mesure par simple ordre de réception. Il distingue explicitement dernière reçue et dernière valide selon RG-07, encore à valider.

Il faut décider si la projection conserve une FK composite vers `mesure`. Une telle FK assure la traçabilité, mais peut bloquer la purge des mesures après 12 mois. Alternatives : ne conserver que les valeurs projetées et leurs métadonnées, ou conserver un identifiant source sans FK comme référence d’audit; le choix dépend de la politique de rétention. Dans tous les cas, la projection reste reconstruisible depuis les mesures encore conservées.

### 4.2 État opérationnel calculé des équipements

**Proposition conditionnelle.** Si l’écran principal demande un état santé immédiat par équipement, matérialiser une vue `etat_courant_equipement` contenant :

- `id_equipement`;
- `etat_sante_calcule` (par exemple normal, à surveiller, en défaut, inconnu);
- `date_evaluation`;
- `motif_principal` ou la référence de l’alerte la plus importante;
- éventuellement `date_derniere_mesure_valide` et l’indicateur de fraîcheur.

Cet état ne remplace **pas** `equipement.statut`, qui décrit le cycle de vie administratif/technique (en service, hors service, retiré). Il est recalculé à partir de la fraîcheur et de la qualité des capteurs affectés, des alertes non closes et des règles approuvées.

**Cohérence.** Le service de supervision recalcule la projection à la réception d’une mesure, au déclenchement/acquittement/clôture d’une alerte, à une modification d’affectation ou au franchissement d’un seuil d’obsolescence. Un recalcul périodique complet sert de contrôle/réconciliation. Le calcul est idempotent et indique la date d’évaluation pour ne pas afficher un état périmé comme courant.

**Décision.** À ne pas construire avant validation de la règle métier d’état de santé et mesure d’un besoin de latence; les quelque 250 équipements permettent potentiellement de calculer à la demande à partir de l’état courant des capteurs et alertes.

### 4.3 Agrégats de consommation par période

**Proposition.** Produire des agrégats horaires et/ou journaliers par capteur et grandeur, puis, lorsque les règles d’attribution temporelle sont stables, par équipement, zone ou bâtiment. Une granularité et des mesures précises sont choisies en fonction des scénarios réels : énergie consommée, minimum, maximum, moyenne, nombre de mesures valides et indicateur de complétude.

Schéma indicatif :

```text
AGREGAT_MESURE(
  PK (debut_periode, granularite, id_capteur),
  fin_periode,
  somme, minimum, maximum, moyenne,
  nombre_mesures, nombre_valides, nombre_invalides,
  date_calcul, version_calcul
)
```

Les agrégats de capteur sont stables par rapport aux déplacements d’équipement; l’agrégation par zone ou bâtiment dépend de l’historique de localisation et de la règle d’allocation de consommation. Le calcul doit joindre la période de mesure à l’affectation capteur-équipement et à la localisation applicable au même instant, ou conserver explicitement la version/référence de cette attribution.

**Cohérence.** Les fenêtres clôturées sont calculées de façon idempotente et recalculables. Une fenêtre récente reste révisable pour accepter les messages tardifs; la durée de correction et la stratégie de reprise doivent être définies. Une matérialized view PostgreSQL est adaptée à un rafraîchissement complet ou périodique de volume modéré; à l’échelle de centaines de millions de lignes, une table d’agrégats alimentée par lots ou flux, avec contrôles de rapprochement, sera généralement plus contrôlable qu’un rafraîchissement complet fréquent.

**Compromis.** La lecture par période et bâtiment devient rapide et réduit le scan de mesures; en contrepartie, il faut maintenir les agrégats, leur version de calcul et les corrections tardives. Les agrégats ne remplacent pas les mesures brutes retenues 12 mois.

### 4.4 Données volontairement non dupliquées

- **Pas de `zone_courante` persistée dans `equipement` à ce stade.** La cardinalité du parc est faible; l’historique et ses contraintes restent la source de vérité.
- **Pas de nom de bâtiment, zone ou équipement dans chaque ligne `mesure`.** Le coût de stockage serait disproportionné et les associations historiques sont temporelles.
- **Pas de `sante_equipement` dans `equipement`.** Ce champ mélangerait le cycle de vie du référentiel et l’état dérivé, sauf s’il s’agit d’une projection explicitement séparée.
- **Pas de copie systématique des attributs de règle dans `alerte`.** La version référencée explique le seuil et l’opérateur appliqués; seule la sévérité constatée est conservée comme instantané d’interprétation historique.
- **Pas de compteur d’alertes ou d’interventions maintenu à l’écriture** sans requête démontrant que les index et agrégations ordinaires ne répondent pas aux objectifs.

## 5. Mise en œuvre et gouvernance de cohérence

### 5.1 Source de vérité et modes d’alimentation

| Donnée | Source de vérité | Projection éventuelle | Mise à jour recommandée |
|---|---|---|---|
| Référentiel campus/bâtiments/zones/équipements/capteurs | Tables normalisées | Aucune par défaut | Transaction de gestion référentielle; invalidation de cache d’application si besoin. |
| Historique de localisation et d’affectation | Tables temporelles normalisées | État courant calculé pour les écrans seulement | Transaction atomique; contraintes d’exclusion en base. |
| Mesures brutes | `mesure` partitionnée | `etat_courant_capteur`, agrégats | Ingestion transactionnelle ou flux durable idempotent, avec surveillance du retard. |
| Santé équipement | Mesures, affectations, alertes et règles | `etat_courant_equipement` si justifié | Recalcul événementiel des sources concernées et réconciliation périodique. |
| Agrégats de consommation | `mesure` et règles d’attribution | `agregat_mesure` | Calcul incrémental par période, recalcul des fenêtres tardives, audit des écarts. |

### 5.2 Garanties de cohérence

1. Une projection expose sa date de calcul et, lorsque le résultat dépend d’une règle, la version de cette règle.
2. Les mises à jour sont idempotentes : retraiter le même message ne doit ni doubler une mesure, ni avancer artificiellement la dernière mesure.
3. Les données en retard sont évaluées selon `horodatage_evenement`, pas uniquement selon `horodatage_reception`.
4. Les projections sont reconstruisibles ou réparables à partir des sources retenues; des contrôles périodiques comparent projection et source.
5. Si un déclencheur SQL est utilisé, sa logique reste courte, déterministe, transactionnelle et testée sous concurrence. Les calculs inter-tables volumineux et l’agrégation analytique relèvent plutôt d’un service ou d’un traitement dédié.
6. La suppression des partitions de mesures tient compte des projections et alertes qui les référencent. Une FK ne doit pas être supprimée ou affaiblie simplement pour faciliter la purge sans décision de rétention et de traçabilité documentée.
7. Les projections sont séparées des tables normalisées et clairement nommées; les consommateurs en lecture seule n’écrivent pas directement dedans.

## 6. Bilan des choix pour SmartCampus

| Choix | Décision | Motif métier / technique | Risque maîtrisé |
|---|---|---|---|
| Référentiel spatial et parc en 3NF/BCNF | **Conserver normalisé** | Peu de lignes; écritures administratives rares; intégrité et absence de divergences prioritaires. | Pas de duplication des noms, codes ou attributs parents. |
| Historique des emplacements et affectations | **Conserver séparé et temporel** | Reconstituer la situation réelle à la date d’une mesure ou intervention. | Pas d’écrasement de l’état passé ni de rattachement courant appliqué rétroactivement. |
| Mesures brutes | **Conserver atomiques, partitionnées** | Nécessaires au diagnostic et à l’historique de 12 mois; index temporels adaptés. | Pas de données descriptives répétées dans plusieurs centaines de millions de lignes. |
| Dernière mesure par capteur | **Projection recommandée si nécessaire, après mesure** | Consultation rapide des 600 capteurs lors des tableaux de bord réactualisés toutes les 5 minutes. | Écriture idempotente, séparation dernière reçue/dernière valide, reconstruction possible. |
| Santé d’équipement | **Projection conditionnelle** | Peut simplifier l’écran de supervision si le calcul multi-sources est trop lent. | Ne pas confondre état de santé calculé et statut de cycle de vie; définir la règle avant implémentation. |
| Agrégats de consommation | **Pré-calcul ciblé** | Évite les scans répétés de périodes longues pour l’analyse énergétique et les tableaux de bord. | Règles d’unité, qualité, attribution temporelle, retard et recalcul documentés. |
| Noms et localisation copiés dans les mesures | **Rejeté** | Forte redondance et risque d’historique faux; le coût de stockage est majeur à l’échelle annuelle. | Jointures ou agrégats temporels plutôt que vérité dupliquée. |

La dénormalisation n’est donc pas appliquée au modèle transactionnel de référence. Elle est réservée à des projections de lecture répondant à un besoin observable : l’état courant à faible latence et les agrégats analytiques sur de longues périodes. Les choix restent conditionnés par des mesures de performance, des objectifs de latence validés et des tests de cohérence sous arrivées tardives.

## 7. Critères de validation avant déploiement

1. Mesurer les requêtes des tableaux de bord avec le volume attendu et l’actualisation de cinq minutes; établir un budget de latence et la concurrence réelle des 50 utilisateurs actifs.
2. Comparer les requêtes sur données normalisées (index et partitions inclus) avec la projection de dernière mesure; ne retenir la projection que si elle apporte un bénéfice significatif et reproductible.
3. Définir les règles de santé, d’obsolescence, d’unités, de qualité et de correction des mesures avant de matérialiser leur résultat.
4. Tester les arrivées désordonnées, répétitions, reconnexions, mouvements d’équipements et affectations de capteurs aux bornes des périodes.
5. Définir la rétention des alertes et des mesures déclencheuses afin de ne pas rendre impossible l’archivage des partitions.
6. Vérifier la réconciliation automatique des projections et le comportement de reprise après arrêt de l’ingestion ou du service d’agrégation.

