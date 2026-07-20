# Box Dice Brawl — Architecture technique v0

> Base : `regles_jeu_box_dice.md` (GDD v0).
> Cible v0 : **web mobile-first, 3D style PS1, solo contre IA, sans backend.**

---

## 1. Décisions de cadrage (validées)

| Sujet | Décision v0 |
|---|---|
| Adversaire | IA locale (heuristique simple) — pas de multijoueur en ligne |
| Ambition 3D | Dés 3D physiques + ring low-poly, boxeurs statiques/simples, UI 2D par-dessus |
| Techno | Three.js vanilla + TypeScript, sans framework UI |
| Backend | Aucun — site 100 % statique |

Conséquence clé : **tout l'état du jeu vit dans le navigateur**. Le multijoueur en ligne
reste possible plus tard sans réécriture si l'on respecte la séparation décrite en §3
(le moteur de règles est pur et sérialisable).

---

## 2. Stack

| Rôle | Choix | Pourquoi |
|---|---|---|
| Langage | **TypeScript** (strict) | Les règles de scoring ont beaucoup de cas ; le typage les verrouille. |
| Build / dev | **Vite** | Dev server instantané, test sur téléphone via IP locale, build statique. |
| 3D | **Three.js** | Choix validé. Contrôle total du pipeline de rendu PS1. |
| Physique des dés | **cannon-es** (léger, ~150 Ko) | Simulation du lancer des d10. Voir §5.2 pour la stratégie résultat. |
| UI 2D | **DOM (HTML/CSS) en overlay** du canvas | Bien plus rapide à itérer qu'une UI in-canvas, accessible, responsive mobile natif. |
| État / flux de jeu | **FSM maison** (pas de lib) | Le déroulé (préparation → combat → scoring) est une machine à états simple et linéaire. |
| Tests | **Vitest** | Le moteur de règles est en fonctions pures → testable exhaustivement. |
| Hébergement | **GitHub Pages / Netlify** | Statique, gratuit, HTTPS (requis pour PWA plus tard). |
| Audio (optionnel v0) | Web Audio API directe | Quelques SFX suffisent. |

Aucune autre dépendance. Bundle cible < 500 Ko gzippé — important pour le mobile.

---

## 3. Architecture logicielle

Principe directeur : **le moteur de règles ne sait pas qu'il y a un écran.**

```
src/
├── core/          # Moteur de règles — PUR, zéro dépendance, zéro DOM
│   ├── types.ts       # GameState, RoundState, Deck, PlayerId, événements…
│   ├── dice.ts        # Lancers (RNG injectable), d10 1–10
│   ├── deck.ts        # Construction du deck (unicité des valeurs), relances
│   ├── combat.ts      # Résolution d'attaque (les 4 cas de scoring du §5.1 du GDD)
│   ├── round.ts       # Déroulé d'un round : 5 tours, tours perdus, KO à 20
│   ├── match.ts       # Partie : 5 rounds, 3 rounds gagnés / 50 pts / KO, égalités
│   └── engine.ts      # Réducteur : (state, action) → { state', events[] }
│
├── ai/            # Décisions de l'adversaire
│   └── opponent.ts    # v0 : heuristique de relance (voir §6)
│
├── render/        # Three.js — consomme les événements du core
│   ├── scene.ts       # Ring, éclairage, caméra
│   ├── dice3d.ts      # Meshes d10, lancer physique cannon-es, lecture de face
│   ├── ps1.ts         # Pipeline rétro : basse résolution, vertex snapping, dithering
│   └── animations.ts  # Réactions visuelles aux événements (coup réussi, raté, KO…)
│
├── ui/            # Overlay DOM
│   ├── hud.ts         # Scores, round/tour courant, decks visibles, tours perdus
│   ├── screens.ts     # Titre, préparation (choix relance), fin de round, fin de partie
│   └── style.css
│
└── game/          # Orchestration
    ├── fsm.ts         # Machine à états du déroulé (voir §4)
    └── main.ts        # Bootstrap : câble core ↔ ai ↔ render ↔ ui
```

