# SmartCampus — Étape 2 : Modèle Conceptuel de Données

## 1. Objet et périmètre

Ce livrable décrit le Modèle Conceptuel de Données (MCD) de la plateforme SmartCampus : entités, identifiants, associations, cardinalités et dictionnaire de données. Il prolonge l’analyse métier de l’étape 1 et couvre le référentiel du campus, les équipements et capteurs, les mesures IoT, les règles et alertes, ainsi que les interventions et leurs techniciens.

Le dimensionnement de référence est de **8 bâtiments**, **environ 160 zones**, **250 équipements** et **600 capteurs**. Chaque capteur émet nominalement une mesure toutes les 30 secondes; les mesures brutes sont conservées pendant 12 mois. Ces volumes n’imposent pas de cardinalités fixes aux référentiels : ce sont des ordres de grandeur, et non des contraintes d’intégrité.

Le MCD est indépendant du moteur de stockage. Les types du dictionnaire sont conceptuels; ils ne constituent pas encore un choix de types SQL ou de stratégie de partitionnement.

## 2. Conventions de lecture

- **Identifiant** : attribut marqué `PK`, unique et obligatoire dans son entité. Les identifiants techniques sont des entiers non signés de 64 bits, sauf les identifiants de protocole externes, qui sont des chaînes.
- **Cardinalité** : les deux nombres placés auprès d’une entité indiquent le minimum et le maximum de participations d’une occurrence à l’association. `1,1` signifie exactement une; `0,N` signifie zéro, une ou plusieurs.
- **Mandatif / Optionnel** : l’optionalité porte sur un attribut ou une association à une occurrence donnée, indépendamment de la nullabilité qui sera choisie en base.
- **Attributs temporels** : les horodatages sont stockés en UTC. Les intervalles d’effet sont semi-ouverts `[début, fin[`; une fin absente désigne l’intervalle actuellement ouvert.
- Les noms de relation tels que `id_zone` dans le dictionnaire désignent les identifiants des entités reliées. Au niveau conceptuel, ils sont portés par les associations, et non répétés comme clés étrangères dans les entités.
- Les entités d’association historisées possèdent un identifiant propre afin de tracer sans ambiguïté chaque période d’affectation.

## 3. Vue d’ensemble du MCD

```text
Campus (1,N) ── contenir ── (1,1) Bâtiment
Bâtiment (1,N) ── contenir ── (1,1) Zone

Equipement (0,N) ── Historique_Zone_Equipement ── (0,N) Zone
    Chaque occurrence d'historique concerne exactement 1 équipement et 1 zone.

Equipement (0,N) ── Affectation_Capteur_Equipement ── (0,N) Capteur
    Chaque occurrence d'affectation concerne exactement 1 équipement et 1 capteur.

Capteur (0,N) ── produire ── (1,1) Mesure

Regle_Alerte (0,N) ── déclencher ── (1,1) Alerte
Alerte (0,1) ── concerner ── (0,N) Equipement
Alerte (0,1) ── concerner ── (0,N) Capteur
    Contrainte XOR : toute alerte concerne exactement un équipement ou un capteur.
Mesure (0,N) ── être_à_l'origine_de ── (0,1) Alerte

Equipement (0,N) ── faire_l'objet_de ── (1,1) Intervention
Utilisateur (0,N) ── demander ── (0,1) Intervention
Intervention (0,N) ── Affectation_Technicien_Intervention ── (0,N) Technicien
Alerte (0,N) ── Lien_Alerte_Intervention ── (0,N) Intervention

Utilisateur (0,1) ── être_le_compte_de ── (1,1) Technicien
```

Cette vue met en évidence les associations historisées. Le détail des cardinalités et les contraintes métier sont précisés à la section 5.

## 4. Entités et identifiants

### 4.1 Référentiel spatial

| Entité | Identifiant | Attributs principaux | Description |
|---|---|---|---|
| **Campus** | `id_campus` (PK) | `code_campus`, `nom`, `adresse`, `statut` | Regroupe les bâtiments supervisés. Le nombre de campus n’est pas fixé par le cas. |
| **Bâtiment** | `id_batiment` (PK) | `code_batiment`, `nom`, `adresse`, `statut` | Bâtiment rattaché à un seul campus. |
| **Zone** | `id_zone` (PK) | `code_zone`, `nom`, `type_zone`, `étage`, `statut` | Espace identifié d’un bâtiment, par exemple un étage, un local technique ou une salle. |

### 4.2 Parc technique et données IoT

