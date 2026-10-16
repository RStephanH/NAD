# NAD — Outils : comparatif et décisions

**Français** | [English](tooling.md)

Ce document compare les outils candidats pour chaque composant de NAD, consigne
la décision prise pour le prototype et indique quand la réexaminer. Il est écrit
pour être lu seul, par une personne ou par un assistant IA qui n'a pas vu le
reste du dépôt.

**Statut :** décisions de prototype, à revalider une fois l'environnement réel
connu. Les éléments marqués **(à vérifier)** dépendent des versions des outils
ou de l'environnement cible et doivent être contrôlés dans la documentation
actuelle.

## 1. Contexte

| Élément | Valeur |
|---|---|
| Objectif | Détecter automatiquement les comportements réseau anormaux (non supervisé) |
| Environnement de référence | Petit réseau d'entreprise, passerelle MikroTik (hAP ac², RouterOS 7), liaison montante Starlink |
| Taille du réseau | Petite ; nombres exacts d'utilisateurs et d'appareils à confirmer sur place |
| Matériel de prototypage | Un hAP ac² utilisé pour prototyper avant d'accéder au site réel |
| Emplacement | NAD tourne sur un hôte Linux séparé, jamais dans le chemin du trafic |
| Profil de l'opérateur | Un ingénieur ; à l'aise avec Linux, Python, Go ; privilégie la ligne de commande |
| Règle absolue | Lecture seule sur les équipements de production : export de télémétrie uniquement |

## 2. Critères d'évaluation

Chaque candidat est jugé avec les mêmes questions :

1. **Passif et en lecture seule :** fonctionne-t-il à partir de la télémétrie
   exportée, sans toucher au routage ni au pare-feu ?
2. **Pièces mobiles :** combien de services, de bases de données et d'agents
   ajoute-t-il ?
3. **Reproductibilité :** peut-il être installé et configuré par Ansible ?
4. **Adapté au ML :** ses données s'exportent-elles sous forme tabulaire (SQL,
   CSV, Parquet) sans décodeur sur mesure ?
5. **Neutralité vis-à-vis du constructeur :** fonctionne-t-il encore si le
   routeur est remplacé ?
6. **Empreinte en ressources :** tient-il sur un hôte Linux modeste ?
7. **Maturité :** est-il maintenu, documenté et largement utilisé ?
8. **Adéquation aux compétences :** correspond-il à Python/Go et au travail en
   ligne de commande ?

## 3. Résumé des décisions

| ID | Composant | Décision | Second choix |
|---|---|---|---|
| D1 | Collecte de flux | goflow2 | nfdump/nfcapd |
| D2 | Métriques de l'équipement (SNMP) | Telegraf `inputs.snmp` | Prometheus + snmp_exporter |
| D3 | Sondes actives | Telegraf `inputs.ping` et `inputs.dns_query` | Scripts fping |
| D4 | Journaux | rsyslog | syslog-ng |
| D5 | Stockage | SQLite (WAL) + export Parquet | InfluxDB |
| D6 | Détecteur | Isolation Forest + référence saisonnière | LOF / One-Class SVM (comparaison uniquement) |
| D7 | Tableau de bord | Streamlit | Grafana |
| D8 | Alertes | Webhook optionnel depuis le détecteur | Grafana / Alertmanager |
| D9 | Déploiement | Ansible + systemd | Docker Compose |
| D10 | Provisionnement du labo (optionnel) | Terraform ou OpenTofu + libvirt | Ansible `community.libvirt` |
| D11 | État de la liaison montante (optionnel) | Outils gRPC de la parabole Starlink | aucun |

---

## 4. Composants

### 4.1 Collecte de flux (D1)

