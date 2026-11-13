# SmartCampus — Étape 6 : architecture polyglotte et résilience

## 1. Objet et principes d’architecture

Ce livrable décrit l’architecture cible SmartCampus et les flux entre la collecte IoT, les services applicatifs, PostgreSQL et les projections de lecture éventuelles. Il prolonge les étapes précédentes :

- **PostgreSQL est la source de vérité** du référentiel, des historiques, des mesures brutes, des règles, alertes et interventions.
- Les mesures sont partitionnées par mois sur `horodatage_evenement`; la rétention brute cible est de 12 mois.
- Le dimensionnement nominal est de **600 capteurs**, une mesure toutes les **30 secondes**, soit **20 messages/s en moyenne**, environ **1 728 000 mesures/jour** et **630 720 000/an** avant retransmissions.
- Les tableaux de bord sont actualisés toutes les **5 minutes**; cette fréquence ne définit pas encore une latence maximale garantie.
- Aucun moteur NoSQL supplémentaire n’est retenu à ce stade. Une projection d’état courant dans PostgreSQL est possible; un cache clé-valeur externe reste une option conditionnelle aux résultats de charge.

Le système doit préserver l’ordre métier des événements, tolérer les reprises et éviter que la panne d’une projection ou d’un service non critique bloque l’enregistrement de référence. Les objectifs chiffrés de disponibilité, RPO et RTO ne sont pas fournis dans le cas initial : les valeurs proposées ci-dessous sont des **cibles de travail à valider**, et non des engagements déjà approuvés.

## 2. Diagramme des composants

```text
┌──────────────────────────── COUCHE TERRAIN / IoT ───────────────────────────┐
│  Capteurs (≈600)                                                            │
│      │ mesures : identifiant source, horodatage événement, valeur, qualité   │
│      v                                                                       │
│  Passerelles IoT                                                            │
│  - authentification de la source                                            │
│  - tampon local durable / reprise après reconnexion                         │
│  - compteur de séquence ou identifiant de message si disponible            │
└──────┬───────────────────────────────────────────────────────────────────────┘
       │ TLS, envoi avec reprise et accusé après persistance durable
       v
┌──────────────────────────── COUCHE COLLECTE ────────────────────────────────┐
│  Point d'entrée IoT / API d'ingestion                                       │
│  - authentification, validation de schéma et limites de taille              │
│  - normalisation des unités/horodatages selon contrat                        │
│       │                                                                      │
│       v                                                                      │
│  File durable / journal d'ingestion [tampon et découplage]                  │
│  - partitions de flux; rétention courte configurable                        │
│  - reprise, back-pressure, métriques de retard                               │
└──────┬────────────────────────────────────────────────────────────────────────┘
       │ mesure durable, ordonnée par clé de partition
       v
┌──────────────────── SERVICES MÉTIER ───────────────────┐
│ Service d'ingestion                                    │
│ - déduplication/idempotence                            │
│ - contrôle qualité et horodatages                      │
│ - écrit les mesures et l'outbox                        │
│ - met à jour projection SQL optionnelle                │
└───────────────────┬────────────────────────────────────┘
                    │ transaction PostgreSQL
                    v
┌──────────────────────────── STOCKAGE DE RÉFÉRENCE ──────────────────────────┐
│ PostgreSQL (HA, sauvegardes et PITR à configurer)                            │
│  - référentiel normalisé et associations historisées                        │
│  - mesure partitionnée par mois                                             │
│  - règles, alertes, interventions, techniciens                              │
│  - inbox/déduplication et outbox transactionnelles                          │
│  - projection relationnelle et agrégats si justifiés                        │
└────────────┬──────────────────────────┬──────────────────────┬───────────────┘
             │                          │                      │
             v                          v                      v
┌────────────────────┐  ┌────────────────────────┐  ┌────────────────────────┐
│ API supervision /  │  │ Service maintenance     │  │ Agrégations / rapports  │
│ tableaux de bord   │  │ interventions           │  │ énergie, séries filtrées│
└─────────┬──────────┘  └────────────┬───────────┘  └────────────────────────┘
          │                          │
          v                          v
  Exploitants / responsables     Techniciens / exploitants
  énergie / administrateurs

PostgreSQL outbox ──> Consommateur règles ──transaction──> PostgreSQL (alertes)
                          │
                          └──> outbox notifications / projection, si nécessaire

OPTION CONDITIONNELLE, hors chemin critique :
┌───────────────────────┐      projection idempotente      ┌───────────────────┐
│ Outbox / événements   │ ───────────────────────────────> │ Cache clé-valeur  │
│ PostgreSQL            │                                  │ état courant      │
└───────────────────────┘                                  └─────────┬─────────┘
                                                                      │ lecture
                                                  API supervision <───┘
                                      (repli PostgreSQL si cache absent/périmé)
```

