# Guide 01 — Mise en place de la télémétrie et conception de l'ingestion

**Français** | [English](01-telemetry-and-ingestion.md)

**Public :** toute personne qui reproduit le prototype, ou un assistant IA qui
reprend le travail.
**Objectif :** faire circuler une télémétrie réelle du routeur passerelle vers
l'hôte NAD, puis concevoir l'ingesteur à partir d'échantillons réels.
**Statut :** en cours. Export de flux testé le 2026-10-15 (mode sûr, annulé
ensuite). SNMP, syslog et le recoupement ne sont pas encore testés. Remplis le
journal des résultats (section 11) au fur et à mesure.

Les éléments marqués **(à vérifier)** dépendent de la version de RouterOS ou de
l'outil et doivent être confrontés à la documentation à jour.

## 1. Pourquoi cet ordre

L'ordre est le suivant : configurer le routeur, préparer l'hôte à recevoir,
capturer des échantillons réels, les recouper, et seulement ensuite écrire
l'ingesteur.

- **Un parseur écrit à partir d'un format imaginé finit réécrit.** Les vrais
  enregistrements de flux ont des noms de champs et des particularités qu'on ne
  découvre qu'après en avoir capturé.
- **Chaque étape produit un résultat visible.** Si quelque chose ne va pas
  (aucun paquet, réponses SNMP vides), tu sais quel maillon de la chaîne est
  cassé.
- **Le recoupement protège tout le projet.** Si les données de flux ne
  correspondent pas à peu près aux compteurs du routeur lui-même, tous les
  résultats suivants seraient bâtis sur du sable.

## 2. Modèle mental

Le routeur communique avec NAD par trois canaux. En termes simples : NAD peut
*poser des questions* au routeur (SNMP), et le routeur peut *envoyer* de lui-même
deux types de messages (résumés de flux et journaux d'événements).

| Canal | Sens | Qui l'initie | Contenu | Port |
|---|---|---|---|---|
| SNMP | NAD vers routeur | NAD interroge | Compteurs, CPU, mémoire | UDP 161 |
| Flux (NetFlow/IPFIX) | Routeur vers NAD | Le routeur pousse | Résumés des conversations | UDP 2055 (convention) |
| Syslog | Routeur vers NAD | Le routeur pousse | Événements (interface coupée, échec de connexion) | UDP 514 |

Ce qui en découle :

- Configurer le routeur revient à lui dire **où envoyer** et **qui peut
  l'interroger**.
- L'hôte doit **écouter avant** que le routeur commence à envoyer.
- Les flux et le syslog passent par UDP : il n'y a pas de renvoi, donc des
  paquets peuvent se perdre sans bruit. C'est pour cela qu'un recoupement existe.
- Le routeur n'est que le **messager** : sur l'hôte, chaque paquet de flux vient
  du routeur, mais son contenu décrit les conversations de tous les
  utilisateurs.

## 3. Prérequis

- Un accès au routeur (SSH ou WinBox) avec le droit de modifier les réglages de
  télémétrie.
- Un hôte Linux sur le même réseau local que le routeur (le futur hôte NAD), de
  préférence en câble plutôt qu'en Wi-Fi (les flux et le syslog passent en UDP,
  sans renvoi), avec une adresse fixe (bail DHCP statique).
- Sur l'hôte : `tcpdump`, `nc` (netcat), `jq`, les outils en ligne de commande
  Net-SNMP (`snmpwalk`, `snmpget`), `rsyslog` et `goflow2` (un binaire publié,
  comme `goflow2-2.2.7-linux-amd64`, fonctionne sans installer Go).
- L'horloge du routeur synchronisée (voir l'étape 1.4).

## 4. Vue d'ensemble du déroulé

```mermaid
flowchart LR
  S0["Étape 0<br/>Sauvegarder la config du routeur"] --> S1["Étape 1<br/>Activer la télémétrie sur le routeur"]
  S1 --> S2["Étape 2<br/>Préparer les récepteurs sur l'hôte"]
  S2 --> S3["Étape 3<br/>Capturer des échantillons réels"]
  S3 --> S4["Étape 4<br/>Recouper flux et SNMP"]
  S4 --> S5["Étape 5<br/>Concevoir l'ingesteur"]
```

En pratique, l'étape 2 (l'hôte qui écoute) doit être prête **avant** que le
routeur commence à exporter à l'étape 1.

