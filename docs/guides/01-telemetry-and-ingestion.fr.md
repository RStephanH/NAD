# Guide 01 — Mise en place de la télémétrie et conception de l'ingestion

**Français** | [English](01-telemetry-and-ingestion.md)

**Public :** toute personne qui reproduit le prototype, ou un assistant IA qui
reprend le travail.
**Objectif :** faire circuler une télémétrie réelle du routeur passerelle vers
l'hôte NAD, puis concevoir l'ingesteur à partir d'échantillons réels.
**Statut :** en cours.

- Export de flux : testé le 2026-10-15 (mode sûr, annulé ensuite).
- SNMP v3 : configuré et testé le 2026-10-16, conservé sur le routeur (lecture
  seule, chiffré, restreint à l'hôte NAD).
- Syslog, mesure de charge du routeur et recoupement : pas encore faits.

Remplis le journal des résultats (section 11) au fur et à mesure. Les éléments
marqués **(à vérifier)** dépendent de la version de RouterOS ou de l'outil et
doivent être confrontés à la documentation à jour.

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
- Sur l'hôte : `tcpdump`, `nc` (netcat), `jq`, les outils clients Net-SNMP
  (`snmpget`, `snmpwalk` : paquet `net-snmp` sur Arch, `snmp` sur
  Debian/Ubuntu), `rsyslog` et `goflow2` (un binaire publié, comme
  `goflow2-2.2.7-linux-amd64`, fonctionne sans installer Go).
- L'horloge du routeur synchronisée (voir l'étape 1.4).

## 4. Vue d'ensemble du déroulé

```mermaid
flowchart LR
  S0["Étape 0<br/>Sauvegarder la config du routeur"] --> S1["Étape 1<br/>Activer la télémétrie sur le routeur"]
  S1 --> S2["Étape 2<br/>Préparer les récepteurs sur l'hôte"]
  S2 --> S3["Étape 3<br/>Capturer des échantillons réels"]
  S3 --> S4["Étape 4<br/>Mesure de charge et recoupement"]
  S4 --> S5["Étape 5<br/>Concevoir l'ingesteur"]
```

En pratique, le côté hôte passe d'abord pour chaque canal : les identifiants
SNMP (étape 2) avant l'utilisateur SNMP du routeur (étape 1.2), et le récepteur
de flux (étape 2) avant l'export de flux (étape 1.1).

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

# Configuration SNMP existante et chaîne input du pare-feu
/snmp print
/snmp community print
/ip firewall filter print where chain=input
```

Depuis l'hôte, télécharge la sauvegarde et garde-la hors du dépôt. Le stockage
du routeur est presque plein sur ce modèle : supprime ensuite le fichier.

```bash
mkdir -p ~/nad-private
scp <admin-user>@<router-ip>:before-nad.rsc ~/nad-private/
```

```
/file remove before-nad.rsc
```

- **Mode sûr :** `Ctrl+X` l'active et l'invite affiche `<SAFE>`. Quitter le
  mode sûr avec `Ctrl+X` **garde** les changements ; `Ctrl+D`, ou une session
  coupée, **annule** tout ce qui a été fait depuis son activation.
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

Le coût réel de l'export de flux sur le CPU du routeur se mesure à l'étape 4.

### 1.2 SNMP (compteurs et santé de l'équipement)

**Pourquoi :** SNMP fournit le débit des interfaces, le CPU et la mémoire. C'est
aussi la référence qui sert à vérifier les données de flux.

Prépare d'abord les identifiants sur l'hôte (étape 2, « Client SNMP »), puis,
sur le routeur, en mode sûr :

**a) Regarder l'existant.** Note si une communauté `public` existe, quelles
adresses elle accepte, et si la chaîne `input` du pare-feu contient des règles
statiques.

```
/snmp print
/snmp community print
/ip firewall filter print where chain=input
```

**b) Désactiver la communauté par défaut, avant d'activer le service.**
`public` est le nom par défaut que tout le monde connaît, ouvert à toutes les
adresses, sans chiffrement. La désactiver d'abord garantit que le routeur ne
répond jamais à tout le monde.

```
# Les entrées par défaut ne peuvent en général pas être supprimées, donc on la désactive (à vérifier)
/snmp community disable [find name=public]

