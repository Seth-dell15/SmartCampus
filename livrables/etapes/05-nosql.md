# SmartCampus — Étape 5 : paradigmes NoSQL et sélection

## 1. Objet et contexte

Cette étape compare les quatre paradigmes NoSQL — clé-valeur, document, colonne (famille de colonnes / *wide-column*) et graphe — au regard des besoins SmartCampus, puis décide s’il est justifié d’ajouter un moteur NoSQL au socle relationnel des étapes précédentes.

Le contexte de dimensionnement est le suivant :

- 8 bâtiments, environ 160 zones, 250 équipements et 600 capteurs;
- une mesure par capteur toutes les 30 secondes;
- environ **20 mesures par seconde en moyenne**, soit **1 728 000 par jour** et **630 720 000 par an** avant retransmissions;
- mesures brutes conservées 12 mois;
- environ 50 utilisateurs actifs en journée et tableaux de bord actualisés toutes les 5 minutes;
- besoins de référentiel intégré, d’historisation des localisations et associations, de lecture d’état courant, de traitement d’alertes et d’analyse.

Le MPD validé s’appuie sur PostgreSQL, ses partitions mensuelles sur `horodatage_evenement`, ses index temporels et les contraintes relationnelles. Un volume annuel élevé en lignes ne suffit pas, à lui seul, à rendre un SGBD NoSQL nécessaire : le débit, les objectifs de latence, les pics de reconnexion, la concurrence, la taille réelle des enregistrements et les capacités d’exploitation doivent être mesurés.

## 2. Comparaison des quatre paradigmes

| Paradigme | Modèle et forces | Limites et coûts | Exemples de besoins adaptés |
|---|---|---|---|
| **Clé-valeur** | Chaque entrée est retrouvée par une clé unique; accès très direct, latence faible et opérations simples. Particulièrement adapté aux valeurs courantes, sessions, caches et états indexés par identifiant. | Requêtes et relations entre valeurs limitées; cohérence transactionnelle et requêtes complexes varient selon le produit. Le cache ajoute une copie à synchroniser, une politique d’expiration et une dépendance d’exploitation. | Dernière mesure/état courant par `id_capteur`, cache de paramètres ou de tableaux de bord. |
| **Document** | Enregistre des objets autonomes en documents, souvent JSON/BSON; schéma souple, évolution aisée des attributs et récupération en une lecture d’un agrégat cohérent. | Répétition de données entre documents, jointures et contraintes référentielles moins naturelles; risque de documents volumineux et de mises à jour multi-documents. La souplesse de schéma ne remplace pas la gouvernance des champs. | Configurations d’équipements à structure variable, règles décrites comme expressions imbriquées, métadonnées propriétaires. |
| **Colonne / famille de colonnes (*wide-column*)** | Données organisées par lignes et familles de colonnes, avec partitionnement distribué et accès efficace lorsqu’on connaît la clé de partition et le parcours visé. Adapté aux écritures distribuées et séries volumineuses dans des modèles conçus par requête. | Exige de concevoir les clés et tables autour des requêtes; transactions, jointures et contraintes référentielles généralement limitées. Mauvaise clé ou partitions trop chaudes créent déséquilibre et difficultés opérationnelles. Ne pas confondre ce paradigme avec les moteurs **colonnaires analytiques** qui stockent les valeurs par colonnes pour les scans OLAP. | Historique de mesures à très grande échelle, distribution multi-nœud et requêtes connues du type capteur + fenêtre temporelle. |
| **Graphe** | Nœuds et arêtes explicites; parcours de relations et de chemins multi-sauts expressifs, y compris relations typées ou à propriétés. | Moteur, compétences et opérations supplémentaires; intégration temporelle, reporting et requêtes analytiques tabulaires doivent être évalués. Pour un graphe petit et des relations peu profondes, les jointures relationnelles suffisent souvent. | Exploration de dépendances complexes, impacts en cascade, topologie, relations dynamiques nombreuses. |

Les familles regroupent des produits aux garanties différentes. La comparaison porte sur les propriétés du paradigme; elle ne vaut pas validation d’un produit, d’une version ou d’un niveau de cohérence particulier.

## 3. Adéquation aux besoins SmartCampus