---

## 5. Étape 0 — Filet de sécurité

**Pourquoi :** les changements de télémétrie sont légers, mais une erreur sur
une passerelle peut couper la connectivité. Une sauvegarde et un moyen
d'annuler rendent l'expérimentation sûre.

```
# Espace libre d'abord : l'export est écrit sur la mémoire flash du routeur.
# Donne aussi le modèle et la version de RouterOS (journal des résultats)
/system resource print

# Enregistrer la configuration actuelle (before-nad.rsc) et vérifier sa présence
/export file=before-nad
/file print where name~"before-nad"

# FastTrack : aucune ligne sous l'en-tête « Flags » = aucune règle FastTrack
/ip firewall filter print where action=fasttrack-connection
/ip firewall connection print count-only where fasttrack

# Organisation du LAN : nom du bridge (souvent pas juste « bridge »), ports
# membres (drapeau H = hardware offload) et sous-réseaux par interface
/interface bridge print
/interface bridge port print
/ip address print
```

Depuis l'hôte, télécharge la sauvegarde et garde-la hors du dépôt. Si le
stockage du routeur est presque plein, supprime ensuite le fichier du routeur :

```bash
mkdir -p ~/nad-private
scp <admin-user>@<router-ip>:before-nad.rsc ~/nad-private/
```

```
/file remove before-nad.rsc
```

- **Mode sûr :** `Ctrl+X` l'active et l'invite affiche `<SAFE>`. Quitter le
  mode sûr avec `Ctrl+X` **garde** les changements ; `Ctrl+D`, ou une session
  coupée, **annule** tout ce qui a été fait depuis son activation. Pendant une
  période de test, ne quitte jamais avec `Ctrl+X`.
- **Ne commite pas le fichier d'export.** Il décrit le réseau et contient le
  numéro de série du routeur.
- **Note**, à partir de `/ip address print` : le sous-réseau LAN, l'adresse WAN
  (elle peut être dynamique) et l'éventuel sous-réseau VPN. L'ingesteur en a
  besoin (étape 5).
- **À propos de FastTrack :** sur les anciennes versions de RouterOS, il
  masquait du trafic à l'export de flux ; la documentation indique que cette
  restriction a été levée en 6.33 **(à vérifier)**. Ne le désactive pas sur un
  routeur de production : note-le, et laisse le recoupement (étape 4) montrer
  s'il a un effet.

## 6. Étape 1 — Activer la télémétrie sur le routeur

Remplace `<nad-ip>` par l'adresse de l'hôte NAD, `<router-lan-ip>` par
l'adresse LAN du routeur et `<lan-bridge>` par le nom du bridge relevé à
l'étape 0. Vérifie chaque commande avec `?` ou `print`, car la syntaxe peut
varier selon la version de RouterOS **(à vérifier)**.

### 1.1 Flux (qui communique avec qui)

**Pourquoi :** les enregistrements de flux montrent les conversations (source,
destination, ports, octets). C'est le signal le plus riche pour détecter les
scans, les inondations et les destinations inhabituelles.

```
# Activer l'export de flux sur le bridge LAN.
# active-flow-timeout : rapporter les longs transferts chaque minute plutôt qu'à la fin
# inactive-flow-timeout : fermer rapidement les flux inactifs
/ip traffic-flow set enabled=yes interfaces=<lan-bridge> active-flow-timeout=1m inactive-flow-timeout=15s

# Envoyer les flux à l'hôte NAD, NetFlow v9, UDP 2055.
# src-address est obligatoire : sans elle, le routeur peut envoyer depuis 0.0.0.0,
# et Linux jette sans bruit les paquets ayant cette adresse source.
/ip traffic-flow target add dst-address=<nad-ip> src-address=<router-lan-ip> port=2055 version=9

# Vérifier
/ip traffic-flow print
/ip traffic-flow target print

# Contrôle de charge : comparer cpu-load et free-memory avec les valeurs de
# l'étape 0, juste après l'activation puis 10 minutes plus tard
/system resource print
```

**Observé sur RouterOS 7.12 (hAP ac²) :** exporter depuis le seul bridge LAN
enregistre **les deux sens**, chaque flux une seule fois. Les flux montants
portent l'adresse du poste interne. Les flux descendants sont enregistrés avant
la traduction NAT inverse : leur destination est l'adresse WAN du routeur, pas
celle du poste. Le cache de flux (32k entrées par défaut) a consommé environ
6 Mo de RAM.

