# WMSDR — Manuel d'utilisation

**Panadapteur, récepteur DSP et tête de commande pour les petits transceivers QRP qui
fournissent une sortie I/Q et un port CAT.**

Version : préliminaire pour Cavaillon, octobre 2026
Auteur : Gabi Mihaila, YO4WM

---

## Sommaire

1. [Ce qu'est le WMSDR](#1-ce-quest-le-wmsdr)
2. [Matériel](#2-matériel)
3. [Architecture logicielle](#3-architecture-logicielle)
4. [Raccorder un transceiver — l'exemple du uSDX triband](#4-raccorder-un-transceiver--lexemple-du-usdx-triband)
5. [Interface CAT](#5-interface-cat)
6. [L'écran, élément par élément](#6-lécran-élément-par-élément)
7. [Les rangées de boutons](#7-les-rangées-de-boutons)
8. [Le menu, onglet par onglet](#8-le-menu-onglet-par-onglet)
9. [Procédures pratiques](#9-procédures-pratiques)
10. [Annexes](#10-annexes)

---

## 1. Ce qu'est le WMSDR

![Face avant du WMSDR : écran principal sur 20 m avec le bandeau d'actualités RSS, le bloc de huit encodeurs en dessous, l'afficheur de la boîte d'accord (unité séparée) à droite](images/wmsdr-front-rss.jpg)

*Figure 1 — Face avant du WMSDR : écran principal sur 20 m avec le bandeau d'actualités RSS, le bloc de huit encodeurs en dessous, l'afficheur de la boîte d'accord (unité séparée) à droite.*

Le WMSDR est une unité autonome de traitement DSP et d'affichage, côté réception. Elle ne
génère pas de HF et n'émet pas. Elle prend la **paire I/Q en bande de base** issue d'un
détecteur à échantillonnage en quadrature (en pratique un mélangeur Tayloe) ainsi qu'une
**liaison série CAT** venant du même poste, et transforme ces deux entrées en tout ce qu'un
petit poste QRP ne peut habituellement pas s'offrir :

- un véritable **panadapteur et une cascade (waterfall)**, avec 8 / 32 / 48 / 96 kHz de
  spectre visible et un zoom-FFT jusqu'à une tranche étroite ;
- la **démodulation logicielle** — LSB, USB, CW, AM, FM et les passe-bandes des modes
  numériques — indépendante de la chaîne audio du poste ;
- des **outils anti-bruit** pour lesquels un poste QRP n'a pas la place : silencieur
  d'impulsions (NB), réducteur de bruit adaptatif LMS, notch automatique et réducteur de
  bruit dans le domaine fréquentiel, tous commutables séparément afin de pouvoir les
  comparer sur le même signal ;
- un **S-mètre calibré**, bande par bande ;
- un **décodeur CW** avec classifieur ML ;
- l'**accord tactile** : touchez un signal sur le spectre ou la cascade et le poste est
  accordé dessus via CAT.

L'intention de conception est explicite : **améliorer un transceiver QRP que vous possédez
déjà**, et non le remplacer. Le poste conserve la HF, le filtrage, l'ampli de puissance et
la chaîne d'émission. Le WMSDR ajoute les yeux et les oreilles.

Le poste de référence pour le développement est le **uSDX triband** de Barb, WB2CBA
(<https://antrak.org.tr/blog/usdx-triband-sdr-all-mode-qrp-transceiver/>), qui sort à la
fois une paire I/Q et un port CAT compatible TS-480. Tout poste faisant de même
conviendra.

> **Ce qui est demandé au poste hôte**
> 1. Une **sortie I/Q** en bande de base — deux voies audio en quadrature, au niveau ligne.
> 2. Un **port série CAT** parlant le dialecte Kenwood TS-480 (au minimum `FA`, `MD`).
> 3. Une masse commune et, idéalement, une alimentation partageable.

---

## 2. Matériel

![La face avant du boîtier : dalle tactile 800×480, bloc de huit encodeurs, bouton d'accord, afficheur de la boîte d'accord (unité séparée)](images/wmsdr-front-atu_01.jpg)

*Figure 2 — La face avant du boîtier : dalle tactile 800×480, bloc de huit encodeurs, bouton d'accord, afficheur de la boîte d'accord (unité séparée).*

| Élément | Détail |
|---|---|
| Processeur | Espressif **ESP32-S3**, double cœur à 240 MHz, 8 Mo de PSRAM |
| Codec | **Wolfson WM8731** — CAN pour l'entrée I/Q, CNA pour la sortie audio |
| Horloge codec | Quartz 12,288 MHz (c'est lui qui fixe les fréquences d'échantillonnage disponibles) |
| Fréquences d'échantillonnage | **8 / 32 / 48 / 96 kHz**, commutables à chaud. C'est aussi la largeur visible. |
| Afficheur | Dalle RGB parallèle **800 × 480**, couleur 16 bits, rétroéclairage sur GPIO 45 |
| Tactile | Contrôleur capacitif **GT911**, I²C sur GPIO 48 / 47 |
| CAT | UART1, TX sur **GPIO 42**, RX sur **GPIO 1**, 9600 / 38400 / 115200 bauds |
| Mémoire | NVS de l'ESP32 (flash) pour tous les réglages, la calibration et les couleurs |
| Sans fil | Wi-Fi pour l'heure SNTP ; liaison **ESP-NOW** vers une télécommande XIAO |
| Boutons de commande (option) | **M5Stack Unit 8Encoder** — huit boutons rotatifs à poussoir, un interrupteur et neuf LED RGB, sur le bus I²C du codec (adresse 0x41), alimenté en **5 V**. Voir §8.10. |
| Chaîne de compilation | ESP-IDF 5.5.3, Xtensa GCC |

La paire I/Q entre dans les entrées ligne du WM8731 sous forme d'un signal stéréo — I sur
une voie, Q sur l'autre. Le choix de la bande latérale dépend donc du **sens** de cette
paire, que le WMSDR détecte automatiquement (voir l'onglet STATUS, §8.12) plutôt que
d'exiger un câblage correct du premier coup.

L'audio démodulé ressort par le CNA du WM8731.

> Le schéma de la carte d'interface — conditionnement de l'entrée I/Q, adaptation de
> niveau du CAT, alimentation — figure en [Annexe A](#annexe-a--schéma-de-la-carte-dinterface).

---

## 3. Architecture logicielle

Tout tourne au plus près du matériel sous FreeRTOS, avec une séparation stricte entre les
deux cœurs.

**Cœur 0 — la chaîne audio temps réel**

1. `i2s_aquire_task` lit 1024 échantillons I/Q stéréo depuis le WM8731.
2. `processing_task` fait le travail : suppression de la composante continue → silencieur
   d'impulsions → réduction de bruit adaptative → FFT 1024 points pour l'affichage →
   démodulation (selon le mode) → décimation vers la fréquence audio → filtrage et
   égalisation → CAG → retour en int16.
3. Réception et analyse des trames CAT.

**Cœur 1 — tout ce que l'opérateur voit**

`out_to_dac_task` alimente le CNA, et quatorze tâches « sprite » indépendantes possèdent
chacune un rectangle de l'écran : spectre, cascade, S-mètre, VFO, texte CW, icônes d'état,
etc. Chacune ne se redessine que lorsque ses propres données changent — c'est ce qui
permet à une dalle 800 × 480 de rester vivante sur un microcontrôleur.

Grâce à cette séparation, un réglage d'affichage coûteux ne peut pas bloquer l'audio, et un
réglage DSP coûteux ne peut pas faire déchirer l'image.

**Démodulation** : la BLU utilise le déphasage par transformée de Hilbert ; la CW utilise
un passe-bande étroit centré sur la tonalité choisie, suivi d'un détecteur de Goertzel et
d'un classifieur ML ; l'AM utilise la détection d'enveloppe ; la FM utilise un
discriminateur avec un squelch dérivé du bruit.

---

## 4. Raccorder un transceiver — l'exemple du uSDX triband

Le uSDX triband (WB2CBA) est le poste avec lequel le WMSDR a été développé et mesuré.

**Câblage**

| uSDX | WMSDR | Remarque |
|---|---|---|
| Sortie I | WM8731 LINE IN, gauche | via la carte d'interface |
| Sortie Q | WM8731 LINE IN, droite | via la carte d'interface |
| CAT TX | GPIO 1 (RX) | adaptation de niveau sur la carte d'interface |
| CAT RX | GPIO 42 (TX) | idem |
| GND | GND | une masse commune, point étoile au poste |

**Première mise sous tension**

1. Réglez la vitesse CAT du uSDX et le `CAT BAUD` du WMSDR (onglet DSP) sur la même
   valeur. 115200 est la valeur par défaut ici.
2. Ouvrez le menu → **STATUS**. `MODE / BAND` et `FREQUENCY` doivent suivre le poste en une
   fraction de seconde. S'ils ne bougent pas, la liaison CAT n'est pas établie.
3. Sur le même onglet, `I/Q SENSE` affichera **STRAIGHT (I,Q)** ou **SWAPPED (Q,I)**. Les
   deux conviennent — le WMSDR compense. L'important est que ce ne soit pas bloqué sur
   `checking...`.
4. Vérifiez l'orientation sur l'air : accordez le poste **vers le bas** en fréquence et
   observez une porteuse sur le spectre. Elle doit se déplacer vers la **droite**. Si elle
   part à gauche, appuyez sur **RE-CHECK I/Q** dans l'onglet STATUS.
5. Calibrez le S-mètre pour la bande en cours — voir l'[onglet CALIB](#88-onglet-calib).

**Un comportement connu du uSDX, qu'il vaut mieux annoncer tout de suite**

Pendant que son propre encodeur tourne, le uSDX cesse largement de répondre au CAT —
mesuré à environ 20 réponses par seconde au repos, tombant à 1 ou 2 par seconde pendant
l'accord. L'affichage de fréquence est donc en retard sur le bouton du poste, puis rattrape
dès que vous vous arrêtez. C'est le poste, pas la chaîne d'affichage. Le WMSDR
**n'interroge délibérément pas plus vite** pour compenser, car demander plus souvent
aggrave la situation : le uSDX analyse le CAT dans la même boucle que celle qui lit son
encodeur.

---

## 5. Interface CAT

Le WMSDR parle le dialecte CAT **Kenwood TS-480** sur UART1, 8-N-1, à 9600, 38400 ou
115200 bauds (choix dans l'onglet DSP, appliqué à chaud).

C'est un **client** : il interroge le poste et le suit. Il n'envoie qu'une seule commande
modifiant l'état du poste — le réglage de fréquence de l'accord tactile.

### Commandes envoyées par le WMSDR

| Commande | Cadence | Rôle |
|---|---|---|
| `FA;` | 10 Hz | Lire la fréquence du VFO A — celle que l'opérateur regarde |
| `FB;` | ~0,5 Hz | Lire la fréquence du VFO B |
| `FR;` | ~0,5 Hz | Lire le VFO de réception (0 = A, 1 = B, 2 = mémoire) |
| `MD;` | 1 Hz | Lire le mode de fonctionnement |
| `FA` + 11 chiffres + `;` | à la demande | **Régler** la fréquence — accord tactile uniquement |

### Réponses comprises par le WMSDR

| Réponse | Format | Effet |
|---|---|---|
| `FA` | `FA` + 11 chiffres + `;` (14 car.) | Fréquence du VFO A. Devient l'OL du panadapteur en réception sur A. |
| `FB` | `FB` + 11 chiffres + `;` (14 car.) | Fréquence du VFO B. Devient l'OL en réception sur B. |
| `FR` | `FR` + 1 chiffre + `;` (4 car.) | VFO de réception. Pilote les icônes VFO-A / VFO-B et la cible de l'accord tactile. |
| `MD` | `MD` + 1 chiffre + `;` (4 car.) | Mode de fonctionnement — voir le tableau ci-dessous. |

**Codes de mode (`MD`)**

| Code | Mode | Libellé WMSDR |
|---|---|---|
| 0 | — | NONE |
| 1 | LSB | LSB |
| 2 | USB | USB |
| 3 | CW | CW-U |
| 4 | FM | FM |
| 5 | AM | AM |
| 6 | FSK | DIGI-U |
| 7 | CW inversée | CW-R |
| 8 | TUNE | NONE |
| 9 | FSK inversée | DIGI-L |

### Détails d'implémentation utiles à connaître

- **Les trames sont réassemblées.** Une réponse arrive couramment répartie sur deux
  lectures UART, et plusieurs réponses arrivent couramment en une seule. Le WMSDR
  accumule les octets et émet chaque trame complète terminée par `;`, en conservant la
  queue partielle pour la lecture suivante. Rien n'est purgé lors d'une coupure.
- **Les champs numériques sont validés.** Une trame de bonne longueur et de bon préfixe
  mais dont le champ de chiffres contient du bruit est rejetée plutôt qu'affichée. Le poste
  étant interrogé en continu, la bonne réponse suivante arrive en quelques millisecondes et
  l'affichage conserve simplement la valeur précédente.
- **L'accord tactile s'adresse toujours à `FA`**, même quand le poste annonce recevoir sur
  le VFO B. Sur le uSDX, `FA` adresse le VFO *actif* et un `FB` en écriture est ignoré
  silencieusement. Un drapeau de compilation (`CAT_SET_FREQ_PER_VFO`) existe pour un vrai
  TS-480, qui lui implémente `FB` comme VFO distinct.
- **Ralentissement des interrogations.** Après 5 secondes de silence total, le WMSDR passe
  à une sonde par seconde. Il n'utilise délibérément *pas* le délai d'une seconde de
  l'icône pour cela, car un poste connecté se tait aussi longtemps de lui-même pendant
  l'accord.
- **Ce que le WMSDR ne fait pas :** il ne passe jamais en émission, ne change jamais le
  mode, ne change jamais de VFO et n'écrit jamais en mémoire.

---

## 6. L'écran, élément par élément

La dalle fait 800 × 480. De haut en bas :

```
 0    ┌───────────────┬────┬────────────────────────┬─────────┬──────┐
      │  S-MÈTRE      │RX  │  MODE / ÉTAT (UPSA)    │ ICÔNES  │ICÔNES│  0..23
      │               │TX  ├────────────────────────┤ INFO    ├──────┤
 32   │               │VFO-A│  VFO-A  /  VFO-B      │ INFO RX │ CONST│ 24..96
      │               │VFO-B│                       │         │ I/Q  │
 98   │               │    │  RANGÉE BANDE (UPSB)   │         ├──────┤
      │               │SQL │                        │         │ DSP  │ 97..120
 127  ├───────────────┴────┴────────────────────────┴─────────┴──────┤
      │  RANGÉE HAUTE — commandes de réception                        │ 127..157
 162  ├───────────────────────────────────────────────────────────────┤
      │  BANDEAU DES LIMITES DE BANDE                                 │ 162..177
 178  ├───────────────────────────────────────────────────────────────┤
      │  SPECTRE                                                      │ 178..268
 270  ├───────────────────────────────────────────────────────────────┤
      │  CASCADE (WATERFALL)                                          │ 270..405
 407  ├───────────────────────────────────────────────────────────────┤
      │  ÉCHELLE DE FRÉQUENCE                                         │
 424  ├───────────────────────────────────────────────────────────────┤
      │  TEXTE DÉCODEUR CW  /  BANDEAU RSS                            │
 450  ├───────────────────────────────────────────────────────────────┤
      │  RANGÉE BASSE — affichage et système                          │ 450..480
      └───────────────────────────────────────────────────────────────┘
```

### 6.1 S-mètre (en haut à gauche)

Une aiguille de style analogique avec un modèle ressort-amortisseur, redessinée à 15 Hz.
Elle indique S1–S9 et S9+dB, et elle est **calibrée par bande** — la conversion du niveau
brut en numéro S provient des deux points de capture pris dans l'onglet CALIB. Sans
calibration, elle n'est qu'indicative. La ligne de pied porte la température du cœur.

*Leviers :* [onglet CALIB](#88-onglet-calib) — NOISE REF, SIGNAL REF, CAPTURE NOISE,
CAPTURE SIGNAL.

### 6.2 Colonne RX / TX / VFO / SQL

Cinq voyants empilés entre le S-mètre et la rangée de mode.

- **RX** / **TX** — état de réception et d'émission. Respectivement vert et rouge ; ces
  deux couleurs ne sont délibérément *pas* personnalisables, car c'est une convention de
  sécurité. Tant que **TX ARM** est activé (onglet TX, §8.11), le voyant TX a un **contour
  orange** : le poste peut émettre. Il passe au rouge quand le PTT est actionné.
- **VFO-A** / **VFO-B** — le VFO sur lequel le poste déclare recevoir, d'après la réponse
  `FR;`.
- **SQL** — le squelch FM. Allumé signifie que la porte est **ouverte**, c'est-à-dire qu'un
  signal occupe le canal : c'est un voyant *occupé*, pas un voyant *muet*. Il n'a de sens
  qu'en FM avec un niveau non nul.

*Leviers :* **touchez le voyant SQL** pour engager/désengager le squelch (FM uniquement —
la touche est ignorée dans tous les autres modes). Le niveau lui-même est `FM SQUELCH` dans
l'[onglet AUDIO](#85-onglet-audio).

### 6.3 Rangée de mode (UPSA) et rangée de bande (UPSB)

Le libellé de mode tel que rapporté par le CAT (LSB / USB / CW-U / CW-R / AM / FM /
DIGI-U / DIGI-L), et la bande amateur dans laquelle tombe la fréquence courante — de 160 m
à 10 m, limites IARU Région 1.

### 6.4 Affichage des VFO

VFO-A et VFO-B en gros chiffres, directement issus des champs numériques CAT. Le VFO de
réception est mis en évidence.

### 6.5 Icônes d'information (en haut à droite, 615–799 × 0–23)

- **U** — la liaison CAT/UART. Allumée tant que des trames arrivent ; s'éteint après une
  seconde de silence.
- **E** — la liaison ESP-NOW vers la télécommande XIAO. Le compagnon apporte aussi l'heure dont
  le FT8 a besoin, et sert les cartes et décodages FT8 à un navigateur (§7.4).
- **UTC** — l'heure en direct, via SNTP sur Wi-Fi.

### 6.6 Panneau info RX (615–717 × 24–123)

Trois indications en colonne :

- la **bande passante de réception**, tracée à partir des limites réelles du filtre DSP —
  pas d'après un tableau, donc elle montre toujours ce que fait réellement la chaîne audio ;
- l'**aide à l'accord CW**, qui montre l'écart entre la tonalité reçue et la tonalité CW
  choisie ;
- le bargraphe de **volume** (VOL, 0,05 … 1,00).

Le panneau continue de se mettre à jour quand le menu ou la fenêtre de décodage est ouvert — il
se trouve au-dessus de la zone recouverte, ce qui permet de voir l'audio réagir pendant qu'on
change un réglage.

### 6.7 Constellation I/Q (726–798 × 24–96)

Un nuage de points 73 × 73 de la paire I/Q entrante, avec un réticule central. Un cercle
propre signifie que I et Q sont équilibrés ; une ellipse signale un déséquilibre de gain ;
une ellipse inclinée signale une erreur de phase. C'est le moyen le plus rapide de voir que
la tête HF va bien.

### 6.8 Info DSP (726–798 × 97–120)

Un affichage compact `SQL` / `AF` — l'état du squelch et le niveau de sortie audio. `AF` est en
dB, **0 dB étant le point où la sortie écrête**. Sur une voix propre, les crêtes se situent vers
**−6 dB** ; si `AF` monte à −2 … 0 dB, on entend une distorsion qui ressemble à de la
réverbération ou à un écho — baissez `AGC TARGET` ou `VOLUME` (voir §8.5). Comme le panneau info
RX, il reste actif menu ouvert.

### 6.9 Spectre (178–268)

Le panadapteur. Pleine largeur, 800 px, réparti sur la largeur courante (8 / 32 / 48 /
96 kHz divisé par le facteur de zoom).

Ce qui peut y être tracé, chaque couche étant commutable indépendamment :

| Couche | Ce que c'est |
|---|---|
| **Trace en direct** | La trame FFT courante, lissée |
| **Plancher de bruit** | Une moyenne lente par bin — le plancher propre à la bande |
| **Traînée (trail)** | Une trace qui s'efface, pour qu'un transitoire laisse une queue visible |
| **Maintien de crête** | Le maximum atteint par chaque bin, décroissant lentement |
| **Remplissage** | Rempli jusqu'à la base au lieu d'un simple trait |
| **Dégradé** | Couleur selon l'amplitude au lieu d'une couleur unie |
| **Masque de bande passante** | Un bandeau translucide montrant la bande passante de réception, relié aux limites réelles du filtre |
| **Marqueurs centre / accord / crête** | Trois marqueurs optionnels |

*Leviers :* tout l'[onglet DISPLAY](#81-onglet-display), l'[onglet SPECTRUM](#82-onglet-spectrum),
l'[onglet TRACE](#83-onglet-trace), et les boutons `FILL`, `SG`, `SAR`, `ZOOM+/-`.

**Accord tactile.** Le spectre et la cascade se comportent comme une seule surface
d'accord. Appuyez, glissez pour positionner le curseur, relâchez — le poste est accordé sur
la fréquence sous votre doigt, arrondie selon `TUNE SNAP`. La commande CAT n'est émise
**qu'au relâchement**, jamais pendant le glissement. Si `FREQ TOUCH` est désactivé, le
curseur suit toujours votre doigt (afin que le panneau n'ait pas l'air mort) mais aucune
commande n'est envoyée.

### 6.10 Cascade (270–405)

135 lignes d'historique. Chaque bin est colorié par rapport à **son propre plancher de
bruit suivi**, et non par rapport à un niveau global unique — c'est ce qui permet à un
signal faible de rester visible à côté d'un signal fort. Sept palettes, à faire défiler
avec `WFG`.

*Leviers :* `WATERFALL SPEED`, `WF BLACK LEVEL`, `WF CONTRAST`, `WF FLOOR BLEND` dans
l'[onglet SPECTRUM](#82-onglet-spectrum) ; `WF FLOOR` par bande dans
l'[onglet CALIB](#88-onglet-calib).

> Les modifications de la cascade **ne sont pas** visibles quand le menu est ouvert — le
> menu couvre cette zone et son rafraîchissement est suspendu. Fermez le menu et laissez
> l'ancien historique défiler (environ 3 s au zoom ×1, environ 23 s au ×8) avant de juger
> un changement.

### 6.11 Échelle de fréquence

La fréquence absolue sous chaque partie du spectre, calculée à partir de l'OL lu en CAT
plus le décalage, en tenant compte du sens de la bande latérale.

### 6.12 Ligne décodeur CW / RSS (424–449)

Une ligne de texte. En CW elle porte les caractères décodés ; sinon elle peut faire défiler
un flux RSS, à sélectionner avec le bouton `RSS`.

*Leviers :* tout l'[onglet CW](#87-onglet-cw).

---

## 7. Les rangées de boutons

Deux rangées de dix. Les boutons munis d'une barre-LED sont des bascules et la barre montre
l'état ; les autres sont momentanés et parcourent une liste, le libellé servant alors
d'affichage.

### 7.1 Rangée haute — commandes de réception (y 127–157)

| Bouton | Type | Rôle |
|---|---|---|
| **ATT** | bascule | Atténuateur d'entrée |
| **AGC** | bascule | CAG audio. Désactivée, la chaîne retombe sur le simple réglage `VOLUME`, qui à son maximum de 1,0 est faible — c'est normal, pas un défaut. `VOLUME` n'est qu'un atténuateur et ne peut pas compenser. |
| **NB** | bascule | Silencieux d'impulsions large bande, appliqué à l'I/Q **avant** décimation. Seuil dans l'onglet AUDIO. |
| **DNR** | bascule | Réduction de bruit adaptative (NLMS) — conserve ce qui est *périodique*. Efficace sur le souffle sous une voix. |
| **FDNR** | bascule | Réduction de bruit dans le domaine fréquentiel (spectrale) — estime le bruit par bin de FFT. Outil **différent** de DNR, pas une version plus forte. Ignorée en CW. Lequel des trois moteurs elle utilise se règle par FDNR ALGO dans l'onglet AUDIO ; la valeur par défaut ne demande aucune attention. |
| **NOTCH** | bascule | Notch automatique — même prédicteur NLMS que DNR, mais en gardant l'*erreur* au lieu de la prédiction. Élimine une porteuse fixe. |
| **FILTER** | momentané | Fait défiler la bande passante de réception. La liste dépend de la classe de mode : **BLU** 3600 / 2800 / 2400 / 1800 / 1200 Hz ; **CW** 1000 / 500 / 250 / 100 / 50 Hz ; **AM** 4500 / 3500 / 2500 Hz. Chaque classe mémorise son propre choix. Le libellé *est* l'affichage. |
| **I/Q** | momentané | Fait défiler la fréquence d'échantillonnage, donc la largeur visible : 8k → 32k → 48k → 96k. Recadence l'I²S, reprogramme le WM8731, et le choix est mémorisé au redémarrage. |
| **FILL** | bascule | Remplissage du spectre jusqu'à la base |
| **MUTE** | bascule | Coupe la sortie, en aval de la CAG afin que le rétablissement n'arrive pas à plein niveau. Non mémorisé — un poste qui démarre muet passe pour cassé. |

> **DNR et FDNR sont commutés séparément à dessein.** L'intérêt d'avoir les deux est de les
> comparer sur le même signal, d'où leur voisinage sur la rangée.

### 7.2 Rangée basse — affichage et système (y 450–480)

| Bouton | Type | Rôle |
|---|---|---|
| **VOL+** / **VOL−** | momentané, répétition | Niveau audio, 0,05 … 1,00 par pas de 0,05. Même valeur que `VOLUME` dans l'onglet AUDIO — un menu ouvert affiche le changement immédiatement. |
| **SSTV** | bascule | Ouvre et ferme la fenêtre de décodage — SSTV, WeFax, RTTY, PSK, FT8 (§7.3). Sa LED indique que la fenêtre est ouverte. |
| **ZOOM+** / **ZOOM−** | momentané | Facteur du zoom-FFT, de 1 à 8. De vrais bins étroits, pas de l'interpolation. Le S-mètre reste délibérément sur le spectre principal. |
| **WFG** | momentané | Fait défiler la palette de la cascade (7 entrées) |
| **SG** | bascule | Dégradé du spectre — couleur selon l'amplitude, ou couleur unie |
| **SAR** | bascule | *Spectrum Auto dB Range* — laisse la fenêtre d'affichage suivre le signal automatiquement |
| **RSS** | momentané | Fait défiler les flux RSS ; LED éteinte quand le flux est sur OFF |
| **MENU** | momentané | Ouvre et ferme le menu de réglages |

### 7.3 La fenêtre de décodage (SSTV / WeFax / RTTY / PSK / FT8)

![La fenêtre de décodage, MODE AUTO, en attente d'un en-tête VIS SSTV](images/menu-sstv.jpg)

*Figure 7.3 — La fenêtre de décodage, MODE AUTO, en attente d'un en-tête VIS SSTV. Mode et actions à gauche, CLEAR / HOLD / TUNE / CLOSE à droite.*

Un décodeur, en réception et affichage seulement, pour les images de télévision à balayage lent,
les cartes météo HF et le texte RTTY et PSK31/PSK63. Elle s'ouvre au même endroit que le menu et le remplace — les deux ne
sont jamais ouverts ensemble. Rien n'est enregistré : la carte n'a pas de carte SD, une image
dure jusqu'à ce qu'on l'efface ou qu'elle défile hors de l'écran.

Les cartes WeFax et les décodages FT8 peuvent être enregistrés depuis un navigateur, via le
compagnon (§7.4).

> **RTTY, WeFax, PSK31 et FT8 sont vérifiés sur l'air.** **Le SSTV n'est pas encore vérifié sur
> une vraie transmission** — il réussit son auto-test intégré (voir *Auto-test* plus bas).

**Disposition.** Quatre boutons en colonne de chaque côté, l'image au milieu, une ligne d'état
au-dessus.

| Colonne gauche | Colonne droite |
|---|---|
| **MODE** — ce qu'il faut décoder ; le libellé montre le choix actuel | **CLEAR** — effacer l'image et attendre à nouveau |
| **START** — démarrer sans attendre d'en-tête, ou arrêter | **HOLD** — conserver l'image ; LED allumée tant qu'il est actif |
| **SLANT−** — corriger la cadence de −20 ppm | **TUNE** — barre de tonalité à la place de l'image ; LED allumée tant qu'il est actif |
| **SLANT+** — corriger la cadence de +20 ppm | **CLOSE** — fermer la fenêtre (le bouton SSTV aussi) |

**MODE** fait défiler :

| Réglage | Décode |
|---|---|
| **AUTO** | SSTV ; l'en-tête VIS choisit Martin 1 ou Martin 2 |
| **M1** / **M2** | SSTV ; l'en-tête VIS décide toujours — le réglage sert à START en l'absence d'en-tête |
| **WEFAX** | Radiofax HF, 120 lignes/min, IOC 576 |
| **TEST** | auto-test : envoie une image Martin 1 synthétique dans le décodeur |
| **RTTY** | texte RTTY (Baudot) ; vitesse et shift selon le préréglage |
| **PSK** | texte PSK31 / PSK63 (Varicode) ; choisissez le signal sur la cascade audio |
| **FT8** | FT8, à chaque créneau UTC de 15 s ; une liste de décodages |

**Largeur de spectre.** Le SSTV a besoin d'un audio à 12 kHz. Si la largeur est 8k ou 32k,
l'ouverture de la fenêtre passe en **48k** et la fermeture remet l'ancienne valeur. Ce changement
n'est pas mémorisé — un redémarrage revient à votre propre réglage. Si vous appuyez sur I/Q
pendant que la fenêtre est ouverte, votre choix est conservé. À 96k, rien ne change.

**L'audio n'est pas modifié.** Les décodeurs prennent une copie de l'audio après la bande
passante FILTER et avant NR, notch et CAG : ces réglages n'influencent pas le décodage, et vous
continuez d'écouter normalement.

#### TUNE — la barre de tonalité

Une barre de 1000 à 2500 Hz avec des repères à **1200** (synchro, rouge), **1500** (noir),
**1900** (en-tête SSTV, jaune) et **2300** (blanc). L'aiguille verte est la tonalité mesurée et la
bande grise sa dispersion sur la dernière demi-seconde ; la fréquence et le niveau sont affichés
dessous. Elle devient grise en l'absence de tonalité.

- **SSTV** au démarrage : l'aiguille reste sur **1900**, puis saute entre 1200 et 1500–2300.
- **Une carte météo** : oscille entre **1500** et **2300**.
- **La voix** : l'aiguille erre et la bande grise respire — normal, la voix n'est pas une
  tonalité unique.

#### Recevoir en SSTV (Martin 1 / Martin 2)

![MODE M1 (Martin 1) sélectionné, en attente d'un signal](images/menu-sstv-m1.jpg)

*Figure 7.3a — MODE M1 (Martin 1) sélectionné, en attente d'un signal.*

![MODE M2 (Martin 2) sélectionné, en attente d'un signal](images/menu-sstv-m2.jpg)

*Figure 7.3b — MODE M2 (Martin 2) sélectionné, en attente d'un signal.*

1. Accordez le poste comme pour la phonie — USB à partir du 20 m, LSB sur 40 et 80 m.
2. Ouvrez la fenêtre, MODE **AUTO**, HOLD désactivé.
3. Au début d'une image, l'en-tête est décodé (`VIS 44 -> M1` dans le journal série) et l'image
   se construit depuis le haut : **114 s** en Martin 1, **58 s** en Martin 2.
4. La ligne d'état indique `RX line n/256`, le nombre d'impulsions de synchro trouvées et
   l'**inclinaison** (slant) en ppm. L'inclinaison est corrigée automatiquement : elle bouge
   pendant les premières dizaines de lignes, puis se stabilise.
5. À la fin : `complete`. L'image reste jusqu'à ce que l'en-tête suivant la remplace — appuyez
   sur **HOLD** pour la garder.

Si l'en-tête est manqué (signal faible ou QSB), appuyez sur **START** quand l'image commence ; la
synchro de ligne est recherchée automatiquement. Si l'image penche, **SLANT±** la redresse. Après
12 lignes sans synchro, l'image s'arrête avec `signal lost` et conserve ce qu'elle a reçu.

#### Recevoir en WeFax

![MODE WEFAX sélectionné, en attente d'une carte](images/menu-sstv-wefax.jpg)

*Figure 7.3c — MODE WEFAX sélectionné, en attente d'une carte.*

1. Accordez en **USB, 1,9 kHz sous la porteuse publiée** (voir *Où écouter*). Sur TUNE, une carte
   oscille entre 1500 et 2300.
2. Ouvrez la fenêtre, MODE **WEFAX**.
3. Une carte commence par un signal de départ de 5 s (`start signal detected`), puis 30 s de
   phasage (`phasing n/56`). Le phasage mesure l'**erreur d'horloge** et aligne la carte ; la
   ligne d'état affiche ensuite `RX line n` et l'horloge en ppm.
4. La carte apparaît comme une **bande qui défile vers le haut**, la ligne la plus récente en bas.
   Chaque rangée regroupe trois lignes de la carte, ce qui respecte les proportions. Elle se
   termine sur le signal d'arrêt (`chart complete`).

**Arrivé en cours de carte ?** Appuyez sur **START**. Les lignes sont reçues sans phasage, la
carte apparaît donc décalée latéralement : touchez la bande à l'endroit où apparaît la marge
gauche de la carte, et cette colonne devient le bord gauche. À savoir sur ce geste :

- seules les rangées suivantes bougent — celles déjà affichées restent telles quelles ;
- le décalage se fait dans un seul sens : pour déplacer la marge un peu vers la droite, touchez
  près de l'extrémité droite ;
- un doigt couvre plusieurs colonnes — touchez à nouveau si c'est légèrement à côté ;
- redressez d'abord toute inclinaison avec **SLANT±**, sinon la marge dérive à nouveau ;
- les touchers ne comptent que pendant la réception d'une carte, et tout toucher involontaire
  sur la bande la décale.

**HOLD** fige la bande pendant que la réception continue ; **CLEAR** l'efface et attend le signal
de départ suivant.

> **Exemple vérifié — DWD Pinneberg, 7880 kHz :** cadran **7878,1 kHz USB**, MODE **WEFAX**.
> Cartes reçues proprement et enregistrées depuis le navigateur (§7.4).

#### Recevoir en RTTY

![MODE RTTY sélectionné, en attente d'un signal](images/menu-sstv-rtty.jpg)

*Figure 7.3d — MODE RTTY sélectionné, en attente d'un signal.*

Du texte radiotélétype — RTTY amateur et bulletins météo — dans une console de 80 × 16 qui
défile vers le haut. Dans les modes RTTY, trois boutons changent de rôle :

| Bouton | En RTTY |
|---|---|
| **REV** (START) | inverse mark et space ; LED allumée tant qu'il est actif |
| **TONE−** / **TONE+** (SLANT−/+) | déplace la paire de tonalités attendue de 10 Hz |
| **CLEAR** | efface le texte |
| **HOLD** | fige la console ; le décodage continue |
| **TUNE** | barre de tonalité avec les repères **M** (mark) et **S** (space) sur la paire réelle |

**Touchez la ligne d'état** pour passer d'un préréglage vitesse / shift au suivant :

| Préréglage | Utilisé par | Mark / space audio |
|---|---|---|
| **45.45/170** | RTTY amateur (par défaut) | 2125 / 2295 Hz |
| **50/170** | certaines stations utilitaires et amateurs | 2125 / 2295 Hz |
| **50/450** | bulletins météo DWD | 1675 / 2125 Hz |
| **75/170** | trafic amateur / utilitaire plus rapide | 2125 / 2295 Hz |
| **50/850** | anciennes stations utilitaires à grand shift | 1475 / 2325 Hz |

La ligne d'état indique par exemple `RTTY 50/450 | M 1714 S 2164 | AFC +39 Hz | LTRS`.

1. Accordez en USB pour que les deux tonalités tombent près des repères **M** et **S** de TUNE —
   l'AFC les rattrape ensuite, jusqu'à ±200 Hz, et la ligne d'état indique l'écart
   (`AFC +n Hz`).
2. Choisissez le préréglage en touchant la ligne d'état.
3. Le texte s'affiche au fil de la réception ; chaque ligne terminée est aussi écrite dans le
   journal série.
4. Rien que du charabia sur un signal propre ? Appuyez sur **REV** — les stations diffèrent sur
   la tonalité qui sert de mark, et changer de bande latérale les inverse aussi.

Détails du décodage : les inversions lettres/chiffres sont suivies, un espace ramène aux lettres
(la convention habituelle), et un caractère dont le bit de départ ou d'arrêt est mauvais, ou
trop faible, est écarté plutôt qu'affiché en charabia.

> **Exemple vérifié — DWD Pinneberg, 10100,8 kHz :** cadran **10098,9–10099,1 kHz USB**, préréglage
> **50/450**, **REV désactivé**, AFC stabilisée à **+39 Hz**. Copie propre du bulletin de la
> station, y compris sa liste de fréquences. Un écart d'AFC de cette taille sur toutes les
> stations désigne l'étalonnage en fréquence du poste ; **TONE+** permet de le pré-régler.

#### Recevoir en PSK31 / PSK63

![MODE PSK sélectionné, en attente d'un signal](images/menu-sstv-psk.jpg)

*Figure 7.3e — MODE PSK sélectionné, en attente d'un signal.*

Du texte PSK de clavier à clavier. Plusieurs signaux partagent généralement les mêmes 3 kHz
d'audio : ce mode affiche donc une **cascade audio** (0–3000 Hz) sous la ligne d'état, un repère
vert au-dessus pour le signal décodé, et les 12 dernières lignes de texte en dessous.

| Bouton / zone | En PSK |
|---|---|
| **SQL** (START) | squelch ; LED allumée tant qu'il est actif (actif par défaut) — écarte les caractères quand la qualité du signal est sous 60 % |
| **FREQ−** / **FREQ+** (SLANT−/+) | déplace la fréquence décodée de 5 Hz |
| **CLEAR** | efface le texte |
| **HOLD** | fige le texte ; le décodage continue |
| **TUNE** | barre de tonalité avec un repère **PSK** sur la fréquence décodée |
| toucher la **cascade** | décode le signal sous le doigt |
| toucher la **ligne d'état** | bascule entre **PSK31** et **PSK63** |

La ligne d'état indique par exemple `PSK31 | 1000 Hz | AFC +2.4 | Q 92% | SQL on`.

1. Accordez en USB sur l'activité PSK — les signaux apparaissent comme de fines traces verticales
   sur la cascade.
2. Touchez une trace. L'AFC rattrape les derniers hertz (jusqu'à ±30 Hz) tant que la qualité est
   bonne.
3. **Q** est la qualité du signal : au-dessus d'environ 60 % le texte s'affiche ; un signal propre
   dépasse 90 %.
4. Du charabia avec SQL désactivé, ou rien avec SQL actif ? Le repère est sans doute à côté de la
   trace — touchez-la à nouveau ou ajustez avec **FREQ±**. Une trace large, « double », est
   généralement du PSK63 : touchez la ligne d'état.

Chaque ligne de texte terminée est aussi écrite dans le journal série.

#### Recevoir en FT8

![MODE FT8 sans heure UTC : l'en-tête l'indique tant que le compagnon n'a pas fourni l'heure](images/menu-sstv-ft8.jpg)

*Figure 7.3f — MODE FT8 sans heure UTC : l'en-tête l'indique tant que le compagnon n'a pas fourni l'heure. Les décodages apparaissent après chaque tranche de 15 s.*

Le FT8 ne demande ni réglage dans la fenêtre ni START : chaque émission dure 15 s et commence sur
une limite UTC (:00, :15, :30, :45) ; le poste enregistre donc chaque créneau et le décode à sa fin.

**Deux conditions :**
- **L'heure UTC du compagnon** (icône **E**, §6.5). Les créneaux FT8 doivent démarrer à une fraction
  de seconde près, et le poste n'a pas d'horloge propre. Sans elle, la ligne d'état affiche
  `no UTC - FT8 needs the companion's time` et rien n'est décodé.
- **Un audio à 12 kHz** — la fenêtre s'en charge (voir *Largeur de spectre* plus haut).

1. Accordez en **USB** sur une fréquence FT8 (voir *Où écouter*), par exemple **14,074 MHz**.
2. Ouvrez la fenêtre, MODE **FT8**.
3. La ligne d'état passe par `waiting for slot hh:mm:ss`, puis `slot hh:mm:ss` pendant
   l'enregistrement, puis `decoding`. Les décodages apparaissent quelques secondes après la fin
   de chaque créneau.

La liste montre les 16 derniers décodages, le plus récent en bas :

```
UTC     SNR   DT   Hz  message
111415   -5 +0.1  822  CQ ON3URT JO10
111415  +11 +0.1 1706  KH8WW IW4EGP JN64
```

- La première ligne de chaque créneau est **blanche**, les suivantes vertes : les créneaux se
  lisent par groupes.
- **SNR** est l'estimation du décodeur lui-même. Il n'est pas étalonné comme celui de WSJT-X :
  comparez les stations entre elles plutôt qu'avec un autre logiciel.
- **DT** est le calage de la station dans le créneau. Le poste mesure son propre retard audio sur
  les créneaux chargés et le corrige ; après une minute ou deux, la plupart des stations sont
  proches de **0.0**. Une station à +1 ou +2 s a elle-même une horloge décalée.
- **Hz** est la fréquence audio — la position de la station dans la bande passante.
- **`<...>`** est un indicatif transmis seulement sous forme de code court. Le poste retient les
  indicatifs complets entendus, si bien que ces champs se complètent (par ex. `<YL3KZ>`) dès que la
  station est apparue en entier.

| Bouton | En FT8 |
|---|---|
| START, SLANT−/+ | inutilisés (vides) — le FT8 suit ses propres créneaux UTC |
| **CLEAR** | vide la liste |
| **HOLD** | fige la liste pour la lire ; le décodage continue |

La ligne d'état indique aussi le créneau précédent, par ex. `last 23 in 7010 ms`. Chaque décodage,
et un résumé par créneau, sont aussi écrits dans le journal série.

**À quoi s'attendre.** Sur une bande 20 m chargée, typiquement **15 à 27 décodages par créneau**.
Le poste en décode moins que WSJT-X sur PC : les signaux les plus faibles, que WSJT-X extrait en
plusieurs passes, restent au-delà de ce que l'ESP32-S3 peut traiter en un créneau.

> **Vérifié sur l'air — 14,074 MHz, septembre 2026 :** 16 à 27 décodages par créneau, chaque
> créneau décodé, DT stabilisé vers 0, indicatifs codés complétés en quelques créneaux.

Les décodages apparaissent aussi en direct dans un navigateur, où l'on peut les filtrer et les
enregistrer (§7.4).

#### Où écouter

| Quoi | Fréquence | Mode | Remarques |
|---|---|---|---|
| SSTV | 3,730–3,735 MHz | LSB | soirées européennes |
| SSTV | 7,165 MHz | LSB | animé le week-end |
| SSTV | 14,230 MHz (aussi 14,227, 14,233) | USB | la fréquence mondiale principale |
| SSTV | 21,340 / 28,680 MHz | USB | quand la bande est ouverte |
| WeFax, DWD Pinneberg | cadran **3853,1** kHz (porteuse 3855) | USB | plutôt la nuit |
| WeFax, DWD Pinneberg | cadran **7878,1** kHz (porteuse 7880) | USB | jour et nuit |
| WeFax, DWD Pinneberg | cadran **13880,6** kHz (porteuse 13882,5) | USB | plutôt le jour |
| RTTY, amateur | 14,080–14,100 MHz | USB | 45.45/170 ; essayez REV |
| RTTY, DWD Pinneberg | cadran **4581,1 / 7644,1 / 10099,1** kHz (porteuses 4583 / 7646 / 10100,8) | USB | 50/450 ; confirmé sur l'air sur 10100,8 |
| RTTY, météo | cadran **11037,3** kHz | USB | 50/450 ; confirmé sur l'air |
| PSK31 | 14,070 MHz | USB | en journée |
| PSK31 | 7,040 / 3,580 MHz | USB | en soirée |
| FT8 | **14,074** MHz | USB | la plus active ; confirmé sur l'air |
| FT8 | 7,074 / 3,573 MHz | USB | le soir et la nuit |
| FT8 | 21,074 / 28,074 MHz | USB | quand la bande est ouverte |

#### Auto-test : TEST

Avec MODE sur **TEST**, **START** envoie une image Martin 1 synthétique parfaite directement dans
le décodeur — le récepteur n'intervient pas — avec une erreur d'horloge volontaire de +50 ppm :
en-tête `VIS 44 -> M1`, puis 8 barres de couleur verticales (blanc, jaune, cyan, vert, magenta,
rouge, bleu, noir) ; l'inclinaison se stabilise près de +50 ppm.

Si le résultat est correct, le décodeur fonctionne et tout problème sur l'air vient de l'accord
ou de la propagation. L'auto-test tourne plus lentement que le temps réel ; cela ne change pas le
résultat.

#### Limites

- Rien n'est enregistré sur le poste — les cartes WeFax et les décodages FT8 peuvent être
  enregistrés depuis un navigateur (§7.4) ; les images SSTV et les textes RTTY / PSK, pas encore.
- SSTV : Martin 1 et Martin 2 uniquement. WeFax : 120 lignes/min, IOC 576 uniquement. RTTY :
  Baudot (ITA2) uniquement — pas de SITOR / NAVTEX. PSK : BPSK31 et BPSK63 uniquement — pas de
  QPSK. FT8 : réception seulement — pas de FT4, pas d'émission.
- Le mode TEST est provisoire et sera retiré une fois le SSTV vérifié sur des signaux réels.

---

### 7.4 Les pages web du compagnon (xiao-dx.local)

Le poste n'a ni stockage ni clavier. Le XIAO compagnon — la carte qui apporte déjà l'heure UTC et
les spots DX par ESP-NOW — reçoit les cartes WeFax et les décodages FT8, garde les plus récents en
mémoire et les sert à tout navigateur de votre réseau Wi-Fi ; il transmet aussi au poste la CW
tapée dans le navigateur. L'enregistrement se fait dans le navigateur, sur votre téléphone ou
votre PC.

Ouvrez **`http://xiao-dx.local/`** :

- **Home** est un tableau de bord à tuiles actualisées — UTC, FT8 (décodages du dernier créneau),
  Images, cluster DX, signal Wi-Fi et durée de fonctionnement du compagnon. Touchez une tuile
  pour ouvrir sa page.
- **☰** (en haut à gauche, sur chaque page) ouvre un tiroir avec toutes les pages : Home, FT8,
  Pictures, **Transmit**, Control panel, RSS feeds, Settings, Status (JSON).
- **☀ / ☾** (en haut à droite) bascule entre un thème sombre et un thème clair ; le choix est
  mémorisé dans ce navigateur.
- **Settings** regroupe les champs Wi-Fi, **CALLSIGN** et cluster DX. L'enregistrement redémarre
  le compagnon ; laisser le mot de passe vide conserve celui enregistré. En première
  configuration (point d'accès `XIAO-DX-Config`, 192.168.4.1), le formulaire s'ouvre directement.

Les pages en direct affichent **LIVE** une fois connectées et se reconnectent d'elles-mêmes.

**Conditions :** le poste et le compagnon à jour, le compagnon sur votre Wi-Fi, et la fenêtre de
décodage ouverte dans le mode concerné — le compagnon ne reçoit que pendant que le poste décode.

#### Images — `/fax.html`

- La carte WeFax **se construit en direct**, ligne par ligne ; **follow live** garde la ligne la
  plus récente à l'écran.
- Les **2 dernières cartes** sont conservées jusqu'au redémarrage du compagnon ; cliquez sur l'une
  d'elles dans la liste pour l'afficher. Une page ouverte en cours de carte charge ce qui a déjà été
  reçu.
- **Save PNG** enregistre la carte affichée, nommée par date et heure
  (par ex. `wefax_20260917_1029Z_3.png`).
- Une carte fait 640 pixels de large, comme sur l'écran du poste.

> **Exemple vérifié — DWD Pinneberg, 7880 kHz :** cadran **7878,1 kHz USB**, MODE **WEFAX**. La
> carte « Atlantic North: sea state » a été reçue en direct dans le navigateur et enregistrée en PNG.

#### FT8 — `/ft8.html`

| Élément | Rôle |
|---|---|
| tableau | créneau le plus récent en haut : UTC, SNR, DT, Hz, message |
| message vert | un **CQ** |
| ligne orange | un message contenant **votre indicatif** — le **CALLSIGN** réglé sur la page de configuration du compagnon |
| **CQ only** / **my call only** | filtres |
| champ de recherche | n'affiche que les messages contenant un indicatif ou un locator, par ex. `JN` |
| **Pause** | fige le tableau pour le lire ; les décodages continuent d'arriver, le bouton les compte, un nouveau clic revient aux plus récents |
| `last slot …: N decoded, M received` | le compte du poste face à ce qui a atteint la page ; **rouge** quand ils diffèrent — des décodages ont été perdus sur la liaison |
| **Save log** | télécharge tous les décodages conservés dans un fichier texte au format `ALL.TXT` de WSJT-X |

Le compagnon garde les **1000 derniers décodages** jusqu'à son redémarrage ; recharger la page les
fait tous revenir.

#### Émission (CW) — `/tx.html`

Émettre en CW depuis n'importe quel navigateur — téléphone, tablette ou PC. Le poste manipule le
uSDX par sa ligne PTT ; la page ne sert qu'à taper.

**C'est le poste qui décide.** Rien n'est émis tant que **TX ARM** n'est pas activé sur le poste
(MENU → TX, §8.11). Le badge en haut indique ce que le poste rapporte, une fois par seconde :

| Badge | Signification |
|---|---|
| **RADIO NOT HEARD** | aucun état reçu du poste depuis 4 s — liaison ou poste hors service |
| **NOT ARMED** | le poste refuse le texte ; armez-le sur le poste |
| **ARMED** (orange) | prêt |
| **SENDING** (rouge) | manipulation en cours, avec les caractères encore en attente |

**La manipulation en direct** (activée par défaut) émet chaque caractère **au fil de la frappe** —
sans Entrée, donc sans trou où une autre station pourrait prendre la parole.

- **Entrée** tape une espace.
- **Retour arrière** reprend les lettres que le poste **n'a pas encore commencées** ; celles déjà
  émises ne peuvent pas être rappelées.
- Seuls les caractères transmissibles en CW sont acceptés : A–Z, 0–9, `. , ? / = + - @`, et les
  signes de procédure entre chevrons — `<AR> <SK> <BT> <KN> <AS> <HH>` (émis une fois le `>` tapé).
- Si vous tapez moins vite que le manipulateur, la transmission **attend 1,5 s** entre les mots
  (tonalité locale active, réception coupée) au lieu de se terminer après chaque mot.
- Fonctionne aussi avec les claviers de téléphone.

Manipulation en direct désactivée : tapez une ligne entière, puis **Entrée** ou **Send**.

- **Vitesse** — le curseur WPM (5–60) suit le poste et le modifie au relâchement.
- **Macros** — CQ, QRZ?, RST, 73 `<SK>`, MY CALL, AGN? ; `{CALL}` est remplacé par le CALLSIGN des
  Settings. *edit macros* les modifie (`Libellé | texte`, une par ligne, mémorisées dans ce
  navigateur).
- **STOP** (ou **Échap**) arrête immédiatement et vide ce qui reste en attente.
- **Sent** journalise ce qui est parti, et en rouge ce qui a été refusé faute d'armement.

> **Vérifié sur l'air (septembre 2026) :** frappe en direct jusqu'à 45 WPM, propre sur la
> tonalité locale et sur un second récepteur.

#### Si une page reste vide

- **LIVE mais rien n'apparaît :** la fenêtre de décodage est-elle ouverte en WEFAX / FT8 sur le
  poste ?
- **FT8 affiche souvent `N decoded, M received` en rouge :** la liaison ESP-NOW perd des trames —
  rapprochez le compagnon ou améliorez son antenne.
- **Le compagnon ne rejoint aucun réseau Wi-Fi** — il voit les réseaux lors d'un scan mais ne se
  connecte à aucun, pas même à son propre point d'accès `XIAO-DX-Config` : vérifiez d'abord son
  **antenne** (le petit connecteur à clipser se détache facilement), puis son alimentation USB.
  C'est arrivé sur le prototype ; une meilleure antenne a réglé le problème.
- **Transmit affiche RADIO NOT HEARD** après un changement de canal Wi-Fi du routeur : le
  firmware actuel du compagnon suit le canal de lui-même ; un firmware plus ancien demandait un
  redémarrage du compagnon. Fixer le routeur sur un canal (11 est celui que le WMSDR écoute en
  premier) évite toute recherche.

---

## 8. Le menu, onglet par onglet

Appuyez sur **MENU** dans la rangée basse. Le menu recouvre le spectre, la cascade,
l'échelle de fréquence et la ligne du décodeur — les rangées de boutons restent actives,
donc **MENU** referme le menu.

**Douze onglets :** DISPLAY · SPECTRUM · TRACE · DSP · AUDIO · EQ · CW · CALIB · COLORS ·
KNOBS · TX · STATUS

**Comment les commandes fonctionnent**

- Les **cellules** munies d'une barre-LED sont des marche/arrêt. Une touche bascule.
- Les **lignes numériques** ont des boutons `−` et `+` avec la valeur entre les deux. Les
  boutons grisent en fin de course. **Appui maintenu** pour répéter : un pas
  immédiatement, puis répétition après 400 ms, puis plus rapide après 1,5 s.
- Le **bouton d'action** en bas au centre change de sens selon l'onglet — le plus souvent
  **RESET DEFAULTS**.

**Comment les réglages sont enregistrés**

Tout sauf l'onglet STATUS est écrit en flash sous forme d'**un seul enregistrement
versionné** (l'onglet TX a son propre enregistrement, et TX ARM n'est jamais enregistré), validé à la **fermeture** du menu — six modifications font une écriture, et
aucune modification ne fait aucune écriture. Les réglages changés hors du menu (boutons
`WFG`, `SG`, `SAR`, `FILL`, `VOL`, appui sur SQL) sont rattrapés par une sauvegarde
différée déclenchée 3 secondes après le dernier changement.

> **RESET DEFAULTS n'est pas un confort, c'est une sécurité.** Plusieurs de ces valeurs
> sont mémorisées *et* capables de rendre l'affichage illisible — un plancher de spectre qui
> pousse la trace hors de la fenêtre, une couleur de texte réglée sur sa propre couleur de
> fond. Une fois enregistré, un cycle d'alimentation ne vous sauve plus. Le bouton de
> réinitialisation est le chemin de retour, et les limites min/max de chaque ligne en sont
> l'autre moitié.

---

### 8.1 Onglet DISPLAY

![Onglet DISPLAY](images/menu-display.jpg)

*Figure 8.1 — Onglet DISPLAY.*

Une grille 3 × 4 de cellules marche/arrêt plus une commande numérique. Tout ici concerne
**ce qui est tracé sur le spectre**, pas le DSP.

| Cellule | Ce qu'elle commande |
|---|---|
| **CENTER PK** | Marqueur sur la fréquence centrale (OL) |
| **TUNED PK** | Marqueur sur la fréquence accordée |
| **SPEC PK** | Marqueur sur la crête la plus forte de la fenêtre |
| **NOISE FL** | Tracer la couche du plancher de bruit suivi |
| **TRAIL** | Tracer la couche de traînée |
| **PEAK HOLD** | Tracer la couche de maintien de crête |
| **LINE NF** | Plancher de bruit en trait (activé) ou rempli (désactivé) |
| **LINE TRL** | Traînée en trait ou remplie |
| **LINE PKH** | Maintien de crête en trait ou rempli |
| **FREQ TOUCH** | **Une touche sur le panadapteur peut-elle réaccorder le poste ?** Désactivé, le curseur suit toujours le doigt mais aucune commande CAT n'est envoyée. À couper si vous craignez une touche involontaire pendant un concours. |

| Commande | Plage | Notes |
|---|---|---|
| **BRIGHTNESS** | 20 – 255, pas de 5, affiché en % | Rétroéclairage. Le minimum n'est délibérément **pas** 0 : un rétroéclairage que l'on peut éteindre complètement masquerait le menu même dont on aurait besoin pour le rallumer. |

**Action :** RESET DEFAULTS.

---

### 8.2 Onglet SPECTRUM

![Onglet SPECTRUM](images/menu-spectrum.jpg)

*Figure 8.2 — Onglet SPECTRUM.*

La fenêtre d'affichage et la mise en couleur de la cascade.

| Ligne | Plage | Unité | Rôle |
|---|---|---|---|
| **SPECTRUM FLOOR** | −90 … −20, pas 1 | dB | Limite basse de la fenêtre d'affichage auto-adaptative. Les signaux en dessous sont au bas de l'écran. |
| **SPECTRUM CEILING** | −20 … +40, pas 1 | dB | Limite haute. Avec FLOOR, elles bornent le déplacement autorisé de la plage automatique — ce ne sont **pas** la fenêtre courante, qui est recalculée à chaque image. |
| **WATERFALL SPEED** | 1 … 5, pas 1 | — | Vitesse de défilement. 1 est la plus lente et montre le plus long historique. |
| **PASSBAND ALPHA** | 0 … 255, pas 4 | — | Opacité du masque translucide de bande passante sur le spectre. À 0 il est invisible — si vous ne voyez pas votre bande passante, regardez ici d'abord. |
| **TUNE SNAP** | 100 … 1000, pas 100 | Hz | Arrondi appliqué à l'accord tactile. 100 pour l'usage courant, 500 ou 1000 pour tomber juste sur une fréquence de réseau. |
| **WF BLACK LEVEL** | 0 … 30, pas 1 | dB | À quelle hauteur au-dessus du plancher propre à chaque bin la cascade commence à colorer. Augmentez pour assombrir une bande bruyante. |
| **WF CONTRAST** | 10 … 80, pas 2 | dB | L'étendue en dB sur laquelle la palette est étirée. Petit = contrasté et vite saturé ; grand = doux. |
| **WF FLOOR BLEND** | 0,0 … 1,0, pas 0,1 | — | Part du plancher **par bin** utilisée par rapport à un plancher global unique. **0 ramène le schéma à un plancher global unique** — à essayer en premier si la cascade a un jour l'air fausse. |

| Cellule | Rôle |
|---|---|
| **AREA FILL** | Remplit la surface sous la trace du spectre |

**Action :** RESET DEFAULTS.

---

### 8.3 Onglet TRACE

![Onglet TRACE](images/menu-trace.jpg)

*Figure 8.3 — Onglet TRACE.*

La forme de la trace elle-même. Ces valeurs se règlent sur l'air face à de vraies
conditions de bande, ce qui explique qu'elles soient mémorisées — et pourquoi le bouton de
réinitialisation compte ici.

| Ligne | Plage | Unité | Rôle |
|---|---|---|---|
| **FLOOR OFFSET** | −30 … +40, pas 1 | dB | Décale toute la trace verticalement. **Celle à manier avec précaution** : une valeur importante pousse la trace hors de la fenêtre visible, et une fois enregistrée, un cycle d'alimentation ne l'efface plus. |
| **PEAK HEADROOM** | 0 … 30, pas 1 | dB | Espace laissé au-dessus de la crête la plus forte. |
| **EDGE KNEE** | 0,10 … 1,00, pas 0,05 | — | Où commence la correction de bord, en fraction de demi-largeur. *Grisée tant que EDGE CORR est désactivé.* |
| **EDGE OUTER** | 0,0 … 30,0, pas 0,5 | dB | Correction appliquée aux bords extrêmes. *Grisée tant que EDGE CORR est désactivé.* |
| **TRAIL DECAY** | 0,05 … 3,00, pas 0,05 | dB / image | Vitesse d'effacement de la traînée. Petit = longue queue. |
| **PEAK DECAY** | 0,01 … 3,00, pas 0,05 | dB / image | Vitesse de retombée du maintien de crête. Des valeurs très faibles en font pratiquement un maximum permanent. |
| **NOISE FL ALPHA** | 0,002 … 0,200, pas 0,005 | — | Vitesse à laquelle le plancher de bruit suivi épouse la bande. Petit = lent et stable ; grand = suit le QSB et peut avaler une porteuse faible et stable. |

| Cellule | Rôle |
|---|---|
| **EDGE CORR** | Active la correction de bord. Les deux lignes EDGE ne sont actives que si elle est activée — grisées sinon, pour qu'elles ne passent pas pour des commandes qui ne font silencieusement rien. |

**Action :** RESET DEFAULTS.

---

### 8.4 Onglet DSP

![Onglet DSP](images/menu-dsp.jpg)

*Figure 8.4 — Onglet DSP. Photographiée avant l'ajout des lignes IQ BALANCE.*

La chaîne de lissage du spectre, en amont de tout ce que règlent les autres onglets
d'affichage — plus la vitesse CAT, qui devait se trouver quelque part d'accessible sans
recompilation.

| Ligne | Plage | Rôle |
|---|---|---|
| **SPECTRUM EMA @48k** | 0,004 … 0,300, pas 0,002 | Lissage exponentiel des amplitudes du spectre, exprimé à 48 kHz. Petit = lisse et lent ; grand = nerveux et réactif. L'onglet affiche en dessous l'**alpha effectif à la fréquence d'échantillonnage courante**, car la valeur se met à l'échelle avec la fréquence et vous ne devriez pas avoir à faire ce calcul. |
| **BIN AVG RADIUS** | 0 … 4, pas 1 | Moyenne chaque bin avec ses voisins. 0 = désactivé. Échange de la résolution en fréquence contre une trace d'aspect plus lisse. |
| **FFT WINDOW** | liste | La fenêtre de FFT d'**affichage** : BLACKMAN-H · NUTTALL · BLACK-NUT · HANN · HAMMING · NONE. Classées d'abord par faibles lobes secondaires, donc descendre la liste échange de la dynamique contre de la résolution. **Les niveaux ne bougent pas d'un choix à l'autre** — chaque fenêtre est normalisée en gain sur la référence Blackman-Harris, donc la calibration du S-mètre reste valable quel que soit le choix. C'est purement de l'affichage ; la chaîne de réception n'a pas de FFT. |
| **CAT BAUD** | 9600 / 38400 / 115200 | La vitesse série CAT. Appliquée immédiatement à l'UART actif, et restaurée avant l'ouverture du port au démarrage. À régler comme le poste. |
| **IQ BALANCE** | OFF / AUTO / MAN | Corrige l'écart de gain et de phase entre les voies I et Q, qui sinon fait apparaître une copie miroir de chaque signal de l'autre côté du centre. **AUTO** (par défaut) le mesure en continu à partir de ce qui se trouve sur la bande et se stabilise en quelques secondes ; la mesure est figée pendant l'émission. **MAN** applique les deux lignes ci-dessous telles quelles. **OFF** n'applique rien. Corrige le spectre et l'audio à l'identique. |
| **IQ GAIN** | −3,00 … +3,00 dB, pas 0,05 | Le déséquilibre de gain **de la carte** (Q par rapport à I), et non la correction. Mis à jour en direct en AUTO — l'onglet se rafraîchit seul. Le modifier en AUTO bascule en MAN, à partir de la valeur mesurée. |
| **IQ PHASE** | −10,0 … +10,0 deg, pas 0,1 | L'erreur de phase de la carte par rapport à 90°. Même comportement que IQ GAIN. |

**Lire les lignes IQ.** Une tête de réception bien appairée affiche quelques dixièmes de dB et
moins d'un degré — environ 40 dB de réjection du miroir avant toute correction. Pour annuler un
miroir à la main : placez une porteuse stable à quelques kHz du centre, passez en MAN, réglez
IQ PHASE pour le miroir le plus faible, puis IQ GAIN. Le mode et les deux valeurs sont conservés
à l'extinction, donc AUTO repart du dernier équilibre connu et non de zéro.

**Action :** RESET DEFAULTS (remet aussi IQ BALANCE sur AUTO, 0 dB, 0°).

---

### 8.5 Onglet AUDIO

![Onglet AUDIO](images/menu-audio.jpg)

*Figure 8.5 — Onglet AUDIO.*

La chaîne audio de réception — celle que vous écoutez réellement.

| Ligne | Plage | Unité | Rôle |
|---|---|---|---|
| **NB THRESHOLD** | 2,0 … 10,0, pas 0,5 | — | Seuil de déclenchement du silencieux d'impulsions. **Le réglage le plus lourd de conséquences de cet onglet**, car ses deux modes de défaillance sont discrets : trop haut, le bouton NB ne fait rien du tout ; trop bas, il rabote le sommet des crêtes de voix et de la manipulation CW, ce qui s'entend comme une légère rugosité et non comme une panne. La bonne valeur dépend du bruit de votre QTH. |
| **AGC TARGET** | 0,05 … 0,25, pas 0,05 | — | Le niveau de sortie *moyen* visé par la CAG ; 0,20 par défaut. Les crêtes de parole montent environ trois fois plus haut, d'où la limite à 0,25 — voir la note sur la marge ci-dessous. |
| **AGC HANG** | 0 … 1000, pas 50 | ms | Durée pendant laquelle le gain est maintenu après une crête, avant de commencer à remonter. Long convient à la voix BLU ; court convient à la CW. |
| **AGC RELEASE** | 100 … 2000, pas 100 | ms | Vitesse de remontée du gain une fois le hang écoulé. |
| **AGC MAX GAIN** | 0 … +45, pas 3 | dB | Plafond du gain de la CAG ; **+34 dB** par défaut. En pratique, il décide **du niveau du bruit de bande dans les pauses** par rapport à la voix : à +34 dB le bruit des pauses reste environ 12 dB sous la voix, à +45 dB il remonte presque au niveau de la parole. Chaque pas s'entend ; au-delà d'environ +42 dB le plafond ne joue plus sur un bruit de bande habituel. |
| **DNR LEVEL** | 0,005 … 0,200, pas 0,005 | — | Pas d'adaptation NLMS du réducteur de bruit. Plus haut suit plus vite mais rend l'audio rugueux ; sur la voix, un son **« aquatique »** est le signe habituel qu'il faut redescendre. |
| **NOTCH LEVEL** | 0,005 … 0,200, pas 0,005 | — | Pas d'adaptation NLMS du notch automatique. Plus haut accroche une porteuse plus vite mais risque davantage de mordre sur la parole. |
| **FDNR STRENGTH** | 0,00 … 1,00, pas 0,05 | — | Agressivité de la suppression spectrale sur chaque bin. |
| **FDNR FLOOR** | −24 … −3, pas 1 | dB | Jusqu'où un bin peut descendre. **Si la FDNR gargouille au lieu de donner un bruit calme, montez cette valeur (vers −3), ne la descendez pas.** Faire taire complètement des bins est ce qui crée le bruit musical ; laisser un lit audible est ce qui rend le résidu naturel. |
| **FDNR ALGO** | MIN-STAT / ROMANIN / **HYBRID** | — | Quel moteur de réduction de bruit le bouton FDNR met en marche. **HYBRID est le réglage par défaut et il n'y a aucune raison de s'en écarter** — les deux autres sont conservés comme points de comparaison, pour entendre soi-même ce qu'apporte chaque moitié de l'algorithme. Voir §8.5.1. Non mémorisé : revient à HYBRID à chaque mise sous tension. |
| **FM SQUELCH** | 0 … 100, pas 1 | — | FM uniquement, ignoré partout ailleurs. **0 = ouvert** (le squelch ne fait rien), 1 = le plus lâche, 100 = le plus serré. Il se déclenche sur le souffle *au-dessus* de la bande audio : le nombre est donc une **tolérance au bruit**, pas un niveau de signal — montez-le jusqu'à ce qu'un canal vide reste silencieux, et arrêtez-vous là. Référence mesurée : environ 0,17 sur un canal vide, environ 0,008 en réception bien quiétée. |
| **VOLUME** | 0,05 … 1,00, pas 0,05 | — | Niveau de sortie après la CAG — uniquement un atténuateur, il n'ajoute jamais de gain. Même valeur que VOL+ / VOL− de la rangée basse. |

> **Marge : gardez AGC TARGET × VOLUME à 0,25 au plus.** La CAG maintient le niveau *moyen* ; les
> crêtes de parole montent environ trois fois plus haut, et au-delà d'un produit d'environ 0,28
> elles atteignent le plafond de sortie. Cela s'entend comme une distorsion qui ressemble à de la
> **réverbération ou un écho**, surtout dans les pauses — et aucun réglage de CAG, de bruit ou de
> filtre n'y remédie. Les plages ci-dessus rendent ce seuil impossible à franchir (0,25 × 1,00),
> et les réglages mémorisés par un ancien firmware sont ramenés dans la plage au démarrage.
> L'indication `AF` (§6.8) montre où vous en êtes : environ −6 dB sur la voix est sain,
> −2 … 0 dB est trop fort.
>
> **La bande passante FILTER n'est délibérément pas dans cet onglet.** C'est une liste par
> mode, déjà affichée comme libellé du bouton FILTER ; une ligne numérique ici ne pourrait
> offrir qu'un simple indice, ce qui serait moins bien que ce que fait déjà le bouton.
>
> **La mise en marche de FDNR est le bouton FDNR**, pas une ligne — c'est une commande à
> l'échelle du QSO, et les lignes coûtent cher sur cet onglet puisque la hauteur de chacune
> est dérivée de leur nombre.

**Action :** RESET DEFAULTS.

#### 8.5.1 Les trois moteurs FDNR

La réduction de bruit spectrale prend deux décisions que l'on regroupe d'ordinaire, mais
qui sont en réalité indépendantes : **comment elle estime le bruit**, et **comment elle
traite les gains par bin qui en résultent**. WMSDR permet d'entendre chacune séparément,
car la combinaison la plus efficace n'est celle à laquelle aucun des deux algorithmes
d'origine n'était parvenu seul.

| Choix | Estimation du bruit | Traitement des gains | Rendu à l'oreille |
|---|---|---|---|
| **MIN-STAT** | Minimum glissant de la puissance de chaque bin sur environ 1,5 s. Les signaux vont et viennent ; le bruit est ce qui est toujours là, donc le minimum, c'est le bruit. | Lissage temporel uniquement. | La suppression la plus forte — voix la plus claire, résidu le plus bas. Mais dans les silences entre les mots, ça gargouille : **une faible « station de radio qui joue en arrière-plan ».** |
| **ROMANIN** | Par bin, la probabilité que la parole soit présente ; l'estimation du bruit n'évolue qu'à hauteur de la probabilité qu'elle soit *absente*. Réagit en quelques dizaines de millisecondes au lieu d'une seconde et demie. | Moyenné sur les **bins voisins**, sur une largeur qui augmente à mesure que le filtre coupe fort. | Aucun gargouillis, mais le bruit de fond reste nettement plus présent. |
| **HYBRID** *(par défaut)* | Celle de MIN-STAT. | Celui de ROMANIN. | La profondeur de coupe de MIN-STAT sans son bruit musical : le bruit descend et y reste, que quelqu'un parle ou non. |

Pourquoi le gargouillis apparaît, et pourquoi HYBRID le supprime : le bruit dans un seul bin
de FFT fluctue de plusieurs décibels d'une trame à l'autre, même lorsque le bruit lui-même
est parfaitement stable. Fixez un seuil : la plupart des bins passent dessous, tandis que
l'un ou l'autre le dépasse par hasard et survit à plein gain. **Ces survivants isolés qui
sonnent chacun pour soi, c'est précisément *cela*, le bruit musical.** Moyenner le gain de
chaque bin avec celui de ses voisins ne peut pas laisser un bin seul debout. Le lissage
temporel — tout ce que fait MIN-STAT — n'y peut rien : un bin qui survit à chaque trame est
parfaitement stable dans le temps, et reste un sifflement.

Le moyennage n'est appliqué que lorsque le filtre coupe réellement fort ; l'audio sur une
bande calme n'est donc jamais brouillé inutilement.

> **Ignorés en CW**, tous les trois. La FDNR ajoute environ 21 ms de retard et adoucit les
> fronts de manipulation — c'est exactement ce à partir de quoi le décodeur CW mesure les
> traits et les espaces.

---

### 8.6 Onglet EQ

![Onglet EQ](images/menu-eq.jpg)

*Figure 8.6 — Onglet EQ.*

Quatre bandes en cloche sur l'audio démodulé.

| Ligne | Plage | À quoi ça sert |
|---|---|---|
| **400 Hz** | −12 … +12 dB, pas 1 | Grave / chaleur. À réduire pour combattre le ronflement et le bruit BF. |
| **900 Hz** | −12 … +12 dB, pas 1 | Corps de la voix. |
| **1600 Hz** | −12 … +12 dB, pas 1 | Présence. À monter pour l'intelligibilité d'un signal faible. |
| **2400 Hz** | −12 … +12 dB, pas 1 | Aigus. À monter pour la netteté, à baisser pour dompter le souffle. |

Les fréquences centrales sont fixées à 400 / 900 / 1600 / 2400 Hz — choisies pour tomber
*à l'intérieur* d'une bande passante de communication plutôt que sur la grille d'octaves
ISO, afin que les quatre fassent réellement quelque chose en BLU.

À 0 dB une section en cloche est une identité exacte, donc **un égaliseur plat est
totalement transparent et est purement et simplement contourné** — le laisser à plat ne
coûte rien.

**Action :** RESET DEFAULTS.

---

### 8.7 Onglet CW

![Onglet CW](images/menu-cw.jpg)

*Figure 8.7 — Onglet CW.*

Réglage du décodeur. La ligne de texte du décodeur sur l'écran principal est votre retour,
donc chacun de ces réglages se juge sur l'air.

| Ligne | Plage | Unité | Rôle |
|---|---|---|---|
| **MAG FLOOR** | 0,1 … 20,0, pas 0,1 | — | L'amplitude absolue en dessous de laquelle le détecteur considère qu'il y a silence. Trop bas, le bruit se décode en caractères aléatoires ; trop haut, les signaux faibles sont ignorés. |
| **TONE** | 400 … 1000, pas 10 | Hz | **La tonalité CW de tout le récepteur**, pas seulement du décodeur. Elle recentre le filtre passe-bande CW, déplace l'aide à l'accord, déplace le marqueur du spectre et fixe le décalage VFO — le tout à partir de ce seul nombre. |
| **HYSTERESIS** | 0,02 … 0,45, pas 0,01 | — | L'écart symétrique entre les seuils de montée et de descente. Plus grand = plus immunisé au scintillement du bruit, mais écrête une manipulation très rapide. |
| **NOISE BLANKER** | 0 … 50, pas 1 | ms | Ignore les événements marque/espace plus courts que cela — du bruit impulsionnel, pas de la manipulation. |
| **MIN MARK** | 10 … 120, pas 5 | ms | Le plus court événement accepté comme un vrai point. Fixe la limite haute pratique en mots/minute. |

La largeur du passe-bande n'est **pas** une ligne ici : le pré-filtre du décodeur suit le
bouton **FILTER**, donc une commande séparée ne pourrait que le contredire. Choisissez la
bande passante CW en face avant et le décodeur suit.

Les valeurs par défaut sont celles calibrées sur l'air ; RESET DEFAULTS y ramène.

**Action :** RESET DEFAULTS.

---

### 8.8 Onglet CALIB

![Onglet CALIB](images/menu-calib.jpg)

*Figure 8.8 — Onglet CALIB.*

Calibration du S-mètre et des planchers d'affichage, par bande. Des commandes à plat, sans
assistant — chaque champ et les deux boutons de capture sont toujours actifs, ce qui permet
de refaire un seul point sans dérouler toute une procédure.

La ligne d'en-tête indique la **BANDE** courante, le **LEVEL** mesuré en direct (rafraîchi
quatre fois par seconde), et si cette bande est **CALIBRATED** ou **UNCALIBRATED**.

| Ligne | Plage | Rôle |
|---|---|---|
| **NOISE REF** | S1 … S15 | Le numéro S que doit indiquer le point de référence *calme*. |
| **SIGNAL REF** | S1 … S15 | Le numéro S que doit indiquer le point de référence *fort*. |
| **SPEC FLOOR** | −30 … +30 dB | Décalage du plancher de spectre par bande — aligne la trace entre des bandes de niveaux de bruit différents. |
| **WF FLOOR** | −30 … +30 dB | Décalage du plancher de la cascade par bande, même principe. |

Deux boutons de capture se trouvent sur les deux lignes de référence :

- **CAPTURE NOISE** — placez le poste sur une portion calme de la bande, réglez NOISE REF
  (typiquement S1), appuyez. Le niveau courant est enregistré comme ce numéro S.
- **CAPTURE SIGNAL** — trouvez un signal de force connue (ou utilisez un générateur
  calibré), réglez SIGNAL REF (typiquement S9), appuyez.

Deux points définissent toute l'échelle. Au-dessus de S9, l'échelle continue à raison de
S9+10 dB par unité.

**Action : SAVE CALIBRATION.** Les décalages de plancher sont modifiés sur place, donc
l'enregistrement est un geste délibéré — sinon un appui maintenu sur une ligne écrirait en
flash toutes les quelques millisecondes. Les deux boutons CAPTURE enregistrent d'eux-mêmes ;
les décalages, non.

---

### 8.9 Onglet COLORS

![Onglet COLORS](images/menu-colors.jpg)

*Figure 8.9 — Onglet COLORS.*

Chaque couleur de l'interface est un emplacement personnalisable. Il y en a plus qu'une
page ne peut en contenir, donc l'onglet montre un **groupe** à la fois et le sélecteur sur
la ligne d'action passe de l'un à l'autre.

Chaque ligne montre le nom de l'emplacement, `−` / `+` pour parcourir la palette, et un
**échantillon de couleur** — plusieurs entrées de la palette sont difficiles à distinguer
par leur nom sur une dalle sombre, et l'échantillon est le seul aperçu honnête pour un
emplacement dont la valeur est hors palette (affichée en hexadécimal). La palette boucle,
il n'y a donc pas de fin de course.

L'endroit où vous avez laissé le sélecteur de groupe n'est pas mémorisé ; les couleurs, si.

**Action : RESET COLORS.** Également une sécurité : la palette contient le NOIR, il est
donc tout à fait possible de régler un emplacement de texte sur sa propre couleur de fond
et de perdre la capacité de lire l'onglet dont on aurait besoin pour revenir en arrière.
Cette action réinitialise **tous** les emplacements, y compris ceux des autres groupes.

---

### 8.10 Onglet KNOBS

![Onglet KNOBS](images/menu-knobs.jpg)

*Figure 8.10 — Onglet KNOBS.*

Configure le boîtier optionnel de boutons **8Encoder** (§2). Sans le boîtier, l'onglet
fonctionne quand même et les réglages sont conservés, prêts pour le jour où il sera branché.

**Comportement des boutons**

- **Tourner** un bouton déplace un réglage d'**un pas de menu par cran** — exactement comme
  les boutons `−` / `+` de l'onglet de ce réglage, avec les mêmes limites. Si cet onglet est
  ouvert, la valeur à l'écran suit le bouton.
- Chaque bouton a jusqu'à **quatre pages**. Une page, c'est un réglage plus une **couleur de
  LED** ; le bouton agit toujours sur sa page courante, et sa LED indique laquelle. C'est ainsi
  qu'un seul bouton couvre tout un groupe — le bouton AGC par défaut est `AGC TARGET` (rouge),
  `AGC HANG` (vert) et `AGC RELEASE` (bleu).
- Le poussoir de chaque bouton distingue deux appuis : **court** (relâché avant 0,6 s) et
  **long** (déclenché à 0,6 s, bouton encore enfoncé — inutile de relâcher). La LED clignote en
  **blanc** sur un appui court et en **orange** sur un appui long, pour confirmer la prise en
  compte.
- L'**interrupteur** du boîtier active ou bloque les huit boutons. Bloqués, les LED des
  boutons s'éteignent et rotations et appuis sont ignorés — pratique pour qu'une main
  maladroite ne modifie rien en plein QSO.
- Au démarrage, chaque bouton repart toujours sur sa **page 1**.

**L'onglet, de haut en bas**

| Zone | Rôle |
|---|---|
| **1 … 8** | Choisit le bouton à configurer. La barre sous chaque numéro est la couleur actuelle de sa LED. **Tourner ou appuyer sur un bouton physique pendant que cet onglet est ouvert le sélectionne ici** — et ne fait rien d'autre, la configuration ne peut donc pas modifier une valeur par erreur. |
| **LED − / +** | Luminosité générale de toutes les LED du boîtier, **5 – 100 %**, pas de 5, **30 %** par défaut. Les pas suivent la perception de l'œil, pas une échelle linéaire, pour que le bas de plage reste utilisable la nuit. Le minimum n'est pas 0 : c'est l'interrupteur qui éteint les LED. |
| **PAGE 1 … 4** | `<` / `>` à gauche parcourent la liste des réglages : `- none -`, puis chaque réglage numérique des onglets SPECTRUM, TRACE, DSP, AUDIO, EQ et CW, affiché `ONGLET: RÉGLAGE`. Appui maintenu pour défiler. `<` / `>` à droite choisissent la couleur de LED : RED, ORANGE, YELLOW, GREEN, CYAN, BLUE, PURPLE, MAGENTA, WHITE. La page active est marquée `*`. |
| **SHORT < >** | Effet d'un appui court : **NONE**, **NEXT PAGE** (page suivante) ou **PREV PAGE** (page précédente). |
| **LONG < >** | Le même choix pour un appui long. |

Modifier une page en fait la page courante du bouton : la LED du boîtier montre donc la couleur
au fur et à mesure qu'on la choisit.

Une page affichant **`? MISSING`** désigne un réglage renommé ou supprimé par une mise à jour du
logiciel. Elle est inactive ; choisissez-lui un autre réglage.

Les réglages de CALIB ne peuvent pas être mis sur un bouton : ils ne sont conservés que par
**SAVE CALIBRATION**, et un bouton y modifierait une valeur qui disparaîtrait sans prévenir au
prochain démarrage.

**Configuration par défaut**

| Bouton | Pages | Appui court |
|---|---|---|
| 1 | VOLUME (vert) | — |
| 2 | AGC TARGET (rouge) · AGC HANG (vert) · AGC RELEASE (bleu) | NEXT PAGE |
| 3 | NB THRESHOLD (orange) | — |
| 4 | DNR LEVEL (magenta) | — |
| 5 | FDNR FLOOR (violet) | — |
| 6 | FM SQUELCH (jaune) | — |
| 7 | CW TONE (rouge) | — |
| 8 | WF CONTRAST (cyan) | — |

**Enregistrement.** L'affectation des boutons et la luminosité des LED sont enregistrées comme
tous les autres réglages — à la fermeture du menu ou par la sauvegarde différée de 3 secondes —
mais dans leurs propres enregistrements : une mise à jour qui modifie l'enregistrement principal
des réglages ne fait pas perdre la configuration des boutons (et inversement).

**Action : RESET KNOBS.** Rétablit la configuration par défaut ci-dessus. Rien d'autre n'est
touché — ni les valeurs commandées par les boutons, ni la luminosité des LED.

---

### 8.11 Onglet TX

![Onglet TX](images/menu-tx.jpg)

*Figure 8.11 — Onglet TX.*

Réglages d'émission. **Rien n'émet tant que TX ARM est sur OFF**, quelle que soit la demande.

| Ligne | Plage | Rôle |
|---|---|---|
| **TX ARM** | OFF / ARMED | Autorise l'émission. **Jamais enregistré — chaque démarrage se fait sur OFF.** Armé, le voyant TX a un contour orange (§6.2). Passer sur OFF coupe le PTT immédiatement et arrête une transmission CW. |
| **CW WPM** | 5–60 | Vitesse du manipulateur (PARIS). Réglable aussi depuis la page Transmit du navigateur. |
| **PTT TIMEOUT** | 5–120 s | Durée maximale d'un seul appui ; le PTT est alors forcé à l'arrêt (journalisé `PTT TIMEOUT`). En CW, chaque point et chaque trait est un appui distinct. |
| **CW WEIGHT** | 25–75 % | Longueur des signes par rapport à l'espace qui suit. 50 % est la norme ; plus lourd sonne plus plein et passe mieux sur un trajet faible ou avec du QSB, plus léger sonne plus net. La vitesse ne change pas. |
| **DAH RATIO** | 2,5–4,5 | Longueur du trait en points (norme 3,0). |
| **LETTER SPACE** | 3–6 unités | Espace entre lettres (norme 3). |
| **WORD SPACE** | 5–14 unités | Espace entre mots (norme 7). |
| **SIDETONE** | 0–10 | Niveau de la tonalité locale CW dans l'audio du WMSDR ; 0 = coupée. |
| **SIDETONE PITCH** | 300–1200 Hz | Hauteur de la tonalité locale — réglage propre, indépendant de TONE dans l'onglet CW. |

Tout sauf TX ARM est enregistré, dans un enregistrement à part (le RESET DEFAULTS du menu n'y
touche pas).

**PTT.** La ligne est la broche P0_0 de l'extenseur AW9523 sur la carte codec, reliée au PTT du
uSDX : **HAUTE au repos, BASSE en émission**. Il n'y a **pas d'entrée PTT manuel** — ne branchez
rien d'autre sur cette ligne, car la broche la pilote activement.

**Émission CW.** Le texte vient de la page Transmit du navigateur (§7.4). Le manipulateur cadence
chaque élément sur une minuterie à la microseconde et pilote directement le PTT : les longueurs
restent exactes jusqu'à au moins 45 WPM. Le uSDX, en mode CW, produit la porteuse ; aucun audio
n'intervient.

**Tonalité locale.** Audible seulement **en mode CW pendant que le manipulateur émet**. Elle rejoue
les vrais fronts du PTT, à l'échantillon près, avec un petit retard fixe, et **remplace l'audio de
réception** pendant la transmission ; l'audio de réception revient à la fin. La sortie audio du
WMSDR alimente aussi l'entrée micro du uSDX, mais en mode CW le uSDX ignore le micro.

> **Le VOX du uSDX doit être désactivé** dès que la sortie audio du WMSDR est reliée à son entrée
> micro : cette sortie transporte l'audio de réception entre les transmissions, et le VOX
> déclencherait le poste dessus.

### 8.12 Onglet STATUS

![Onglet STATUS](images/menu-status.jpg)

*Figure 8.12 — Onglet STATUS.*

Diagnostics en lecture seule, rafraîchis deux fois par seconde.

| Ligne | Ce qu'elle vous dit |
|---|---|
| **I/Q SENSE** | `STRAIGHT (I,Q)` (vert) ou `SWAPPED (Q,I)` (jaune), ou `checking...` pendant la détection. Les deux résultats conviennent — le WMSDR compense. C'est la confirmation la plus rapide que la paire I/Q arrive bien. |
| **MODE / BAND** | Ce que dit le poste, via CAT |
| **FREQUENCY** | L'OL courant, en kHz |
| **SAMPLE RATE** | La fréquence courante en Hz, et la largeur qui en résulte |
| **FREE MEM** | DRAM interne libre et PSRAM libre, en Ko |

**Action : RE-CHECK I/Q.** Réarme la détection automatique du sens. Elle s'exécute depuis
l'intérieur de la trame DSP, là où les tampons qu'elle inspecte sont réellement valides, et
le résultat apparaît donc un instant après l'appui. À utiliser après un changement de mode,
un changement de bande, ou chaque fois que la bande latérale semble inversée.

Rien de cet onglet n'est mémorisé — un état connu au démarrage y est plus utile qu'un état
mémorisé.

---

## 9. Procédures pratiques

### Premier passage sur l'air

1. `CAT BAUD` (onglet DSP) accordé au poste · vérifiez que STATUS montre la fréquence qui
   bouge.
2. **RE-CHECK I/Q** dans STATUS. Confirmez sur l'air : accordez le poste **vers le bas**,
   la porteuse doit se déplacer vers la **droite**.
3. Bouton `I/Q` → choisissez une largeur. 48k est le choix polyvalent.
4. Onglet CALIB → CAPTURE NOISE à S1, CAPTURE SIGNAL à S9, **SAVE CALIBRATION**.
5. `FILTER` → 2.4k pour la BLU.
6. `AGC` activée, `VOLUME` à votre goût — gardez l'indication `AF` vers −6 dB sur la voix.

### La bande est bruyante

1. `NB` activé. Surveillez le compteur `NB <n>` dans le journal DSP pendant le réglage de
   `NB THRESHOLD` — proche de zéro sur une bande propre, c'est ce qu'il faut viser.
2. Essayez `DNR` puis `FDNR` **l'un après l'autre** sur le même signal. Ce sont des outils
   différents.
3. Si la FDNR gargouille — une faible « station de radio » dans les silences entre les mots
   — vérifiez d'abord que `FDNR ALGO` est sur **HYBRID** ; MIN-STAT fait cela par
   construction. Si le gargouillis persiste en HYBRID, montez `FDNR FLOOR` vers −3.
4. Un hétérodyne fixe : `NOTCH`.

### La cascade a l'air fausse

1. Mettez `WF FLOOR BLEND` à **0**. Cela ramène le schéma par bin à un plancher global
   unique — à peu près l'aspect classique. Si l'image redevient correcte, c'est bien le
   suivi par bin qui était en cause, et c'est sur `WF BLACK LEVEL` / `WF CONTRAST` qu'il
   faut travailler.
2. Rappelez-vous que le menu suspend le rafraîchissement de la cascade — fermez-le et
   attendez que l'ancien historique défile avant de juger quoi que ce soit.

### L'affichage est devenu illisible

Ouvrez le menu, trouvez l'onglet propriétaire du réglage, appuyez sur **RESET DEFAULTS**
(ou **RESET COLORS** dans l'onglet COLORS). Un cycle d'alimentation ne vous sauvera *pas* —
ces valeurs sont mémorisées.

### Configurer un bouton

1. Vérifiez que l'interrupteur du boîtier est activé — les LED des boutons sont allumées.
2. Menu → **KNOBS**. Tournez le bouton physique voulu ; son numéro est sélectionné à l'écran.
3. **PAGE 1** → `<` / `>` jusqu'au réglage, puis choisissez une couleur.
4. Plusieurs réglages sur ce bouton ? Remplissez **PAGE 2** et suivantes, et réglez **SHORT**
   (ou **LONG**) sur **NEXT PAGE**.
5. Fermez le menu et essayez. Trop lumineux ou trop sombre ? **LED − / +** dans le même onglet.

### Recevoir une image SSTV

1. Accordez comme pour la phonie sur une fréquence SSTV (§7.3) — USB sur 20 m, LSB sur 40 / 80 m.
2. Bouton **SSTV** → MODE **AUTO**, HOLD désactivé.
3. **TUNE** : attendez que l'aiguille se pose sur 1900 — un en-tête arrive. Appuyez à nouveau sur
   TUNE pour voir l'image.
4. Rien après le 1900 ? Appuyez sur **START** au début de l'image.
5. **HOLD** pour garder une image terminée.

### Recevoir une carte météo

1. Accordez DWD en USB, 1,9 kHz sous la porteuse (7878,1 kHz pour 7880).
2. Bouton **SSTV** → MODE **WEFAX**.
3. Attendez `start signal detected` et le phasage — ou, en cours de carte, appuyez sur **START**
   et touchez la marge gauche de la carte.
4. Les lignes penchent ? D'abord **SLANT±**, puis touchez à nouveau la marge si nécessaire.

### Recevoir en RTTY

1. Accordez en USB pour que les tonalités tombent sur les repères **M** / **S** de TUNE (DWD :
   cadran 10099,1 kHz pour 10100,8, ou 11037,3 kHz).
2. Bouton **SSTV** → MODE **RTTY** ; touchez la ligne d'état pour le préréglage (DWD **50/450**,
   amateur **45.45/170**).
3. Du charabia sur un signal propre ? **REV**.
4. Laissez l'AFC se stabiliser ; **CLEAR** vide la console, **HOLD** la fige.

### Recevoir en FT8

1. Vérifiez que l'icône **E** est allumée — le FT8 a besoin de l'heure UTC du compagnon.
2. Accordez **14,074 MHz USB** (ou une autre fréquence FT8, §7.3).
3. Bouton **SSTV** → MODE **FT8**. Les décodages apparaissent après le premier créneau complet de
   15 s.
4. Pour lire tranquillement : **HOLD** sur le poste, ou ouvrez `xiao-dx.local/ft8.html` et
   utilisez **Pause**, les filtres et **Save log** (§7.4).

### Émettre en CW

1. uSDX sur charge fictive ou antenne, **mode CW**, **VOX désactivé**.
2. MENU → **TX** → **TX ARM → ARMED**. Réglez CW WPM et SIDETONE. Fermez le menu — le voyant TX a
   un contour orange.
3. Ouvrez `xiao-dx.local` → ☰ → **Transmit**. Le badge indique **ARMED**.
4. Tapez — chaque caractère part au fil de la frappe. Macros pour CQ / RST / 73. **STOP** ou
   **Échap** arrête immédiatement.
5. Terminé : **TX ARM → OFF**.

---

## 10. Annexes

### Annexe A — Schéma de la carte d'interface

*À insérer.*

Couvre : conditionnement et mise à niveau de l'entrée I/Q vers les entrées ligne du WM8731,
adaptation de niveau du CAT entre le poste et l'UART de l'ESP32-S3, l'étage de sortie
audio et l'agencement de l'alimentation.

### Annexe B — Boîtier proposé

![Face avant du boîtier](images/wmsdr-front-atu_02.jpg)

*Figure B.1 — Face avant du boîtier : dalle tactile, bloc de huit encodeurs avec voyants, bouton d'accord, afficheur de la boîte d'accord (unité séparée), connecteur rond multibroche en bas à gauche. Voir aussi les figures 1 et 2.*

Plans cotés : *à insérer.*

### Annexe C — Tableaux de référence

*À insérer.*

Suggestion : récapitulatif du brochage, valeurs par défaut de chaque ligne de menu, et une
carte de référence rapide d'une page pour les deux rangées de boutons.

---

*WMSDR — YO4WM. Présenté à Cavaillon, octobre 2026.*