Le diagramme montre une file durable entre la collecte et les consommateurs. Son choix de produit et son niveau de réplication restent à arrêter selon les objectifs de reprise et les compétences d’exploitation. À ce débit, elle sert surtout à absorber les rafales, isoler les consommateurs et rejouer les messages après incident; elle ne remplace ni PostgreSQL ni les tampons des passerelles.

## 3. Rôle des composants

| Composant | Responsabilités principales | Données détenues et garanties |
|---|---|---|
| **Capteurs** | Produire une mesure avec identifiant, horodatage d’événement, valeur et qualité lorsqu’elle est disponible. | Les capteurs ne sont pas la source de vérité de la plateforme; leur horloge et l’ordre de transmission peuvent être imparfaits. |
| **Passerelles IoT** | Authentifier les équipements, convertir le protocole terrain, tamponner localement les messages non transmis, les retransmettre après reconnexion et exposer leur état de liaison. | Tampon durable borné en capacité et durée. Accusé d’envoi à la passerelle seulement après persistance selon le contrat d’ingestion; alerte avant saturation. |
| **Point d’entrée / API d’ingestion** | Terminer les connexions sécurisées, authentifier la passerelle, limiter le débit et la taille, valider le contrat, rejeter explicitement les messages illisibles. | Aucun succès fictif : réponse d’acceptation seulement si le message est écrit dans un stockage durable (file ou base selon le mode retenu). |
| **File durable / journal** | Découpler l’arrivée des messages des consommateurs, lisser les rafales, conserver les messages non traités pendant la durée convenue et permettre le rejeu. | Ordre garanti au moins au niveau d’une partition/clé, pas nécessairement global. Chaque message porte une clé de partition, un identifiant source et des horodatages. |
| **Service d’ingestion** | Dédupliquer, contrôler le format et la qualité, appliquer les règles de normalisation, écrire les mesures et maintenir au besoin la projection d’état courant. | PostgreSQL reste la vérité. Les écritures sont transactionnelles et rejouables; les messages refusés sont tracés et routés en quarantaine plutôt que perdus silencieusement. |
| **Consommateur de règles** | Évaluer les règles actives et versionnées sur les mesures et l’état requis, déterminer la cible (capteur ou équipement) et produire un événement de déclenchement. | L’évaluation utilise la version de règle effective pour l’événement. Les alertes persistées et leurs transitions sont dans PostgreSQL. |
| **Service supervision** | Fournir états courants, alertes, localisation et historiques aux exploitants et responsables énergie. | Lit les projections éventuelles pour l’affichage rapide, mais peut revenir à PostgreSQL; expose fraîcheur et statut d’indisponibilité des sources. |
| **Service maintenance** | Créer, affecter et suivre les interventions; prendre en charge les participations de plusieurs techniciens et le lien aux alertes. | PostgreSQL transactionnel avec clés étrangères; modification des statuts attribuée à un utilisateur et horodatée selon les règles d’accès. |
| **PostgreSQL** | Stocker et garantir le référentiel, les historiques temporels, mesures partitionnées, règles, alertes, interventions, clés d’idempotence et outbox. | Source de vérité durable, contraintes d’intégrité, sauvegardes et restauration à configurer. Le mécanisme HA et le site de secours restent à définir. |
| **Outbox / projection** | Publier durablement des événements métier après transaction et alimenter des vues de lecture/caches. | L’outbox est atomique avec la transaction source; les consommateurs sont idempotents, surveillés et rattrapables. |
| **Cache clé-valeur éventuel** | Accélérer la lecture de la dernière mesure ou état par capteur si le benchmark justifie un moteur supplémentaire. | Donnée dérivée et reconstructible; jamais utilisé comme autorité ni comme unique destination d’ingestion. |
| **Agrégateur analytique** | Calculer séries et indicateurs par période, équipement ou bâtiment et alimenter des rapports. | Agrégats recalculables avec version de calcul et traitement des mesures tardives; les mesures brutes restent la référence durant leur rétention. |
| **Supervision technique** | Mesurer débit, retard, erreurs, backlog, santé des partitions, espace disque, sauvegardes, réplication et fraîcheur des projections. | Alertes techniques indépendantes des alertes métier; traces corrélées par identifiant de message et de traitement. |

