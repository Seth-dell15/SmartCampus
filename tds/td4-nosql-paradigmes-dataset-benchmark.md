# TD 4 — Paradigmes NoSQL, dataset et protocole de benchmark

## 1. Objet et périmètre

Ce document étudie l'adéquation de quatre paradigmes NoSQL aux besoins de SmartCampus : base orientée documents, clé-valeur, wide-column et graphe. Il définit ensuite un dataset de référence et un protocole reproductible pour comparer une solution relationnelle à des solutions NoSQL sur des accès représentatifs.

Les technologies citées sont des candidates à évaluer, et non des choix déjà validés. PostgreSQL demeure la source de vérité relationnelle du projet; un moteur spécialisé ne serait ajouté que si un besoin mesuré le justifie. Les volumes, cadences et durées ci-dessous sont des hypothèses de dimensionnement, pas des mesures de production ni des résultats de benchmark.

## 2. Scénarios NoSQL

### 2.1 Scénario A — Base orientée documents : fiche équipement complète

**Besoin.** Consulter rapidement une fiche d'équipement qui rassemble ses caractéristiques et les informations courantes utiles à l'interface.

**Document principal proposé.** Un document par équipement peut contenir :

- les attributs relativement stables : identifiant, nom, type, marque, référence constructeur, date d'installation et statut;
- une configuration propre à l'équipement, sous forme d'objet structuré;
- un résumé de localisation courante (identifiants et libellés de bâtiment et de zone) pour la lecture, en le considérant comme une projection et non comme la source de vérité;
- une liste bornée de capteurs associés, avec leurs identifiants, types, unités et états utiles à la fiche.

Exemple de forme logique :

```json
{
  "_id": "EQ-0250",
  "nom": "CTA-Est-01",
  "type": "centrale_traitement_air",
  "marque": "Exemple",
  "reference_constructeur": "CTA-450",
  "date_installation": "2024-06-15",
  "statut": "en_service",
  "configuration": {
    "puissance_nominale_kw": 18.5,
    "parametres": {}
  },
  "localisation_courante": {
    "id_batiment": "BAT-02",
    "nom_batiment": "Bâtiment Est",
    "id_zone": "Z-021"
  },
  "capteurs": [
    {
      "id_capteur": "CAP-0101",
      "type": "temperature",
      "unite": "degC",
      "actif": true
    }
  ]
}
```

Les propriétés imbriquées sont adaptées aux données consultées avec l'équipement et dont le nombre reste raisonnablement borné. Les mesures historiques, les interventions et les autres historiques non bornés doivent rester dans des collections ou tables séparées, indexées par équipement et période. Les embarquer dans la fiche entraînerait une croissance continue du document, des écritures coûteuses et des limites pratiques de taille.

**Avantages et risques.** La lecture de la fiche peut se faire en une requête avec peu de jointures. En contrepartie, les libellés de bâtiment ou de zone et les résumés de capteurs peuvent être dupliqués. Une modification de référentiel doit donc être propagée, idéalement par événements/outbox et traitement idempotent, avec surveillance et possibilité de reconstruire la projection. Une architecture document doit également définir clairement quelle source fait autorité pour chaque attribut.

### 2.2 Scénario B — Base clé-valeur : dernière valeur d'un capteur

**Besoin.** Lire très rapidement la valeur la plus récente reçue pour un capteur, sans parcourir l'historique des mesures.

**Convention de clé :**

```text
smartcampus:v1:capteur:{id_capteur}:latest
```

Exemple : `smartcampus:v1:capteur:CAP-0101:latest`.

**Valeur logique proposée :**

```json
{
  "id_capteur": "CAP-0101",
  "valeur": 21.7,
  "unite": "degC",
  "horodatage_mesure_utc": "2026-11-10T14:32:00Z",
  "horodatage_reception_utc": "2026-11-10T14:32:01Z",
  "statut": "valide",
  "qualite": "bonne",
  "id_mesure": "mes-8f12"
}
```

La mise à jour doit être atomique et conditionnée à l'ordre des mesures : remplacer la valeur courante uniquement si l'horodatage de la nouvelle mesure est plus récent. En cas d'horodatage identique, un numéro de séquence ou un identifiant de mesure stable sert de critère de départage. Une mesure reçue en retard peut être enregistrée dans l'historique sans faire régresser l'état courant. Le mécanisme doit être idempotent afin que le rejeu d'un événement ne produise pas d'effet incorrect.