| Besoin SmartCampus | Clé-valeur | Document | Wide-column | Graphe |
|---|---|---|---|---|
| **Mesures brutes et historique sur 12 mois** | Inadapté comme stockage analytique principal : recherche ponctuelle possible, mais parcours temporels et agrégations limités. | Possible, mais une mesure par document ajoute de la flexibilité qui n’est pas utile au format essentiellement régulier; index et agrégations restent à dimensionner. | Bonne adéquation si la croissance, le débit ou les objectifs de disponibilité dépassent le socle relationnel et si les requêtes sont principalement par capteur et plage temporelle. | Inadapté au stockage primaire des séries; le graphe peut référencer des événements, mais n’est pas optimisé pour des centaines de millions de mesures simples. |
| **Référentiel campus, bâtiments, zones, équipements, capteurs** | Faible : les relations et recherches par attribut exigent des clés ou index spécifiques. | Possible si le référentiel était constitué d’objets autonomes, mais relations, unicités et intégrité temporelle sont centrales ici. | Possible mais peu approprié aux mises à jour référentielles et aux jointures nécessaires. | Peut représenter le réseau d’entités, mais ne remplace pas naturellement les contraintes, formulaires et listes relationnelles. |
| **Configurations d’équipements et règles d’alerte** | Adapté à la lecture par identifiant si l’objet est opaque; pas au besoin d’édition, versionnement et recherche par attribut. | Bonne adéquation si les configurations sont réellement variables, imbriquées et peu reliées. Le MCD actuel a des attributs connus et des règles versionnées relationnelles; PostgreSQL `JSONB` peut héberger un complément flexible sans service séparé. | Faible pour l’administration et la validation de configurations. | Utile seulement si les règles expriment des dépendances/chemins entre objets plutôt que des critères locaux. |
| **Dernier état d’un capteur** | Très bonne adéquation fonctionnelle à une clé `id_capteur`; peut fournir une lecture à faible latence. | Possible, mais inutilement complexe pour un état plat par capteur. | Possible, mais surdimensionné pour environ 600 entrées et peu de mises à jour. | Non adapté à la simple dernière valeur. |
| **Parcours de dépendances** | Faible : les liens doivent être lus séparément et assemblés par le service. | Peut contenir un agrégat préassemblé, mais les dépendances partagées et historisées sont dupliquées. | Peu adapté aux traversées multi-relations. | Très bonne adéquation si les parcours sont nombreux, variables, multi-sauts et orientent réellement les opérations. |
| **Agrégats de consommation et rapports** | Inadapté aux scans et regroupements multi-dimensions. | Possible pour petits agrégats préconstruits, mais pas un moteur analytique implicite. | Adapté à certains accès pré-modélisés; un moteur colonne analytique ou des agrégats PostgreSQL peuvent aussi convenir selon la concurrence et les fenêtres. | Inadapté aux agrégations numériques générales. |

### 3.1 Lecture des volumes et des accès

La cadence nominale correspond à environ 20 écritures/s, avec une rafale possible de 600 messages toutes les 30 secondes si les capteurs sont synchronisés ou se reconnectent ensemble. Le volume de 630,72 millions de mesures/an impose un plan de stockage, de rétention, d’indexation et de sauvegarde robuste; il ne prouve pas qu’un cluster *wide-column* est nécessaire. Le partitionnement mensuel du MPD, l’index `(id_capteur, horodatage_evenement)` et les requêtes limitées aux périodes utiles sont une première stratégie cohérente à benchmarker.

Le référentiel et les dépendances comptent quelques centaines d’équipements et capteurs, ainsi qu’environ 160 zones. Les clés étrangères et historiques temporels PostgreSQL donnent une représentation directe et contrôlée. En l’absence de parcours réellement complexes ou de latence démontrée comme problématique, le moteur graphe ajouterait une seconde représentation sans bénéfice établi.

## 4. Sélection recommandée

### 4.1 Décision : pas de nouveau moteur NoSQL au démarrage

**La recommandation pour le périmètre et les volumes de référence est de ne sélectionner aucun moteur NoSQL supplémentaire en production à ce stade.** PostgreSQL reste la source de vérité pour le référentiel, les mesures brutes, les associations historisées, les règles, les alertes et les interventions.

Cette décision est volontaire et restrictive :