## 4. Flux de données

### 4.1 Flux d’ingestion des mesures

1. Le capteur transmet une mesure à la passerelle; le message contient au minimum l’identifiant externe du capteur, l’horodatage d’événement, la valeur et l’unité. L’identifiant source du message et la qualité sont utilisés lorsqu’ils sont disponibles.
2. La passerelle valide le minimum de protocole et place les messages dans son tampon local tant qu’ils ne sont pas confirmés par la plateforme.
3. Le point d’entrée authentifie la passerelle et vérifie taille, schéma, type de données et droits de la source. Une erreur de validation retourne un refus explicite avec motif; elle n’est pas transformée en valeur de mesure par défaut.
4. Le message est écrit dans la file durable (ou directement dans une inbox/base durable dans une variante sans courtier). L’accusé à la passerelle n’est envoyé qu’après la confirmation de cette persistance.
5. Le service d’ingestion consomme le message, résout l’identifiant externe vers `capteur.id_capteur`, normalise et valide l’unité/horodatage, puis effectue une insertion idempotente de la mesure. L’horodatage d’événement est utilisé pour la partition et les analyses temporelles; l’horodatage de réception est généré par la plateforme.
6. Dans la même transaction, il peut actualiser la projection relationnelle de dernière mesure et écrire les événements nécessaires dans l’outbox. Le consommateur n’acquitte la file qu’après commit durable de la transaction.
7. Les erreurs transitoires entraînent une nouvelle tentative avec temporisation exponentielle et jitter. Les erreurs permanentes (identifiant inconnu, format invalide, unité interdite) sont isolées dans une file de quarantaine avec motif et métadonnées, puis exposées pour traitement; elles ne provoquent pas une boucle infinie.

### 4.2 Flux de traitement des alertes

1. Le consommateur de règles lit les mesures acceptées, ainsi que les versions de règles actives et les associations temporelles nécessaires.
2. Il évalue la règle au regard de l’horodatage d’événement et de sa validité. Pour une mesure tardive, la politique détermine si elle peut créer une alerte rétroactive, enrichir une alerte existante ou être exclue du déclenchement temps réel; cette règle doit être approuvée.
3. Le consommateur écrit l’alerte et son lien de déclenchement, ainsi qu’un événement d’outbox, dans une transaction PostgreSQL. Une clé de traitement idempotente empêche de créer plusieurs fois la même alerte lors d’un rejeu.
4. L’API de supervision consulte les alertes; un utilisateur habilité peut acquitter, commenter ou clôturer selon les transitions autorisées. Le service maintenance peut créer une intervention liée à une ou plusieurs alertes.
5. Une erreur de calcul est réessayée; une donnée non interprétable est mise en quarantaine et signalée. Les événements ne sont acquittés qu’après persistance de leur résultat ou de leur statut d’échec durable.

### 4.3 Flux de gestion des interventions

1. Un exploitant ou un processus autorisé soumet une demande via l’API maintenance; la demande référence l’équipement et peut être liée à une alerte.
2. Dans une transaction, le service crée l’intervention et son événement d’audit/outbox. Les références et statuts sont validés par PostgreSQL.
3. Un ou plusieurs techniciens sont affectés dans `affectation_technicien_intervention`, avec les dates et rôles applicables. Une réaffectation clôt la période précédente et crée une nouvelle occurrence; elle n’écrase pas l’historique.
4. Le technicien met à jour statut, dates et compte rendu. Le service applique les transitions permises et conserve l’identité de l’auteur dans la couche d’audit.
5. Une panne de notification ou d’index de recherche n’annule pas la transaction métier : les événements d’outbox sont rejoués après reprise.

### 4.4 Flux de consultation des tableaux de bord

1. Le client de supervision s’authentifie auprès de l’API et demande les indicateurs correspondant à son périmètre et ses filtres.
2. L’API lit les alertes, interventions et référentiels dans PostgreSQL; elle lit l’état courant depuis une projection SQL, ou un cache clé-valeur uniquement si celui-ci est ultérieurement retenu.
3. Les plages historiques sont filtrées par capteur et fenêtre temporelle afin de bénéficier du partitionnement mensuel et de l’index `(id_capteur, horodatage_evenement)`. Les agrégats sont préférés aux scans bruts répétés pour les rapports de longue période lorsqu’ils ont été validés.
4. La réponse indique l’heure de dernière donnée reçue/valide et la fraîcheur de la projection. Une donnée périmée ou indisponible est présentée comme telle, pas comme un état actuel certain.
5. Les tableaux de bord se rafraîchissent toutes les cinq minutes selon le besoin métier; les limites de débit et caches applicatifs empêchent une actualisation simultanée de tous les utilisateurs de créer une rafale inutile.