| Entité | Identifiant | Attributs principaux | Description |
|---|---|---|---|
| **Equipement** | `id_equipement` (PK) | `code_equipement`, `libelle`, `type_equipement`, `marque`, `modele`, `numero_serie`, `date_mise_en_service`, `statut` | Actif technique maintenu et supervisé. Sa localisation est portée par son historique, pas par un emplacement courant écrasable. |
| **Historique_Zone_Equipement** | `id_historique_zone` (PK) | `date_debut`, `date_fin`, `motif_deplacement` | Occurrence d’occupation d’une zone par un équipement pendant un intervalle donné. |
| **Capteur** | `id_capteur` (PK) | `identifiant_externe`, `type_capteur`, `libelle`, `unite_nominale`, `statut`, `date_mise_en_service` | Source de mesures IoT. Un capteur peut être sans équipement associé, par exemple avant sa mise en service ou après son retrait. |
| **Affectation_Capteur_Equipement** | `id_affectation_capteur` (PK) | `date_debut`, `date_fin`, `motif_affectation` | Période durant laquelle un capteur est rattaché à un équipement donné. |
| **Mesure** | `id_mesure` (PK) | `horodatage_evenement`, `horodatage_reception`, `valeur`, `unite`, `qualite`, `cle_deduplication` | Observation d’un capteur. La date de l’événement et la date de réception sont distinctes pour gérer les retards de transmission. |

### 4.3 Alertes et interventions

| Entité | Identifiant | Attributs principaux | Description |
|---|---|---|---|
| **Regle_Alerte** | `id_regle_alerte` (PK) | `code_regle`, `numero_version`, `libelle`, `portee_cible`, `indicateur`, `operateur`, `seuil`, `severite`, `date_debut_validite`, `date_fin_validite`, `statut` | Version d’une règle de déclenchement. L’alerte conserve le lien vers la version exacte appliquée. |
| **Alerte** | `id_alerte` (PK) | `date_creation`, `severite_constatee`, `statut`, `date_acquittement`, `date_cloture`, `commentaire` | Événement détecté par une règle et concernant un seul équipement ou un seul capteur. |
| **Intervention** | `id_intervention` (PK) | `reference`, `type_intervention`, `priorite`, `statut`, `date_ouverture`, `date_planifiee`, `date_debut`, `date_cloture`, `description`, `compte_rendu` | Action de maintenance portant sur un équipement, demandée par un utilisateur ou créée dans un autre contexte (par exemple préventif/automatique). |
| **Technicien** | `id_technicien` (PK) | `matricule`, `specialite`, `statut` | Profil d’un utilisateur habilité à réaliser des interventions. |
| **Affectation_Technicien_Intervention** | `id_affectation_technicien` (PK) | `date_affectation`, `date_fin_affectation`, `role_dans_intervention`, `commentaire` | Participation d’un technicien à une intervention sur une période; permet plusieurs techniciens et l’historique des réaffectations. |
| **Lien_Alerte_Intervention** | `id_lien_alerte_intervention` (PK) | `date_liaison`, `commentaire` | Relie une alerte à une intervention sans imposer une correspondance un-à-un. |

### 4.4 Identité des demandeurs et utilisateurs

| Entité | Identifiant | Attributs principaux | Description |
|---|---|---|---|
| **Utilisateur** | `id_utilisateur` (PK) | `courriel`, `prenom`, `nom`, `statut`, `date_creation` | Identité humaine utilisée pour tracer une demande d’intervention et rattacher un profil technicien. Les rôles et permissions détaillés relèvent de la gestion des habilitations. |

L’association `Utilisateur—Technicien` représente un profil technicien rattaché à un compte utilisateur. Les capteurs et passerelles ne sont pas modélisés comme utilisateurs humains : leur identité technique et leur authentification relèvent de l’intégration IoT.

## 5. Associations, sens et cardinalités

### 5.1 Campus, bâtiments et zones

| Association | Cardinalités | Sens et règles |
|---|---|---|
| **Contenir** (Campus — Bâtiment) | Campus `1,N` ; Bâtiment `1,1` | Un campus regroupe un ou plusieurs bâtiments; chaque bâtiment appartient à un seul campus. Le minimum `1` côté campus traduit la règle de gestion « un campus regroupe plusieurs bâtiments ». Si le référentiel doit autoriser la création d’un campus vide, le minimum devra devenir `0`. |
| **Contenir** (Bâtiment — Zone) | Bâtiment `1,N` ; Zone `1,1` | Un bâtiment contient une ou plusieurs zones; chaque zone appartient à un seul bâtiment. Les quelque 160 zones du cas ne fixent pas une quantité minimale ou maximale par bâtiment au-delà de cette règle métier. |

### 5.2 Localisation et affectation temporelles

| Association | Cardinalités | Attributs propres et règles |
|---|---|---|
| **Historique_Zone_Equipement** (Equipement — Zone) | Equipement `0,N` ; Zone `0,N` | L’occurrence relie exactement un équipement à exactement une zone. `date_debut` est obligatoire; `date_fin` et `motif_deplacement` sont optionnels. Un équipement peut avoir plusieurs emplacements successifs; une zone peut accueillir plusieurs équipements au fil du temps ou simultanément. |
| **Affectation_Capteur_Equipement** (Equipement — Capteur) | Equipement `0,N` ; Capteur `0,N` | L’occurrence relie exactement un équipement à exactement un capteur. `date_debut` est obligatoire; `date_fin` et `motif_affectation` sont optionnels. Un équipement peut recevoir plusieurs capteurs; un capteur peut être réaffecté à plusieurs équipements au cours de son cycle de vie, mais au plus à un équipement à un instant donné. |