# Vérifier : la ligne « public » doit porter un X (désactivée)
/snmp community print
```

Si la commande est refusée, restreins-la plutôt :
`/snmp community set [find name=public] addresses=<nad-ip>/32`.

**c) Créer l'utilisateur SNMPv3** (une seule ligne, avec les valeurs `AUTH` et
`PRIV` générées sur l'hôte) :

```
/snmp community add name=nad addresses=<nad-ip>/32 read-access=yes write-access=no security=private authentication-protocol=SHA1 authentication-password=<AUTH> encryption-protocol=AES encryption-password=<PRIV>
```

Dans RouterOS, `/snmp community` contient des profils d'accès. Avec
`security=private`, l'entrée est un **utilisateur SNMPv3** (authentifié et
chiffré), et `name` devient le nom d'utilisateur : il doit correspondre à
`defSecurityName` sur l'hôte. Les versions récentes de RouterOS proposent des
algorithmes plus forts que SHA1 **(à vérifier)** ; si tu en choisis un, change
aussi `defAuthType` sur l'hôte.

**d) Activer le service et vérifier :**

```
/snmp set enabled=yes
/snmp print
/snmp community print
```

`nad` doit être actif et restreint à `<nad-ip>/32` ; `public` doit être
désactivée. `trap-community: public` dans `/snmp print` est sans conséquence :
sans `trap-target`, aucune alerte n'est envoyée.

**e) Garder ou annuler.** Une fois les tests de l'étape 2 réussis, la
configuration SNMP peut rester : lecture seule, SNMPv3, chiffrée, une seule
adresse autorisée. Quitte le mode sûr avec `Ctrl+X` pour la garder. Sinon, une
session coupée en mode sûr annulerait SNMP au milieu d'une longue mesure
(étape 4).

**Observé (hAP ac², RouterOS 7.12) :** le service SNMP était désactivé, la
communauté `public` existait (par défaut, toutes les adresses `::/0`, sécurité
`none`), et la chaîne `input` ne contenait que des règles dynamiques créées par
le Hotspot. Sans règle statique, rien ne bloque ni ne protège l'accès au
routeur lui-même.

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

### 1.5 Annuler

**Flux, après chaque session de test :**

```
# Supprimer la cible d'export et remettre les valeurs par défaut
/ip traffic-flow target remove [find dst-address=<nad-ip>]
/ip traffic-flow set enabled=no interfaces=all active-flow-timeout=30m inactive-flow-timeout=15s

# Preuve : un export ne montre que ce qui diffère des valeurs par défaut,
# donc aucune ligne /ip traffic-flow ne doit apparaître sous l'en-tête
/ip traffic-flow export
```

Quitte ensuite le mode sûr avec `Ctrl+D` (annuler), jamais `Ctrl+X` (garder).

**SNMP, seulement s'il faut le retirer complètement :**

```
/snmp set enabled=no
/snmp community remove [find name=nad]
```

Même dans ce cas, laisse `public` désactivée : c'est plus sûr que l'état
d'origine.

---

## 7. Étape 2 — Préparer les récepteurs sur l'hôte

**Pourquoi :** le routeur pousse les flux et le syslog sans attendre. Si rien
n'écoute, ces messages sont perdus. Pour SNMP, c'est l'hôte qui interroge : il
lui faut les identifiants avant que l'utilisateur existe sur le routeur.

Si un pare-feu est actif sur l'hôte (vérifie avec `sudo ufw status verbose`),
autorise le routeur à joindre les récepteurs de flux et de syslog. La règle ne
correspond que si le routeur envoie vraiment depuis `<router-lan-ip>` (voir
`src-address` à l'étape 1.1). C'est l'hôte qui initie SNMP : aucune règle
entrante n'est nécessaire pour lui.

```bash
# Autoriser les flux et le syslog venant du routeur uniquement
sudo ufw allow from <router-lan-ip> to any port 2055 proto udp comment 'NAD netflow'
sudo ufw allow from <router-lan-ip> to any port 514 proto udp comment 'NAD syslog'
sudo ufw status numbered | grep NAD
```

### Client SNMP

```bash
# Outils clients uniquement. N'active PAS le service snmpd : c'est l'agent,
# dont NAD n'a pas besoin.
sudo pacman -S --needed net-snmp      # Debian/Ubuntu : sudo apt install snmp
snmpget --version
```

Génère deux phrases de passe (Net-SNMP exige au moins 8 caractères) et
range-les dans un fichier lisible par toi seul :

```bash
AUTH=$(openssl rand -base64 24 | tr -d '/+=' | head -c 20)
PRIV=$(openssl rand -base64 24 | tr -d '/+=' | head -c 20)

