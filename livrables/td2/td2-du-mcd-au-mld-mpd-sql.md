# TD 2 — Du MCD au MLD / MPD SQL

## Projet SmartCampus

**Objet :** traduction du MCD en schéma relationnel, définition du MPD PostgreSQL, contraintes, index et vérifications d’intégrité.  
**SGBD de référence :** PostgreSQL 15 ou ultérieur.  
**Références :** [MCD du projet](../etapes/02-mcd.md) et [MPD détaillé](../etapes/03-mld-mpd.md).

## Périmètre et conventions

Le schéma reprend le périmètre SmartCampus établi au TD 1 : campus, bâtiments, zones, équipements, capteurs, mesures, alertes et maintenance. Les liens de localisation d’un équipement et d’affectation d’un capteur sont historisés avec des intervalles semi-ouverts `[début, fin[`. Le modèle physique prévoit le partitionnement mensuel de `mesure` par horodatage d’événement.

Les identifiants techniques sont des `BIGINT` générés par identité. Les horodatages sont des instants `TIMESTAMPTZ`, stockés et exploités en UTC. Dans les tableaux, **NN** signifie `NOT NULL`, **NULL** signifie facultatif. Les valeurs d’énumération données dans les domaines sont les valeurs acceptées par les `CHECK` du MPD de référence. Les vocabulaires sont volontairement contrôlés en ligne; ils pourront devenir des tables de référence s’ils doivent être administrés.

## Étape 1 — Traduire le MCD en MLD

### 1.1 Règles de passage appliquées

| Règle de passage | Application dans SmartCampus |
|---|---|
| Une entité devient une table. | Les entités du MCD deviennent notamment `campus`, `batiment`, `zone`, `equipement`, `capteur`, `mesure`, `regle_alerte`, `alerte`, `intervention`, `technicien` et `utilisateur`. |
| Un attribut devient une colonne. | Les propriétés métier sont des colonnes typées dans la table correspondante. Un identifiant externe de capteur reste une chaîne distincte de l’identifiant technique interne. |
| Un identifiant devient une clé primaire. | Chaque table de référentiel et d’association porte une clé primaire technique. Exception physique : la clé primaire de `mesure` est composée de `(horodatage_evenement, id_mesure)` car PostgreSQL impose que la clé unique d’une table partitionnée inclue la clé de partition. |
| Une association 1,N est portée par une FK du côté N. | `batiment.id_campus`, `zone.id_batiment`, `mesure.id_capteur`, `intervention.id_equipement` et `alerte.id_regle_alerte` référencent leur parent. |
| Une association 1,1 est matérialisée par une FK `UNIQUE`. | `technicien.id_utilisateur` est obligatoire et `UNIQUE` : un profil technicien correspond à un compte utilisateur, et un compte ne porte au plus qu’un profil. |
| Une association N,N devient une table associative. | Les relations avec attributs propres sont des tables : `historique_zone_equipement`, `affectation_capteur_equipement`, `affectation_technicien_intervention` et `lien_alerte_intervention`. |
| Une association facultative devient une FK nullable. | `intervention.id_demandeur`, les deux FK de cible d’alerte et la référence facultative à une mesure déclencheuse acceptent `NULL`, avec les contraintes métier appropriées. |
| Une association porteuse d’historique conserve les occurrences passées. | Une nouvelle ligne représente chaque période; la ligne précédente est close et n’est pas écrasée par une nouvelle affectation. |

### 1.2 Schéma relationnel (MLD)

`PK` : clé primaire; `FK` : clé étrangère; `UQ` : unicité; `NN` : non nul.

```text
CAMPUS(PK id_campus, UQ code_campus, nom NN, adresse, statut NN)
BATIMENT(PK id_batiment, FK id_campus NN -> CAMPUS, UQ code_batiment,
         nom NN, adresse, statut NN)
ZONE(PK id_zone, FK id_batiment NN -> BATIMENT, code_zone NN, nom NN,
     type_zone NN, etage, statut NN, UQ(id_batiment, code_zone))
EQUIPEMENT(PK id_equipement, UQ code_equipement, libelle NN,
           type_equipement NN, marque, modele, numero_serie,
           date_mise_en_service, statut NN)
HISTORIQUE_ZONE_EQUIPEMENT(PK id_historique_zone,
    FK id_equipement NN -> EQUIPEMENT, FK id_zone NN -> ZONE,
    date_debut NN, date_fin, motif_deplacement,
    CHECK date_debut < date_fin si date_fin existe,
    EXCLUDE même équipement et périodes qui se chevauchent)
CAPTEUR(PK id_capteur, UQ identifiant_externe, type_capteur NN,
        libelle NN, unite_nominale NN, statut NN, date_mise_en_service)
AFFECTATION_CAPTEUR_EQUIPEMENT(PK id_affectation_capteur,
    FK id_capteur NN -> CAPTEUR, FK id_equipement NN -> EQUIPEMENT,
    date_debut NN, date_fin, motif_affectation,
    CHECK date_debut < date_fin si date_fin existe,
    EXCLUDE même capteur et périodes qui se chevauchent)
MESURE(PK physique(horodatage_evenement, id_mesure),
       id_mesure NN, FK id_capteur NN -> CAPTEUR,
       horodatage_evenement NN, horodatage_reception NN,
       valeur NN, unite NN, qualite NN, cle_deduplication)
REGLE_ALERTE(PK id_regle_alerte, UQ(code_regle, numero_version),
    code_regle NN, numero_version NN, libelle NN, portee_cible NN,
    indicateur NN, operateur NN, seuil NN, severite NN,
    date_debut_validite NN, date_fin_validite, statut NN)
ALERTE(PK id_alerte, FK id_regle_alerte NN -> REGLE_ALERTE,
    FK id_equipement_cible -> EQUIPEMENT, FK id_capteur_cible -> CAPTEUR,
    FK (horodatage_mesure_declencheuse, id_mesure_declencheuse) -> MESURE,
    date_creation NN, severite_constatee NN, statut NN,
    date_acquittement, date_cloture, commentaire,
    CHECK exactement une cible : équipement XOR capteur)
UTILISATEUR(PK id_utilisateur, UQ courriel, prenom NN, nom NN,
            statut NN, date_creation NN)
TECHNICIEN(PK id_technicien, FK UQ id_utilisateur NN -> UTILISATEUR,
           UQ matricule, specialite, statut NN)
INTERVENTION(PK id_intervention, UQ reference,
    FK id_equipement NN -> EQUIPEMENT,
    FK id_demandeur -> UTILISATEUR, type_intervention NN,
    priorite NN, statut NN, date_ouverture NN, date_planifiee,
    date_debut, date_cloture, description NN, compte_rendu)
AFFECTATION_TECHNICIEN_INTERVENTION(PK id_affectation_technicien,
    FK id_intervention NN -> INTERVENTION, FK id_technicien NN -> TECHNICIEN,
    date_affectation NN, date_fin_affectation, role_dans_intervention,
    commentaire)
LIEN_ALERTE_INTERVENTION(PK id_lien_alerte_intervention,
    FK id_alerte NN -> ALERTE, FK id_intervention NN -> INTERVENTION,
    date_liaison NN, commentaire, UQ(id_alerte, id_intervention))
```