**Contraintes temporelles communes :**

1. Pour un équipement, les intervalles de son historique de localisation ne se chevauchent pas. Une seule occurrence peut être en cours à une date donnée.
2. Pour un capteur, les intervalles d’affectation à un équipement ne se chevauchent pas. Un capteur ne peut donc pas être simultanément rattaché à deux équipements.
3. Un intervalle respecte `date_debut < date_fin` lorsque la fin est renseignée. Les bornes sont semi-ouvertes pour permettre deux affectations consécutives sans chevauchement.
4. Les identifiants des zones et équipements référencés ne sont pas supprimés physiquement tant qu’un historique ou une mesure en dépend; un statut d’inactivité ou un archivage préserve la traçabilité.
5. Au déploiement, la règle « tout équipement en service a exactement une localisation courante » est à appliquer comme contrainte conditionnelle de statut, et non comme participation minimale générale : un équipement peut être créé avant installation ou rester en parc après retrait.

### 5.3 Capteurs, mesures, règles et alertes

| Association | Cardinalités | Attributs propres et règles |
|---|---|---|
| **Produire** (Capteur — Mesure) | Capteur `0,N` ; Mesure `1,1` | Chaque mesure provient d’un seul capteur; un capteur peut n’avoir encore aucune mesure ou en produire de très nombreuses. La mesure porte les horodatages, la valeur, l’unité et la qualité. |
| **Déclencher** (Regle_Alerte — Alerte) | Regle_Alerte `0,N` ; Alerte `1,1` | Chaque alerte est rattachée à une version précise d’une règle; une version peut ne déclencher aucune alerte ou en déclencher plusieurs. `code_regle` et `numero_version` forment une clé candidate unique. |
| **Concerner** (Equipement — Alerte) | Equipement `0,N` ; Alerte `0,1` | Une alerte concerne au plus un équipement via cette association. |
| **Concerner** (Capteur — Alerte) | Capteur `0,N` ; Alerte `0,1` | Une alerte concerne au plus un capteur via cette association. |
| **Etre_a_l_origine_de** (Mesure — Alerte) | Mesure `0,N` ; Alerte `0,1` | Une mesure peut être à l’origine de plusieurs alertes. Le lien est optionnel pour permettre les alertes non directement issues d’une mesure unique; pour une alerte automatiquement déclenchée par une mesure, il doit être renseigné. |

**Contrainte XOR sur la cible d’une alerte :** chaque alerte est liée à **exactement un** équipement **ou** à **exactement un** capteur, jamais aux deux simultanément et jamais à aucun des deux. Cette contrainte globale complète les cardinalités locales `0,1` des deux associations. La cible doit être compatible avec `portee_cible` de la version de règle. Un déclenchement sur une mesure n’impose pas que la cible soit le capteur producteur : le moteur de règles peut, par exemple, créer une alerte portant sur l’équipement auquel le capteur était rattaché à l’instant considéré.

### 5.4 Interventions, demandeurs et techniciens

| Association | Cardinalités | Attributs propres et règles |
|---|---|---|
| **Faire_l_objet_de** (Equipement — Intervention) | Equipement `0,N` ; Intervention `1,1` | Une intervention porte sur exactement un équipement; un équipement peut n’avoir aucune intervention ou en avoir plusieurs. Le choix d’un équipement obligatoire est celui retenu pour la maintenance du parc; les interventions visant uniquement une zone ou un capteur sont hors périmètre à ce stade. |
| **Demander** (Utilisateur — Intervention) | Utilisateur `0,N` ; Intervention `0,1` | Une intervention peut avoir un demandeur humain ou être créée sans demandeur utilisateur (par exemple par une planification préventive ou un traitement automatique). Un utilisateur peut demander plusieurs interventions. |
| **Etre_le_compte_de** (Utilisateur — Technicien) | Utilisateur `0,1` ; Technicien `1,1` | Tout profil technicien est rattaché à exactement un compte utilisateur; un utilisateur a zéro ou un profil technicien. Cette association est exclusive et n’implique pas que tout utilisateur ait le rôle de technicien. |
| **Affectation_Technicien_Intervention** (Technicien — Intervention) | Technicien `0,N` ; Intervention `0,N` | Chaque occurrence relie exactement un technicien à une intervention. `date_affectation` est obligatoire; `date_fin_affectation`, `role_dans_intervention` et `commentaire` sont optionnels. L’historique permet les affectations successives, la participation multiple et plusieurs techniciens simultanés. |
| **Lien_Alerte_Intervention** (Alerte — Intervention) | Alerte `0,N` ; Intervention `0,N` | Association facultative et multiple dans les deux sens : une intervention peut traiter plusieurs alertes et une alerte peut être suivie par plusieurs interventions. `date_liaison` est obligatoire; `commentaire` est optionnel. |