1. Le modèle a besoin d’intégrité référentielle, d’historisation temporelle et d’interrogations jointes, toutes déjà couvertes par le MLD/MPD.
2. Le débit moyen nominal est modéré; le principal enjeu est le volume historique annuel et son exploitation, déjà pris en compte par les partitions et index temporels.
3. Le besoin d’état courant peut être servi d’abord par les index PostgreSQL et, si les tests le justifient, par la projection relationnelle `etat_courant_capteur` recommandée à l’étape 4.
4. Les configurations et règles ne sont pas décrites comme fortement variables au point de justifier un magasin de documents indépendant.
5. Les dépendances sont limitées et connues; une base graphe n’est pas justifiée par le seul fait que le domaine comporte des relations.
6. Chaque moteur supplémentaire apporte synchronisation, sauvegarde, supervision, contrôle d’accès, reprise après incident, coûts et compétences à maintenir. Aucun bénéfice mesuré ne compense actuellement ces coûts.

Ainsi, la sélection effective est **relationnelle seule** pour le scénario de départ. Le paradigme clé-valeur est le candidat NoSQL le plus plausible pour une évolution ultérieure; il n’est pas activé d’emblée.

### 4.2 Candidat conditionnel : clé-valeur pour l’état courant

Un magasin clé-valeur peut être évalué si le besoin d’état courant dépasse ce que fournissent les index et la projection PostgreSQL, par exemple si des tests en charge démontrent que la latence de lecture ou l’isolation de la charge des tableaux de bord ne respecte pas un objectif approuvé.

La valeur serait dérivée et indexée par `id_capteur`, avec au minimum :

```text
clé : smartcampus:capteur:{id_capteur}:etat
valeur :
  id_mesure_derniere
  horodatage_evenement_derniere
  horodatage_reception_derniere
  id_mesure_valide
  horodatage_evenement_valide
  valeur_valide
  unite
  qualite
  date_calcul
  version_regle_etat
```

Cette structure distingue la dernière mesure reçue de la dernière mesure considérée valide. Il ne faut pas utiliser la seule dernière arrivée : une mesure ancienne reçue après une coupure ne doit pas faire régresser l’état temporel.

**Critère de décision.** L’ajout n’est accepté qu’après benchmark reproductible avec les partitions, index, projections SQL, rafraîchissement des tableaux de bord toutes les cinq minutes, pics de reconnexion et concurrence réaliste. Le benchmark doit montrer un gain pertinent sur un objectif de latence ou d’isolation défini, et intégrer le coût complet d’exploitation. Les valeurs de latence cibles n’étant pas spécifiées dans l’étape 1, aucun seuil arbitraire n’est fixé ici.

## 5. Articulation technique avec PostgreSQL

### 5.1 Architecture de référence

```text
Capteurs / passerelles
        |
        v
Service d'ingestion
  - validation, horodatage, idempotence
  - résolution de l'identifiant externe
        |
        v
PostgreSQL (source de vérité)
  - référentiel et associations temporelles
  - mesures partitionnées par mois
  - alertes, règles, interventions
  - projection SQL d'état courant si nécessaire
        |
        +--> tableaux de bord / API
        +--> agrégations analytiques et exports

Extension future conditionnelle :
Service d'ingestion / projection --> magasin clé-valeur
                                      (cache/projection reconstruisible)
```

Dans le scénario retenu, aucune écriture à double destination n’est nécessaire : l’ingestion valide et persiste les mesures dans PostgreSQL, puis l’état courant peut être calculé ou maintenu dans une projection relationnelle au sein de la même transaction, conformément à l’étape 4. Les tableaux de bord lisent le référentiel et les historiques dans PostgreSQL et exploitent les projections/agrégats quand ils existent.

### 5.2 Si le candidat clé-valeur est ultérieurement retenu

Le magasin clé-valeur ne devient jamais l’autorité pour les mesures, le référentiel ou les règles. Il ne contient qu’une projection lisible et reconstruisible :