mkdir -p ~/.snmp
(umask 077; cat > ~/.snmp/snmp.conf <<EOF
defVersion 3
defSecurityLevel authPriv
defSecurityName nad
defAuthType SHA
defAuthPassphrase $AUTH
defPrivType AES
defPrivPassphrase $PRIV
EOF
)

# Les afficher une fois : elles servent à la commande du routeur (étape 1.2 c)
echo "AUTH=$AUTH"; echo "PRIV=$PRIV"
```

Efface l'écran après avoir copié les deux valeurs. Avec ce fichier, les
commandes se réduisent à `snmpget <ip> <oid>`, et aucun mot de passe
n'apparaît dans l'historique du shell ni dans `ps`.

### Tests SNMP, du plus simple au plus utile

```bash
R=<router-lan-ip>

# 1) Le routeur répond-il ? Description du système et durée de fonctionnement
snmpget $R 1.3.6.1.2.1.1.1.0 1.3.6.1.2.1.1.3.0

# 2) Noms des interfaces avec leur index (l'index est le dernier nombre de l'OID)
snmpwalk $R 1.3.6.1.2.1.31.1.1.1.1

# 3) Charge CPU, une valeur par cœur
snmpwalk -Oqv $R 1.3.6.1.2.1.25.3.3.1.2

# 4) Stockage : description, unité d'allocation, taille, utilisé
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.3
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.4
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.5
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.6