Le trafic entre deux ports LAN en hardware offload est commuté par la puce du
switch sans atteindre le CPU : il n'apparaît pas dans les flux. C'est une limite
connue, acceptable car l'essentiel du trafic va vers Internet.

### 1.2 SNMP (compteurs et santé de l'équipement)

**Pourquoi :** SNMP fournit le débit des interfaces, le CPU et la mémoire. C'est
aussi la référence qui sert à vérifier les données de flux. **Pas encore
testé.**

```
# Activer le service SNMP
/snmp set enabled=yes

# Utilisateur SNMPv3 « nad » : lecture seule, seul l'hôte NAD peut interroger,
# authentifié et chiffré
# (remplace les deux mots de passe ; ne commite jamais les vrais)
/snmp community add name=nad addresses=<nad-ip>/32 read-access=yes write-access=no \
    security=private authentication-protocol=SHA1 authentication-password=CHANGE_ME \
    encryption-protocol=AES encryption-password=CHANGE_ME_TOO

# Vérifier
/snmp print
/snmp community print
```

**Pourquoi SNMPv3 et une restriction en /32 :** les anciennes versions de SNMP
envoient un mot de passe partagé en clair. Restreindre à une seule adresse et à
la lecture seule limite les dégâts si les identifiants fuient.

### 1.3 Syslog (événements)

**Pourquoi :** les événements de l'équipement (changements d'interface, échecs
de connexion) apportent un contexte utile. NAD les utilise sous forme de
comptages par fenêtre de temps. **Pas encore testé.**

```
# Définir une destination : l'hôte NAD, port UDP 514
/system logging action add name=nad target=remote remote=<nad-ip> remote-port=514 src-address=<router-lan-ip>

# Envoyer les messages de niveau info et plus graves vers cette destination
/system logging add topics=info,warning,error,critical action=nad

# Vérifier
/system logging action print
/system logging print
```

Indiquer `src-address` ici aussi évite le même problème d'adresse source que
pour les flux **(à vérifier)**.

### 1.4 Heure

**Pourquoi :** NAD agrège tout en fenêtres d'une minute. Si l'horloge du routeur
est décalée, ses données tombent dans la mauvaise fenêtre.

```
# Vérifier que l'horloge du routeur est synchronisée
/system ntp client print
/system clock print
```

### 1.5 Annuler après une session de test

**Pourquoi :** pendant la période de test, rien ne doit rester sur le routeur de
production.

```
# Supprimer la cible d'export et remettre les valeurs par défaut
/ip traffic-flow target remove [find dst-address=<nad-ip>]
/ip traffic-flow set enabled=no interfaces=all active-flow-timeout=30m inactive-flow-timeout=15s

# Preuve : un export ne montre que ce qui diffère des valeurs par défaut,
# donc aucune ligne /ip traffic-flow ne doit apparaître sous l'en-tête
/ip traffic-flow export
```

Quitte ensuite le mode sûr avec `Ctrl+D` (annuler), jamais `Ctrl+X` (garder).

---

## 7. Étape 2 — Préparer les récepteurs sur l'hôte

**Pourquoi :** le routeur pousse les flux et le syslog sans attendre. Si rien
n'écoute, ces messages sont perdus.

Si un pare-feu est actif sur l'hôte (vérifie avec `sudo ufw status verbose`),
autorise le routeur à joindre les récepteurs. La règle ne correspond que si le
routeur envoie vraiment depuis `<router-lan-ip>` (voir `src-address` à
l'étape 1.1) :

```bash
# Autoriser les flux et le syslog venant du routeur uniquement
sudo ufw allow from <router-lan-ip> to any port 2055 proto udp comment 'NAD netflow'
sudo ufw allow from <router-lan-ip> to any port 514 proto udp comment 'NAD syslog'
sudo ufw status numbered | grep NAD
```

### Récepteur de flux

```bash
mkdir -p ~/nad-samples

# Recevoir NetFlow sur UDP 2055, le décoder, afficher des lignes JSON et en garder une copie
# (vérifie les options exactes avec -h : elles changent selon les versions)
./goflow2-2.2.7-linux-amd64 -listen netflow://:2055 -format json | tee ~/nad-samples/flows.jsonl
```

```bash
# Dans un autre terminal : les paquets bruts arrivent-ils, et depuis quelle source ?
sudo tcpdump -ni <host-interface> udp port 2055
```

Utilise l'interface réelle (par exemple `eth0`) plutôt que `-i any` : la
pseudo-interface `any` affiche un avertissement sans conséquence (« promiscuous
mode not supported »).

Diagnostique en trois niveaux, dans cet ordre :

1. **`tcpdump` ne montre rien :** configuration du routeur ou chemin réseau.
2. **`tcpdump` montre des paquets, mais `nc` ne reçoit rien :** quelque chose
   les jette entre la carte réseau et l'application. `tcpdump` capture **avant**
   le pare-feu de l'hôte : voir des paquets ne prouve donc pas qu'ils sont
   livrés. Vérifie l'adresse source dans `tcpdump` (`0.0.0.0` est jetée par
   Linux) et la règle de pare-feu.