Un TTL n'est pas obligatoire pour une donnée dont l'objectif est de représenter le dernier état. Si un TTL est utilisé pour libérer le cache, l'absence de clé ne doit pas être interprétée comme une valeur nulle ou comme une preuve que le capteur n'a jamais transmis. La fraîcheur doit être évaluée à partir de l'horodatage et d'un seuil métier, avec un état « périmé » ou « indisponible » explicite. Le cache ne remplace jamais l'historique durable des mesures.

### 2.3 Scénario C — Base wide-column : historique des mesures

**Besoin.** Ingestion soutenue et lecture de plages temporelles pour un capteur, sans créer de partitions démesurées.

**Clé de partitionnement et tri proposés :**

```text
Partition : (id_capteur, jour_utc)
Clustering : horodatage_evenement DESC, id_mesure ASC
```

La clé de partition contient le capteur et le jour UTC afin de répartir les données dans le temps et de limiter la taille de chaque partition. Exemple de forme Cassandra/CQL :

```sql
CREATE TABLE mesures_par_capteur_jour (
    id_capteur text,
    jour_utc date,
    horodatage_evenement timestamp,
    id_mesure uuid,
    valeur double,
    unite text,
    qualite text,
    PRIMARY KEY ((id_capteur, jour_utc), horodatage_evenement, id_mesure)
) WITH CLUSTERING ORDER BY (horodatage_evenement DESC, id_mesure ASC);
```

