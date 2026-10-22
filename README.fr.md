# NAD — Network Anomaly Detection

**Français** | [English](README.md)

> Apprend à quoi ressemble un fonctionnement « normal » sur un réseau
> d'entreprise, puis signale automatiquement les comportements inhabituels.

**Statut :** en conception — prototype, pas prêt pour la production

## Pourquoi

Les réseaux de petite et moyenne taille ont rarement quelqu'un qui surveille des
tableaux de bord toute la journée. Les seuils d'alerte fixes cessent de
fonctionner dès que l'usage change entre un lundi matin et un dimanche soir.
NAD apprend le rythme normal du réseau à partir de ses propres données et
signale ce qui s'en écarte.

## Ce que fait NAD

- Collecte la télémétrie réseau : débit, latence et perte de paquets,
  enregistrements de flux (qui parle à qui), journaux du routeur.
- Construit une référence du comportement normal, y compris les cycles
  quotidiens et hebdomadaires.
- Note le nouveau trafic par rapport à cette référence avec un détecteur non
  supervisé.
- Affiche des statistiques et les anomalies mises en évidence dans un tableau de
  bord simple.

## Environnement de référence

Un réseau d'entreprise d'une trentaine d'utilisateurs. Le routeur principal est
un MikroTik RB1100AHx4 connecté à la liaison Starlink (double NAT) ; tous les
switchs et autres routeurs se branchent en aval de celui-ci. Le routeur exporte
la télémétrie ; NAD tourne sur un hôte Linux séparé et ne se trouve jamais dans
le chemin du trafic.

## Architecture

![Architecture de NAD](docs/diagrams/nad-containers-fr.png)

| Composant | Rôle | Implémentation |
|---|---|---|
| Collecteur SNMP et sondes | Compteurs du routeur, latence, perte de paquets | Telegraf |
| Collecteur de flux | Décode les enregistrements de flux | goflow2 |
| Récepteur de journaux | Stocke les journaux du routeur | rsyslog |
| Ingesteur | Analyse les fichiers bruts, normalise en fenêtres temporelles | Python |
| Stockage | Fenêtres, scores, événements d'anomalie, export du jeu de données | SQLite + Parquet |
| Détecteur | Construction des caractéristiques, scoring, référence | Python, scikit-learn |
| Tableau de bord | Statistiques et anomalies mises en évidence | Streamlit |
| Alerte | Notification optionnelle lors de nouvelles anomalies | Webhook |
| Transport | Achemine la télémétrie vers l'hôte NAD quand il n'est pas sur le réseau local | WireGuard (optionnel) |
| Déploiement | Configure l'hôte de façon reproductible | Ansible + systemd |

### Topologies de déploiement

Le routeur atteint l'hôte NAD de l'une de ces deux façons. La chaîne de
traitement est la même dans les deux cas ; seul le chemin de la télémétrie
change.

| Topologie | Fonctionnement | Convient quand |
|---|---|---|
| **Machine locale** | Une machine Linux du réseau local héberge NAD et reçoit la télémétrie directement du routeur | Le site peut héberger une machine ; le plus simple, et aucune donnée ne quitte le site |
| **VPN (WireGuard)** | L'hôte NAD est distant (par exemple un VPS). Une machine ou le routeur du site ouvre un tunnel WireGuard vers lui, et la télémétrie circule dans le tunnel | Pas de machine disponible sur place pour NAD, ou NAD doit être joignable de l'extérieur |

Remarques :

- Le tunnel est toujours ouvert depuis le site vers l'hôte distant, il traverse
  donc le NAT. WireGuard natif sur le routeur demande RouterOS 7 ; sinon, une
  petite machine Linux du réseau local porte le tunnel et le routeur lui exporte
  la télémétrie en local.
- Les sondes ICMP doivent partir d'une machine du réseau surveillé. Un hôte NAD
  distant mesurerait sa propre liaison, pas celle de l'entreprise.

## Spécification

### Sources de données

