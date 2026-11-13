# SmartCampus — Étape 1 : analyse métier et dimensionnement

## 1. Objet et périmètre

Ce livrable formalise les acteurs, les besoins, les règles de gestion, les exigences non fonctionnelles, les volumes et les principaux scénarios d'accès de la plateforme SmartCampus.

Le périmètre de travail est celui du cas pratique : une plateforme de supervision technique pour **8 bâtiments**, environ **160 zones**, **250 équipements** et **600 capteurs**, avec une mesure nominale par capteur toutes les **30 secondes**, une conservation des mesures brutes pendant **12 mois**, environ **50 utilisateurs actifs en journée** et des tableaux de bord actualisés toutes les **5 minutes**.

Ces nombres sont des hypothèses pédagogiques de dimensionnement, pas des mesures observées. Les décisions non précisées dans le cas sont signalées comme **hypothèses à valider** ou **zones grises**.

## 2. Acteurs

| Acteur | Rôle et objectifs | Données consultées | Données créées ou modifiées | Contraintes et contrôles |
|---|---|---|---|---|
| **Exploitant** | Surveille l’état du parc, identifie les incidents visibles et déclenche une demande d’intervention. | État courant des capteurs et équipements, alertes, localisation, interventions et historique utile au diagnostic. | Demandes d’intervention, commentaires ou informations de contexte. | Ne modifie pas le référentiel technique ni les mesures brutes. Une demande doit être traçable et reliée à un équipement, sauf décision contraire pour les incidents concernant un capteur ou une zone. |
| **Technicien** | Prend en charge les interventions, diagnostique et renseigne les opérations réalisées. | Interventions affectées, équipement et localisation, historique des mesures et alertes associés. | Prise en charge, statut, dates, compte rendu, opérations effectuées et, éventuellement, pièces ou observations. | N’accède qu’aux interventions nécessaires à son activité selon son périmètre. Les changements de statut et comptes rendus doivent être attribués à un utilisateur et horodatés. |
| **Responsable énergie** | Suit les consommations, analyse les tendances et compare bâtiments, périodes et types d’équipements. | Mesures pertinentes pour l’énergie, agrégats, indicateurs, référentiel bâtiments/zones/équipements et historique. | Éventuellement des paramètres de rapports ou annotations analytiques, si cette fonction est retenue. | Accès principalement en lecture. La définition des indicateurs, unités, périodes de comparaison et droits d’export reste à préciser. |
| **Administrateur** | Maintient le référentiel et les comptes, configure les droits et contribue à l’exploitation de la plateforme. | Référentiel complet, utilisateurs, rôles, configurations et journaux d’administration. | Bâtiments, zones, équipements, capteurs, associations et règles configurables; comptes, rôles et habilitations. | Les changements doivent être contrôlés, auditables et protégés contre les incohérences temporelles ou référentielles. Les opérations sensibles devraient être séparées des comptes d’usage courant. |
| **Capteurs / passerelles IoT** | Produisent et transmettent les mesures horodatées vers la plateforme, directement ou via une passerelle. | Configuration minimale nécessaire à la transmission et, selon le protocole, accusés de réception ou état de synchronisation. | Mesures, horodatage d’événement, identifiant du capteur et, si disponible, indicateur de qualité et métadonnées de transmission. | Identité d’équipement technique authentifiée, validation du format, gestion des doublons et retards, et mise en file temporaire en cas d’indisponibilité réseau. Les actions d’administration ne relèvent pas de cet acteur. |

## 3. Besoins fonctionnels et contraintes