Les statuts et transitions de cycle de vie (alerte et intervention), les vocabulaires de priorité/sévérité et les rôles d’affectation doivent être définis dans un référentiel contrôlé ou par des contraintes applicatives. Ils ne modifient pas les cardinalités du MCD.

## 6. Dictionnaire de données

### 6.1 Entités de référentiel

#### Campus

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_campus` (PK) | Campus | Entier 64 bits | Mandatif | Identifiant technique stable du campus. |
| `code_campus` | Campus | Chaîne (80) | Mandatif | Code métier unique du campus. |
| `nom` | Campus | Chaîne (160) | Mandatif | Nom usuel du campus. |
| `adresse` | Campus | Chaîne (500) | Optionnel | Adresse ou description de localisation. |
| `statut` | Campus | Enumération / chaîne (30) | Mandatif | État du campus (actif, inactif, archivé, etc.). |

#### Bâtiment

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_batiment` (PK) | Bâtiment | Entier 64 bits | Mandatif | Identifiant technique stable du bâtiment. |
| `code_batiment` | Bâtiment | Chaîne (80) | Mandatif | Code métier unique du bâtiment. |
| `nom` | Bâtiment | Chaîne (160) | Mandatif | Nom usuel du bâtiment. |
| `adresse` | Bâtiment | Chaîne (500) | Optionnel | Adresse ou indication d’accès au bâtiment. |
| `statut` | Bâtiment | Enumération / chaîne (30) | Mandatif | État du bâtiment dans le référentiel. |

#### Zone

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_zone` (PK) | Zone | Entier 64 bits | Mandatif | Identifiant technique stable de la zone. |
| `code_zone` | Zone | Chaîne (80) | Mandatif | Code unique de la zone à l’intérieur de son bâtiment. |
| `nom` | Zone | Chaîne (160) | Mandatif | Nom d’affichage de la zone. |
| `type_zone` | Zone | Chaîne contrôlée (50) | Mandatif | Catégorie de zone (salle, local technique, étage, etc.). |
| `etage` | Zone | Chaîne (40) | Optionnel | Étage ou niveau du bâtiment. |
| `statut` | Zone | Enumération / chaîne (30) | Mandatif | État de la zone (active, fermée, archivée, etc.). |

#### Equipement

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_equipement` (PK) | Equipement | Entier 64 bits | Mandatif | Identifiant technique stable de l’équipement. |
| `code_equipement` | Equipement | Chaîne (100) | Mandatif | Référence interne unique de l’équipement. |
| `libelle` | Equipement | Chaîne (200) | Mandatif | Désignation compréhensible par les exploitants. |
| `type_equipement` | Equipement | Chaîne contrôlée (80) | Mandatif | Famille fonctionnelle (ex. HVAC, éclairage, comptage). |
| `marque` | Equipement | Chaîne (100) | Optionnel | Fabricant ou marque. |
| `modele` | Equipement | Chaîne (100) | Optionnel | Modèle déclaré. |
| `numero_serie` | Equipement | Chaîne (120) | Optionnel | Numéro de série du fabricant. |
| `date_mise_en_service` | Equipement | Date | Optionnel | Date de mise en service opérationnelle. |
| `statut` | Equipement | Enumération / chaîne (30) | Mandatif | État du parc (en préparation, en service, hors service, retiré, etc.). |

#### Utilisateur

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_utilisateur` (PK) | Utilisateur | Entier 64 bits | Mandatif | Identifiant technique de l’utilisateur. |
| `courriel` | Utilisateur | Chaîne (254) | Mandatif | Adresse unique servant d’identifiant de contact ou de compte. |
| `prenom` | Utilisateur | Chaîne (100) | Mandatif | Prénom de l’utilisateur. |
| `nom` | Utilisateur | Chaîne (100) | Mandatif | Nom de l’utilisateur. |
| `statut` | Utilisateur | Enumération / chaîne (30) | Mandatif | État du compte (actif, suspendu, désactivé). |
| `date_creation` | Utilisateur | Horodatage UTC | Mandatif | Date de création du compte. |

### 6.2 Entités IoT, mesures et historisation

#### Historique_Zone_Equipement — association historisée

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_historique_zone` (PK) | Historique_Zone_Equipement | Entier 64 bits | Mandatif | Identifiant d’une période de localisation. |
| `id_equipement` | Historique_Zone_Equipement | Entier 64 bits (identifiant Equipement) | Mandatif | Équipement localisé pendant la période. |
| `id_zone` | Historique_Zone_Equipement | Entier 64 bits (identifiant Zone) | Mandatif | Zone occupée pendant la période. |
| `date_debut` | Historique_Zone_Equipement | Horodatage UTC | Mandatif | Début d’effet de la localisation. |
| `date_fin` | Historique_Zone_Equipement | Horodatage UTC | Optionnel | Fin exclusive; absente tant que la localisation est en cours. |
| `motif_deplacement` | Historique_Zone_Equipement | Chaîne (500) | Optionnel | Motif du changement de zone ou de la fin de localisation. |