| Source | Contenu | Transport |
|---|---|---|
| Compteurs | Débit des interfaces, CPU, mémoire | Interrogation SNMP |
| Flux | Source, destination, ports, octets, paquets | NetFlow/IPFIX sur UDP |
| Sondes | Temps d'aller-retour et perte vers plusieurs cibles | ICMP |
| Journaux | Événements du routeur (changements d'interface, connexions) | Syslog sur UDP |

### Fenêtres normalisées

Toutes les sources sont agrégées en fenêtres de durée fixe (1 minute par défaut,
configurable). Une ligne par fenêtre, un seul schéma quel que soit
l'équipement source.

### Caractéristiques

- Trafic : octets et paquets montants/descendants, nombre de flux, rapport
  envoi/réception
- Diversité : hôtes internes distincts, destinations distinctes, ports
  distincts, part de l'hôte le plus bavard
- Qualité de la liaison : temps d'aller-retour (moyenne et p95), perte de
  paquets, par cible
- Santé de l'équipement : CPU, mémoire, nombre d'événements de journal
- Contexte temporel : heure du jour et jour de la semaine (encodage cyclique)

Les caractéristiques par hôte sont calculées à partir des flux sortants. Les
flux entrants sont exportés avant la traduction inverse du NAT : ils portent
donc l'adresse WAN de la passerelle au lieu de l'hôte interne ; ils comptent
quand même dans les totaux.

### Détection

- Une Isolation Forest note chaque fenêtre par rapport à la référence apprise.
- Une référence saisonnière (statistiques par heure de la semaine) détecte les
  écarts que la forêt ne peut pas voir, faute de notion du temps.
- Les deux scores sont combinés en un score d'anomalie ; le seuil est
  configurable.
- Chaque événement d'anomalie enregistre un horodatage, un score et ses
  principales caractéristiques contributrices, pour pouvoir être expliqué.
- Les caractéristiques de qualité de liaison sont gardées séparées pour
  distinguer les problèmes de liaison d'un comportement anormal à l'intérieur du
  réseau. Sonder à la fois le routeur Starlink (premier saut) et des cibles sur
  Internet permet de distinguer la liaison locale de la liaison satellite.

### Tableau de bord

Séries temporelles avec les anomalies mises en évidence, état courant, hôtes les
plus bavards et tableau des événements d'anomalie.

### Exigences non fonctionnelles

- Lecture seule sur les équipements de production : export de télémétrie
  uniquement, aucun changement de routage ni de pare-feu.
- Tourne sur un seul hôte Linux modeste.
- Le déploiement est automatisé et reproductible.
- Aucun identifiant ni clé dans le dépôt ; configuration par fichiers
  d'environnement.
- La télémétrie qui quitte le site est chiffrée (WireGuard), et les services de
  NAD écoutent uniquement sur des adresses internes ou du tunnel, jamais sur une
  adresse publique.
- Les données brutes ne quittent jamais l'hôte ; les données partagées sont
  anonymisées.

## Décisions technologiques

| Composant | Décision | Pourquoi | Si l'environnement offre plus |
|---|---|---|---|
| Compteurs et sondes | Telegraf | Configuration seule, aucun code à maintenir, généré par Ansible | Prometheus avec un exporteur SNMP, ou réutiliser Zabbix/LibreNMS s'ils existent |
| Flux | goflow2 | Léger, écrit en Go, gère NetFlow v9, IPFIX et sFlow | Même collecteur avec un routeur plus capable ; port miroir + Zeek pour plus de visibilité |
| Journaux | rsyslog | Récepteur fiable le plus simple | Loki ou ELK |
| Stockage | SQLite + Parquet | Un seul fichier, aucune administration, Parquet adapté au ML | TimescaleDB ou VictoriaMetrics ; ClickHouse pour de gros volumes de flux |
| Détecteur | Isolation Forest + référence saisonnière | Peu d'hypothèses, explicable, gère de nombreuses caractéristiques | Ajouter LOF ou One-Class SVM en comparaison |
| Tableau de bord | Streamlit | Contrôle total du rendu des anomalies, Python pur | Grafana + Alertmanager |
| Transport | WireGuard, hôte distant comme point de rencontre (ou réseau local simple) | Peu de pièces mobiles, natif sur RouterOS 7, un seul port UDP | Tailscale ou Headscale pour de nombreux appareils |
| Déploiement | Ansible + systemd | Un seul hôte, aucune couche de conteneurs à maintenir | Docker Compose sur un serveur |

Le comparatif complet des outils candidats, avec leurs forces et leurs limites,
se trouve dans [`docs/tools/tooling.fr.md`](docs/tools/tooling.fr.md).

## Modes dégradés

Le détecteur n'a besoin que d'un flux normalisé de fenêtres ; la chaîne se
dégrade donc proprement lorsqu'une source est indisponible :

1. **Pas d'export de flux :** compteurs SNMP et sondes uniquement. Les pics et
   la dérive de latence sont toujours détectés, sans attribution à un hôte.
2. **Pas d'accès au routeur :** capture passive depuis un port miroir de
   commutateur.
3. **Pas encore d'historique local :** développer et tester le détecteur sur des
   jeux de données publics et sur un réseau de labo jusqu'à disposer de données
   réelles.

## Principes de conception

1. **Des adaptateurs en périphérie.** Changer de routeur ou de source de
   données ne touche que les collecteurs, pas le reste de la chaîne.
2. **La référence d'abord.** La collecte démarre tôt pour que le détecteur
   s'appuie sur un vrai historique.
3. **Non supervisé par défaut.** Aucune attaque étiquetée n'est nécessaire. Les
   anomalies de test injectées servent uniquement à valider la détection.
4. **Lecture seule sur la production.** NAD ne modifie jamais le routage ni les
   règles de pare-feu.
5. **Reproductible.** Le déploiement est décrit sous forme de code et peut être
   rejoué.

## Périmètre

| Dans le périmètre | Hors périmètre |
|---|---|
| Détection et rapport | Blocage ou remédiation automatiques |
| Métadonnées (compteurs, flux, latence) | Inspection du contenu des paquets |
| Un site, un routeur principal | Corrélation multi-sites |

## Organisation du dépôt

```
README.md       Documentation (anglais)
README.fr.md    Documentation (français)
Makefile        Raccourcis (make diagrams)
infra/          Automatisation du déploiement (Ansible) ; labo optionnel (Terraform, reporté)
src/nad/        Code applicatif : collect, features, model, detect
scenarios/      Anomalies injectées avec les étiquettes attendues
data/           Local uniquement (ignoré par git) plus de petits échantillons anonymisés
docs/           Architecture, outils, rapport
  concepts/     Explications des notions clés (qu'est-ce qu'une anomalie)
  diagrams/     Sources PlantUML (.puml) et images générées (.png, .svg)
  tools/        Comparatifs d'outils et choix technologiques
  guides/       Guides de procédure pas à pas (installation, justification, commandes)
```

## Validation

Des anomalies sont injectées volontairement (pic de trafic, scan de ports,
saturation de connexions) dans une fenêtre autorisée. La détection est mesurée
par rapport à la chronologie connue : taux de détection et fausses alertes par
jour.

## Sécurité et confidentialité

Les données de flux sont des métadonnées sur l'activité de personnes réelles.
Les données brutes restent hors de Git, et tout jeu de données partagé est
d'abord anonymisé (adresses, noms d'hôtes). L'export de télémétrie sur le
routeur est restreint à l'hôte NAD (ou à l'adresse du tunnel), avec un compte
dédié en lecture seule. Les clés WireGuard sont générées par appareil, jamais
réutilisées et jamais commitées.

## Feuille de route

- [ ] Télémétrie activée sur le routeur principal
- [ ] Transport sécurisé vers l'hôte NAD (tunnel WireGuard, ou machine du réseau local)
- [ ] Collecte en fonctionnement sur l'hôte NAD
- [ ] Ingestion en fenêtres normalisées
- [ ] Détecteur et référence
- [ ] Validation avec des anomalies injectées
- [ ] Tableau de bord
- [ ] Déploiement Ansible
- [ ] Automatisation du labo (optionnel, reporté)
- [ ] État de la parabole comme source supplémentaire (optionnel)