| Domaine | Besoin | Contraintes et implications de données |
|---|---|---|
| **Référentiel** | Décrire le campus, ses bâtiments et zones, les équipements, les capteurs, les utilisateurs et les interventions. | Données structurées, identifiants stables, intégrité référentielle, historique des changements qui affectent l’interprétation du passé. Le cas ne précise pas les champs détaillés du référentiel. |
| **Mesures IoT** | Ingérer des mesures fréquentes, horodatées, avec valeur, unité et statut de qualité. | 600 flux nominaux, une mesure toutes les 30 s par capteur; tolérer les arrivées en rafale, retards, répétitions et coupures. Les plages de valeurs et vocabulaires qualité restent à définir par type de capteur. |
| **État courant** | Afficher rapidement la dernière valeur par capteur et l’état de santé ou d’anomalie pertinent. | Distinguer « dernière mesure reçue » de « dernière mesure valide » et exposer sa date; définir une règle d’obsolescence si un capteur cesse d’émettre. |
| **Historique** | Consulter les séries temporelles par capteur et période et effectuer des agrégations temporelles. | Conservation des mesures brutes pendant 12 mois. Définir les granularités d’agrégation, le traitement des données tardives et le sort des agrégats au-delà de cette durée. |
| **Dépendances** | Parcourir les relations entre équipements, zones, capteurs, alertes et interventions. | Une relation doit être interprétable dans le temps : l’emplacement ou l’association en vigueur aujourd’hui ne doit pas fausser l’analyse historique. |
| **Analytique** | Produire des indicateurs par bâtiment, période, type d’équipement et intervention; comparer les bâtiments. | Nécessite unités comparables, règles d’agrégation explicites et gestion des mesures manquantes/invalides. Les indicateurs métier ne sont pas définis dans le cas. |
| **Exploitation** | Sauvegarder, sécuriser, superviser et exploiter la plateforme à un coût et avec des compétences maîtrisés. | Besoin de surveiller l’ingestion, les retards, les erreurs, le stockage, les sauvegardes et les accès. Disponibilité cible, RPO/RTO, localisation des données et budget ne sont pas fournis. |

## 4. Règles de gestion

Les règles ci-dessous consolident les règles initiales du cas et explicitent les décisions nécessaires à leur mise en œuvre. Les éléments marqués **À valider** ne doivent pas être considérés comme des exigences définitivement approuvées.

| ID | Règle de gestion | Ambiguïté ou décision à valider |
|---|---|---|
| **RG-01 — Campus / bâtiment** | Un campus regroupe plusieurs bâtiments; chaque bâtiment appartient à un seul campus. | Le cas ne donne ni le nombre de campus ni la possibilité qu’un bâtiment soit créé avant son rattachement. Le modèle peut imposer exactement un campus par bâtiment. |
| **RG-02 — Bâtiment / zone** | Un bâtiment contient plusieurs zones; chaque zone appartient à un seul bâtiment. Le dimensionnement de référence est d’environ 20 zones par bâtiment, soit environ 160 zones. | « Plusieurs » peut signifier au minimum une zone; le nombre 20 est une moyenne, pas une contrainte fixe. |
| **RG-03 — Équipement / localisation** | Un équipement est rattaché à une zone à un instant donné. Son historique de localisation est conservé afin de restituer son emplacement pour une date passée. | Le cas exige l’historique, mais ne précise pas les dates d’effet, les déplacements rétroactifs, les intervalles ouverts ni si les zones peuvent être supprimées. À valider : intervalles de validité non chevauchants et conservation de l’historique après déplacement. |
| **RG-04 — Équipement / capteurs** | Un équipement peut être associé à plusieurs capteurs; un capteur est associé à un équipement à un instant donné. | L’association est-elle obligatoire ou facultative? Un capteur peut-il être partagé entre plusieurs équipements, associé directement à une zone ou réaffecté dans le temps? À valider avant de fixer les cardinalités et l’historique. |
| **RG-05 — Mesure** | Chaque mesure identifie un capteur et possède une valeur, une unité, un horodatage et un statut de qualité. | Le cas ne définit ni types numériques, ni précision, ni catalogue des unités/statuts. À valider : horodatage d’événement fourni par le capteur et horodatage de réception enregistrés séparément; unité stable par capteur ou portée par chaque mesure. |
| **RG-06 — Doublons et retard** | L’ingestion doit pouvoir distinguer les mesures répétées et les arrivées tardives sans supprimer silencieusement une mesure distincte. | La clé de déduplication n’est pas donnée (identifiant de message ou couple capteur/horodatage, par exemple), pas plus que la tolérance d’horloge et la politique de correction. Décision à formaliser au niveau du protocole IoT. |
| **RG-07 — État courant** | L’état courant d’un capteur correspond à sa dernière mesure exploitable et doit indiquer l’horodatage de cette mesure. | À valider : une valeur invalide plus récente remplace-t-elle la dernière valeur valide affichée? Seuil d’obsolescence et état « indisponible » non spécifiés. |
| **RG-08 — Alerte** | Une alerte est déclenchée par une règle et concerne un équipement ou un capteur. | Le cas ne précise pas si une alerte peut concerner les deux, ni les seuils, priorités, états, acquittement, clôture, dédoublonnage ou historique des versions de règle. À valider avant d’en déduire le modèle. |
| **RG-09 — Intervention** | Une intervention concerne un équipement et possède un statut, des dates, une priorité et un compte rendu. | Le lien aux alertes est plausible mais non imposé par le cas. À valider : statuts autorisés, dates obligatoires, règles de clôture et possibilité d’interventions préventives planifiées. |
| **RG-10 — Affectation des techniciens** | Une intervention peut mobiliser plusieurs techniciens et un technicien peut participer à plusieurs interventions. | Le cas établit une relation plusieurs-à-plusieurs, mais pas les rôles dans l’intervention, les dates d’affectation ni le technicien responsable principal. |
| **RG-11 — Conservation** | Les mesures brutes sont conservées pendant 12 mois. | « 12 mois » peut signifier une durée glissante exacte ou une rétention par mois civil; fuseau horaire, archivage et suppression des sauvegardes ne sont pas définis. Les autres données (alertes, interventions, audit, agrégats) ont des durées non précisées. |
| **RG-12 — Accès et traçabilité** | Les utilisateurs consultent ou modifient les données selon leur rôle; les capteurs/passerelles n’écrivent que les données IoT autorisées. | Le cas cite les rôles mais ne définit pas la matrice détaillée, les périmètres bâtiment, la fédération d’identité, la durée de conservation des journaux ni le processus de révocation. |