**En termes simples :** le routeur peut envoyer un court résumé de chaque
conversation qu'il transporte (qui a parlé à qui, sur quel port, combien
d'octets). Un collecteur de flux reçoit ces résumés et les transforme en
enregistrements lisibles.

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **goflow2** | Collecteur léger écrit en Go ; décode NetFlow v5/v9, IPFIX et sFlow ; sortie JSON | Tu construis ta propre chaîne et veux une sortie propre et facile à lire | Ne stocke, n'agrège et n'affiche rien ; l'ingestion est à écrire |
| nfdump / nfcapd | Collecteur classique, stockage binaire, CLI puissante pour l'agrégation | Tu préfères interroger en ligne de commande, ou analyser hors ligne | Format binaire nécessitant une conversion ; moins pratique en flux continu |
| pmacct (nfacctd) | Agrégation dès la collecte, nombreuses sorties (SQL, fichiers, brokers) | Tu veux des données pré-agrégées écrites directement en base | Configuration plus lourde, apprentissage plus raide |
| ntopng | Interface web avec statistiques d'hôtes en temps réel | Visibilité rapide ou démonstration en direct | Peu adapté à l'export d'un jeu de données ; certaines fonctions dépendent de la licence **(à vérifier)** |
| Telegraf `inputs.netflow` | Même agent que le SNMP et les sondes | Tu veux un seul agent pour tout | Disponibilité selon la version de Telegraf **(à vérifier)** ; moins de contrôle sur les champs bruts |
| Filebeat / Logstash netflow | Intégration dans une pile Elastic | ELK existe déjà | Lourd pour cet usage |
| Zeek / Suricata sur trafic miroir | Capture passive avec journaux riches par connexion | Un port miroir de commutateur est disponible | Nécessite un port miroir ou un tap ; CPU et stockage plus élevés |

**Décision D1 : goflow2.** Il fait une seule chose, se prête bien au script
(lignes JSON) et correspond à des compétences Go/Python. Chaque enregistrement
de flux est lu par l'ingesteur Python et agrégé en fenêtres temporelles.

**À réévaluer si :**

- L'entrée netflow de Telegraf est confirmée sur la version installée :
  fusionner flux et SNMP dans un seul agent supprime un composant.
- Un port miroir existe : ajouter Zeek pour des métadonnées plus fines par
  connexion.
- L'entreprise utilise déjà ntopng ou un autre outil de flux : le réutiliser
  plutôt que de dupliquer.

**À vérifier sur place :** si les flux exportés contiennent les adresses
internes ou les adresses traduites (NAT). L'attribution par utilisateur en
dépend. Vérifier aussi si l'accélération du routage (par exemple FastTrack sur
MikroTik) masque une partie du trafic à l'export de flux, et si l'export de flux
augmente sensiblement le CPU du routeur.

### 4.2 Métriques de l'équipement via SNMP (D2)