3. **`nc` reçoit, mais `goflow2` n'écrit rien :** décodage. Quelques
   avertissements `template error` au démarrage sont normaux : les données
   NetFlow v9 ne peuvent pas être décodées avant que le routeur envoie son
   template.

```bash
# Test du niveau 2 : arrête d'abord goflow2, puis écoute avec un outil minimal
timeout 15 nc -ul 2055 | xxd | head -5
```

goflow2 publie aussi des métriques sur le port 8080. Si un autre service occupe
déjà ce port, lance goflow2 avec `-addr :9081`.

### Vérification SNMP

```bash
# Parcourir la table « octets reçus par interface » (ifInOctets)
# -v3 -l authPriv : SNMPv3 avec authentification et chiffrement
snmpwalk -v3 -l authPriv -u nad -a SHA -A 'CHANGE_ME' -x AES -X 'CHANGE_ME_TOO' \
    <router-lan-ip> 1.3.6.1.2.1.2.2.1.10
```

C'est l'hôte qui initie SNMP : aucune règle de pare-feu entrante n'est
nécessaire pour lui.

### Récepteur syslog

```bash
sudo mkdir -p /var/log/nad
sudoedit /etc/rsyslog.d/10-nad.conf
```

```
# Écouter le syslog sur UDP 514
module(load="imudp")
input(type="imudp" port="514")

# Les messages du routeur vont dans leur propre fichier ; « & stop » les tient
# à l'écart du journal système général
if $fromhost-ip == '<router-lan-ip>' then /var/log/nad/router.log
& stop
```

```bash
sudo systemctl restart rsyslog
tail -f /var/log/nad/router.log
```

Si le fichier reste vide, provoque un événement sur le routeur (par exemple une
connexion) et vérifie que l'utilisateur syslog peut écrire dans le fichier de
journal.

---

## 8. Étape 3 — Capturer des échantillons réels

**Pourquoi :** ce sont les vrais enregistrements qui serviront à concevoir
l'ingesteur.

- Laisse tourner les récepteurs 20 à 30 minutes pendant une utilisation normale
  du réseau.
- Garde les captures brutes **hors de Git**. Seul un petit extrait anonymisé a
  sa place dans `data/samples/`.
- Inspecte un enregistrement pour connaître les noms des champs, puis résume la
  capture :

```bash
F=~/nad-samples/flows.jsonl

# Afficher joliment le premier enregistrement de flux
head -n 1 $F | jq .

# Répartition par sens (adapte le préfixe LAN et l'adresse WAN)
jq -r 'if (.src_addr|startswith("<lan-prefix>")) then "montant"
       elif .dst_addr=="<wan-ip>" then "descendant"
       else "autre" end' $F | sort | uniq -c

# Nombre de postes internes distincts vus
jq -r 'select(.src_addr|startswith("<lan-prefix>")) | .src_addr' $F | sort -u | wc -l

# Volume, pour estimer le stockage par jour
ls -lh $F

# La capture décrit l'activité de personnes réelles
chmod 600 $F
```

**Volume :** chaque ligne JSON brute fait environ 1,3 Ko, en majorité des champs
inutiles (BGP, MPLS, IPv6). Sur le réseau de référence, cela représente de
l'ordre de 1 Go de JSON brut par jour, d'où la sélection des champs et la
rotation des fichiers dans l'ingesteur.

Une session de test valide la **chaîne de traitement**, pas la référence. La
référence sera construite plus tard à partir de la collecte continue.

