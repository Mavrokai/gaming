# Design — HTML5 Gaming (ThreeJS Action Top-Down)

**Date :** 2026-07-27  
**Projet :** W-WEB-403-LIL-4-1-gaming-7  
**Sujet :** `html5_gaming.pdf` (HTML/CSS/JS · ThreeJS ou Phaser)

## 1. Vision

**Titre de travail :** *Shade Rift* (modifiable).

Jeu d’action **3D top-down** inspiré de League of Legends (caméra, sorts, HUD), en **PvE** : vagues d’ennemis + boss, pas de multi ni de lanes.

- **Techno :** ThreeJS (vanilla modules ES + Vite)
- **Thème :** dark / horror light, low-poly soigné
- **Ambition :** Standard+ (finissable, au-dessus du MVP)

## 2. Contraintes sujet (obligatoires)

- Règles compréhensibles et claires (écran Règles)
- Menu : lancer, redémarrer, options / touches
- Jouabilité sans bugs bloquants
- Gameplay agréable
- Aspect visuel finalisé
- Au minimum : **3 niveaux**, **boss**, **élimination d’ennemis**, **tirs / sorts**
- Code entièrement maîtrisé / explicable (pas de générateur opaque)

## 3. Décisions produit

| Choix | Décision |
|---|---|
| Librairie | ThreeJS |
| Genre | Action top-down (esprit LoL, PvE) |
| Thème | Dark / horror light |
| Scope | Standard+ |
| Déplacement | Clic au sol (click-to-move) |
| Sorts | **A Z E R** (AZERTY), direction = curseur |
| HUD | Barre basse type LoL |

## 4. Architecture

```
src/
  main.js
  core/           # Game, Input, SceneManager, Audio, Settings
  scenes/         # Boot, Menu, Rules, HeroSelect, Game, Pause, GameOver, Victory
  entities/       # Hero, Enemy, Boss, Projectile, Pickup, Arena
  systems/        # ClickMove, Combat, WaveSpawner, AI, Collision, CameraFollow
  ui/             # HUD LoL, menus DOM overlay
  data/           # heroes.js, enemies.js, levels.js
assets/
  hud/            # portraits + icônes de sorts
  audio/
  models/         # si besoin (sinon géométrie procédurale)
```

**Boucle :** `SceneManager` empile/remplace des scènes. La scène `Game` exécute update systems → rendu ThreeJS. L’UI (menus + HUD) est en **HTML/CSS** par-dessus le canvas.

**Stack :** Vite + ThreeJS (npm). Livraison GitHub des sources, pas de binaires inutiles. Bonus éventuels dans `bonus/`.

## 5. Flux de scènes

```
Boot → Menu → HeroSelect → Game(Level 1..3)
                ↳ Rules
                ↳ Options
Game ⇄ Pause (Reprendre / Options / Recommencer / Menu)
Game → LevelClear → Game(next) | Victory (après niveau 3)
Game → GameOver → Recommencer | Menu
```

## 6. Gameplay

### 6.1 Boucle de niveau

1. Spawn dans une arène fermée
2. Vagues d’ennemis (WaveSpawner)
3. Clear de vague → vague suivante
4. Boss de fin de niveau (phases)
5. Victoire de niveau → niveau suivant (×3)

### 6.2 Héros (×3 au select)

| Héros | Rôle | Particularité |
|---|---|---|
| 1 | Bruiser | Plus de HP, sorts mêlée / zone courte |
| 2 | Mage | Fragile, dégâts distance / AoE |
| 3 | Assassin | Rapide, burst, moins tanky |

Chacun a : portrait HUD, stats de base, 4 sorts (A Z E R) avec icônes dédiées.

### 6.3 Sorts

- **A** — skillshot / auto renforcée (faible CD, faible mana)
- **Z** — contrôle (slow / root court)
- **E** — mobilité (dash) ; le bruiser peut avoir un shield à la place (défini par héros dans `heroes.js`)
- **R** — ultime (gros CD, gros impact, contour doré HUD)
- Direction de cast = **position curseur** au moment de la touche
- Coûts mana + cooldowns ; impossible de cast si CD / mana insuffisant