# 5) Compteurs d'octets 64 bits (entrée, sortie) d'une interface
snmpget $R 1.3.6.1.2.1.31.1.1.1.6.<idx> 1.3.6.1.2.1.31.1.1.1.10.<idx>
```

| Test | Résultat attendu |
|---|---|
| 1 | Une description qui mentionne le modèle du routeur, puis la durée de fonctionnement |
| 2 | Une ligne par interface, comme `...1.8 = STRING: bridge1` |
| 3 | Un pourcentage par cœur (quatre sur le hAP ac²) |
| 4 | Trouve « main memory » dans les descriptions ; % utilisé = `used / size` sur la même ligne (index 65536 sur ce routeur) |
| 5 | Deux grands nombres qui **augmentent** quand tu relances la commande 10 secondes plus tard |

Notes :

- Note les **index** du bridge LAN et de l'interface WAN dans le journal des
  résultats : le recoupement en a besoin.
- Les compteurs se lisent **du point de vue du routeur** : sur le bridge LAN,
  « entrée » est ce que les machines du LAN envoient vers l'extérieur
  (montant), « sortie » est ce que le routeur leur livre (descendant).
- Les OID numériques fonctionnent sans MIB installée. Les noms comme `ifName`
  exigent des fichiers MIB, que certaines distributions ne fournissent pas.
- Les index d'interface peuvent changer quand des interfaces dynamiques
  apparaissent (sessions VPN). Pour la collecte continue, convertis les index en
  noms (`ifName`) à chaque lecture au lieu de te fier à un numéro fixe.

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

## 9. Étape 4 — Mesure de charge et recoupement

### 4.1 Charge du routeur (phases A/B/C)

**Pourquoi :** l'export de flux coûte du CPU et de la RAM sur un routeur
modeste qui sert toute l'entreprise. Un seul relevé ne suffit pas : la charge
varie avec l'usage. Trois phases de 10 minutes chacune, mesurées de la même
façon, séparent le coût de l'export de la variation naturelle :

| Phase | Export de flux | Rôle |
|---|---|---|
| A | Éteint | Charge de référence |
| B | Allumé | Charge avec l'export |
| C | Éteint à nouveau | Vérifie que la charge revient à la référence (contrôle) |

Évite les gros téléchargements pendant la mesure, et enchaîne les trois phases
à une heure comparable de la journée.

```bash
R=<router-lan-ip>
mkdir -p ~/nad-samples
# Colonnes : heure UTC, CPU moyen %, cœur le plus chargé %, mémoire utilisée %
while true; do
  cpu=$(snmpwalk -Oqv $R 1.3.6.1.2.1.25.3.3.1.2 | awk '{s+=$1; if ($1>m) m=$1} END {printf "%.1f %d", s/NR, m}')
  mem=$(snmpget -Oqv $R 1.3.6.1.2.1.25.2.3.1.6.65536 1.3.6.1.2.1.25.2.3.1.5.65536 | paste -sd' ' | awk '{printf "%.1f", 100*$1/$2}')
  echo "$(date -u +%FT%TZ) $cpu $mem"
  sleep 10
done | tee -a ~/nad-samples/cpu.log
```

Le cœur le plus chargé compte : un cœur peut saturer alors que la moyenne
paraît basse. Note l'heure de début de chaque phase, puis calcule les moyennes
par phase :

```bash
# CPU moyen, maximum du cœur le plus chargé et mémoire moyenne entre deux heures UTC
awk -v s="<debut>" -v e="<fin>" '$1>=s && $1<e {c+=$2; if ($3>m) m=$3; r+=$4; n++}
  END {printf "mesures=%d cpu_moy=%.1f coeur_max=%d mem_moy=%.1f\n", n, c/n, m, r/n}' ~/nad-samples/cpu.log
```

### 4.2 Recouper les flux avec SNMP

**Pourquoi :** l'export de flux peut sous-compter (transfert accéléré, hardware
offload) ou sur-compter (le même trafic exporté depuis plusieurs interfaces), et
UDP peut perdre des paquets. Les compteurs SNMP viennent de l'interface
elle-même et servent de référence. **Pas encore testé.**

Utilise uniquement le bridge LAN : ses deux compteurs couvrent les deux sens,
et contrairement au WAN, ils excluent l'encapsulation VPN et le trafic refusé
par le pare-feu.

| Compteur SNMP du bridge LAN | Correspond aux flux avec | Sens |
|---|---|---|
| Entrée (`ifHCInOctets`, OID ...31.1.1.1.6) | `in_if` = index du bridge | Montant |
| Sortie (`ifHCOutOctets`, OID ...31.1.1.1.10) | `out_if` = index du bridge | Descendant |

```bash
R=<router-lan-ip>
B=<lan-ifindex>

# 1) Lire les deux compteurs, lancer la capture de flux sur un fichier neuf,
#    attendre 5 minutes, relire les deux compteurs
snmpget -Oqv $R 1.3.6.1.2.1.31.1.1.1.6.$B 1.3.6.1.2.1.31.1.1.1.10.$B
sleep 300
snmpget -Oqv $R 1.3.6.1.2.1.31.1.1.1.6.$B 1.3.6.1.2.1.31.1.1.1.10.$B