## 5. Exigences non fonctionnelles

### 5.1 Exigences dérivées du cas

- **Actualisation** : les tableaux de bord sont actualisés toutes les 5 minutes. Il s’agit d’une fréquence d’actualisation, pas d’une garantie de délai bout en bout; la latence maximale d’ingestion/affichage reste à spécifier.
- **Rétention** : les données de mesure brutes restent disponibles pendant 12 mois.
- **Charge nominale** : 600 capteurs émettent une mesure toutes les 30 secondes; environ 50 utilisateurs sont actifs en journée.
- **Intégrité** : les relations entre le référentiel, les mesures, les alertes et les interventions doivent préserver une interprétation correcte des données.
- **Accès contrôlé** : les lectures et écritures sont limitées aux responsabilités métier et aux identités des équipements IoT.

### 5.2 Exigences à fixer avant conception détaillée

Le cas ne donne pas de cible chiffrée pour la disponibilité, les temps de réponse interactifs, la perte de données tolérée (RPO), le délai de reprise (RTO), la reprise après sinistre, le chiffrement, la résidence des données, les sauvegardes ou la croissance future. Ces valeurs doivent être approuvées par le métier et l’exploitation plutôt que d’être inventées. Les objectifs de capacité ci-dessous sont donc des ordres de grandeur nominaux, pas des SLA.

La solution devra au minimum surveiller le débit et les erreurs d’ingestion, les retards de réception, le retard d’affichage des tableaux de bord, l’utilisation du stockage et le résultat des sauvegardes. Une perte de connectivité des passerelles doit être distinguable d’une absence de variation de mesure.

## 6. Volumes et dimensionnement

### 6.1 Volumes de référence

| Élément | Valeur de travail | Commentaire |
|---|---:|---|
| Bâtiments | 8 | Valeur du cas |
| Zones | ~160 | 20 par bâtiment en moyenne |
| Équipements | 250 | Valeur du cas |
| Capteurs | 600 | Valeur du cas |
| Utilisateurs actifs | ~50 en journée | Ne signifie pas nécessairement 50 sessions simultanées |
| Fréquence nominale de mesure | 1 / 30 s / capteur | Hypothèse nominale, hors retransmissions |
| Rétention brute | 12 mois | Calcul annuel ci-dessous sur 365 jours |
| Actualisation du tableau de bord | 5 min | Fréquence demandée, latence effective à préciser |

### 6.2 Nombre de mesures

Pour un capteur :

- **2 mesures/minute** (60 / 30)
- **120 mesures/heure**
- **2 880 mesures/jour** (120 × 24)
- **1 051 200 mesures/an** (2 880 × 365)

Pour 600 capteurs :

- **1 728 000 mesures/jour** (2 880 × 600)
- **630 720 000 mesures/an** (1 728 000 × 365)
- Sur une année bissextile de 366 jours : **632 448 000 mesures**.