#### Capteur

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_capteur` (PK) | Capteur | Entier 64 bits | Mandatif | Identifiant interne stable du capteur. |
| `identifiant_externe` | Capteur | Chaîne (160) | Mandatif | Identité unique transmise par le capteur ou sa passerelle. |
| `type_capteur` | Capteur | Chaîne contrôlée (80) | Mandatif | Grandeur ou famille mesurée (température, énergie, etc.). |
| `libelle` | Capteur | Chaîne (200) | Mandatif | Nom lisible du capteur dans l’interface. |
| `unite_nominale` | Capteur | Chaîne (30) | Mandatif | Unité attendue, sans remplacer l’unité effectivement portée par chaque mesure. |
| `statut` | Capteur | Enumération / chaîne (30) | Mandatif | État opérationnel du capteur. |
| `date_mise_en_service` | Capteur | Date | Optionnel | Date de début d’exploitation du capteur. |

#### Affectation_Capteur_Equipement — association historisée

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_affectation_capteur` (PK) | Affectation_Capteur_Equipement | Entier 64 bits | Mandatif | Identifiant d’une période de rattachement capteur-équipement. |
| `id_capteur` | Affectation_Capteur_Equipement | Entier 64 bits (identifiant Capteur) | Mandatif | Capteur concerné. |
| `id_equipement` | Affectation_Capteur_Equipement | Entier 64 bits (identifiant Equipement) | Mandatif | Équipement supervisé pendant la période. |
| `date_debut` | Affectation_Capteur_Equipement | Horodatage UTC | Mandatif | Début d’effet du rattachement. |
| `date_fin` | Affectation_Capteur_Equipement | Horodatage UTC | Optionnel | Fin exclusive; absente tant que le rattachement est en cours. |
| `motif_affectation` | Affectation_Capteur_Equipement | Chaîne (500) | Optionnel | Motif d’installation, de réaffectation ou de retrait. |

#### Mesure

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_mesure` (PK) | Mesure | Entier 64 bits | Mandatif | Identifiant interne de la mesure. À dimensionner pour plusieurs centaines de millions de lignes. |
| `id_capteur` | Mesure | Entier 64 bits (identifiant Capteur) | Mandatif | Capteur producteur de la mesure. |
| `horodatage_evenement` | Mesure | Horodatage UTC | Mandatif | Instant auquel la mesure a été observée par le capteur. |
| `horodatage_reception` | Mesure | Horodatage UTC | Mandatif | Instant où la plateforme a reçu la mesure. |
| `valeur` | Mesure | Décimal haute précision | Mandatif | Valeur numérique observée. La précision maximale dépend du type de capteur. |
| `unite` | Mesure | Chaîne (30) | Mandatif | Unité de la valeur au moment de la mesure. |
| `qualite` | Mesure | Enumération / chaîne (30) | Mandatif | Indique la validité ou la qualité (valide, invalide, estimée, etc.). |
| `cle_deduplication` | Mesure | Chaîne (160) | Optionnel | Identifiant de message fourni par la source, s’il existe; facilite la déduplication des retransmissions. |

Une mesure est identifiée par `id_mesure`, et non par le seul couple capteur/horodatage : deux observations peuvent partager un horodatage selon la précision de l’horloge, et les règles de déduplication ne sont pas encore spécifiées. Si une clé source fiable est disponible, son unicité est contrôlée dans le périmètre de la source.

### 6.3 Entités d’alertes et d’interventions

#### Regle_Alerte

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_regle_alerte` (PK) | Regle_Alerte | Entier 64 bits | Mandatif | Identifiant de la version de règle. |
| `code_regle` | Regle_Alerte | Chaîne (100) | Mandatif | Code stable identifiant la règle logique. |
| `numero_version` | Regle_Alerte | Entier | Mandatif | Numéro de version; unique avec `code_regle`. |
| `libelle` | Regle_Alerte | Chaîne (200) | Mandatif | Nom de la règle. |
| `portee_cible` | Regle_Alerte | Enumération (EQUIPEMENT, CAPTEUR) | Mandatif | Type de cible attendu par la règle. |
| `indicateur` | Regle_Alerte | Chaîne (80) | Mandatif | Grandeur ou indicateur évalué. |
| `operateur` | Regle_Alerte | Enumération / chaîne (20) | Mandatif | Comparaison appliquée au seuil (>, >=, <, <=, etc.). |
| `seuil` | Regle_Alerte | Décimal haute précision | Mandatif | Valeur de seuil de la règle. |
| `severite` | Regle_Alerte | Enumération / chaîne (30) | Mandatif | Niveau de sévérité configuré. |
| `date_debut_validite` | Regle_Alerte | Horodatage UTC | Mandatif | Début d’application de cette version. |
| `date_fin_validite` | Regle_Alerte | Horodatage UTC | Optionnel | Fin exclusive de validité. |
| `statut` | Regle_Alerte | Enumération / chaîne (30) | Mandatif | État de publication de la version. |

