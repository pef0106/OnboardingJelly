# Règles du jeu — **BOX DICE BRAWL** (Version 0.1 — Game Design Document)

> Cette version 0.1 reprend le GDD initial et **intègre les arbitrages** pris avec
> le game designer sur trois zones d'ombre du texte d'origine. Les décisions sont
> signalées par des blocs **« Décision v0 »** et récapitulées au §11.

## 1. Présentation générale
**Box Dice Brawl** est un jeu vidéo compétitif en duel (1 contre 1) basé sur un système de combat de boxe et de lancer de dés. Chaque partie se déroule en **5 rounds**, eux-mêmes composés de plusieurs phases : préparation du deck, combat en 5 tours, puis calcul des points.
Le jeu est jouable sur ordinateur ou sur téléphone.

Ce document constitue la **version 0 du game design**, servant de référence pour les équipes design, tech, UX, UI et narrative.

---

# 2. Structure d’une partie
Une partie se déroule en **5 rounds maximum**, chacun comprenant :
- **Phase 1 : Préparation (constitution du deck)**
- **Phase 2 : Combat (5 tours)**
- **Phase 3 : Calcul des points du round**

Un joueur peut aussi remporter la partie avant la fin :
- s’il atteint **20 points dans un round (KO)**;
- ou **50 points cumulés sur l’ensemble des rounds**.

---

# 3. Phase de préparation — Constitution du deck
Chaque joueur commence un round en lançant **6 dés à 10 faces**.

## 3.1 Définition du deck
Le deck représente les « coups disponibles » d’un joueur durant le round.

### Règle : conserver une seule occurrence de chaque valeur
Pour constituer le deck initial :
- Le joueur conserve **toutes les valeurs obtenues** lors du lancer.
- Lorsqu’une même valeur apparaît plusieurs fois, elle n’est conservée **qu’une seule fois** dans le deck.
- Le deck ne peut donc jamais contenir plusieurs occurrences d’une même valeur.

#### Exemple :
Lancers du joueur 1 :
- Dé 1 = 2
- Dé 2 = 9
- Dé 3 = 2
- Dé 4 = 4
- Dé 5 = 5
- Dé 6 = 1

Valeurs retenues : **2, 9, 4, 5, 1** → Deck = {1, 2, 4, 5, 9}
La valeur **2** apparaît deux fois, mais elle est **conservée une seule fois** dans le deck (il n’est pas possible d’avoir deux occurrences de la même valeur dans le deck).

---

## 3.2 Relance optionnelle
Une fois le deck initial formé, chaque joueur a le choix :
- **Valider son deck**, ou
- **Effectuer une relance** (1 ou 2 dés maximum)

### Effet des relances
- Les valeurs obtenues sur ces relances sont ajoutées au deck **uniquement si elles sont uniques**.
- **Chaque dé relancé retire 1 tour de combat** au joueur dans ce round.

#### Exemple :
Joueur 1 relance **1 dé** et obtient :
- Dé = 4

Mais le joueur a déjà un **4** dans son deck → **aucune nouvelle valeur ajoutée**.
Il perd tout de même **1 tour de combat**.

---

# 4. Phase de combat — 5 tours
Chaque round comporte **5 tours**. Dans chaque tour, les deux joueurs agissent.

## 4.1 Détermination du premier joueur
- **Round 1, Tour 1** : tirage aléatoire.
- Ensuite, l’ordre alterne automatiquement d’un tour à l’autre.

## 4.2 Tours perdus
Un joueur ayant relancé des dés :
- Perd **autant de tours que de dés relancés**.
- Lors d’un tour perdu, il ne peut **pas attaquer**, mais peut **se défendre**.

> **Décision v0 — Nature de la défense.**
> Dans les règles de scoring (§5.1), le défenseur ne réalise **aucune action** : c'est
> la présence ou l'absence de la valeur dans son deck qui le « défend » automatiquement.
> « Se défendre » lors d'un tour perdu est donc **passif** : le deck protège
> normalement, exactement comme à n'importe quel tour. Il n'y a **aucune mécanique de
> défense active** en v0 (pas de blocage, pas de réduction). À l'écran, la défense n'est
> qu'une **animation**.

> **Décision v0 — Ordre des tours perdus.**
> Les tours perdus sont les **N premiers tours** du round (N = nombre de dés relancés).
> Exemple : un joueur qui relance 2 dés ne peut pas attaquer aux tours 1 et 2, puis
> attaque normalement aux tours 3, 4 et 5.

---

# 5. Déroulement d’un tour
Chaque joueur lance un **dé de combat (d10)** lorsqu’il attaque.

## 5.1 Règles de score d’une attaque
Le score dépend de 2 conditions :
- Le joueur possède-t-il la valeur tirée dans son deck ?
- Son adversaire possède-t-il cette valeur dans son deck ?