## 9. Étape 4 — Recouper les flux avec SNMP

**Pourquoi :** l'export de flux peut sous-compter (transfert accéléré, hardware
offload) ou sur-compter (le même trafic exporté depuis plusieurs interfaces), et
UDP peut perdre des paquets. Les compteurs SNMP viennent de l'interface
elle-même et servent de référence. **Pas encore testé.**

```bash
# 1) Lister les noms et index des interfaces (ifDescr)
snmpwalk -v3 -l authPriv -u nad -a SHA -A 'CHANGE_ME' -x AES -X 'CHANGE_ME_TOO' \
    <router-lan-ip> 1.3.6.1.2.1.2.2.1.2

# 2) Lire deux fois le compteur 64 bits d'octets reçus, à 5 minutes d'intervalle
#    (les compteurs 64 bits évitent le retour à zéro sur les liens rapides)
IFX=<ifIndex>
OID=1.3.6.1.2.1.31.1.1.1.6.$IFX
snmpget -v3 -l authPriv -u nad -a SHA -A 'CHANGE_ME' -x AES -X 'CHANGE_ME_TOO' <router-lan-ip> $OID
sleep 300
snmpget -v3 -l authPriv -u nad -a SHA -A 'CHANGE_ME' -x AES -X 'CHANGE_ME_TOO' <router-lan-ip> $OID
```

Compare **par sens** : SNMP compte par interface, alors que les flux mélangent
les deux sens. Les octets reçus sur le bridge LAN correspondent aux flux
montants ; les octets reçus sur l'interface WAN correspondent aux flux
descendants. Utilise `in_if` pour les séparer :

```bash
# 3) Sur les mêmes 5 minutes (pars d'un fichier neuf), additionner les octets
#    des flux par interface d'entrée
jq -s --argjson lan <lan-ifindex> 'map(select(.in_if == $lan) | .bytes) | add' ~/nad-samples/flows-crosscheck.jsonl
jq -s --argjson wan <wan-ifindex> 'map(select(.in_if == $wan) | .bytes) | add' ~/nad-samples/flows-crosscheck.jsonl
```

Attends-toi à un accord approximatif, pas à une égalité. Le compteur du bridge
est un peu plus élevé que les flux montants, car il compte aussi le trafic
destiné au routeur lui-même (requêtes DNS, administration). Un grand écart
renvoie à l'une des causes de la section 12.

## 10. Étape 5 — Concevoir l'ingesteur

### Principe

En termes simples, l'ingesteur est un centre de tri. Il lit les fichiers bruts,
dépose chaque événement dans la boîte de sa minute, compte ce que contient
chaque boîte, et écrit une ligne par boîte.

Quatre étapes, indépendantes les unes des autres pour qu'on puisse remplacer une
source sans toucher au reste :

1. **Lire :** un adaptateur par source (flux, sortie SNMP, sondes, journaux).
2. **Normaliser :** convertir chaque événement dans une forme interne commune
   (horodatage UTC, sens, octets, adresses).
3. **Agréger :** affecter les événements à des fenêtres d'une minute et calculer
   les caractéristiques.
4. **Écrire :** enregistrer une ligne par fenêtre dans la base de données.

### Décisions de conception

| Décision | Pourquoi |
|---|---|
| Tous les horodatages en UTC, fenêtres alignées sur la minute | Évite les confusions de fuseau horaire et d'heure d'été ; les fenêtres des différentes sources s'alignent |
| Fermer une fenêtre seulement après un délai | Les flux arrivent en retard (délais du routeur) ; fermer immédiatement sous-compterait |
| Écritures idempotentes (upsert sur la clé de fenêtre) | Relancer l'ingesteur ne doit jamais dupliquer de lignes |
| Mémoriser la position de lecture par fichier | Reprendre après un arrêt sans relire ni sauter de données |
| Sens déterminé par `in_if`/`out_if`, pas par les adresses | L'adresse WAN est dynamique (DHCP) ; les index d'interface sont stables |
| Caractéristiques par hôte calculées sur les flux montants uniquement | Les flux descendants portent l'adresse WAN (enregistrés avant le NAT inverse) |
| Classes d'adresses en configuration : interne (LAN, VPN), amont (LAN Starlink), Internet | Le trafic VPN est interne à l'entreprise ; le LAN Starlink n'est ni interne ni Internet |
| Ne garder que les champs utiles dès la lecture | Les enregistrements bruts font ~1,3 Ko, surtout des champs inutiles |
| Affecter un flux à la fenêtre de son heure de fin | Simple et déterministe ; avec un délai actif d'une minute, un flux ne s'étend jamais sur plus de deux fenêtres |
| Fichiers bruts compressés et archivés par rotation après ingestion | Les flux bruts sont la partie volumineuse ; les fenêtres sont minuscules |