#### Alerte

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_alerte` (PK) | Alerte | Entier 64 bits | Mandatif | Identifiant unique de l’alerte. |
| `id_regle_alerte` | Alerte | Entier 64 bits (identifiant Regle_Alerte) | Mandatif | Version précise de la règle ayant produit l’alerte. |
| `date_creation` | Alerte | Horodatage UTC | Mandatif | Date de création dans la plateforme. |
| `severite_constatee` | Alerte | Enumération / chaîne (30) | Mandatif | Sévérité associée à cette occurrence. |
| `statut` | Alerte | Enumération / chaîne (30) | Mandatif | État de traitement de l’alerte. |
| `date_acquittement` | Alerte | Horodatage UTC | Optionnel | Date à laquelle l’alerte a été acquittée. |
| `date_cloture` | Alerte | Horodatage UTC | Optionnel | Date de clôture de l’alerte. |
| `commentaire` | Alerte | Chaîne longue | Optionnel | Contexte ou commentaire de suivi. |
| `id_equipement_cible` (association) | Alerte | Entier 64 bits (identifiant Equipement) | Optionnel | Renseigné si la cible est un équipement; exclusif avec `id_capteur_cible`. |
| `id_capteur_cible` (association) | Alerte | Entier 64 bits (identifiant Capteur) | Optionnel | Renseigné si la cible est un capteur; exclusif avec `id_equipement_cible`. |
| `id_mesure_declencheuse` (association) | Alerte | Entier 64 bits (identifiant Mesure) | Optionnel | Mesure directement à l’origine de l’alerte, lorsqu’elle existe. |

Les deux identifiants de cible sont présentés ici pour rendre explicite la contrainte XOR dans une représentation logique possible. Au MCD, ils correspondent aux deux associations `Concerner`; la contrainte « exactement une cible » est nécessairement contrôlée au niveau conceptuel et applicatif.

#### Intervention

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_intervention` (PK) | Intervention | Entier 64 bits | Mandatif | Identifiant technique de l’intervention. |
| `reference` | Intervention | Chaîne (100) | Mandatif | Référence lisible et unique de suivi. |
| `id_equipement` | Intervention | Entier 64 bits (identifiant Equipement) | Mandatif | Équipement concerné par l’intervention. |
| `id_demandeur` (association) | Intervention | Entier 64 bits (identifiant Utilisateur) | Optionnel | Utilisateur à l’origine de la demande, le cas échéant. |
| `type_intervention` | Intervention | Enumération / chaîne (30) | Mandatif | Nature de l’action (corrective, préventive, etc.). |
| `priorite` | Intervention | Enumération / chaîne (30) | Mandatif | Niveau de priorité opérationnelle. |
| `statut` | Intervention | Enumération / chaîne (30) | Mandatif | État du cycle de vie de l’intervention. |
| `date_ouverture` | Intervention | Horodatage UTC | Mandatif | Date de création ou d’ouverture. |
| `date_planifiee` | Intervention | Horodatage UTC | Optionnel | Date prévue pour la réalisation. |
| `date_debut` | Intervention | Horodatage UTC | Optionnel | Date de début effectif. |
| `date_cloture` | Intervention | Horodatage UTC | Optionnel | Date de fin effective. |
| `description` | Intervention | Chaîne longue | Mandatif | Motif, symptôme ou objet de l’intervention. |
| `compte_rendu` | Intervention | Chaîne longue | Optionnel | Travail réalisé et résultat constaté. |

#### Technicien

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_technicien` (PK) | Technicien | Entier 64 bits | Mandatif | Identifiant du profil technicien. |
| `id_utilisateur` (association) | Technicien | Entier 64 bits (identifiant Utilisateur) | Mandatif | Compte humain associé au profil. Unique : un compte ne porte au plus qu’un profil technicien. |
| `matricule` | Technicien | Chaîne (80) | Mandatif | Référence professionnelle unique du technicien. |
| `specialite` | Technicien | Chaîne contrôlée (100) | Optionnel | Domaine de compétence du technicien. |
| `statut` | Technicien | Enumération / chaîne (30) | Mandatif | Disponibilité administrative ou opérationnelle du profil. |

#### Affectation_Technicien_Intervention — association porteuse d’attributs

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_affectation_technicien` (PK) | Affectation_Technicien_Intervention | Entier 64 bits | Mandatif | Identifiant de l’occurrence d’affectation. |
| `id_intervention` | Affectation_Technicien_Intervention | Entier 64 bits (identifiant Intervention) | Mandatif | Intervention à laquelle le technicien participe. |
| `id_technicien` | Affectation_Technicien_Intervention | Entier 64 bits (identifiant Technicien) | Mandatif | Technicien affecté. |
| `date_affectation` | Affectation_Technicien_Intervention | Horodatage UTC | Mandatif | Début de participation ou d’affectation. |
| `date_fin_affectation` | Affectation_Technicien_Intervention | Horodatage UTC | Optionnel | Fin exclusive de l’affectation en cours ou passée. |
| `role_dans_intervention` | Affectation_Technicien_Intervention | Chaîne contrôlée (40) | Optionnel | Rôle éventuel (principal, soutien, spécialiste, etc.). |
| `commentaire` | Affectation_Technicien_Intervention | Chaîne longue | Optionnel | Information contextuelle sur la participation. |