### Cas 1 : l’attaquant possède la valeur
➡️ **L’attaque réussit**
- L’adversaire possède aussi la valeur → **1 point**
- L’adversaire ne possède pas la valeur → **points = valeur du dé**

### Cas 2 : l’attaquant ne possède pas la valeur
➡️ **L’attaque échoue**, sauf situation avantageuse
- L’adversaire possède cette valeur → **0 point**
- L’adversaire ne possède pas cette valeur → **1 point**

### Tableau récapitulatif du scoring

| Attaquant possède la valeur | Défenseur possède la valeur | Points marqués |
|:---:|:---:|:---:|
| Oui | Oui | **1** |
| Oui | Non | **valeur du dé** |
| Non | Oui | **0** |
| Non | Non | **1** |

---

## 5.2 Exemples détaillés
### Exemple 1 :
Deck Joueur 1 : {1, 2, 4, 5, 9}
Deck Joueur 2 : {1, 3, 4, 10}

**Tour 1 — Joueur 1 attaque :**
Lancer = 9
- J1 possède 9
- J2 ne le possède pas
→ J1 marque **9 points**

**Tour 1 — Joueur 2 attaque :**
Lancer = 1
- J2 possède 1
- J1 possède 1
→ J2 marque **1 point**

---

### Exemple 2 — cas avec relance :
Joueur 1 a relancé 1 dé → il perd **son premier tour d’attaque**.

**Tour 1 :**
- J2 attaque normalement
- J1 ne peut **que défendre** (passif — son deck le protège)

---

# 6. Fin du round — Calcul des points
À la fin des 5 tours :
- On additionne les points marqués.
- Le joueur ayant le total le plus élevé remporte le round.

> **Décision v0 — Durée du round.**
> Le round dure **toujours 5 tours complets**. La mention « ou si un joueur n'a plus de
> tours disponibles » du GDD d'origine est **sans effet** : avec 2 relances maximum sur
> 5 tours, un joueur ne peut jamais tomber à zéro tour. Le joueur pénalisé passe
> simplement ses N premières attaques pendant que l'adversaire attaque normalement aux
> 5 tours.

## 6.1 KO
Si un joueur atteint **20 points dans le round**, il remporte immédiatement le round par KO.

> **Décision v0 — Portée du KO.**
> Un KO (20 points dans un round) fait gagner **le round ET la partie** immédiatement.
> Le combat s'arrête sur-le-champ. (Lecture retenue du §7.)

---

# 7. Fin de la partie
La partie s’achève si :
- Un joueur remporte **3 rounds (sur 5)** → victoire aux points
- Ou atteint **50 points cumulés** sur l’ensemble des rounds → victoire technique
- Ou remporte un round par **KO (20 points)** → victoire immédiate (voir §6.1)

En cas d’égalité en nombre de rounds :
➡️ Le vainqueur est celui ayant **le plus de points cumulés**.

---

# 8. Paramètres du jeu (pour l’équipe tech & UI)
- Dés à 10 faces : valeurs **1 à 10**
- Lancer initial : **6 dés**
- Relances possibles : **1 ou 2 dés**, une seule fois
- Combat : **5 tours maximum par round**
- Tour = attaque du joueur A + attaque du joueur B (sauf tours perdus)
- Score maximum dans un round : **20 (KO)**
- Score maximum total : **50**

---

# 9. Points d’attention pour la production
- Le calcul des valeurs uniques doit être **lisible, clair et en temps réel**.
- La perte de tours doit être **visualisée clairement** pour comprendre l’impact des relances.
- Les animations d’attaque/défense doivent refléter les **issues de chaque règle de scoring**.
- L’ordre des tours et les alternances doivent être affichés pour suivre le combat.

---

# 10. Idées d’évolution (version future)
- Types de dés alternatifs (pouvoirs, effets spéciaux)
- Compétences uniques par combattant
- Mode entraînement
- Classification en ligne
- Version multi local
- Multijoueur en ligne temps réel

---

# 11. Récapitulatif des décisions v0 (issues des échanges design)

1. **Défense passive** — « Se défendre » lors d'un tour perdu ne déclenche aucune
   action : le deck protège automatiquement comme à tout autre tour. Défense = simple
   animation, aucune règle supplémentaire. (§4.2)
2. **Ordre des tours perdus** — Les tours perdus sont les N premiers tours du round
   (N = nombre de dés relancés). (§4.2)
3. **Round de 5 tours complets** — Le round va toujours jusqu'au tour 5 ; la clause
   « plus de tours disponibles » du GDD d'origine est sans effet. (§6)
4. **KO = victoire de la partie** — Atteindre 20 points dans un round gagne le round
   ET la partie, immédiatement. (§6.1 / §7)
5. **d10 = 1 à 10** — Les faces vont de 1 à 10 (pas 0 à 9). (§8)

---

# Fin du document V0.1