# 2) Sur les mêmes 5 minutes, additionner les octets des flux par sens
F=~/nad-samples/flows-crosscheck.jsonl
jq -s --argjson b $B 'map(select(.in_if == $b) | .bytes) | add' $F    # montant
jq -s --argjson b $B 'map(select(.out_if == $b) | .bytes) | add' $F   # descendant
```

Compare chaque différence SNMP avec le total de flux correspondant.
Attends-toi à un accord approximatif, pas à une égalité : les compteurs du
bridge incluent aussi le trafic vers et depuis le routeur lui-même (DNS,
administration, l'interrogation SNMP) et l'encapsulation Ethernet, que les
flux ne comptent pas. La comparaison est aussi faussée si la capture rate les
premières ou dernières secondes. Un grand écart renvoie à l'une des causes de
la section 12.

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
| Sens déterminé par `in_if`/`out_if`, pas par les adresses | L'adresse WAN est dynamique (DHCP) ; les index d'interface sont plus stables |
| Index d'interface résolus à partir des noms au démarrage | Les index peuvent changer quand des interfaces dynamiques (sessions VPN) apparaissent |
| Caractéristiques par hôte calculées sur les flux montants uniquement | Les flux descendants portent l'adresse WAN (enregistrés avant le NAT inverse) |
| Classes d'adresses en configuration : interne (LAN, VPN), amont (LAN Starlink), Internet | Le trafic VPN est interne à l'entreprise ; le LAN Starlink n'est ni interne ni Internet |
| Ne garder que les champs utiles dès la lecture | Les enregistrements bruts font ~1,3 Ko, surtout des champs inutiles |
| Affecter un flux à la fenêtre de son heure de fin | Simple et déterministe ; avec un délai actif d'une minute, un flux ne s'étend jamais sur plus de deux fenêtres |
| Fichiers bruts compressés et archivés par rotation après ingestion | Les flux bruts sont la partie volumineuse ; les fenêtres sont minuscules |

Champs utilisés dans goflow2 (v2.2.7) : `time_flow_start_ns`,
`time_flow_end_ns`, `src_addr`, `dst_addr`, `src_port`, `dst_port`, `proto`,
`bytes`, `packets`, `in_if`, `out_if`, `tcp_flags`, `sampler_address`. Les
horodatages sont en nanosecondes.

Valeurs SNMP utilisées : CPU par cœur (`hrProcessorLoad`, OID
1.3.6.1.2.1.25.3.3.1.2), mémoire principale (`hrStorageSize` et
`hrStorageUsed`, index 65536 sur ce routeur), compteurs d'interface 64 bits
(`ifHCInOctets`, `ifHCOutOctets`), noms d'interface (`ifName`).

Exemple de configuration (valeurs du réseau de référence) :

```yaml
internal_subnets:
  - 192.168.30.0/24   # LAN utilisateurs
  - 10.0.0.0/24       # VPN WireGuard (trafic interne à l'entreprise)
upstream_subnets:
  - 192.168.1.0/24    # LAN du routeur Starlink (ni interne, ni Internet)
lan_interface: bridge1   # résolu en index au démarrage (8 le 2026-10-16)
wan_interface: ether1    # résolu en index au démarrage (1, à confirmer)
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
  cpu_pct           REAL,                 -- moyenne des cœurs
  cpu_core_max_pct  REAL,                 -- cœur le plus chargé
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