#### Lien_Alerte_Intervention — association porteuse d’attributs

| Nom de l’élément | Entité/association | Type de données | Optionalité | Description métier |
|---|---|---|---|---|
| `id_lien_alerte_intervention` (PK) | Lien_Alerte_Intervention | Entier 64 bits | Mandatif | Identifiant du lien traçable entre alerte et intervention. |
| `id_alerte` | Lien_Alerte_Intervention | Entier 64 bits (identifiant Alerte) | Mandatif | Alerte prise en compte ou traitée. |
| `id_intervention` | Lien_Alerte_Intervention | Entier 64 bits (identifiant Intervention) | Mandatif | Intervention liée à l’alerte. |
| `date_liaison` | Lien_Alerte_Intervention | Horodatage UTC | Mandatif | Date de création du lien. |
| `commentaire` | Lien_Alerte_Intervention | Chaîne longue | Optionnel | Motif ou contexte du lien. |

### 6.4 Associations simples sans attribut propre

Les identifiants des occurrences participantes sont obligatoires dans les associations ci-dessous. Ils sont des références aux identifiants des entités et non des attributs métier supplémentaires.

| Association | Éléments identifiants reliés | Type | Optionalité | Description métier |
|---|---|---|---|---|
| Contenir (Campus — Bâtiment) | `id_campus`, `id_batiment` | Identifiants Entier 64 bits | Mandatif | Rattachement de chaque bâtiment à son campus unique. |
| Contenir (Bâtiment — Zone) | `id_batiment`, `id_zone` | Identifiants Entier 64 bits | Mandatif | Rattachement de chaque zone à son bâtiment unique. |
| Demander (Utilisateur — Intervention) | `id_utilisateur`, `id_intervention` | Identifiants Entier 64 bits | Association optionnelle côté Intervention | Trace le demandeur humain, s’il existe; aucun attribut propre supplémentaire. |
| Etre_le_compte_de (Utilisateur — Technicien) | `id_utilisateur`, `id_technicien` | Identifiants Entier 64 bits | Mandatif pour Technicien | Relie un profil technicien à son compte utilisateur. |
| Concerner (Equipement — Alerte) | `id_equipement`, `id_alerte` | Identifiants Entier 64 bits | Association optionnelle côté Alerte | Première branche de la cible d’alerte; exclusive avec la branche Capteur. |
| Concerner (Capteur — Alerte) | `id_capteur`, `id_alerte` | Identifiants Entier 64 bits | Association optionnelle côté Alerte | Deuxième branche de la cible d’alerte; exclusive avec la branche Equipement. |
| Etre_a_l_origine_de (Mesure — Alerte) | `id_mesure`, `id_alerte` | Identifiants Entier 64 bits | Association optionnelle côté Alerte | Relie éventuellement l’alerte à la mesure déclencheuse. |
| Faire_l_objet_de (Equipement — Intervention) | `id_equipement`, `id_intervention` | Identifiants Entier 64 bits | Mandatif pour Intervention | Identifie l’équipement faisant l’objet de l’intervention. |
| Déclencher (Regle_Alerte — Alerte) | `id_regle_alerte`, `id_alerte` | Identifiants Entier 64 bits | Mandatif | Relie l’alerte à la version de règle qui l’a déclenchée. |

Les associations `Historique_Zone_Equipement`, `Affectation_Capteur_Equipement`, `Affectation_Technicien_Intervention` et `Lien_Alerte_Intervention` ont des attributs propres; leur dictionnaire détaillé figure dans les sections précédentes.

## 7. Note méthodologique : historisation

### 7.1 Localisation des équipements

La zone courante n’est pas stockée comme un simple attribut modifiable de `Equipement`. Une telle conception écraserait l’emplacement antérieur et rendrait incorrectes les analyses rétrospectives : une mesure, une alerte ou une intervention passée pourrait être présentée dans la zone actuelle plutôt que dans la zone où l’équipement se trouvait à la date de l’événement.