La mesure est identifiée conceptuellement par `id_mesure`, mais, sur la table partitionnée, la garantie d’unicité déclarée par la base porte sur le couple `(horodatage_evenement, id_mesure)`. Toute FK vers une mesure doit donc conserver les deux valeurs. Le couple `(id_capteur, horodatage_evenement)` n’est pas unique : plusieurs observations peuvent partager le même instant.

### 1.3 Historisation et absence de chevauchements

Les deux relations temporelles sont représentées par des tables associatives. Une fin `NULL` est interprétée comme une borne supérieure infinie. La convention `[)` inclut la borne de début et exclut la borne de fin : deux localisations peuvent donc se succéder exactement à la même date/heure sans chevauchement.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE smartcampus.historique_zone_equipement
  ADD CONSTRAINT ck_hze_intervalle
  CHECK (date_fin IS NULL OR date_debut < date_fin),
  ADD CONSTRAINT ex_hze_equipement_periode
  EXCLUDE USING gist (
    id_equipement WITH =,
    tstzrange(date_debut, date_fin, '[)') WITH &&
  );

ALTER TABLE smartcampus.affectation_capteur_equipement
  ADD CONSTRAINT ck_ace_intervalle
  CHECK (date_fin IS NULL OR date_debut < date_fin),
  ADD CONSTRAINT ex_ace_capteur_periode
  EXCLUDE USING gist (
    id_capteur WITH =,
    tstzrange(date_debut, date_fin, '[)') WITH &&
  );