| Élément | Test des flux (2026-10-15) | Mise en place SNMP (2026-10-16) | Collecte continue |
|---|---|---|---|
| Modèle du routeur | hAP ac² (RBD52G-5HacD2HnD), 4 × ARM 672 MHz, 128 Mo RAM | Confirmé par SNMP | |
| Version de RouterOS | 7.12 (stable) | | |
| Durée de fonctionnement au moment de la mesure | 22 min, puis 1 h 21 | 23 h 45 | |
| Stockage libre du routeur | ~0,4 Mo sur 15,2 Mo | 0,32 Mo (97,9 % utilisé) | |
| FastTrack activé | Non (aucune règle configurée) | | |
| Charge CPU, export éteint | 10 % (juste après un redémarrage) | 8 % en moyenne (cœurs 6/4/10/12), un seul relevé | |
| Charge CPU, export allumé | 31 % (pas comparable) | | |
| Mémoire libre, export éteint | 46,9 Mo (22 min de fonctionnement) | 36,6 Mo (71,4 % utilisé) | |
| Mémoire libre, export allumé | 41,1 Mo | | |
| Index d'interface | Flux : descendants `in_if` 1, montants `in_if` 8 | `ether1`–`ether5` = 1–5, `wlan1`/`wlan2` = 6/7, `bridge1` = 8, puis `wg-vpn`, `l2tp-vpn`, `ovpn-vpn` | |
| Interface WAN | `ether1` (porte l'adresse WAN) | Index 1, à confirmer avec les compteurs | |
| Compteurs du bridge LAN | | Entrée 8,7 Go, sortie 55,5 Go sur 23 h 45 (≈ 0,8 Mbit/s montant, 5,2 Mbit/s descendant) | |
| Interface choisie pour l'export de flux | Bridge LAN seul : les deux sens, sans double comptage | | |
| Total des flux vs compteurs SNMP (ratio) | | Pas encore mesuré | |
| Adresses dans les flux | Montants : postes internes. Descendants : adresse WAN du routeur | | |
| Configuration SNMP | | Utilisateur v3 `nad`, authPriv (SHA1/AES), restreint à l'hôte NAD ; `public` désactivée ; conservée sur le routeur | |
| Format syslog observé | Pas encore testé | | |
| Notes | Double NAT derrière le routeur Starlink ; ports LAN en hardware offload | Hotspot actif (règles `input` dynamiques) ; aucune règle `input` statique ; trois interfaces VPN (WireGuard, L2TP, OpenVPN) | |

## 12. Dépannage

### Flux

| Symptôme | Cause probable | À vérifier |
|---|---|---|
| Aucun paquet dans `tcpdump` sur le port 2055 | Routeur qui n'exporte pas, mauvaise adresse cible | `/ip traffic-flow target print`, même réseau local |
| `tcpdump` montre des paquets, `nc` et `goflow2` ne reçoivent rien | Adresse source `0.0.0.0` (jetée par Linux, paquet « martien ») ou pare-feu de l'hôte | Source dans `tcpdump` ; `src-address` de la cible ; `sudo ufw status` |
| Règle de pare-feu ajoutée mais toujours rien | La règle vise l'adresse du routeur, les paquets viennent de `0.0.0.0` | Régler d'abord `src-address` sur la cible |
| Avertissements `template error` dans goflow2 | Template pas encore reçu | Normal les premières secondes ; relancer goflow2 si ça dure plus de 2 minutes |
| Avertissement « promiscuous mode not supported » | `tcpdump -i any` | Sans conséquence ; utiliser l'interface réelle |
| Métriques goflow2 injoignables sur le port 8080 | Port occupé par un autre service | `sudo ss -tlnp \| grep 8080` ; lancer goflow2 avec `-addr :9081` |
| Octets des flux bien inférieurs à SNMP | Transfert accéléré ou hardware offload, perte de paquets | Règle FastTrack, drapeaux des ports du bridge, pertes UDP sur l'hôte |
| Octets des flux supérieurs à SNMP | Même trafic exporté depuis plusieurs interfaces | Choix de l'interface d'export (`<lan-bridge>` ou `all`) |
| Le CPU du routeur augmente nettement | Charge de l'export de flux | Phases A/B/C (étape 4.1) ; ajuster les délais ou activer l'échantillonnage |

### SNMP

| Message ou symptôme | Cause probable | À vérifier |
|---|---|---|
| `Timeout: No Response from <ip>` | Service désactivé, mauvaise adresse dans la communauté, règle `input` du pare-feu | `/snmp print` ; `addresses=` égal à l'IP de l'hôte ; chaîne `input` |
| `Unknown user name` | Le nom de la communauté ne correspond pas à `defSecurityName` | `/snmp community print` ; `~/.snmp/snmp.conf` |
| `Authentication failure` | Phrase `AUTH` ou algorithme différents (SHA1 contre un autre) | Même valeur et même algorithme des deux côtés |
| `Decryption error` | Phrase `PRIV` ou chiffrement différents | Même valeur des deux côtés |
| `No Such Object available` | Cet OID n'est pas exposé par le routeur | `snmpwalk` de la branche parente |
| Un compteur ne change pas | Mauvais index d'interface | Refaire le test 2 |
| Avertissement « Unknown Object Identifier » | Noms utilisés sans fichiers MIB | Utiliser les OID numériques |
| `command not found` | Outils clients non installés | `sudo pacman -S --needed net-snmp` |

### Autres

| Symptôme | Cause probable | À vérifier |
|---|---|---|
| Fichier syslog vide | Port 514 bloqué, mauvaise adresse dans le filtre, permissions du fichier | `tcpdump` sur UDP 514, valeur de `fromhost-ip`, propriétaire de `/var/log/nad` |
| Fenêtres décalées entre les sources | Horloge du routeur non synchronisée | `/system ntp client print` |
| Configuration disparue après fermeture de SSH | Session fermée alors que le mode sûr était actif | Normal pour les tests ; `Ctrl+X` garde les changements quand tu les veux permanents |

## 13. Prochaines étapes

1. Confirmer l'index de l'interface WAN avec ses compteurs.
2. Charge du routeur, phases A/B/C (étape 4.1).
3. Recoupement des flux avec les compteurs du bridge LAN (étape 4.2).
4. Syslog (étape 1.3).
5. Configuration de Telegraf pour SNMP et les sondes, avec résolution des index
   en noms (guide 02).
6. Squelette de l'ingesteur dans `src/nad/collect`, avec les adaptateurs et des
   tests sur les échantillons capturés.
7. Rôles Ansible qui installent et configurent les récepteurs sur l'hôte NAD.

Durcissement ultérieur, hors de ce guide : une règle `input` statique qui
n'accepte l'UDP 161 que depuis l'hôte NAD. Pas maintenant : le routeur fait
tourner un Hotspot, et modifier le pare-feu d'une passerelle de production est
un autre chantier.

## Glossaire

- **Flux (flow) :** le résumé d'une conversation entre deux machines (adresses,
  ports, octets, paquets).