Le MCD représente donc chaque période de localisation par une occurrence de `Historique_Zone_Equipement`, avec un équipement, une zone, une date de début et une date de fin. La localisation à une date `t` se calcule en sélectionnant l’occurrence dont l’intervalle contient `t`. Une localisation courante est une occurrence dont `date_fin` est absente. La règle de non-chevauchement assure qu’il ne peut pas exister deux localisations simultanées pour un même équipement.

Ce choix conserve les mouvements et permet de corréler les événements historiques à l’emplacement valide à leur date. Les déplacements rétroactifs, corrections d’intervalle et éventuelle traçabilité de l’auteur du changement sont des règles d’audit à préciser avant le modèle logique.

### 7.2 Association capteur-équipement dans le temps

La relation capteur-équipement est elle aussi une association historisée, et non une référence permanente inscrite directement dans `Capteur`. Le capteur peut être remplacé, déplacé ou réaffecté; conserver `date_debut` et `date_fin` permet d’interpréter les mesures selon l’équipement auquel le capteur était réellement rattaché lors de leur émission.

Pour associer une mesure à l’équipement correspondant, on recherche l’affectation du capteur dont l’intervalle contient `horodatage_evenement` (et non seulement la date de réception, qui peut être tardive). Une seule affectation d’équipement peut être active pour un capteur à un instant donné. L’absence d’affectation reste autorisée afin de représenter les capteurs non installés, en stock ou retirés.

Le modèle retenu suit l’hypothèse de l’étape 1 : **un équipement peut avoir plusieurs capteurs; un capteur est rattaché à au plus un équipement à un instant donné**. L’association à une zone sans équipement, le partage d’un capteur et l’obligation d’affectation pendant toute la durée de vie restent à confirmer par le métier.

### 7.3 Mesures, cible des alertes et conservation

Les mesures portent séparément l’heure d’observation et l’heure de réception pour distinguer l’ordre métier de l’ordre d’arrivée. L’identifiant technique d’une mesure n’impose pas une règle de déduplication non validée; la clé de déduplication reste optionnelle dans l’attente de la définition du protocole IoT. L’association capteur-équipement historisée donne le contexte de la mesure à sa date d’événement.

La cible d’une alerte est modélisée séparément du capteur qui a éventuellement produit la mesure déclencheuse. Ainsi, une règle peut signaler une anomalie de l’équipement à partir d’une mesure d’un de ses capteurs, sans confondre cible et source. La version de règle est conservée pour permettre d’expliquer après coup le seuil et la logique appliqués.

La rétention de 12 mois concerne les mesures brutes. Elle ne justifie pas la suppression des associations historiques, alertes ou interventions encore nécessaires à l’audit et à l’interprétation du parc; leurs durées de conservation sont à définir séparément.

## 8. Hypothèses et décisions à valider

1. **Campus et cardinalités minimales** : confirmer si un campus vide peut exister avant la création de son premier bâtiment; le MCD retient actuellement `Campus (1,N)`.
2. **Zones** : valider qu’une zone appartient à un seul bâtiment pendant tout son cycle de vie. Si une zone peut changer de bâtiment, sa propre localisation devra être historisée.
3. **Localisation** : définir les règles de déplacement rétroactif, les corrections d’historique et la traçabilité de l’auteur; valider la règle d’une localisation pour tout équipement en service.
4. **Capteur-équipement** : confirmer si un capteur peut être associé directement à une zone, être partagé, ou être obligatoirement affecté pendant son fonctionnement.
5. **Mesures** : définir précision numérique, unités autorisées, vocabulaire de qualité, tolérance d’horloge, identité de message et politique de déduplication/correction.
6. **Règles et alertes** : préciser les opérateurs, fenêtres d’évaluation, valeurs de seuil, sévérités, transitions de statut, acquittement, dédoublonnage et éventuels déclenchements manuels.
7. **Alertes** : confirmer qu’une alerte concerne exactement une cible (équipement ou capteur); le MCD impose cette décision à partir de la règle « équipement ou capteur » de l’étape 1.
8. **Interventions** : confirmer le caractère obligatoire de l’équipement associé et les cas autorisés sans demandeur humain; préciser les statuts, dates et champs obligatoires à la clôture.
9. **Techniciens** : préciser les rôles d’affectation, la notion de technicien principal et si plusieurs périodes d’affectation du même technicien à la même intervention doivent être conservées.
10. **Identité et habilitations** : définir la gestion des rôles et des périmètres bâtiment; les permissions ne sont pas détaillées dans ce MCD fonctionnel.
11. **Conservation** : préciser les durées applicables aux alertes, interventions, liens d’audit, règles et historiques au-delà de la rétention brute de 12 mois.

Ces décisions devront être arbitrées avant de dériver le Modèle Logique de Données (MLD), les contraintes physiques (unicité, exclusion d’intervalles, règles XOR) et les mécanismes de stockage adaptés au volume nominal d’environ 631 millions de mesures par an.