```

L’opérateur `&&` détecte le chevauchement des plages. L’index GiST créé par chaque exclusion interdit deux intervalles qui se chevauchent pour le même équipement dans son historique spatial, ou pour le même capteur dans ses affectations. Il accepte une borne finale ouverte, refuse les intervalles inversés et autorise les intervalles contigus. La deuxième règle exprime « au plus un équipement à la fois par capteur »; elle n’interdit pas qu’un équipement possède plusieurs capteurs.

### 1.4 Choix physiques associés

- `mesure` est partitionnée par `RANGE (horodatage_evenement)`, sur des partitions mensuelles. Toute clé unique et toute FK qui la référence inclut donc l’horodatage.
- Le partitionnement nécessite une supervision des partitions futures et une politique de purge compatible avec les alertes qui référencent éventuellement une mesure.
- Les noms SQL sont en `snake_case`; les identifiants techniques utilisent `BIGINT GENERATED BY DEFAULT AS IDENTITY`.

## Étape 2 — Définir les types de données

Les tableaux suivants couvrent **toutes les colonnes métier et techniques déclarées** dans les 15 tables du MLD. `DEFAULT` indique une valeur fournie par la base. Les domaines sont ceux du MPD de référence; ils doivent être élargis par migration si de nouveaux états métier sont approuvés.

### 2.1 Référentiel spatial

| Table | Colonne | Type SQL | NULL | Justification et domaine |
|---|---|---|---|---|
| `campus` | `id_campus` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique 64 bits, généré par PostgreSQL; PK. |
| `campus` | `code_campus` | `VARCHAR(80)` | NN | Code stable d’intégration; `UNIQUE`. |
| `campus` | `nom` | `VARCHAR(160)` | NN | Nom affichable avec longueur bornée. |
| `campus` | `adresse` | `VARCHAR(500)` | NULL | Adresse facultative; longueur maximale adaptée à une adresse complète. |
| `campus` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `ACTIF`, `INACTIF`, `ARCHIVE`. |
| `batiment` | `id_batiment` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; PK. |
| `batiment` | `id_campus` | `BIGINT` | NN | FK vers `campus.id_campus`; relation obligatoire. |
| `batiment` | `code_batiment` | `VARCHAR(80)` | NN | Code métier stable; `UNIQUE`. |
| `batiment` | `nom` | `VARCHAR(160)` | NN | Désignation obligatoire du bâtiment. |
| `batiment` | `adresse` | `VARCHAR(500)` | NULL | Adresse facultative, distincte du nom. |
| `batiment` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `ACTIF`, `INACTIF`, `ARCHIVE`. |
| `zone` | `id_zone` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; PK. |
| `zone` | `id_batiment` | `BIGINT` | NN | FK vers `batiment.id_batiment`; relation obligatoire. |
| `zone` | `code_zone` | `VARCHAR(80)` | NN | Code local au bâtiment; `UNIQUE(id_batiment, code_zone)`. |
| `zone` | `nom` | `VARCHAR(160)` | NN | Nom lisible de la zone. |
| `zone` | `type_zone` | `VARCHAR(50)` | NN | Catégorie de zone; domaine contrôlé par le référentiel applicatif. |
| `zone` | `etage` | `VARCHAR(40)` | NULL | Peut être absent (extérieur, sous-sol non numéroté, etc.); chaîne pour gérer des valeurs non numériques. |
| `zone` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `ACTIVE`, `FERMEE`, `ARCHIVEE`. |

### 2.2 Équipements, capteurs et mesures

| Table | Colonne | Type SQL | NULL | Justification et domaine |
|---|---|---|---|---|
| `equipement` | `id_equipement` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; PK. |
| `equipement` | `code_equipement` | `VARCHAR(100)` | NN | Identifiant métier unique dans le parc; `UNIQUE`. |
| `equipement` | `libelle` | `VARCHAR(200)` | NN | Désignation courte nécessaire à l’exploitation. |
| `equipement` | `type_equipement` | `VARCHAR(80)` | NN | Famille technique contrôlée par le domaine métier. |
| `equipement` | `marque` | `VARCHAR(100)` | NULL | Le fabricant peut être inconnu ou sans objet. |
| `equipement` | `modele` | `VARCHAR(100)` | NULL | Modèle éventuellement non renseigné. |
| `equipement` | `numero_serie` | `VARCHAR(120)` | NULL | Certains actifs ne possèdent pas de numéro exploitable. |
| `equipement` | `date_mise_en_service` | `DATE` | NULL | Date civile, sans heure ni fuseau. |
| `equipement` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `EN_PREPARATION`, `EN_SERVICE`, `HORS_SERVICE`, `RETIRE`, `ARCHIVE`. |
| `historique_zone_equipement` | `id_historique_zone` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant de l’occurrence d’historique; PK. |
| `historique_zone_equipement` | `id_equipement` | `BIGINT` | NN | FK vers l’équipement localisé. |
| `historique_zone_equipement` | `id_zone` | `BIGINT` | NN | FK vers la zone; une localisation porte toujours sur une zone. |
| `historique_zone_equipement` | `date_debut` | `TIMESTAMPTZ` | NN | Instant UTC de début; début inclus. |
| `historique_zone_equipement` | `date_fin` | `TIMESTAMPTZ` | NULL | Fin exclusive; `NULL` signifie période toujours ouverte. |
| `historique_zone_equipement` | `motif_deplacement` | `VARCHAR(500)` | NULL | Motif facultatif, borné pour le texte court. |
| `capteur` | `id_capteur` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; PK. |
| `capteur` | `identifiant_externe` | `VARCHAR(160)` | NN | Identité transmise par la source IoT; `UNIQUE`. |
| `capteur` | `type_capteur` | `VARCHAR(80)` | NN | Type/grandeur mesurée; domaine métier. |
| `capteur` | `libelle` | `VARCHAR(200)` | NN | Nom affichable du capteur. |
| `capteur` | `unite_nominale` | `VARCHAR(30)` | NN | Unité attendue à l’installation; format court. |
| `capteur` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `EN_STOCK`, `EN_SERVICE`, `INDISPONIBLE`, `RETIRE`, `ARCHIVE`. |
| `capteur` | `date_mise_en_service` | `DATE` | NULL | Date civile facultative de mise en service. |
| `affectation_capteur_equipement` | `id_affectation_capteur` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant d’une période d’affectation; PK. |
| `affectation_capteur_equipement` | `id_capteur` | `BIGINT` | NN | FK du capteur affecté. |
| `affectation_capteur_equipement` | `id_equipement` | `BIGINT` | NN | FK de l’équipement supervisé pendant la période. |
| `affectation_capteur_equipement` | `date_debut` | `TIMESTAMPTZ` | NN | Début inclus de l’affectation. |
| `affectation_capteur_equipement` | `date_fin` | `TIMESTAMPTZ` | NULL | Fin exclusive; NULL tant que l’affectation reste ouverte. |
| `affectation_capteur_equipement` | `motif_affectation` | `VARCHAR(500)` | NULL | Motif facultatif d’installation ou de réaffectation. |
| `mesure` | `id_mesure` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; partie de la PK physique composée avec l’horodatage de partition. |
| `mesure` | `id_capteur` | `BIGINT` | NN | FK vers le capteur producteur. |
| `mesure` | `horodatage_evenement` | `TIMESTAMPTZ` | NN | Instant de mesure et clé de partition; nécessaire dans la PK physique. |
| `mesure` | `horodatage_reception` | `TIMESTAMPTZ` | NN, `DEFAULT CURRENT_TIMESTAMP` | Instant d’arrivée côté plateforme, utile pour mesurer le retard de transmission. |
| `mesure` | `valeur` | `NUMERIC(20,6)` | NN | Décimal exact; précision et échelle initiales à valider selon les capteurs. |
| `mesure` | `unite` | `VARCHAR(30)` | NN | Unité réellement portée par cette observation; distincte de l’unité nominale. |
| `mesure` | `qualite` | `VARCHAR(30)` | NN | `CHECK` : `VALIDE`, `INVALIDE`, `ESTIMEE`, `INCONNUE`. |
| `mesure` | `cle_deduplication` | `VARCHAR(160)` | NULL | Identifiant de message source optionnel; pas d’unicité sans périmètre de déduplication approuvé. |

### 2.3 Alertes et règles

| Table | Colonne | Type SQL | NULL | Justification et domaine |
|---|---|---|---|---|
| `regle_alerte` | `id_regle_alerte` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant de version de règle; PK. |
| `regle_alerte` | `code_regle` | `VARCHAR(100)` | NN | Code stable d’une règle, répété entre ses versions. |
| `regle_alerte` | `numero_version` | `INTEGER` | NN | Petit entier ordonné; `CHECK > 0`, `UNIQUE(code_regle, numero_version)`. |
| `regle_alerte` | `libelle` | `VARCHAR(200)` | NN | Désignation de la version de règle. |
| `regle_alerte` | `portee_cible` | `VARCHAR(20)` | NN | `CHECK` : `EQUIPEMENT` ou `CAPTEUR`; doit correspondre à la cible de l’alerte. |
| `regle_alerte` | `indicateur` | `VARCHAR(80)` | NN | Identifie la grandeur comparée au seuil. |
| `regle_alerte` | `operateur` | `VARCHAR(20)` | NN | `CHECK` : `>`, `>=`, `<`, `<=`, `=`, `<>`. |
| `regle_alerte` | `seuil` | `NUMERIC(20,6)` | NN | Seuil décimal exact, cohérent avec la précision des mesures. |
| `regle_alerte` | `severite` | `VARCHAR(30)` | NN | `CHECK` : `INFO`, `MINEURE`, `MAJEURE`, `CRITIQUE`. |
| `regle_alerte` | `date_debut_validite` | `TIMESTAMPTZ` | NN | Début inclus de validité de la version. |
| `regle_alerte` | `date_fin_validite` | `TIMESTAMPTZ` | NULL | Fin exclusive éventuelle; doit être postérieure au début. |
| `regle_alerte` | `statut` | `VARCHAR(20)` | NN | `CHECK` : `BROUILLON`, `ACTIVE`, `SUSPENDUE`, `EXPIREE`. |
| `alerte` | `id_alerte` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; PK. |
| `alerte` | `id_regle_alerte` | `BIGINT` | NN | FK vers la version exacte ayant déclenché l’alerte. |
| `alerte` | `id_equipement_cible` | `BIGINT` | NULL | FK équipement; exactement l’une des deux cibles doit être présente. |
| `alerte` | `id_capteur_cible` | `BIGINT` | NULL | FK capteur; XOR avec `id_equipement_cible`. |
| `alerte` | `id_mesure_declencheuse` | `BIGINT` | NULL | Identifiant d’une mesure déclencheuse; FK composite avec son horodatage. |
| `alerte` | `horodatage_mesure_declencheuse` | `TIMESTAMPTZ` | NULL | Composante de la FK vers la PK partitionnée de `mesure`; nullable avec l’identifiant. |
| `alerte` | `date_creation` | `TIMESTAMPTZ` | NN, `DEFAULT CURRENT_TIMESTAMP` | Instant de création de l’alerte. |
| `alerte` | `severite_constatee` | `VARCHAR(30)` | NN | `CHECK` : `INFO`, `MINEURE`, `MAJEURE`, `CRITIQUE`. |
| `alerte` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `OUVERTE`, `ACQUITTEE`, `EN_COURS`, `CLOTUREE`, `IGNOREE`. |
| `alerte` | `date_acquittement` | `TIMESTAMPTZ` | NULL | Instant d’acquittement, absent tant que non acquittée. |
| `alerte` | `date_cloture` | `TIMESTAMPTZ` | NULL | Instant de clôture, absent tant que non clôturée. |
| `alerte` | `commentaire` | `TEXT` | NULL | Commentaire de longueur variable; aucune limite métier établie. |

### 2.4 Utilisateurs, techniciens et interventions

| Table | Colonne | Type SQL | NULL | Justification et domaine |
|---|---|---|---|---|
| `utilisateur` | `id_utilisateur` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; PK. |
| `utilisateur` | `courriel` | `VARCHAR(254)` | NN | Taille maximale pratique pour une adresse de courrier; `UNIQUE`. |
| `utilisateur` | `prenom` | `VARCHAR(100)` | NN | Prénom d’affichage; donnée personnelle à accès restreint. |
| `utilisateur` | `nom` | `VARCHAR(100)` | NN | Nom d’affichage; donnée personnelle à accès restreint. |
| `utilisateur` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `ACTIF`, `SUSPENDU`, `DESACTIVE`. |
| `utilisateur` | `date_creation` | `TIMESTAMPTZ` | NN, `DEFAULT CURRENT_TIMESTAMP` | Instant de création du compte. |
| `technicien` | `id_technicien` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique du profil; PK. |
| `technicien` | `id_utilisateur` | `BIGINT` | NN | FK `UNIQUE` vers `utilisateur`; matérialise le 1:1 optionnel côté utilisateur. |
| `technicien` | `matricule` | `VARCHAR(80)` | NN | Identifiant professionnel; `UNIQUE`. |
| `technicien` | `specialite` | `VARCHAR(100)` | NULL | Spécialité éventuelle; domaine contrôlé par le métier. |
| `technicien` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `ACTIF`, `INDISPONIBLE`, `INACTIF`. |
| `intervention` | `id_intervention` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique; PK. |
| `intervention` | `reference` | `VARCHAR(100)` | NN | Référence lisible; `UNIQUE`. |
| `intervention` | `id_equipement` | `BIGINT` | NN | FK obligatoire vers l’équipement concerné par l’intervention. |
| `intervention` | `id_demandeur` | `BIGINT` | NULL | FK facultative vers l’utilisateur demandeur; une intervention peut être préventive ou automatique. |
| `intervention` | `type_intervention` | `VARCHAR(30)` | NN | `CHECK` : `CORRECTIVE`, `PREVENTIVE`, `INSPECTION`, `AUTRE`. |
| `intervention` | `priorite` | `VARCHAR(30)` | NN | `CHECK` : `BASSE`, `NORMALE`, `HAUTE`, `URGENTE`. |
| `intervention` | `statut` | `VARCHAR(30)` | NN | `CHECK` : `A_PLANIFIER`, `PLANIFIEE`, `EN_COURS`, `EN_ATTENTE`, `TERMINEE`, `ANNULEE`. |
| `intervention` | `date_ouverture` | `TIMESTAMPTZ` | NN, `DEFAULT CURRENT_TIMESTAMP` | Instant de création/ouverture. |
| `intervention` | `date_planifiee` | `TIMESTAMPTZ` | NULL | Instant prévu, non obligatoire à la création. |
| `intervention` | `date_debut` | `TIMESTAMPTZ` | NULL | Début effectif, absent avant réalisation. |
| `intervention` | `date_cloture` | `TIMESTAMPTZ` | NULL | Clôture effective facultative. |
| `intervention` | `description` | `TEXT` | NN | Description libre, sans taille maximale métier fixée. |
| `intervention` | `compte_rendu` | `TEXT` | NULL | Compte rendu ajouté après l’intervention. |
| `affectation_technicien_intervention` | `id_affectation_technicien` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant de l’occurrence d’affectation; PK. |
| `affectation_technicien_intervention` | `id_intervention` | `BIGINT` | NN | FK vers l’intervention. |
| `affectation_technicien_intervention` | `id_technicien` | `BIGINT` | NN | FK vers le technicien. |
| `affectation_technicien_intervention` | `date_affectation` | `TIMESTAMPTZ` | NN | Début inclus de participation. |
| `affectation_technicien_intervention` | `date_fin_affectation` | `TIMESTAMPTZ` | NULL | Fin exclusive facultative; doit être postérieure au début si renseignée. |
| `affectation_technicien_intervention` | `role_dans_intervention` | `VARCHAR(40)` | NULL | Rôle facultatif de l’intervenant. |
| `affectation_technicien_intervention` | `commentaire` | `TEXT` | NULL | Note de participation, texte libre. |
| `lien_alerte_intervention` | `id_lien_alerte_intervention` | `BIGINT GENERATED BY DEFAULT AS IDENTITY` | NN | Identifiant technique de l’association; PK. |
| `lien_alerte_intervention` | `id_alerte` | `BIGINT` | NN | FK vers l’alerte reliée. |
| `lien_alerte_intervention` | `id_intervention` | `BIGINT` | NN | FK vers l’intervention reliée. |
| `lien_alerte_intervention` | `date_liaison` | `TIMESTAMPTZ` | NN, `DEFAULT CURRENT_TIMESTAMP` | Instant de création de l’association. |
| `lien_alerte_intervention` | `commentaire` | `TEXT` | NULL | Commentaire facultatif sur le lien. |

## Étape 3 — Définir les contraintes d’intégrité

### 3.1 Contraintes appliquées

| Contrainte | Application | Justification |
|---|---|---|
| `PRIMARY KEY` | Une PK par table; `(horodatage_evenement, id_mesure)` pour `mesure`. | Garantit l’identification non nulle et unique des lignes, en conformité avec le partitionnement PostgreSQL. |
| `FOREIGN KEY` | Toutes les relations parent-enfant du MLD, dont les associations historiques. FK de mesure déclencheuse composite. | Interdit les références orphelines et protège la cohérence temporelle référençable. |
| `ON DELETE RESTRICT` | Toutes les FK. | Préserve l’historique et empêche qu’une suppression en cascade efface un actif, une alerte ou une trace de maintenance. Utiliser archivage/statut plutôt que suppression. |
| `ON UPDATE RESTRICT` | Toutes les FK. | Les identifiants techniques sont stables et ne doivent pas être propagés par mise à jour. |
| `NOT NULL` | Identifiants, champs indispensables, FK des associations obligatoires, `date_debut`, valeur/unité/qualité des mesures, équipement d’une intervention. | Traduit les participations minimales `1,1` et empêche les objets incomplets. |
| `UNIQUE` | Codes de campus, bâtiment et équipement; `identifiant_externe`; matricule, courriel, référence d’intervention; `(code_regle, numero_version)`; `(id_batiment, code_zone)`; `(id_alerte, id_intervention)`. | Protège les identifiants métier et empêche les doublons incohérents dans les associations. |
| `CHECK` de domaine | Statuts, sévérités, types, priorités, portée, opérateurs, qualités et `numero_version > 0`. | Ferme les valeurs invalides au niveau de la base sans multiplier les petites tables de référence. |
| `CHECK` d’intervalle | `date_fin IS NULL OR date_debut < date_fin`, même règle sur validité de règle et affectation technicien. | Rejette les durées nulles ou inversées. |
| `EXCLUDE USING gist` | Historique de zone par équipement; affectation par capteur. | Garantit l’absence de chevauchement malgré les écritures concurrentes, là où un contrôle applicatif seul serait sujet aux courses. |
| `CHECK` XOR | Sur `alerte`, exactement une FK cible non nulle. | Autorise une et une seule cible : équipement ou capteur. |
| FK composite nullables par paire | Sur la mesure déclencheuse, avec un `CHECK` de cohérence des deux colonnes. | PostgreSQL ignore une FK si l’une de ses colonnes est `NULL`; le `CHECK` impose que les deux soient soit présentes, soit absentes. |

### 3.2 XOR d’une alerte

```sql
CONSTRAINT ck_alerte_cible_xor CHECK (
  (id_equipement_cible IS NOT NULL)
  <> (id_capteur_cible IS NOT NULL)
)
```

L’opérateur booléen `<>` est un XOR : deux cibles absentes et deux cibles présentes sont refusées; une seule cible présente est acceptée. Une vérification supplémentaire par déclencheur compare cette cible à `regle_alerte.portee_cible`. Un `CHECK` SQL ne peut pas consulter une autre table.

## Étape 4 — Requêtes métier et index nécessaires

Les requêtes sont écrites avec des paramètres nommés (`:...`) à remplacer par des paramètres préparés du pilote. Le partitionnement de `mesure` n’annule pas l’intérêt des index locaux; une borne temporelle explicite permet au planificateur d’élaguer les partitions non concernées.

### Requête 1 — Équipements d’un bâtiment

**Usage et règle temporelle.** La localisation n’est pas une colonne de `equipement`; on consulte l’emplacement valide à l’instant demandé. Pour l’emplacement courant, fournir `CURRENT_TIMESTAMP` ou une valeur UTC. La contrainte d’exclusion garantit au plus une ligne de localisation pour un équipement à cet instant.

```sql
SELECT e.id_equipement, e.code_equipement, e.libelle, e.type_equipement,
       z.id_zone, z.code_zone, z.nom AS nom_zone