### Le contrat central : actions entrantes, événements sortants

Le `core` expose un unique point d'entrée de type réducteur :

```ts
applyAction(state: GameState, action: Action): { state: GameState; events: GameEvent[] }
```

- **Actions** (intentions) : `ROLL_INITIAL`, `VALIDATE_DECK`, `REROLL { diceCount }`,
  `ATTACK`, `NEXT_TURN`, `NEXT_ROUND`…
- **Événements** (faits) : `DiceRolled`, `DeckBuilt { duplicatesDropped }`,
  `TurnLost`, `AttackResolved { roll, cases, points }`, `KO`, `RoundWon`, `MatchWon`…

`render` et `ui` sont de purs **consommateurs d'événements** ; l'IA est un **producteur
d'actions** comme le joueur. Bénéfices directs :

1. Les règles se testent sans navigateur (Vitest, 100 % des cas du GDD couverts).
2. Le §9 du GDD (« les animations doivent refléter chaque règle de scoring ») est
   trivial : chaque `AttackResolved` porte le cas exact (1 pt / valeur du dé / 0 pt).
3. Le RNG est injecté → parties rejouables, tests déterministes, replays futurs.
4. Passage au multi en ligne plus tard = déplacer `engine.ts` côté serveur, rien d'autre.

---

## 4. Machine à états (déroulé d'une partie)

```
TITLE
  └─ start ▶ MATCH_SETUP (tirage premier joueur du round 1)
       └─▶ ROUND_PREP        lancer 6d10 → deck → choix relance (0/1/2 dés)
             └─▶ COMBAT       boucle de 5 tours max
                   ├─ TURN_ATTACK (joueur actif lance le dé de combat)
                   ├─ TURN_SKIPPED (tour perdu pour cause de relance)
                   └─ ko? ─▶ ROUND_END
             └─▶ ROUND_END    scores du round, vainqueur
                   ├─ partie finie? (3 rounds / 50 pts / KO) ─▶ MATCH_END
                   └─ sinon ─▶ ROUND_PREP (round suivant)
MATCH_END
  └─ rejouer ▶ MATCH_SETUP
```

La FSM (`game/fsm.ts`) séquence les écrans et **attend la fin des animations** avant
d'envoyer l'action suivante au core — le core, lui, est synchrone et instantané.

---

## 5. Rendu 3D style PS1

### 5.1 Le look rétro — trois techniques, par ordre d'impact

1. **Basse résolution** : rendu dans un `WebGLRenderTarget` de ~320×240 (ratio adapté
   au portrait mobile, ex. 240×320), upscalé plein écran en **nearest-neighbor**.
   C'est ce qui donne 80 % du look, et ça rend le jeu *très* léger en GPU mobile.
2. **Vertex snapping** : dans le vertex shader (via `onBeforeCompile` ou
   `ShaderMaterial`), arrondir les positions projetées sur une grille → le
   « tremblement » de géométrie caractéristique de la PS1.
3. **Textures** : petites (64×64 max), `NearestFilter`, pas de mipmaps, palette
   réduite + dithering ordonné en post. L'affine texture mapping exact est coûteux à
   simuler ; le snapping + basse résolution en donne l'illusion suffisante en v0.

Éclairage : `MeshLambertMaterial` / éclairage par sommet, pas d'ombres portées
temps réel (un simple disque sombre sous les dés et les boxeurs).

### 5.2 Les dés : physique réelle, résultat contrôlé

Point de design important : **le résultat du dé est tiré par le core (RNG), pas par la
physique.** Déroulé d'un lancer :

1. Le core tire la valeur (ex. 7) et émet `DiceRolled`.
2. `dice3d.ts` simule un lancer cannon-es visuellement crédible (impulsion + torque
   aléatoires, rebonds sur le ring).
3. À l'arrêt du dé, on lit la face supérieure obtenue et on **réindexe les textures des
   faces** (ou on applique une rotation de correction) pour que la face visible affiche
   la valeur tirée.