Champs utilisés dans goflow2 (v2.2.7) : `time_flow_start_ns`,
`time_flow_end_ns`, `src_addr`, `dst_addr`, `src_port`, `dst_port`, `proto`,
`bytes`, `packets`, `in_if`, `out_if`, `tcp_flags`, `sampler_address`. Les
horodatages sont en nanosecondes.

Exemple de configuration (valeurs du réseau de référence) :

```yaml
internal_subnets:
  - 192.168.30.0/24   # LAN utilisateurs
  - 10.0.0.0/24       # VPN WireGuard (trafic interne à l'entreprise)
upstream_subnets:
  - 192.168.1.0/24    # LAN du routeur Starlink (ni interne, ni Internet)
lan_ifindex: <lan-ifindex>
wan_ifindex: <wan-ifindex>
```

### Schéma provisoire (à ajuster après avoir vu de vrais enregistrements)

```sql
-- Une ligne par fenêtre d'une minute, issue des flux, des compteurs et des journaux
CREATE TABLE IF NOT EXISTS windows (
  window_start      INTEGER PRIMARY KEY,  -- secondes unix, UTC, alignées sur la minute
  flows             INTEGER,
  bytes_up          INTEGER,
  bytes_down        INTEGER,
  packets_up        INTEGER,
  packets_down      INTEGER,
  distinct_hosts    INTEGER,              -- à partir des flux montants
  distinct_dests    INTEGER,
  distinct_ports    INTEGER,
  top_talker_share  REAL,                 -- à partir des flux montants
  cpu_pct           REAL,
  mem_pct           REAL,
  log_events        INTEGER,
  ingested_at       INTEGER
);

-- Une ligne par fenêtre et par cible de sonde (plusieurs cibles par fenêtre)
CREATE TABLE IF NOT EXISTS probe_windows (
  window_start  INTEGER,
  target        TEXT,                     -- ex. routeur Starlink (premier saut), cibles Internet
  rtt_mean_ms   REAL,
  rtt_p95_ms    REAL,
  loss_pct      REAL,
  PRIMARY KEY (window_start, target)
);
```

Les sondes ont leur propre table, car chaque fenêtre a une ligne par cible.
Sonder à la fois le routeur Starlink et des cibles Internet permet de
distinguer un problème de liaison locale d'un problème de liaison satellite.

## 11. Journal des résultats

Remplis-le après chaque session. Ces valeurs alimentent les décisions de
`docs/tools/tooling.fr.md`.

| Élément | Session de test (mode sûr) | Collecte continue |
|---|---|---|
| Date | 2026-10-15 | |
| Modèle du routeur | hAP ac² (RBD52G), 4 × ARM 672 MHz, 128 Mo RAM | |
| Version de RouterOS | 7.12 (stable) | |
| Stockage libre du routeur | ~0,4 Mo sur 15,2 Mo | |
| FastTrack activé | Non (aucune règle configurée) | |
| Charge CPU avant télémétrie | 10 % (22 min après un redémarrage) | |
| Charge CPU avec télémétrie | 31 % (1 h plus tard, pas encore comparable) | |
| Mémoire libre avant / avec télémétrie | 46,9 Mo / 41,1 Mo | |
| Interface choisie pour l'export de flux | Bridge LAN seul : les deux sens, sans double comptage | |
| Total des flux vs compteurs SNMP (ratio) | Pas encore mesuré | |
| Noms des champs de flux relevés | Voir l'étape 5 | |
| Adresses dans les flux | Montants : postes internes. Descendants : adresse WAN du routeur | |
| Format syslog observé | Pas encore testé | |
| Notes | Double NAT derrière le routeur Starlink ; VPN WireGuard sur le routeur ; ports LAN en hardware offload | |

## 12. Dépannage