FROM smartcampus.zone AS z
JOIN smartcampus.historique_zone_equipement AS h
  ON h.id_zone = z.id_zone
JOIN smartcampus.equipement AS e
  ON e.id_equipement = h.id_equipement
WHERE z.id_batiment = :id_batiment
  AND h.date_debut <= :a_date
  AND (h.date_fin IS NULL OR :a_date < h.date_fin)
ORDER BY z.code_zone, e.code_equipement;
```

| Index adapté | Type | Rôle |
|---|---|---|
| PK `zone(id_zone)`; `ix_zone_id_batiment(id_batiment)` | B-tree | Récupère les zones du bâtiment. |
| `ix_hze_zone_date_debut` sur `(id_zone, date_debut DESC) INCLUDE (id_equipement, date_fin)` | B-tree couvrant | Cherche, par zone et date, les localisations candidates et limite les retours à la table. |
| PK `equipement(id_equipement)` | B-tree | Jointure vers l’équipement. |
| Index GiST créé par `ex_hze_equipement_periode` | GiST | Assure l’exclusion; utile aux recherches temporelles commençant par l’équipement, mais n’est pas à lui seul l’index principal de cette requête qui part du bâtiment/zone. |

Pour les listes « uniquement aujourd’hui », un index partiel `(id_zone, id_equipement) WHERE date_fin IS NULL` peut être ajouté si les mesures démontrent que cette variante domine. Il ne remplace pas l’index permettant les recherches historiques.

### Requête 2 — Dernières mesures d’un capteur

```sql
SELECT id_mesure, id_capteur, horodatage_evenement,
       horodatage_reception, valeur, unite, qualite