## 5. Synchronisation, idempotence et ordre des événements

### 5.1 Identité des messages et déduplication

Le MPD actuel a une colonne facultative `cle_deduplication`, mais ne fixe pas encore de clé source fiable ni de contrainte d’unicité. L’étape de mise en œuvre doit donc définir un contrat explicite avant d’activer l’idempotence forte :

- identifiant unique de message émis par le capteur ou la passerelle, idéalement stable lors de chaque retransmission;
- identifiant de la source et génération/époque de séquence si un compteur peut repartir à zéro;
- portée d’unicité (par appareil, par passerelle ou globalement) et durée de conservation des clés;
- comportement en cas de réutilisation d’une clé avec un contenu différent (rejet et alerte, pas écrasement silencieux).

La déduplication est réalisée par une table d’inbox/registre d’ingestion avec clé unique sur `(source_id, message_id)` ou par une contrainte équivalente après ajout du périmètre manquant au MPD. Si aucun identifiant fiable n’est disponible, le couple capteur/horodatage/valeur ne doit pas être déclaré unique par défaut : des observations distinctes peuvent partager un horodatage et les capteurs peuvent avoir une précision limitée. Une stratégie de rapprochement probabiliste ne remplace pas l’identifiant d’idempotence.

L’effet de traitement suit le schéma **au moins une livraison + consommateurs idempotents** : les systèmes distribués peuvent retransmettre après perte d’accusé; l’exactement-une-fois de bout en bout n’est pas supposée. La transaction PostgreSQL inscrit le marqueur d’inbox, la mesure, la mise à jour de projection SQL et les événements d’outbox liés. En cas de replay, la contrainte d’unicité retourne le résultat déjà traité sans doubler les effets métier.

### 5.2 Mesures tardives et désordonnées

- Conserver séparément `horodatage_evenement` et `horodatage_reception`.
- Valider l’écart d’horloge et conserver, si nécessaire, un indicateur de dérive; rejeter ou isoler les dates impossibles selon des seuils configurés et documentés.
- Les mesures retardées restent interrogeables comme données historiques si elles sont valides et dans la rétention; leur période est déterminée par l’heure d’événement.
- La mise à jour de l’état courant compare l’horodatage d’événement : une mesure plus ancienne reçue après une nouvelle ne fait pas reculer l’état courant. Conserver séparément « dernière reçue » et « dernière valide ».
- Les agrégats gardent une fenêtre de correction configurable; les périodes récentes peuvent être recalculées en cas d’arrivée tardive, au-delà la politique peut archiver une correction ou rejeter la modification.
- L’ordre global entre capteurs n’est pas requis. Si l’ordre par capteur est nécessaire aux règles (variation, seuil consécutif), la partition de file est déterminée par identifiant de capteur et la règle traite les numéros de séquence/temps selon la tolérance définie.
- Définir les règles métier pour les mesures tardives qui auraient déclenché une alerte : création rétroactive, annotation, ou non-déclenchement. L’historique des données et la notification temps réel sont deux décisions séparées.

### 5.3 Outbox, inbox et projections

- **Inbox** : registre de réception/traitement permettant de détecter les redélivrances et d’en suivre le statut (`RECU`, `TRAITE`, `QUARANTAINE`, motif, horodatages).
- **Outbox transactionnelle** : même transaction que l’alerte/intervention ou la mesure dont elle publie l’événement; un relay peut republier plusieurs fois, donc le consommateur déduplique aussi.
- **Projections** : la projection SQL de dernière mesure peut être mise à jour atomiquement avec l’insertion. Le cache externe éventuel est mis à jour depuis l’outbox, jamais par un second commit réseau non coordonné après l’écriture source.
- **Ordre** : chaque événement transporte un identifiant de source, de message, de mesure, un horodatage d’événement et de réception, et, si disponible, une séquence. Les consommateurs détectent doublons et retards.
- **Réconciliation** : comparer régulièrement le nombre d’événements persistés et traités, l’état de dernière mesure et les agrégats aux sources; une projection divergente est reconstruite ou corrigée depuis PostgreSQL.