À titre de dimensionnement, une journée représente environ 2 880 lignes par capteur à une cadence de 30 secondes, ou 8 640 lignes à une cadence de 10 secondes. Le volume en octets dépend du format, des métadonnées et du moteur; il doit être mesuré avec le schéma et la charge réels. Si les partitions dépassent la cible de taille définie lors des essais, le bucket peut être réduit (par exemple à l'heure) ou le modèle revu. Le choix du jour reste une hypothèse initiale à valider avec les tailles de lignes et les parcours de requêtes.

La rétention cible est de 12 mois. Une stratégie TTL doit être calculée selon l'âge événementiel de la mesure; un TTL par défaut calculé depuis l'heure d'insertion ne garantit pas à lui seul une conservation exacte de 12 mois en présence de mesures retardées. Dans Cassandra et les moteurs compatibles, l'expiration par TTL et les suppressions produisent des *tombstones* (marques de suppression) qui restent nécessaires jusqu'à ce qu'ils puissent être purgés sans risque pour la cohérence entre réplicas. Sur des centaines de millions de lignes, leur accumulation peut augmenter le travail de lecture et de compactage, dégrader les latences et accroître temporairement les entrées-sorties. Il faut donc éviter les purges massives mal planifiées, suivre le volume de tombstones et coordonner la rétention, les réparations/réplications et les paramètres de purge. Une stratégie de compactage orientée séries temporelles, telle que TWCS (*Time Window Compaction Strategy*) lorsqu'elle est prise en charge par le moteur et sa version, regroupe les données par fenêtres temporelles et peut faciliter l'expiration des fichiers devenus entièrement périmés. Elle ne supprime toutefois pas à elle seule tous les coûts des tombstones : ses fenêtres, le TTL et le cycle de réparation doivent être configurés et validés ensemble par mesure.

### 2.4 Scénario D — Base graphe : dépendances entre équipements

**Besoin.** Explorer des relations multi-niveaux et comprendre l'impact d'une panne ou d'une alerte au-delà des jointures directes.

**Types de nœuds possibles :** `Campus`, `Batiment`, `Zone`, `Equipement`, `Capteur`, `CircuitElectrique`, `InstallationCentrale`, `Alerte`, `Intervention` et `Technicien`.

**Relations principales :**

- `(Campus)-[:CONTIENT]->(Batiment)`
- `(Batiment)-[:CONTIENT]->(Zone)`
- `(Zone)-[:ACCUEILLE]->(Equipement)`
- `(Equipement)-[:POSSEDE_CAPTEUR]->(Capteur)` ou `(Capteur)-[:SURVEILLE]->(Equipement)`, selon le sens retenu pour les parcours;
- `(CircuitElectrique)-[:ALIMENTE]->(Equipement)`
- `(Equipement)-[:DEPEND_DE]->(Equipement|CircuitElectrique|InstallationCentrale)`
- `(Zone)-[:DEPEND_DE]->(Equipement|InstallationCentrale)`
- `(Alerte)-[:CIBLE_EQUIPEMENT|CIBLE_CAPTEUR]->(Equipement|Capteur)`
- `(Intervention)-[:TRAITE]->(Alerte)` et `(Technicien)-[:REALISE]->(Intervention)`.

La relation `DEPEND_DE` est orientée de l'entité dépendante vers sa dépendance : par exemple, une zone pointe vers l'installation centrale dont elle dépend. Les dépendances temporelles ou changeantes doivent porter des propriétés de validité (`valide_depuis`, `valide_jusqua`) ou être historisées afin de ne pas confondre la topologie actuelle avec celle qui existait à une date passée. Il faut également prévenir les cycles non voulus dans les dépendances ou limiter les parcours en profondeur.

**Parcours de recherche pertinents :**

1. **Quels équipements seront impactés par la panne d'un circuit électrique ?** Le graphe suit d'abord les relations `ALIMENTE` depuis le circuit vers les équipements directement alimentés, puis remonte les relations `DEPEND_DE` depuis les équipements qui dépendent de ces équipements. Par exemple, le motif Cypher suivant renvoie les équipements directement alimentés et leurs dépendants transitifs :

   ```cypher
   MATCH (c:CircuitElectrique {id: $idCircuit})-[:ALIMENTE]->(alimente:Equipement)
   MATCH (impacte:Equipement)-[:DEPEND_DE*0..]->(alimente)
   RETURN DISTINCT impacte
   ```

   Dans un modèle relationnel, le même résultat est calculable, mais demande généralement une CTE récursive sur les dépendances, puis des jointures avec les circuits et les équipements; le graphe exprime directement le parcours et peut aussi retourner le chemin expliquant l'impact.

2. **Quelles zones dépendent d'une installation centrale ?** Le parcours part de l'installation centrale et suit à rebours les relations `DEPEND_DE`, orientées de la zone dépendante vers sa dépendance. Cela permet de retrouver les zones qui en dépendent directement ou à travers un équipement intermédiaire :

   ```cypher
   MATCH (zone:Zone)-[:DEPEND_DE*1..]->(i:InstallationCentrale {id: $idInstallation})
   RETURN DISTINCT zone
   ```

   Dans une base relationnelle, il faudrait parcourir récursivement la table des dépendances et joindre les identifiants obtenus aux tables des zones, équipements et installations. Le graphe rend la dépendance transitive explicite dans le modèle et évite de coder une succession de jointures auto-référentes pour chaque profondeur.

3. **Contexte d'une alerte :** remonter de l'alerte au capteur ou à l'équipement, puis à la zone et au bâtiment, et retrouver les interventions antérieures pertinentes.
4. **Dépendances communes :** partir d'un équipement et identifier les circuits, équipements ou autres composants partagés susceptibles d'expliquer plusieurs alertes simultanées.

Le graphe est particulièrement utile pour ces parcours relationnels; il n'est pas nécessairement le meilleur stockage pour les mesures brutes ou les simples lectures de fiches.

## 3. Comparaison des paradigmes

| Critère | Document | Clé-valeur | Wide-column | Graphe |
|---|---|---|---|---|
| **Structure des données** | Documents autonomes JSON/BSON, éventuellement imbriqués | Valeur opaque adressée par une clé unique | Lignes regroupées par clé de partition et ordonnées par colonnes de clustering | Nœuds et relations typés, tous deux porteurs de propriétés |
| **Type d'accès privilégié** | Lecture/écriture d'un agrégat fonctionnel | Accès direct par clé connue | Écritures et lectures de plages dans des partitions préconçues | Parcours de relations et recherche de chemins |
| **Avantage principal** | Rassemble les données nécessaires à une fiche en peu de lectures | Très faible coût d'accès à un état connu par clé | Débit et distribution adaptés aux séries temporelles | Parcours multi-niveaux naturels pour dépendances et impacts |
| **Limite principale** | Duplication, taille des documents et cohérence des données imbriquées | Requêtes secondaires limitées et dépendance au cache pour la durabilité | Requêtes contraintes par le modèle de partition; gestion de partitions et rétention | Modélisation et exploitation plus spécifiques; moins adapté aux agrégations massives |
| **Cas SmartCampus** | Fiche complète d'un équipement | Dernier état d'un capteur | Historique des mesures par capteur et période | Dépendances d'équipements et diagnostic d'impact |
| **Technologies candidates** | MongoDB | Redis | Cassandra ou ScyllaDB | Neo4j |

Ces candidats n'ont pas les mêmes garanties, coûts d'exploitation ni sémantiques de cohérence. La comparaison doit porter sur une charge et une configuration explicites, et non sur une hiérarchie universelle entre technologies.

## 4. Dataset SmartCampus de référence

### 4.1 Population et hypothèses de charge

| Élément | Hypothèse de référence |
|---|---:|
| Bâtiments | 8 |
| Zones | Environ 160 |
| Équipements | Environ 250 |
| Capteurs | Environ 600 |
| Période d'historique | 12 mois, convention de calcul de 365 jours |
| Cadence nominale | 1 mesure toutes les 30 secondes par capteur |
| Cadence haute | 1 mesure toutes les 10 secondes par capteur |

Les nombres d'entités décrivent le périmètre de dimensionnement du cas. Les répartitions entre bâtiments, zones, équipements et capteurs doivent être produites de façon déterministe pour rendre les essais reproductibles. L'historique couvre une année synthétique; une année bissextile ou une fenêtre glissante de 12 mois doit être paramétrée explicitement si elle est retenue.

### 4.2 Volumétrie calculée

| Cadence par capteur | Mesures par capteur et par an | Total pour 600 capteurs | Débit moyen global |
|---|---:|---:|---:|
| 30 secondes | 1 051 200 | **630 720 000** | **20 mesures/s** |
| 10 secondes | 3 153 600 | **1 892 160 000** | **60 mesures/s** |

Ces totaux supposent une émission continue, sans panne ni lacune. La taille physique sur disque ne peut pas être déduite du seul nombre de lignes : elle doit être mesurée pour chaque moteur, schéma, index, réplication et politique de rétention.

### 4.3 Données et reproductibilité

Le générateur de dataset doit produire les mêmes entités de référence, mesures et horodatages pour chaque solution comparée. Il faut documenter la graine aléatoire, les formats, les unités, les règles de qualité, les valeurs manquantes ou aberrantes, ainsi que l'ordre d'arrivée des événements.

Pour tester les cas temporels, inclure des mesures reçues en retard et des corrections d'anciennes valeurs. Les mesures doivent conserver leur horodatage événementiel UTC et un horodatage de réception distinct. Les associations historiques de localisation doivent permettre d'attribuer une mesure au bâtiment et à la zone valides au moment de sa production, plutôt qu'à la localisation courante du capteur.

## 5. Protocole de benchmark

### 5.1 Solutions à comparer

Établir une référence relationnelle (PostgreSQL, avec le schéma et le partitionnement prévus pour le cas) et une solution candidate adaptée au scénario testé. Les options NoSQL peuvent être évaluées séparément : MongoDB pour la fiche, Redis pour l'état courant, Cassandra/ScyllaDB pour l'historique et Neo4j pour les parcours de graphe.

Une comparaison n'est équitable que si les garanties fonctionnelles et de persistance sont documentées. Pour chaque essai, préciser les versions, le matériel, le système, la configuration, les index, le niveau de réplication, le niveau de durabilité des écritures, la concurrence cliente et les paramètres réseau. Ne pas comparer une écriture durable à une écriture acquittée sans persistance équivalente sans signaler explicitement la différence.

### 5.2 Scénario de test 1 — Ingestion continue

1. Charger le même référentiel initial dans chaque solution.
2. Émettre le flux de mesures à **20 mesures/s** (cadence 30 secondes par capteur), puis répéter à **60 mesures/s** (cadence 10 secondes).
3. Utiliser des identifiants stables, une distribution uniforme documentée et des tailles de lots identiques lorsqu'elles sont pertinentes.
4. Mesurer la latence d'écriture jusqu'à l'acquittement correspondant au niveau de durabilité choisi, le débit réellement accepté, les erreurs, les reprises et le retard d'ingestion.
5. Vérifier après chaque essai le nombre d'événements, l'absence de perte ou de doublon inattendu et l'état courant des capteurs, notamment lorsque des événements sont rejoués ou arrivent en retard.

### 5.3 Scénario de test 2 — Reporting analytique mensuel

Exécuter une requête portant sur une période définie, par exemple les consommations mensuelles pour un bâtiment sur 12 mois, puis comparer des variantes à une granularité plus fine (zone, jour ou capteur). Les mêmes règles métier doivent s'appliquer : unités converties de manière cohérente, mesures invalides traitées pareillement, et localisation attribuée selon la validité historique au moment de la mesure.

Pour les moteurs dont le modèle ne correspond pas directement à l'agrégation, inclure toutes les étapes nécessaires dans la mesure de latence : lecture, transfert, transformation et agrégation côté client ou service. Rapporter séparément les coûts de préparation éventuels (tables d'agrégats, vues matérialisées, projections ou pré-calculs) et leur mécanisme de mise à jour.