FROM smartcampus.mesure
WHERE id_capteur = :id_capteur
  AND horodatage_evenement >= :depuis
  AND horodatage_evenement < :jusqua
ORDER BY horodatage_evenement DESC
LIMIT :limite;
```

| Index adapté | Type | Rôle |
|---|---|---|
| `ix_mesure_capteur_evenement` sur `(id_capteur, horodatage_evenement DESC) INCLUDE (valeur, unite, qualite, horodatage_reception)` | B-tree composite couvrant, local aux partitions | Filtre d’abord le capteur, renvoie les observations dans l’ordre demandé et peut couvrir la projection sans inclure les colonnes dans la clé de tri. |
| Partitionnement mensuel par `horodatage_evenement` | RANGE | Élimine les partitions hors période grâce aux deux bornes du prédicat. |

Sans filtre temporel, toutes les partitions encore présentes peuvent devoir être consultées pour trouver les dernières mesures; limiter la période lorsque le besoin fonctionnel le permet. L’index est maintenu sur chaque partition et augmente le coût d’ingestion.

### Requête 3 — Interventions d’un équipement

```sql
SELECT i.id_intervention, i.reference, i.type_intervention,
       i.priorite, i.statut, i.date_ouverture, i.date_cloture
FROM smartcampus.intervention AS i
WHERE i.id_equipement = :id_equipement
ORDER BY i.date_ouverture DESC
LIMIT :limite;
```

| Index adapté | Type | Rôle |
|---|---|---|
| `ix_intervention_equipement_ouverture` sur `(id_equipement, date_ouverture DESC) INCLUDE (reference, type_intervention, priorite, statut, date_cloture)` | B-tree composite couvrant | Filtre par équipement et fournit directement l’ordre chronologique décroissant. |
| PK `intervention(id_intervention)` | B-tree | Sert aux liens et autres consultations par identifiant. |

Un index `(id_equipement, statut, date_ouverture)` ne remplace pas celui proposé si la requête ne filtre pas sur `statut`, car le statut intermédiaire empêche l’ordre global par date pour un équipement.

### Requête 4 — Interventions d’un technicien

Une intervention peut contenir plusieurs lignes d’affectation historiques pour un même technicien. `EXISTS` évite de répéter l’intervention dans le résultat.

```sql
SELECT i.id_intervention, i.reference, i.id_equipement,
       i.statut, i.date_ouverture, i.date_cloture