**En termes simples :** le SNMP est une façon standard de demander à un
équipement « combien de trafic est passé par ce port, à quel point ton CPU est-il
occupé ? ». Un interrogateur pose la question toutes les quelques secondes et
enregistre les réponses.

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **Telegraf `inputs.snmp`** | Interrogateur uniquement par configuration, nombreux formats de sortie | Tu veux zéro code pour une tâche banale | La configuration des tables SNMP est verbeuse |
| Python (pysnmp ou équivalent) | Contrôle total dans ton propre code | OID inhabituels ou logique sur mesure | À écrire et maintenir ; maintenance des bibliothèques inégale **(à vérifier)** |
| Prometheus + snmp_exporter | Supervision standard avec un large écosystème | Tu veux l'univers Prometheus (Grafana, Alertmanager) | Ajoute une base de séries temporelles et un modèle « pull » |
| Zabbix / LibreNMS / PRTG | Suites de supervision complètes (découverte, alertes) | L'entreprise en utilise déjà une | Lourd à installer pour cet usage seul ; extraire les données pour le ML passe par une API ou une base |
| API du constructeur (API RouterOS) | Données plus riches et spécifiques (baux DHCP, clients sans fil, files d'attente) | Tu dois associer une IP à un appareil ou un utilisateur | Spécifique au constructeur ; un adaptateur dédié par constructeur |

**Décision D2 : Telegraf SNMP.** L'interrogateur devient un fichier de
configuration qu'Ansible génère à partir d'un modèle, ce qui est à la fois plus
rapide et plus reproductible qu'un code sur mesure. L'API du constructeur reste
un **complément optionnel** pour associer IP et machine (baux DHCP).

**Notes de sécurité :** utiliser SNMPv3 si l'équipement le permet, sinon
restreindre la communauté à l'adresse de l'hôte NAD ; utiliser un compte en
lecture seule.

**À réévaluer si :** l'entreprise a déjà un système de supervision basé sur
SNMP. Récupérer ses données plutôt que d'interroger deux fois.

### 4.3 Sondes actives (D3)

**En termes simples :** l'hôte NAD envoie régulièrement un ping vers quelques
destinations et mesure le temps de réponse et la perte. C'est la vue la plus
directe de la santé de la liaison montante.

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **Telegraf `inputs.ping` + `inputs.dns_query`** | Même agent que le SNMP ; mesure aussi le temps de résolution DNS | Tu veux les sondes dans la même chaîne | Nécessite le droit d'envoyer de l'ICMP |
| fping / scripts ping | Très simple | Quelques cibles | Planification et analyse à ta charge |
| Smokeping | Historique long de la latence et de la gigue, bons visuels | Revue humaine de la qualité de la liaison | Le stockage RRD se prête mal à l'export pour le ML |
| blackbox_exporter | Sondes ICMP, HTTP, DNS et TCP | Tu utilises déjà Prometheus | Nécessite Prometheus |

**Décision D3 : Telegraf.** Sonder plusieurs cibles (la passerelle, le saut
suivant, et deux ou trois destinations publiques) pour distinguer un problème
local d'un problème en amont. Ajouter des sondes DNS et HTTP plus tard si des
métriques d'expérience utilisateur sont nécessaires.

### 4.4 Journaux (D4)

**En termes simples :** le routeur peut signaler des événements (interface
tombée, échec de connexion) à un récepteur de journaux. NAD les utilise surtout
sous forme de comptages par fenêtre temporelle.

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **rsyslog** | Répandu, récepteur simple écrivant dans des fichiers | Une boîte de réception fiable suffit | L'analyse est à ta charge |
| syslog-ng | Filtrage et analyse intégrés | Tu veux des champs structurés dès le départ | Un peu plus de configuration |
| Telegraf `inputs.syslog` | Garde tout dans un seul agent | Conception à agent unique | Attend un format syslog précis ; la compatibilité avec l'équipement est à contrôler **(à vérifier)** |
| Loki + agent | Stockage de journaux léger, par étiquettes | Pile centrée sur Grafana | Services supplémentaires |
| ELK / OpenSearch / Graylog | Recherche plein texte et analyses | Analyse de journaux poussée ou déploiement existant | Gourmand en ressources |

**Décision D4 : rsyslog.** Les journaux n'alimentent que des caractéristiques de
type comptage (événements par fenêtre, par catégorie) ; un récepteur simple
suffit.

### 4.5 Stockage (D5)

**En termes simples :** un endroit où conserver le tableau « une ligne par
minute » que lit le détecteur, plus un moyen de l'exporter comme jeu de données
du projet.

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **SQLite (mode WAL)** | Un fichier, SQL, aucune administration | Un seul écrivain, volume modéré | Écritures concurrentes limitées ; pas pour une ingestion intensive |
| **Fichiers Parquet** | Fichiers en colonnes compressés, adaptés au ML | Export du jeu de données et notebooks | Pas conçus pour des ajouts ligne par ligne |
| DuckDB | SQL analytique rapide sur Parquet et SQLite | Exploration et notebooks | Un seul écrivain ; pas un stockage de flux continu |
| InfluxDB | Séries temporelles natives, rétention et sous-échantillonnage | Métriques continues avec tableau de bord | Service supplémentaire ; le langage de requête diffère selon les versions majeures **(à vérifier)** |
| TimescaleDB | PostgreSQL avec fonctions de séries temporelles | L'équipe utilise déjà PostgreSQL | Surcharge d'administration |
| VictoriaMetrics / Prometheus TSDB | Stockage efficace de métriques | Chaînes limitées aux métriques | Mal adapté aux enregistrements de flux |
| ClickHouse | Très gros volumes de données de flux | Rétention longue ou grands réseaux | Surdimensionné à cette échelle |
| Elasticsearch / OpenSearch | Recherche et analyses | Une pile ELK existe | Gourmand en ressources |

**Décision D5 : SQLite en mode WAL pour le stockage opérationnel, Parquet pour
les exports.** Les fenêtres agrégées sont minuscules (une ligne par minute). La
partie volumineuse, ce sont les fichiers de flux bruts, à compresser et à faire
tourner après ingestion. DuckDB est un outil optionnel pratique pour les
notebooks.

**À réévaluer si :** plusieurs processus doivent écrire en même temps, la
rétention passe à plusieurs mois, ou le volume de flux devient important.

### 4.6 Détecteur (D6)

**En termes simples :** le détecteur apprend à quoi ressemble une minute normale
à un moment donné de la semaine et signale les minutes qui paraissent
inhabituelles. Deux méthodes complémentaires sont combinées : l'une regarde de
nombreuses mesures ensemble, l'autre connaît le rythme de la semaine.

| Méthode | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **Isolation Forest** | Rapide, peu d'hypothèses, nombreuses caractéristiques | Premier modèle ; caractéristiques numériques variées | Aucune notion du temps ; le seuil demande des réglages |
| **Référence saisonnière** (z-score robuste par heure de la semaine, ou STL / Holt-Winters via statsmodels) | Cycles quotidiens et hebdomadaires explicites, très explicable | Rythmes d'usage marqués | Besoin d'assez d'historique par case ; les cases peu remplies demandent un repli (par exemple heure du jour seule) |
| Local Outlier Factor | Comparaison de densité locale | Groupes de densités différentes | Sensible au nombre de voisins ; nécessite le mode novelty pour scorer de nouvelles données |
| One-Class SVM | Frontière autour des données normales | Petits jeux de données | Passe mal à l'échelle ; sensible au noyau et aux paramètres ; demande une mise à l'échelle |
| Elliptic Envelope / Mahalanobis | Peu coûteux, statistique | Caractéristiques à peu près gaussiennes | Faible sur un trafic multimodal |
| Modèles à résidus de prévision (type Prophet) | Saisonnalité plus jours fériés | Longs historiques | Dépendance plus lourde ; plus de mise en place |
| Autoencodeur / LSTM | Motifs temporels complexes | Grands jeux de données | Gourmand en données, plus difficile à expliquer, réglages lourds |
| PyOD | API unifiée sur de nombreux détecteurs | Expériences de comparaison côte à côte | Une dépendance de plus |

**Décision D6 : Isolation Forest combinée à une référence saisonnière robuste.**
La forêt gère les combinaisons de caractéristiques ; la référence gère le temps.
Chaque événement d'anomalie enregistre ses principales caractéristiques
contributrices pour pouvoir être expliqué. LOF et One-Class SVM ne sont
exécutés qu'en comparaison dans un notebook.

**Instabilité de la liaison montante :** les liaisons satellites ont des
variations naturelles de latence et de brèves coupures. Garder les
caractéristiques de qualité de liaison séparées et les juger par rapport à leur
propre référence, pour que la variation normale de la liaison ne soit pas
signalée comme un comportement anormal interne.

**Comportement des petits réseaux :** sur un réseau comptant peu d'appareils, un
seul appareil peut dominer le trafic, si bien que les pics normaux paraissent
importants. Privilégier des statistiques robustes (médiane et MAD plutôt que
moyenne et écart-type), envisager des fenêtres plus larges (par exemple 5
minutes) pour lisser les pics courts, et commencer avec des seuils tolérants en
les resserrant selon le taux de fausses alertes observé. Les caractéristiques de
diversité (hôtes distincts) apportent moins de signal ici, tandis que des
profils par appareil deviennent envisageables et pourront être ajoutés plus
tard.

**À réévaluer si :** l'historique est assez long pour des méthodes à base de
prévision, ou si les cases saisonnières sont trop peu remplies (repasser à des
cases plus grossières).

### 4.7 Tableau de bord (D7)

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **Streamlit** | Interface web 100 % Python | Affichage sur mesure (anomalies mises en évidence, tableau d'événements) | Pas d'alertes intégrées ; pas d'authentification intégrée |
| Grafana | Tableaux de bord de supervision standard, alertes, nombreuses sources | Les données sont dans un stockage de séries temporelles compatible | Mettre en évidence les scores du modèle demande une configuration en plus ; le support SQLite passe par un plugin **(à vérifier)** |
| Dash (Plotly) | Plus de contrôle que Streamlit | Une application structurée et soignée | Plus de code répétitif |
| Textual / Rich (interface terminal) | Vue d'état en ligne de commande | Vérifications rapides en SSH | Pas adapté au partage avec des non-techniciens |
| Kibana | Intégré à ELK | ELK existe déjà | Utile seulement avec cette pile |

**Décision D7 : Streamlit.** La vue des anomalies est une logique sur mesure,
plus simple à écrire en Python. Streamlit n'a pas d'authentification propre : le
lier à une interface interne, ou le placer derrière un reverse proxy ou un VPN.

### 4.8 Alertes (D8)

| Option | Convient quand | Limites |
|---|---|---|
| **Table d'événements + tableau de bord (par défaut)** | Rien de plus n'est nécessaire | Quelqu'un doit consulter le tableau de bord |
| Webhook depuis le détecteur (outil de messagerie, ntfy, bot de type Telegram) | Notifications push simples | Le choix du canal dépend de l'entreprise |
| E-mail (SMTP) | L'entreprise s'appuie sur l'e-mail | Nécessite un relais de messagerie joignable |
| Alertes Grafana / Alertmanager | Une pile Grafana ou Prometheus existe | Nécessite cette pile |

**Décision D8 :** écrire les événements dans le stockage et les afficher sur le
tableau de bord ; ajouter un webhook optionnel une fois connu le canal préféré
de l'entreprise.

### 4.9 Déploiement et configuration (D9)

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **Ansible + systemd** | Sans agent, rôles idempotents via SSH | Un ou quelques hôtes, sans couche de conteneurs | Nécessite Python sur la cible ; les modifications manuelles créent une dérive |
| Docker Compose | Piles multi-services empaquetées et portables | Un serveur avec un moteur de conteneurs | Ajoute un moteur ; les récepteurs UDP demandent une configuration soignée des ports ou du réseau hôte |
| Ansible déployant Compose | Le meilleur des deux | Un hôte de classe serveur | Deux couches à comprendre |
| Scripts shell | Rapide | Installations jetables | Non idempotents |
| Puppet / Salt / Chef | Grandes flottes | Nombreux hôtes | Surdimensionnés ici |
| Kubernetes / Nomad | Orchestration à grande échelle | Grandes plateformes | Très au-delà de ce périmètre |

**Décision D9 : Ansible avec des services systemd.** Les mêmes rôles doivent
fonctionner pour l'hôte du labo et pour un hôte réel ; ils ne doivent donc pas
supposer comment l'hôte a été créé. Garder un inventaire par environnement et
les secrets hors de Git (Ansible Vault ou fichiers d'environnement).

### 4.10 Provisionnement du labo, optionnel (D10)

Le labo recrée la pile dans des machines virtuelles pour la tester sans toucher
au réseau réel. Il n'est pas nécessaire en production.

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **Terraform + provider libvirt** | VM déclaratives sur KVM | Tu veux de l'infrastructure as code pour le labo | Gestion de l'état ; le schéma du provider a changé selon les versions, donc fixer la version **(à vérifier)** |
| OpenTofu | Fork open source de Terraform, largement compatible | Préférence de licence | Mêmes réserves sur le provider |
| Ansible `community.libvirt` | Gérer les VM avec Ansible seul | Tu veux moins d'outils | Moins déclaratif que Terraform |
| Vagrant | VM de développement rapides | Installations locales simples | Maintenance des plugins ; licence HashiCorp |
| virt-install + scripts cloud-init | Minimal | VM ponctuelles | Pas de suivi d'état |
| Incus / LXD | Conteneurs système, légers et rapides | Labos à faibles ressources | Pas de vraies VM |
| Images de routeurs virtuels (par exemple RouterOS CHR) | Tester l'automatisation d'un routeur sans risque | Tu as besoin d'un routeur sans toucher au matériel | Image x86 ; pas de Wi-Fi ni certaines fonctions matérielles |

**Décision D10 : Terraform ou OpenTofu avec libvirt, optionnel.** À traiter
comme un confort de labo, jamais comme une dépendance du produit.

### 4.11 État de la liaison montante, optionnel (D11)

| Outil | Spécialité | Convient quand | Limites |
|---|---|---|---|
| **Outils gRPC de la parabole Starlink (communautaires)** | État de la parabole : latence, obstruction, coupures, débit | Distinguer les pannes de la liaison montante des problèmes internes | Interface non officielle susceptible de changer avec le firmware ; joignable seulement depuis le réseau de la parabole **(à vérifier)** |

**Décision D11 :** une source supplémentaire optionnelle derrière un adaptateur
de collecte. Elle ne doit pas se trouver sur le chemin critique.

---

## 5. Profils de pile selon l'environnement

La chaîne n'a besoin que d'un flux normalisé de fenêtres ; la pile peut donc
être échangée selon l'environnement en changeant les collecteurs et, si besoin,
le stockage et le tableau de bord.

| Composant | A. Minimal (prototype actuel) | B. Routeur puissant + serveur | C. Supervision existante | D. Accès restreint |
|---|---|---|---|---|
| Flux | goflow2 | goflow2, plus port miroir et Zeek si disponibles | goflow2 ou réutiliser l'outil de flux de l'entreprise | Capture passive sur port miroir, ou aucune |
| Métriques de l'équipement | Telegraf SNMP | Prometheus + snmp_exporter, ou Telegraf | Récupérer depuis le système existant (API ou base) | SNMP seulement si autorisé |
| Sondes | Telegraf | Telegraf ou blackbox_exporter | Réutiliser les sondes existantes si comparables | Telegraf (aucun accès au routeur nécessaire) |
| Journaux | rsyslog | Loki ou ELK | Plateforme de journaux existante | rsyslog si l'export syslog est autorisé |
| Stockage | SQLite + Parquet | TimescaleDB / VictoriaMetrics ; ClickHouse si le volume de flux est élevé | Stockage existant plus SQLite pour les fenêtres | SQLite + Parquet |
| Détecteur | Isolation Forest + saisonnier | Idem, avec comparaisons | Idem | Idem |
| Tableau de bord | Streamlit | Grafana (+ Alertmanager) | Tableaux de bord existants plus Streamlit pour les scores | Streamlit |
| Déploiement | Ansible + systemd | Ansible + Docker Compose | Ansible | Ansible |

## 6. Modes dégradés

| Condition | Comment la repérer | Repli | Ce qui est perdu |
|---|---|---|---|
| Export de flux indisponible ou incomplet | Aucun paquet de flux n'arrive, ou les compteurs divergent du SNMP | Compteurs SNMP et sondes uniquement | Attribution aux hôtes et aux destinations |
| SNMP bloqué | L'interrogateur renvoie des délais dépassés | Flux plus sondes ; API du constructeur si autorisée | Métriques d'interfaces et de CPU |
| Export syslog non autorisé | Aucun événement de journal | Abandonner les caractéristiques de journaux | Contexte des événements de l'équipement |
| CPU du routeur trop élevé avec l'export activé | CPU élevé après activation de la télémétrie | Réduire le périmètre d'export ou l'échantillonnage ; interroger moins souvent | Finesse |
| Aucun accès au routeur | Demande refusée | Capture passive depuis un port miroir de commutateur | Métriques de santé du routeur |
| Pas encore assez d'historique | Cases saisonnières peu remplies | Cases plus grossières (heure du jour) ; développer sur des jeux publics et du trafic de labo | Précision du cycle hebdomadaire |
| Conservation des flux bruts non autorisée | Décision de politique interne | Garder seulement les fenêtres agrégées ; supprimer les flux bruts après ingestion | Possibilité de ré-agréger ou d'investiguer |
| État de la parabole injoignable | L'outil gRPC ne peut pas se connecter | Ignorer la source optionnelle | Attribution des pannes de liaison montante |

## 7. Questions à poser sur place

Elles décident quel profil de la section 5 s'applique.

| Question | Pourquoi c'est important | Influence |
|---|---|---|
| Modèle du routeur et version de RouterOS ? | Fonctions de télémétrie et marge de CPU | D1, D2 |
| L'accélération du routage (FastTrack) est-elle activée ? | Peut masquer du trafic à l'export de flux | D1 |
| Les flux exportés montrent-ils les adresses internes ou traduites (NAT) ? | Attribution par utilisateur | D1, caractéristiques |
| Combien d'appareils sont connectés, et lesquels sont partagés ou non gérés ? | Taille des fenêtres, seuils et utilité des caractéristiques de diversité | D6, caractéristiques |
| Existe-t-il un commutateur administrable avec port miroir ? | La capture passive et Zeek deviennent possibles | Modes dégradés, profils B/D |
| Quel matériel serveur est disponible, et supporte-t-il la virtualisation ? | Conteneurs, labo, stockage plus lourd | D9, D10, profil B |
| Une supervision ou une gestion de journaux existe-t-elle déjà ? | Réutiliser plutôt que dupliquer | Profil C |
| Règles de rétention et de confidentialité pour les données de flux ? | Ce qui peut être stocké et pour combien de temps | D5, section 6 |
| Où tournent le DHCP, le DNS et le Wi-Fi ? | Associer les IP aux utilisateurs et aux appareils | API RouterOS optionnelle, caractéristiques |
| L'adresse de gestion de la parabole est-elle joignable depuis le LAN ? | Source optionnelle de la liaison montante | D11 |
| Qui autorise les tests d'injection d'anomalies, et quand ? | La validation nécessite une fenêtre | Plan de validation |

## 8. Vérifications à faire dès maintenant sur le routeur de prototypage

Avant d'écrire du code, confirmer la chaîne de télémétrie de bout en bout sur le
matériel de prototypage. Ce sont des vérifications en lecture seule sur le
routeur :

```
/system resource print
/ip firewall filter print where action=fasttrack-connection
/ip traffic-flow print
/snmp print
```

Puis sur l'hôte NAD :

```bash
# Do flow packets arrive? (2055 is the common NetFlow/IPFIX port, check your config)
sudo tcpdump -ni any udp port 2055

# Does SNMP answer? (replace the address and credentials)
snmpwalk -v2c -c <community> <router-ip> 1.3.6.1.2.1.2.2.1.10

# Do syslog messages arrive?
sudo tcpdump -ni any udp port 514
```

Noter le modèle du routeur, la version de RouterOS, la charge CPU avec et sans
export activé, et si les octets comptés par les flux correspondent à peu près
aux compteurs d'interface SNMP. Ces résultats alimentent directement les
décisions D1 et D2 ainsi que les questions à poser sur place ci-dessus.

## 9. Points ouverts à vérifier

- Si l'entrée netflow de Telegraf est disponible et suffisante sur la version
  installée (ce qui permettrait de fusionner D1 dans D2/D3).
- Le format syslog envoyé par le routeur comparé à celui qu'attend chaque
  récepteur.
- L'effet de l'accélération du routage sur l'exactitude des flux.
- Si les flux exportés montrent des adresses avant ou après NAT.
- L'état actuel de la version du provider libvirt choisie pour Terraform ou
  OpenTofu.
- La joignabilité de l'interface de gestion de la parabole depuis le LAN.