- **SNMP :** un protocole standard pour demander à un équipement ses compteurs
  et son état.
- **Communauté (SNMP v1/v2c) :** un mot de passe partagé, envoyé en clair avec
  chaque requête. `public` est la valeur par défaut universelle.
- **Utilisateur SNMPv3 :** un compte avec authentification et, au niveau
  `authPriv`, chiffrement. Dans RouterOS, c'est une entrée de
  `/snmp community` avec `security=private`.
- **OID :** l'adresse numérique d'une valeur en SNMP, comme
  `1.3.6.1.2.1.1.3.0` pour la durée de fonctionnement.
- **MIB :** un catalogue qui donne des noms aux OID. Les OID numériques
  fonctionnent sans lui.
- **ifIndex :** le numéro qu'utilisent SNMP et les flux pour identifier une
  interface.
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
- **Hotspot :** une fonction MikroTik d'accès invité avec page de connexion ;
  elle ajoute des règles de pare-feu dynamiques.
- **Template (NetFlow v9) :** un message qui décrit l'organisation des champs ;
  les données ne peuvent pas être décodées avant son arrivée.
- **Mode sûr (safe mode) :** un mode de session RouterOS où les changements
  sont annulés sauf si tu choisis explicitement de les garder.
- **Paquet martien :** un paquet avec une adresse source impossible (comme
  `0.0.0.0`), jeté par le noyau Linux.