FROM smartcampus.intervention AS i
WHERE EXISTS (
  SELECT 1
  FROM smartcampus.affectation_technicien_intervention AS ati
  WHERE ati.id_intervention = i.id_intervention
    AND ati.id_technicien = :id_technicien
)
ORDER BY i.date_ouverture DESC
LIMIT :limite;
```

| Index adapté | Type | Rôle |
|---|---|---|
| `ix_ati_technicien_intervention` sur `(id_technicien, id_intervention)` | B-tree composite | Recherche les interventions liées à un technicien sans balayer toutes les affectations. |
| `ix_ati_intervention` sur `(id_intervention)` | B-tree | Rend efficace la vérification de l’`EXISTS` côté intervention et les opérations référentielles. |
| PK `intervention(id_intervention)`; optionnel `ix_intervention_ouverture` sur `(date_ouverture DESC)` | B-tree | Récupération et tri des interventions. Le tri après le filtre `EXISTS` peut rester nécessaire; l’index de date seul n’est rentable que si le volume justifie une recherche par ordre. |

### Requête 5 — Consommation mensuelle d’un bâtiment

La requête résout la localisation **à la date de chaque mesure** : capteur → équipement historique → zone historique → bâtiment. Les contraintes temporelles garantissent au plus un équipement associé par capteur et une zone par équipement à l’instant de l’observation.

**Hypothèse de calcul :** chaque mesure valide en `kWh` représente une consommation de période (delta) et peut être additionnée. Le MCD ne précise pas si le capteur envoie des deltas ou un compteur cumulatif; dans ce dernier cas, `SUM(valeur)` serait incorrect et il faut calculer les différences successives, traiter les remises à zéro et définir la période de rattachement des différences.

```sql
SELECT date_trunc('month', m.horodatage_evenement AT TIME ZONE 'UTC')::date
         AS mois_utc,
       SUM(m.valeur) AS consommation_kwh