Pourquoi : les règles restent 100 % dans le core (testables, équitables, rejouables), la
physique n'est que du feedback visuel — et un d10 (trapèzoèdre pentagonal) qui devrait
faire physiquement autorité pose des problèmes de faces ambiguës qu'on évite entièrement.

Modèles 3D : ring, coins, cordes et d10 modélisés en low-poly (< 500 tris par objet),
format glTF. Deux silhouettes de boxeurs statiques en fond suffisent pour la v0.

---

## 6. IA adverse (v0)

Une heuristique, pas du ML :

- **Décision de relance** : relancer coûte des tours (grosse pénalité). Règle simple :
  relancer 1 dé seulement si le deck a ≤ 3 valeurs **et** que l'espérance de gain de
  valeur unique est élevée (beaucoup de doublons tirés). Sinon valider.
- **Attaque** : aucun choix à faire (on lance le dé), donc rien à coder.
- Petits délais artificiels (500–900 ms) pour donner l'impression d'une réflexion.

L'interface `ai/opponent.ts` reçoit l'état visible et retourne une `Action` — le jour où
on veut une IA plus fine (espérance de score exacte, calculable en fermé), on ne touche
que ce fichier.

---

## 7. Mobile

- **Portrait first** : ring vu de ¾ haut, dés au premier plan, HUD en haut, boutons
  d'action dans la zone du pouce en bas.
- Interactions : uniquement des **taps** (lancer, choisir les dés à relancer, valider).
  La sélection des dés à relancer se fait en tapant les dés 3D (raycasting Three.js).
- Performance : la basse résolution du rendu PS1 est notre meilleure alliée — cible
  60 fps même sur un téléphone milieu de gamme.
- `viewport-fit=cover`, gestion des safe areas iOS, blocage du double-tap zoom.
- **PWA** (manifest + service worker) en fin de v0 : installable et jouable hors-ligne —
  gratuit à ajouter puisqu'il n'y a pas de backend.

---

## 8. Points de règles tranchés (validés avec le game designer)

Ambiguïtés du GDD résolues — le core les encode telles quelles :

1. **« Défendre » pendant un tour perdu (§4.2)** : la défense est **passive**. Le deck
   protège automatiquement comme à n'importe quel tour ; « se défendre » est une
   animation à l'écran, aucune règle supplémentaire.
2. **Tours perdus (§4.2 / §6)** : le round dure **toujours 5 tours complets**. Un
   joueur ayant relancé N dés passe ses attaques des N premiers tours ; l'adversaire
   attaque normalement aux 5 tours. La clause « si un joueur n'a plus de tours
   disponibles » du §6 est sans effet (2 relances max sur 5 tours).
3. **KO à 20 points (§6.1 / §7)** : le KO fait gagner le round **et la partie**
   immédiatement.
4. **Le d10 affiche 1–10** (pas 0–9) — confirmé par le §8, encodé tel quel.

---

## 9. Jalons proposés

| Jalon | Contenu | Critère de done |
|---|---|---|
| **J1 — Moteur de règles** | `core/` complet + tests Vitest de tous les exemples du GDD | Partie complète jouable en console/texte |
| **J2 — Boucle jouable** | FSM + UI DOM brute (pas de 3D) branchée sur le core + IA | Partie complète jouable au doigt sur téléphone |
| **J3 — Scène 3D** | Ring, dés physiques cannon-es, lecture/forçage de face | Lancer de dés crédible synchronisé au core |
| **J4 — Look PS1** | Basse résolution, vertex snapping, textures, dithering | Direction artistique validée sur mobile |
| **J5 — Finitions v0** | Animations d'attaque/KO, SFX, écrans titre/fin, PWA, déploiement | URL publique jouable |

L'ordre est volontaire : **le gameplay est validable dès J2**, avant tout
investissement 3D — si les règles ne sont pas fun, on l'apprend au plus tôt.

---

## 10. Ce qui est explicitement hors v0

- Multijoueur en ligne (préparé par l'architecture, non implémenté)
- Comptes, classement, persistance serveur
- Boxeurs animés, compétences par combattant, dés à effets (§10 du GDD)
- Mode entraînement, multi local