## 6. Indisponibilités et résilience

### 6.1 Coupure du réseau terrain

1. La passerelle détecte la perte de connexion à partir de la supervision de transport et conserve les messages dans un spool local durable, avec capacité mesurée et politique FIFO/temps.
2. Elle signale sa connectivité et son niveau de tampon indépendamment des valeurs mesurées; une absence de données n’est pas interprétée comme une valeur stable.
3. À reconnexion, elle transmet les lots en respectant les limites de débit. La plateforme applique la déduplication et le traitement temporel; les événements tardifs ne sont pas rejetés uniquement parce qu’ils sont anciens.
4. Le dimensionnement du spool dépend de la durée maximale de coupure approuvée, du volume réel de message et d’une marge de reprise. À la cadence nominale, le parc produit **1 728 000 messages par jour**; une journée de tampon théorique correspond donc à ce nombre de messages, avant enveloppes, index et marge.
5. Lorsque le spool approche de sa capacité, alerter et appliquer une politique explicite (extension de stockage, limitation, ou perte contrôlée avec journal). Ne jamais supprimer silencieusement les plus anciennes mesures.

### 6.2 Indisponibilité du point d’entrée, de la file ou des consommateurs

- Si le point d’entrée est inaccessible, les passerelles conservent et réessaient selon backoff avec jitter; le service ne répond pas par un faux accusé de succès.
- Si la file durable est indisponible, le point d’entrée refuse l’acceptation ou utilise un tampon durable de secours défini. Un tampon mémoire seul ne justifie jamais un accusé à la source.
- Si le service d’ingestion est indisponible, la file conserve les messages et applique back-pressure; les alertes de backlog préviennent avant saturation.
- Si le consommateur de règles est indisponible, les mesures continuent d’être persistées; les alertes sont retardées, puis rejouées dans l’ordre requis. Les tableaux de bord affichent le retard du traitement des règles.
- Les messages illisibles ou définitivement non conformes sont isolés dans une file de quarantaine avec accès restreint, durée de conservation définie et procédures de correction/rejeu.

### 6.3 Indisponibilité de PostgreSQL

- La file durable et les tampons des passerelles absorbent temporairement les écritures; la durée réelle dépend du dimensionnement du backlog et de la capacité de rattrapage.
- Le service d’ingestion ralentit/cesse la consommation si PostgreSQL n’accepte plus les écritures; il ne confirme les messages que selon la règle d’accusé durable décrite au §4.1.
- Les API de supervision passent en lecture seule dégradée si une réplique disponible et cohérente est prévue; sinon elles affichent une indisponibilité explicite. Les écritures d’intervention ne sont pas acquittées si elles ne sont pas durablement enregistrées.
- Après bascule ou restauration, les consommateurs rejouent les messages non confirmés; inbox, clés uniques et écritures transactionnelles empêchent les doubles mesures et alertes.
- Vérifier les partitions temporelles futures, l’espace disque, les index, le retard de réplication, les sauvegardes et les journaux WAL. Une partition manquante doit produire une alerte avant l’échéance; ne pas acheminer silencieusement les mesures vers une partition de secours non gouvernée.

### 6.4 Panne du cache ou d’une projection

Le cache clé-valeur n’est pas déployé dans l’architecture de départ. S’il est ajouté, sa panne ne doit pas bloquer ingestion, alertes ou interventions : l’API revient à PostgreSQL ou présente une donnée dégradée avec sa date de fraîcheur. Le cache est rempli à nouveau depuis l’outbox ou les tables sources. Une projection SQL endommagée est reconstruite depuis les mesures encore conservées; l’état de santé de cette projection fait l’objet d’une métrique séparée.

### 6.5 RPO, RTO et stratégie de reprise

Le cas initial ne fixe pas de RPO/RTO. Les valeurs suivantes sont des **propositions de travail à arbitrer par le métier et l’exploitation**, après estimation du coût de réplication synchrone, du stockage de sauvegarde et des scénarios de perte de site :

