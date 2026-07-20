# Prompt de relance — développement de Box Dice Brawl

> Copie-colle le bloc ci-dessous comme **premier message** de la nouvelle conversation
> Claude Code (démarrée sur le dépôt cible, idéalement `pef0106/BoxDice`).
> Joins aussi les deux fichiers `.md` : `regles_jeu_box_dice.md` (les règles v0.1) et,
> si tu veux, `ARCHITECTURE.md`.

---

Je développe **Box Dice Brawl**, un jeu web en 3D style PS1, mobile-first, jouable en
solo contre une IA, sans backend. Les règles complètes (v0.1, avec les arbitrages de
design déjà tranchés) sont dans le fichier `regles_jeu_box_dice.md` que je te fournis —
lis-le, c'est la référence.

## Stack validée
- **TypeScript** (strict) + **Vite**
- **Three.js** (vanilla, pas de framework) pour la 3D
- **cannon-es** pour la physique des dés
- **UI en DOM/CSS** en overlay du canvas (pas d'UI dans le canvas)
- **Vitest** pour les tests du moteur de règles
- Hébergement **statique** (GitHub Pages / Netlify), aucun backend

## Principe d'architecture (à respecter absolument)
Le moteur de règles (`core/`) est **pur** : zéro dépendance, zéro DOM. Il expose un
réducteur `applyAction(state, action) -> { state, events }`. Le rendu 3D, l'UI et l'IA
sont de simples consommateurs d'événements / producteurs d'actions branchés dessus. Le
RNG est **injectable** (parties déterministes et testables). Objectif : pouvoir tester
100 % des règles sans navigateur, et pouvoir passer au multijoueur en ligne plus tard
sans réécrire le core.

Arborescence cible :
```
src/
├── core/    # règles pures : types, dice, deck, combat, round, match, engine
├── ai/      # adversaire (heuristique de relance)
├── render/  # Three.js : scène, dés 3D physiques, pipeline PS1, animations
├── ui/      # overlay DOM : HUD, écrans
└── game/    # FSM du déroulé + bootstrap
```

## Décisions de règles déjà tranchées (ne pas re-poser ces questions)
1. **Défense passive** : le deck protège automatiquement, « se défendre » = animation.
2. **Ordre des tours perdus** : ce sont les N premiers tours (N = dés relancés).
3. **Round de 5 tours complets** : le round va toujours jusqu'au tour 5.
4. **KO à 20 pts** : gagne le round ET la partie immédiatement.
5. **d10 = 1 à 10**.

## Rendu PS1 (pour le jalon dédié, pas maintenant)
Rendu en basse résolution (~240×320 portrait) upscalé en nearest-neighbor + vertex
snapping en shader + textures pixelisées. Le résultat du dé vient du **RNG du core**, la
physique cannon-es n'est que du feedback visuel (on oriente la face pour afficher la
valeur tirée).

## Ce que je veux maintenant : JALON 1
Implémente le **moteur de règles pur** (`src/core/`) en TypeScript, couvrant l'intégralité
des règles v0.1, avec des **tests Vitest** qui vérifient tous les exemples chiffrés du GDD
(construction du deck avec doublons, les 4 cas de scoring, effet des relances sur les
tours, KO, conditions de fin de partie, égalités). À la fin du jalon, je dois pouvoir
jouer une partie complète en mode texte/console pour valider les règles avant toute 3D.

Initialise aussi le projet (Vite + TS strict + Vitest) proprement. Ne fais **pas** encore
de 3D ni d'UI : on suit les jalons dans l'ordre (J1 règles → J2 boucle jouable → J3 scène
3D → J4 look PS1 → J5 finitions + PWA + déploiement).

Développe sur la branche `claude/game-web-3d-architecture-9v4e5y`, commit au fil de l'eau.