### 5.4 Indicateurs à collecter

- **Latence :** p50 et p95 des opérations, séparément pour les écritures et les lectures.
- **Débit :** mesures écrites ou requêtes terminées par seconde.
- **Fiabilité :** taux d'erreur, timeouts, pertes, doublons et reprises.
- **Ressources :** CPU, mémoire et espace de stockage consommé, en précisant ce qui est inclus (index, journaux, réplication, données temporaires).
- **Qualité de service :** retard d'ingestion, fraîcheur des projections ou agrégats et exactitude des résultats, lorsqu'applicable.

Les résultats doivent être accompagnés des paramètres de charge, de la taille du dataset et de la configuration complète. Aucun résultat n'est présumé avant l'exécution du protocole.

### 5.5 Déroulement et équité des essais

1. **Préparation :** valider les schémas, index, paramètres et scripts; remplir les moteurs avec le même jeu de données et vérifier un échantillon de résultats.
2. **Échauffement :** exécuter la charge sans comptabiliser les résultats afin de stabiliser caches, connexions et structures internes. Garder la même politique de cache pour les solutions comparées et préciser si un essai à froid est également effectué.
3. **Mesure :** exécuter chaque scénario à charge et durée fixées; répéter au moins trois fois (cinq si le temps le permet), dans un ordre alterné ou randomisé pour limiter les biais liés à l'environnement.
4. **Réinitialisation :** entre les essais, restaurer le même snapshot ou reconstruire un état équivalent; supprimer les données de test de façon contrôlée, rétablir les mêmes caches lorsque le protocole le requiert et vérifier l'état avant la reprise.
5. **Analyse :** comparer les distributions de latence, les débits soutenus, les erreurs, les ressources et la justesse des résultats. Documenter les écarts entre répétitions et conserver scripts, configurations, journaux et manifeste de dataset.

Le matériel, les ressources allouées, le nombre de clients, le niveau de concurrence, la taille des requêtes, la durée de chauffe et la durée de mesure doivent rester identiques dans les essais comparables. Les résultats d'un essai à cache chaud ne doivent pas être mélangés à ceux d'un essai à cache froid.

## 6. Conclusion

Les quatre paradigmes répondent à des formes d'accès différentes : agrégat de fiche, accès direct par clé, séries temporelles et exploration de dépendances. Aucun paradigme ni produit n'est universellement supérieur. La décision doit découler des requêtes réellement attendues, des garanties de cohérence et de durabilité, de la charge mesurée et du coût opérationnel.

Pour SmartCampus, PostgreSQL peut rester la source de vérité tant que les essais ne démontrent pas un besoin justifiant un moteur supplémentaire. Une architecture polyglotte éventuelle doit limiter les données dupliquées à des projections explicites, préciser leur synchronisation et prévoir leur reconstruction. Le benchmark proposé sert à valider ces arbitrages; il ne préjuge pas de ses résultats.