FROM smartcampus.mesure AS m
JOIN smartcampus.affectation_capteur_equipement AS ace
  ON ace.id_capteur = m.id_capteur
 AND ace.date_debut <= m.horodatage_evenement
 AND (ace.date_fin IS NULL OR m.horodatage_evenement < ace.date_fin)
JOIN smartcampus.historique_zone_equipement AS h
  ON h.id_equipement = ace.id_equipement
 AND h.date_debut <= m.horodatage_evenement
 AND (h.date_fin IS NULL OR m.horodatage_evenement < h.date_fin)
JOIN smartcampus.zone AS z
  ON z.id_zone = h.id_zone
WHERE z.id_batiment = :id_batiment
  AND m.horodatage_evenement >= :debut
  AND m.horodatage_evenement < :fin
  AND m.qualite = 'VALIDE'
  AND m.unite = 'kWh'
GROUP BY date_trunc('month', m.horodatage_evenement AT TIME ZONE 'UTC')::date
ORDER BY mois_utc;
```

| Index adapté | Type | Rôle |
|---|---|---|
| `ix_mesure_capteur_evenement` `(id_capteur, horodatage_evenement DESC) INCLUDE (valeur, unite, qualite)` | B-tree composite couvrant | Lecture par capteur et fenêtre de temps; le filtre explicite sur l’événement permet aussi l’élagage des partitions. |
| `ix_ace_capteur_debut` `(id_capteur, date_debut DESC) INCLUDE (id_equipement, date_fin)` | B-tree composite couvrant | Résout l’affectation capteur-équipement à l’instant de la mesure. |
| Index GiST d’exclusion `(id_capteur, tstzrange(...))` | GiST | Garantit l’absence d’affectations simultanées; l’index B-tree reste utile pour le parcours par capteur et début. |
| `ix_hze_equipement_debut` `(id_equipement, date_debut DESC) INCLUDE (id_zone, date_fin)` | B-tree composite couvrant | Résout la zone historique d’un équipement identifié à la mesure. |
| Index GiST d’exclusion `(id_equipement, tstzrange(...))` | GiST | Garantit l’absence de localisations simultanées et aide aux recherches de période par équipement. |
| `ix_zone_id_batiment(id_batiment)` | B-tree | Filtre les zones du bâtiment; selon les cardinalités, le plan peut partir des partitions temporelles des mesures ou des zones. |

L’agrégat étant spatial et historique, des agrégats matérialisés ne sont à envisager qu’après validation de la définition énergétique et mesure du plan. Confirmer les plans avec `EXPLAIN (ANALYZE, BUFFERS)` sur un jeu représentatif.

### Synthèse des index proposés

| Nom indicatif | Définition | Requêtes |
|---|---|---|
| `ix_batiment_id_campus` | `batiment(id_campus)` | Jointures / liste des bâtiments d’un campus. |
| `ix_zone_id_batiment` | `zone(id_batiment)` | 1 et 5. |
| `ix_hze_zone_date_debut` | `historique_zone_equipement(id_zone, date_debut DESC) INCLUDE (id_equipement, date_fin)` | 1. |
| `ix_hze_equipement_debut` | `historique_zone_equipement(id_equipement, date_debut DESC) INCLUDE (id_zone, date_fin)` | 5 et historique par équipement. |
| `ix_ace_capteur_debut` | `affectation_capteur_equipement(id_capteur, date_debut DESC) INCLUDE (id_equipement, date_fin)` | 5 et historique du capteur. |
| `ix_mesure_capteur_evenement` | `mesure(id_capteur, horodatage_evenement DESC) INCLUDE (valeur, unite, qualite, horodatage_reception)` | 2 et 5; index local à chaque partition. |
| `ix_intervention_equipement_ouverture` | `intervention(id_equipement, date_ouverture DESC) INCLUDE (reference, type_intervention, priorite, statut, date_cloture)` | 3. |
| `ix_ati_technicien_intervention` | `affectation_technicien_intervention(id_technicien, id_intervention)` | 4. |
| `ix_ati_intervention` | `affectation_technicien_intervention(id_intervention)` | 4, jointures et contrôles FK. |

Les PK et contraintes `UNIQUE` créent déjà leurs index B-tree : ne pas les dupliquer. Les FK seules ne créent pas automatiquement un index côté enfant dans PostgreSQL. Indexer les FK lorsque les jointures, recherches entrantes ou opérations référentielles le justifient. Chaque index secondaire consomme du stockage et ralentit les insertions IoT; mesurer avant d’ajouter des index partiels ou couvrants supplémentaires.

## Étape 5 — Contrôler les anomalies (tests d’intégrité)

Les résultats ci-dessous découlent du DDL PostgreSQL de référence. PostgreSQL fournit un SQLSTATE stable, mais le texte complet du message varie selon la version, le nom de contrainte et la langue du serveur; les textes cités sont les formes attendues en anglais (`lc_messages` par défaut), accompagnées du SQLSTATE.

| Cas | Opération de test | Résultat | Contrainte / message attendu |
|---|---|---|---|
| 1. Équipement avec zone inexistante | Le MLD historisé ne possède pas de `id_zone` dans `equipement`. Créer l’équipement puis tenter d’insérer une ligne `historique_zone_equipement(id_equipement, id_zone, date_debut)` avec un `id_zone` absent. Si la création et la localisation sont dans la même transaction, annuler la transaction en cas d’échec. | L’insertion de l’équipement seul est valide; l’insertion de localisation est **refusée** (donc la transaction complète peut être annulée). Un `INSERT ... equipement(id_zone, ...)` est lui-même refusé car la colonne n’existe pas. | FK `historique_zone_equipement.id_zone -> zone.id_zone`; SQLSTATE `23503`, `violates foreign key constraint "..."`. Sur une insertion directe dans `equipement`, SQLSTATE `42703`, `column "id_zone" of relation "equipement" does not exist`. |
| 2. Bâtiment sans nom | `INSERT INTO batiment (..., nom, ...) VALUES (..., NULL, ...)` ou omission de `nom`. | **Refusée** dans les deux cas, car la colonne est `NOT NULL` et sans valeur par défaut. | `NOT NULL` sur `batiment.nom`; SQLSTATE `23502`, `null value in column "nom" ... violates not-null constraint`. |
| 3. Deux capteurs avec le même identifiant métier | Insérer deux lignes `capteur` avec des `identifiant_externe` identiques. | Première insertion acceptée si l’identifiant est libre; seconde **refusée**, y compris en concurrence. | `UNIQUE(capteur.identifiant_externe)`; SQLSTATE `23505`, `duplicate key value violates unique constraint "capteur_identifiant_externe_key"` (nom exact selon le DDL). |
| 4. Fin d’affectation antérieure au début | Insérer `affectation_capteur_equipement` avec `date_fin < date_debut`. | **Refusée** avant même le contrôle de chevauchement. | `CHECK ck_ace_intervalle`; SQLSTATE `23514`, `violates check constraint "ck_ace_intervalle"`. `date_fin = date_debut` est également refusée. |
| 5. Intervention sans équipement associé | Insérer une ligne `intervention` avec `id_equipement = NULL` ou omettre cette colonne. | **Refusée** : l’association équipement–intervention est obligatoire dans le MCD retenu. | `NOT NULL` sur `intervention.id_equipement`; SQLSTATE `23502`. Si une valeur non nulle mais inexistante est fournie, la FK intervient à la place (SQLSTATE `23503`). |

Pour vérifier également les intervalles simultanés, deux localisations qui se chevauchent pour le même équipement déclenchent SQLSTATE `23P01` (`exclusion_violation`) sur `ex_hze_equipement_periode`; l’équivalent pour un même capteur déclenche `ex_ace_capteur_periode`. Deux intervalles contigus `[t0,t1[` et `[t1,t2[` sont autorisés.

## Conclusion

Le passage au MLD conserve les cardinalités et les associations historisées du MCD; le MPD traduit ces règles par des FK, `NOT NULL`, `UNIQUE`, `CHECK` et contraintes d’exclusion. Les index sont alignés sur les cinq lectures demandées et sur la résolution de l’emplacement historique au moment de la mesure. Le principal point fonctionnel restant à arbitrer pour les consommations est la sémantique du signal énergétique (delta ou compteur cumulatif) et ses unités de calcul.