1. **Écriture de référence** : valider et enregistrer la mesure dans PostgreSQL.
2. **Publication durable** : dans la même transaction, enregistrer un événement d’outbox ou utiliser un mécanisme durable équivalent. Éviter un simple `COMMIT` PostgreSQL suivi d’une écriture réseau non garantie : une panne entre les deux laisserait la projection en retard.
3. **Projection idempotente** : un consommateur traite les événements, compare l’horodatage d’événement et l’identité de mesure, puis met à jour la clé du capteur sans régression temporelle.
4. **Lecture avec contrat explicite** : le tableau de bord accepte une éventuelle cohérence différée et peut afficher `date_calcul`/retard de projection. Les opérations d’administration ou d’audit relisent la source relationnelle.
5. **Reprise et réconciliation** : les événements peuvent être rejoués; une tâche compare ou reconstruit les états à partir des mesures récentes. Une panne du magasin ne doit jamais interrompre ou compromettre l’écriture de la source PostgreSQL.
6. **Expiration contrôlée** : la politique TTL ne supprime pas un état actif par inadvertance. Le TTL éventuel correspond à une règle explicite d’obsolescence et doit être distingué de la rétention des mesures brutes.

Une projection peut aussi être rafraîchie par le service d’ingestion à partir de la table relationnelle, à condition que l’écriture soit transactionnellement fiable ou récupérable. Un simple cache à invalidation non durable est acceptable seulement si sa perte n’affecte pas l’état métier et si une lecture de repli sur PostgreSQL existe.

### 5.3 Pourquoi les autres paradigmes ne sont pas déployés

- **Document** : les configurations actuelles sont suffisamment structurées pour les tables relationnelles; un attribut imbriqué réellement variable peut être stocké en `JSONB` dans PostgreSQL si le besoin apparaît. Cela conserve les transactions, sauvegardes et accès sans introduire un second système. Un moteur document distinct ne sera retenu que si les profils de configuration deviennent autonomes, très variables et principalement lus/écrits comme documents complets.
- **Wide-column** : le besoin de série temporelle est réel, mais la cadence nominale, les partitions mensuelles et les requêtes par capteur/période ne suffisent pas à établir un besoin de base distribuée. Une évaluation deviendrait pertinente si les mesures de capacité démontrent que l’ingestion, les lectures historiques, la disponibilité ou la croissance dépassent durablement PostgreSQL après optimisation. Elle exigerait un contrat de cohérence, de rétention et de double écriture/rejeu ainsi qu’un plan de migration.
- **Graphe** : les relations sont temporelles mais peu nombreuses, et les parcours connus peuvent être réalisés par jointures et contraintes relationnelles. Un moteur graphe serait réévalué si des parcours multi-sauts dynamiques (par exemple impacts en cascade sur de nombreuses dépendances) deviennent un cas métier fréquent et mesuré, et si leur coût SQL est limitant.

## 6. Exploitabilité, sécurité et critères de réévaluation

Avant l’introduction d’un moteur supplémentaire, l’équipe devra comparer le coût global, pas seulement la latence d’une requête :

- sauvegardes, restauration, réplication, mises à jour et supervision du moteur;
- disponibilité, RPO/RTO, reprise de flux, gestion des doublons et des retards;
- authentification, autorisation, chiffrement et audit alignés sur PostgreSQL;
- cohérence entre source et projection, délai maximal de rattrapage et mécanisme de reconstruction;
- taille et croissance réelles, distribution des clés et comportement lors des rafales;
- compétence d’exploitation, coût de licence/hébergement et dépendance fournisseur;
- benchmark avec les requêtes réelles, les index/partitions, la rétention de 12 mois et une concurrence représentative.

Les indicateurs à suivre sur le socle actuel comprennent le débit et les erreurs d’ingestion, le retard entre événement et réception, la latence des requêtes d’état courant et historiques, la durée de rafraîchissement des tableaux de bord, l’utilisation des partitions/index et l’espace restant. Ces mesures apporteront une justification factuelle à une éventuelle évolution.

## 7. Conclusion

Les quatre paradigmes ont des cas d’usage distincts, mais **SmartCampus n’a pas actuellement besoin d’accumuler quatre moteurs ni d’introduire un NoSQL par principe**. Le modèle relationnel PostgreSQL normalisé reste adapté aux données de référence, aux relations historisées et au stockage initial des mesures partitionnées.

La seule piste NoSQL retenue comme option conditionnelle est le **clé-valeur pour la projection de l’état courant**, si des benchmarks démontrent un besoin de latence ou d’isolation que PostgreSQL ne satisfait pas après optimisation. Le paradigme *wide-column* reste une option de mise à l’échelle à réexaminer en cas de croissance ou de limites mesurées; le document et le graphe ne répondent pas à un besoin actuel assez fort pour justifier leur coût opérationnel.