| Scénario | RPO de travail proposé | RTO de travail proposé | Mesures à mettre en place |
|---|---:|---:|---|
| Panne d’un processus applicatif ou consommateur | 0 pour les transactions déjà validées; messages non validés rejoués depuis la file | ≤ 15 min | Redémarrage supervisé, file durable, inbox/outbox, tests de rejeu. |
| Panne du nœud PostgreSQL principal dans le même site | 0 si réplication synchrone et bascule testées; sinon RPO égal au retard de réplication mesuré | ≤ 15 min | Réplica HA, bascule contrôlée, fencing de l’ancien primaire et vérification d’intégrité. |
| Perte du site principal / reprise après sinistre | ≤ 5 min à confirmer avec réplication asynchrone ou sauvegarde continue | ≤ 4 h à confirmer | Copie hors site des sauvegardes et WAL, infrastructure de secours, procédures et exercices de restauration. |
| Perte temporaire de connectivité d’une passerelle | Pas de perte des messages encore présents dans le spool; borne de perte liée à la capacité/panne du spool | Reprise à la restauration réseau; délai métier à définir | Tampon local durable dimensionné, alertes de capacité et retransmission idempotente. |

Ces cibles ne sont garanties que si l’architecture correspondante est déployée et testée. La réplication synchrone peut augmenter la latence et réduire la disponibilité en cas de perte du nœud secondaire; la réplication asynchrone peut perdre les dernières transactions au sinistre. La décision HA/DR doit arbitrer explicitement ces compromis.

Plan minimal de sauvegarde et reprise :

1. sauvegarde complète périodique et archivage continu des WAL pour restauration à un instant donné;
2. copie des sauvegardes hors du domaine de panne principal, chiffrement et contrôle d’accès;
3. sauvegarde cohérente des configurations de passerelles, définitions de règles et secrets, avec leur processus de rotation;
4. tests réguliers de restauration réelle, mesure du RTO et vérification du nombre de mesures/alertes restaurées;
5. procédures documentées de bascule, fencing, reprise de file, réconciliation des projections et réouverture des écritures;
6. rétention des sauvegardes alignée sur les exigences légales et opérationnelles, distincte de la rétention de 12 mois des mesures brutes.

## 7. Observabilité et exploitation

Les tableaux de bord techniques doivent au minimum exposer :

- messages reçus, acceptés, refusés et mis en quarantaine par source;
- débit entrant, rafales, taille des lots, répétitions détectées et taux de déduplication;
- âge du plus ancien message dans la file, taille du backlog et vitesse de rattrapage;
- différence entre `horodatage_evenement` et `horodatage_reception`, taux de retard et dérive d’horloge;
- latence d’écriture PostgreSQL, erreurs de commit, connexions, utilisation disque et présence des partitions à venir;
- retard de réplication et réussite/âge des sauvegardes;
- âge des états courants, délai de traitement des règles et des alertes, écart entre projection et source;
- durée des requêtes des tableaux de bord et volume de requêtes simultanées.

Les logs applicatifs sont corrélés par `source_id`, `message_id`, `id_capteur`, `id_mesure` et identifiant de traitement; ils ne doivent pas contenir de secrets. Les alarmes d’exploitation doivent rester distinctes des alertes métier générées par les règles de supervision des bâtiments.

## 8. Décisions restant à valider

1. **RPO/RTO officiels**, SLA de disponibilité et tolérance à une lecture dégradée.
2. **Technologie et topologie de la file durable** : partitions, réplication, rétention, ordre, dimensionnement, quarantaine et limites de backlog.
3. **Capacité de spool des passerelles** : durée de coupure à supporter, stockage local, priorités et politique si capacité épuisée.
4. **Identifiant de message source** et portée de déduplication; modification nécessaire au modèle physique si l’inbox ou l’unicité doit être persistée.
5. **Tolérance d’horloge et traitement des événements tardifs**, notamment pour les alertes rétroactives et la correction d’agrégats.
6. **Politique HA/DR PostgreSQL** : réplication synchrone ou asynchrone, site de secours, copies de WAL et responsable de bascule.
7. **Projection d’état et cache externe** : uniquement après benchmark; règles d’état courant, cohérence attendue, repli et reconstruction.
8. **Rétention et dépendances** : mesure déclencheuse référencée par une alerte, durée de vie de l’outbox/inbox, quarantaine et sauvegardes.
9. **Sécurité de bout en bout** : identité des passerelles, rotation des certificats/secrets, chiffrement, segmentation réseau et autorisation par rôle.

L’architecture peut être déployée progressivement : commencer par une source relationnelle PostgreSQL bien instrumentée et une collecte durable adaptée aux coupures; introduire des projections ou moteurs supplémentaires seulement lorsqu’un besoin mesuré et un objectif de reprise approuvé le justifient.