| Symptôme | Cause probable | À vérifier |
|---|---|---|
| Aucun paquet dans `tcpdump` sur le port 2055 | Routeur qui n'exporte pas, mauvaise adresse cible | `/ip traffic-flow target print`, même réseau local |
| `tcpdump` montre des paquets, `nc` et `goflow2` ne reçoivent rien | Adresse source `0.0.0.0` (jetée par Linux, paquet « martien ») ou pare-feu de l'hôte | Source dans `tcpdump` ; `src-address` de la cible ; `sudo ufw status` |
| Règle de pare-feu ajoutée mais toujours rien | La règle vise l'adresse du routeur, les paquets viennent de `0.0.0.0` | Régler d'abord `src-address` sur la cible |
| Avertissements `template error` dans goflow2 | Template pas encore reçu | Normal les premières secondes ; relancer goflow2 si ça dure plus de 2 minutes |
| Avertissement « promiscuous mode not supported » | `tcpdump -i any` | Sans conséquence ; utiliser l'interface réelle |
| Métriques goflow2 injoignables sur le port 8080 | Port occupé par un autre service | `sudo ss -tlnp \| grep 8080` ; lancer goflow2 avec `-addr :9081` |
| SNMP ne répond pas (timeout) | Mauvais identifiants, restriction d'adresse, service désactivé | `/snmp print`, la restriction de la community correspond à l'adresse de l'hôte |
| Fichier syslog vide | Port 514 bloqué, mauvaise adresse dans le filtre, permissions du fichier | `tcpdump` sur UDP 514, valeur de `fromhost-ip`, propriétaire de `/var/log/nad` |
| Octets des flux bien inférieurs à SNMP | Transfert accéléré ou hardware offload, perte de paquets | Règle FastTrack, drapeaux des ports du bridge, pertes UDP sur l'hôte |
| Octets des flux supérieurs à SNMP | Même trafic exporté depuis plusieurs interfaces | Choix de l'interface d'export (`<lan-bridge>` ou `all`) |
| Fenêtres décalées entre les sources | Horloge du routeur non synchronisée | `/system ntp client print` |
| Le CPU du routeur augmente nettement | Charge de l'export de flux | `/tool profile duration=10s` ; ajuster les délais ou activer l'échantillonnage |
| Configuration disparue après fermeture de SSH | Session fermée alors que le mode sûr était actif | Normal pendant les tests ; `Ctrl+X` garde les changements quand tu les veux permanents |

## 13. Prochaines étapes

1. Tester SNMP (étape 1.2), puis le recoupement (étape 4).
2. Configuration de Telegraf pour SNMP et les sondes (guide 02).
3. Squelette de l'ingesteur dans `src/nad/collect`, avec les adaptateurs et des
   tests sur les échantillons capturés.
4. Rôles Ansible qui installent et configurent les récepteurs sur l'hôte NAD,
   pour rendre reproductibles les étapes manuelles de ce guide.

## Glossaire

- **Flux (flow) :** le résumé d'une conversation entre deux machines (adresses,
  ports, octets, paquets).
- **SNMP :** un protocole standard pour demander à un équipement ses compteurs
  et son état.
- **Syslog :** un protocole standard qui permet aux équipements d'envoyer des
  messages d'événement.
- **Fenêtre :** une tranche de temps fixe (une minute) dans laquelle tous les
  événements sont regroupés.
- **Idempotent :** exécuter une opération deux fois donne le même résultat que
  l'exécuter une seule fois.
- **FastTrack :** une fonction MikroTik qui accélère le transfert en
  contournant une partie du traitement des paquets.
- **NAT :** le routeur remplace les adresses internes par la sienne avant
  d'envoyer le trafic vers l'extérieur. Derrière le routeur Starlink, il y a
  deux couches de NAT (double NAT).
- **Bridge :** un commutateur virtuel qui regroupe plusieurs ports en une seule
  interface LAN.
- **Hardware offload :** trafic commuté par la puce du switch sans passer par le
  CPU ; il n'apparaît pas dans l'export de flux.
- **Template (NetFlow v9) :** un message qui décrit l'organisation des champs ;
  les données ne peuvent pas être décodées avant son arrivée.
- **Mode sûr (safe mode) :** un mode de session RouterOS où les changements
  sont annulés sauf si tu choisis explicitement de les garder.
- **Paquet martien :** un paquet avec une adresse source impossible (comme
  `0.0.0.0`), jeté par le noyau Linux.
