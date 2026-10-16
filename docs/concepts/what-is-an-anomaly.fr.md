# Qu'est-ce qu'une anomalie réseau ?

**Français** | [English](what-is-an-anomaly.md)

Ce document définit ce que NAD appelle une « anomalie », afin que ses résultats
puissent être lus et jugés correctement.

## En termes simples

Une anomalie réseau est **un comportement qui ne ressemble pas à ce que le réseau
fait d'habitude**. Ce n'est pas forcément une attaque, ni même un problème. C'est
un écart.

Imagine un gardien qui connaît le rythme d'un bâtiment. Une porte qui s'ouvre à
14 h un mardi ne le surprend pas ; la même porte ouverte à 3 h du matin un
dimanche, si. NAD joue ce rôle : il apprend le rythme normal, puis signale ce qui
en sort.

## Trois formes d'anomalie

| Forme | Description simple | Exemple |
|---|---|---|
| **Anomalie ponctuelle** | Une seule valeur extrême | Un débit dix fois supérieur à la normale pendant une minute |
| **Anomalie contextuelle** | Une valeur banale en soi, mais inhabituelle *à ce moment-là* | Un fort trafic à 3 h du matin un dimanche, normal un lundi matin |
| **Anomalie collective** | Chaque minute paraît normale isolément, mais la série est étrange | Une machine qui contacte une nouvelle destination toutes les 60 secondes pile |

Les anomalies contextuelles expliquent pourquoi NAD apprend des rythmes
hebdomadaires (par heure de la semaine) plutôt que de s'appuyer sur des seuils
fixes.

## Ce qui peut en être la cause

| Origine | Exemple | Ce que NAD voit |
|---|---|---|
| Panne ou dégradation | Liaison satellite qui perd des paquets, routeur saturé | Latence et perte de paquets en hausse, CPU élevé |
| Mauvaise configuration | Boucle réseau, service qui réessaie sans fin | Explosion soudaine du nombre de flux ou de paquets |
| Usage inhabituel mais légitime | Grosse mise à jour, sauvegarde, streaming vidéo | Volume montant ou descendant élevé |
| Événement de sécurité | Scan de ports, inondation de connexions, machine compromise, données qui sortent du réseau | Nombreux ports ou destinations distincts, très grand nombre de flux, volume montant anormal |
| Événement nouveau et légitime | Nouvel appareil, nouvelle application | Une fausse alerte à examiner |

## Ce que NAD dit, et ce qu'il ne dit pas

NAD répond à **« est-ce inhabituel ? »**, pas à **« est-ce malveillant ? »**. Une
alerte est un point de départ pour une investigation humaine.

Pour aider cette investigation, chaque événement d'anomalie enregistre les
caractéristiques qui y ont le plus contribué. On peut alors le lire ainsi :
« signalé à 14 h 32 parce que le nombre de ports de destination distincts a été
multiplié par quarante », ce qui oriente vers un scan plutôt qu'une panne.

## Limites à garder en tête

- **Ce qui était présent pendant l'apprentissage devient « normal ».** Si le
  réseau fonctionnait mal pendant la collecte de la référence, ce comportement
  est appris comme une habitude.
- **Les attaques discrètes peuvent rester sous le seuil.** Une activité qui reste
  dans les valeurs habituelles, comme une fuite de données très lente, peut ne
  pas ressortir.
- **Le contenu des paquets n'est pas inspecté.** NAD ne voit que des métadonnées
  (compteurs, flux, latence). C'est volontaire et hors du périmètre du projet.
- **Il ne remplace pas la détection par signatures.** Les systèmes de détection
  d'intrusion à base de règles (par exemple Suricata) reconnaissent des motifs
  connus ; NAD signale les écarts même sans règle. Les deux approches se
  complètent.

## Définition de travail

> Une anomalie est une fenêtre temporelle dont le vecteur de caractéristiques
> (trafic, diversité, qualité de liaison, santé de l'équipement) s'écarte de la
> référence apprise pour son contexte horaire dans la semaine, au-delà d'un seuil
> configurable.

Cette définition rend le sujet mesurable, ce sur quoi s'appuie la validation.

## Lien avec la validation

La validation injecte des anomalies connues pendant une fenêtre autorisée et
vérifie qu'elles sont détectées. Chacune illustre une catégorie ci-dessus.

| Anomalie injectée | Outil | Catégorie illustrée | Signal attendu |
|---|---|---|---|
| Transfert massif | iperf3 | Usage inhabituel mais légitime | Volume en forte hausse |
| Scan de ports | nmap | Événement de sécurité | Ports et destinations distincts en hausse |
| Inondation de connexions | hping3 | Événement de sécurité, mauvaise configuration | Nombre de flux et de paquets en explosion |

Les pannes de la liaison montante ne peuvent pas être injectées, mais elles se
produisent naturellement et s'observent. Elles servent à vérifier que les
problèmes de liaison sont distingués des anomalies internes au réseau, grâce aux
caractéristiques de qualité de liaison tenues à part des autres.

## Documents liés

- [README](../../README.fr.md) : vue d'ensemble du projet et architecture
- [Comparatif des outils](../tools/tooling.fr.md) : outils et décisions
- [Guide 01](../guides/01-telemetry-and-ingestion.md) : mise en place de la
  télémétrie et conception de l'ingesteur (en anglais)