Le nombre annuel suppose 365 jours de fonctionnement continu, sans période de maintenance ni perte d’émission. Il ne comprend pas les retransmissions; leur effet dépend de la politique de reprise et de déduplication.

### 6.3 Volumétrie des mesures

Le format de stockage et les index ne sont pas définis. Pour donner un ordre de grandeur reproductible, on estime ici une ligne de mesure logique à **64 octets** en moyenne (identifiant capteur, horodatage, valeur, qualité/unité codées et coût de représentation/payload). Cette hypothèse ne constitue pas un benchmark d’un moteur donné.

| Base de calcul | Calcul | Volume décimal estimé |
|---|---:|---:|
| Mesures/jour à 32 octets chacune (borne basse illustrative) | 1 728 000 × 32 | ~55,3 Mo/jour |
| Mesures/jour à 64 octets chacune (hypothèse centrale) | 1 728 000 × 64 | ~110,6 Mo/jour |
| Mesures/an à 32 octets chacune | 630 720 000 × 32 | ~20,2 Go/an |
| Mesures/an à 64 octets chacune | 630 720 000 × 64 | **~40,4 Go/an** |
| Mesures/an à 128 octets chacune (borne haute illustrative) | 630 720 000 × 128 | ~80,7 Go/an |

Ces valeurs représentent une **estimation logique brute** des enregistrements, en unités décimales (1 Go = 10⁹ octets). Elles excluent index, journaux transactionnels, métadonnées moteur, réplication, sauvegardes, agrégats et espace libre opérationnel. Avec un facteur de planification indicatif de **1,5 à 2** pour l’empreinte primaire (à mesurer sur le moteur choisi), l’hypothèse centrale correspond à environ **61 à 81 Go** pour les seules données et structures primaires associées. Une réplica, les sauvegardes, les agrégats et la marge de croissance s’ajoutent; ils ne sont pas inclus dans cette estimation primaire.

Une moyenne de taille réelle ne pourra être confirmée qu’après fixation du schéma, du format de mesure, de la compression, des index et un test avec des données représentatives. La taille des référentiels (250 équipements, 600 capteurs, etc.) est faible par rapport aux centaines de millions de mesures, mais les historiques d’audit et d’intervention ne sont pas quantifiés.

### 6.4 Charge d’écriture

- **Débit moyen nominal** : 1 728 000 / 86 400 = **20 mesures par seconde**.
- **Écriture en rafale possible** : si les 600 capteurs sont synchronisés, une arrivée groupée de **600 mesures toutes les 30 secondes** est à absorber.
- À l’échelle de 12 mois, l’ingestion produit environ **631 millions de lignes** avant expiration de la rétention.
- Le dimensionnement doit prendre en compte les reconnexions des passerelles, les retransmissions et les pointes; leur facteur de charge n’est pas fourni et doit être mesuré ou spécifié.

Le débit moyen est modeste, mais le volume historique et les rafales, ainsi que l’indexation temporelle, conditionnent davantage la stratégie de stockage et de partitionnement.

### 6.5 Charge de lecture

- Une actualisation de tous les utilisateurs actifs représente environ **50 rafraîchissements toutes les 5 minutes**, soit **10 rafraîchissements/minute** ou **0,17 rafraîchissement/seconde en moyenne** si les consultations sont réparties.
- Pour donner un ordre de grandeur sur une journée de travail de 8 heures (**hypothèse**), cela représente **4 800 rafraîchissements de tableau de bord par jour** (50 × 12 × 8). Sur une journée entière de 24 h, la borne théorique serait 14 400.
- Les requêtes réelles peuvent effectuer plusieurs lectures par rafraîchissement; elles dépendent du nombre de widgets, des filtres, du cache et du niveau d’agrégation. Si les 50 utilisateurs rafraîchissent simultanément, la plateforme peut recevoir un pic d’environ **50 chargements** au même moment.
- La lecture « dernière valeur » porte au plus sur les **600 capteurs** du parc. Les lectures historiques peuvent scanner un volume beaucoup plus important; la période, le nombre d’utilisateurs, la granularité, les agrégations et la concurrence ne sont pas définis et ne peuvent pas être chiffrés précisément.

Pour des tableaux de bord, privilégier la lecture de l’état courant et d’agrégats préparés plutôt que de relire systématiquement toute la série brute sur 12 mois. Le taux de cache, les agrégats nécessaires et le pic attendu doivent être validés par les scénarios réels.