### 6.4 Ennemis & boss

- **2–3 types** : melee rusher, ranged, tanky
- **IA :** idle → chase joueur → attack à portée ; léger évitement entre eux
- **Boss :** 1 par niveau, phases sur seuils HP, AoE télégraphiée (indicateur au sol)

### 6.5 Économie légère

- Kills → **or** + **score**
- Pickups : vie, boost vitesse (durée limitée)
- Slots inventaire HUD (6) pour afficher consommables ramassés
- **D / F** : usages simplifiés (ex. potion / dash utilitaire), pas summoners LoL complets

### 6.6 Score

Kills + bonus clear vague + bonus boss + composante temps (à calibrer).

## 7. Contrôles

| Input | Action |
|---|---|
| Clic gauche (sol) | Déplacer le héros vers le point |
| A Z E R | Lancer les sorts vers le curseur |
| D / F | Utilitaires HUD |
| Échap | Pause |
| Options | Remapping des sorts + volumes |

Click-to-move : raycast sur le plan du sol → cible ; déplacement à vitesse constante ; stop à proximité (~0.3 unité). Obstacles arène = AABB simples (pas de navmesh).

## 8. HUD (validé)

Barre basse inspirée LoL :

- Mini-stats (AD / AP / Armor / MS)
- Portrait circulaire + niveau
- Icônes sorts **A Z E R** (images) + coût mana + état cooldown
- Barres HP (vert) et Mana (bleu)
- Grille inventaire 6 slots + or
- Score / vague en overlay **haut** (séparé de la barre)

Assets de référence mockup : `assets/hud/` (`hud-portrait.png`, `spell-a|z|e|r.png`). En jeu : **1 set d’images par héros**.

## 9. Menus & settings

- **Menu :** Jouer, Options, Règles
- **Options (localStorage) :** volume musique/SFX, rebind A Z E R, option qualité (ombres)
- **Pause :** Reprendre, Options, Recommencer niveau, Quitter au menu
- **Règles :** texte clair (objectif, contrôles, victoire/défaite)

## 10. Rendu & audio

- Caméra top-down légèrement inclinée, follow héros
- Fog, lumières faibles, silhouettes, arènes distinctes par niveau
- Géométrie low-poly (procédurale ou modèles simples)
- BGM menu/combat + SFX cast/hit/pickup/death
- Pooling projectiles / ennemis pour la perf

## 11. Données

Fichiers data purs (pas de logique ThreeJS) :

- `heroes.js` — stats, sorts, chemins assets
- `enemies.js` — types, HP, dégâts, vitesse, comportement
- `levels.js` — arène, composition des vagues, boss, seuils

## 12. Hors scope (volontaire)

- Multijoueur / matchmaking
- Jungle, shop d’items complet type LoL
- 5v5, skill points / arbre de talents
- Navmesh complexe / open world
- Clone pixel-perfect de l’UI Riot

## 13. Critères de done

- [ ] Menu + Règles + Options (rebind + volumes persistés)
- [ ] Select 3 héros jouables
- [ ] Click-to-move + sorts A Z E R fonctionnels
- [ ] HUD LoL (images portrait/sorts, CD, HP/Mana)
- [ ] 3 niveaux avec vagues + boss à phases
- [ ] Pickups + score + or
- [ ] Pause / Game Over / Victory
- [ ] Ambiance dark cohérente, pas de bug bloquant
- [ ] README (lancer le jeu, contrôles, règles)

## 14. Approche technique retenue

**Vanilla ThreeJS + modules ES + Vite** — contrôle total, code explicable à l’oral, collisions/AABB custom suffisantes pour arènes fermées. Pas de moteur physique lourd (Cannon/Rapier) sauf besoin avéré plus tard.