## 7. Scénarios d’accès

| Scénario | Acteur | Lecture | Écriture | Fréquence / charge de référence |
|---|---|---|---|---|
| **Ingestion nominale** | Capteur / passerelle IoT | Configuration minimale et accusés de réception, si disponibles. | Mesures horodatées et métadonnées de qualité; éventuellement état de transmission. | Une mesure / capteur / 30 s : 20 mesures/s en moyenne pour 600 capteurs; possibilité de rafales de 600 mesures / 30 s. |
| **Supervision du parc** | Exploitant | Référentiel utile, état courant, alertes et interventions associées. | Demande d’intervention et observations. | Tableau de bord actualisé toutes les 5 min; jusqu’à ~50 rafraîchissements par intervalle si tous les utilisateurs actifs l’utilisent. |
| **Diagnostic et intervention** | Technicien | Affectations, équipement, zone, mesures et alertes historiques pertinentes. | Statut, prise en charge, dates, compte rendu et opérations effectuées. | À l’ouverture et à chaque étape d’intervention; fréquence journalière non fournie. |
| **Analyse énergétique** | Responsable énergie | Séries et agrégats par bâtiment, période et type d’équipement; référentiel. | Paramètres de rapports ou annotations uniquement si cette fonction est retenue. | Consultations périodiques et comparaisons; fréquence, période et exports non spécifiés. |
| **Administration du référentiel** | Administrateur | Référentiel, utilisateurs, droits, configurations et journaux d’audit. | Création/mise à jour des bâtiments, zones, équipements, capteurs, associations et droits; règles si habilité. | Événementiel, probablement bien moins fréquent que l’ingestion; volume non spécifié. |
| **Consultation de l’historique** | Exploitant, technicien, responsable énergie selon habilitation | Mesures filtrées par capteur, équipement, zone ou bâtiment et période; agrégations temporelles. | Aucune écriture de mesure par les utilisateurs. | À la demande; fréquence et concurrence à mesurer. La rétention brute disponible est de 12 mois. |
| **Traitement d’une alerte** | Service de règles et utilisateur habilité | Mesures, configuration/version de règle, référentiel et état d’alerte. | Création et évolution de l’alerte; acquittement/commentaire par un utilisateur, si ces états sont validés. | Déclenchement à chaque franchissement selon des règles à définir; fréquence inconnue. |
| **Sauvegarde et exploitation** | Exploitation technique / administrateur habilité | Données et configurations nécessaires à la sauvegarde, au contrôle et à la restauration. | Journaux techniques, sauvegardes et opérations de maintenance contrôlées. | Planification et objectifs de reprise non fournis; à définir avec RPO/RTO et exigences de rétention. |

## 8. Hypothèses et décisions ouvertes prioritaires

1. **Association capteur/équipement** : association unique ou non, facultative ou obligatoire, directe à une zone possible, et conservation des périodes d’association.
2. **Historique de localisation** : définition des dates d’effet, règles de chevauchement, traitement des déplacements et politique de suppression des entités.
3. **Mesures** : horloge de référence, fuseau, précision, vocabulaires d’unité et de qualité, validation de plages, doublons, retards et corrections.
4. **État courant** : définition d’une mesure valide, gestion d’une valeur invalide plus récente et seuil rendant un capteur obsolète.
5. **Alertes** : règles/seuils, versions, sévérités, dédoublonnage, acquittement, clôture, lien aux interventions et durée de conservation.
6. **Interventions** : cycle de statuts, intervention préventive, équipement obligatoire, responsable principal et règles d’affectation des techniciens.
7. **Rétention** : durée exacte et périmètre (mesures brutes seulement ou autres données), agrégats à conserver et politique applicable aux sauvegardes.
8. **Exigences d’exploitation** : SLA, latence, disponibilité, RPO/RTO, sécurité et audit, croissance, budget et capacité de reprise des passerelles.
9. **Profils de lecture** : nombre réellement simultané d’utilisateurs, widgets par tableau de bord, plages historiques usuelles, pics et politique de cache.

Ces décisions doivent être confirmées avec les parties prenantes avant de figer le MCD/MLD et le choix de stockage. Les hypothèses de volumétrie devront ensuite être vérifiées sur un jeu représentatif et un protocole de charge reproductible.
