# Éclosion : documentation complète

Ce document décrit toute l'application, pour qu'une autre personne ou un autre assistant puisse l'améliorer sans rien casser.

- **App en ligne** : https://sfnt656f64-tech.github.io/eclosion-/
- **Code** : https://github.com/sfnt656f64-tech/eclosion-
- **Fichier principal** : `index.html` (tout le code de l'app est dedans)

---

## 1. Le concept

Application personnelle de progression inspirée des RPG, pour atteindre des objectifs de vie réels.

- Chaque **objectif** (ex. « Trouver un CDI en France ») est représenté par une **créature** mignonne et originale qui naît d'un œuf et évolue.
- Chaque objectif contient des **compétences** qui montent en niveau, des **actions** récurrentes rapides à valider (XP) et des **étapes** concrètes (beaucoup d'XP).
- L'**étape finale** fait passer la créature à sa forme **légendaire**.
- L'utilisateur crée, modifie et supprime lui-même objectifs, compétences, actions et étapes.

**Priorités de l'utilisateur, par ordre d'importance :**
1. Beau et personnel : univers coloré et chaleureux, créatures attachantes et animées.
2. Très stimulant, comme un bon jeu mobile : confettis, sons, XP qui s'envole, critiques, évolutions spectaculaires, séries, quêtes, succès.
3. Rapide au quotidien : valider une action en 1 ou 2 touches.
4. Les données ne doivent jamais se perdre.
5. Utilisable sur téléphone (installée comme une app) et sur PC.

L'utilisateur **ne sait pas coder** : il faut tout lui expliquer simplement, en français, et lui dire précisément comment tester.

---

## 2. Technique

| Élément | Choix |
|---|---|
| Langages | HTML, CSS et JavaScript « vanilla », sans framework ni étape de compilation |
| Structure | Une seule page `index.html` (≈ 178 Ko) qui contient le style, le code et les dessins |
| Dessins | Créatures et icônes dessinées en SVG, générées par le code |
| Effets | Particules sur un `<canvas>` plein écran (`#fx`) + lucioles de fond (`#amb`) |
| Sons et musique | Générés en direct avec la Web Audio API, sans aucun fichier audio |
| Polices | Google Fonts : « Baloo 2 » (titres) et « Nunito » (texte) |
| Sauvegarde | `localStorage`, clé `eclosion.save.v1` (un seul objet JSON) |
| Installation | PWA : `manifest.webmanifest`, `sw.js` (service worker), icônes PNG |
| Hors connexion | Le service worker garde l'app en cache : réseau d'abord, cache si hors ligne |
| Hébergement | GitHub Pages, branche `main`, dossier racine |

### Fichiers du dépôt

- `index.html` : l'app complète.
- `manifest.webmanifest` : nom, couleurs et icônes de l'app installée.
- `sw.js` : service worker. **Changer la constante `CACHE` (ex. `eclosion-v9`) à chaque mise à jour** pour forcer le rafraîchissement.
- `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`, `apple-touch-icon.png` : icônes (un œuf à taches colorées).
- `.nojekyll` : empêche GitHub Pages de transformer les fichiers.

### Deux versions de l'app

1. **Version installable (ce dépôt)** : sauvegarde locale uniquement, pas de synchronisation entre appareils. Transfert manuel par Exporter / Importer.
2. **Version « artifact » Claude** (claude.ai) : même code, mais elle synchronise les données via la base de données de l'artifact (`window.claude.use('db')`, document `data/users/<id>/save`). Dans la version installable, `window.claude` n'existe pas : le code le détecte et passe en mode local.

### Organisation du code dans `index.html`

Le script est dans une seule fonction anonyme `(function(){ ... })()`, découpée en sections commentées `/* ===== ... ===== */` :

1. **Réglages du jeu** : `DIFF`, `MSIZE`, `STAGES`, `SPECIES`, `COLORS`, `ACH` (succès), `QUESTS`.
2. **Petits outils** : `esc()` (échappement HTML, **à utiliser pour tout texte saisi par l'utilisateur**), `uid()`, dates (`dayKey`, `daysBetween`, `weekKey`), `seeded()` (aléatoire reproductible), `mix()` (mélange de couleurs).
3. **Icônes** (`IC`) et coffre (`CHEST`).
4. **Accessoires** (`ACC`, `accSVG`) et **dessin des créatures** (`eggSVG`, `creatureSVG`).
5. **Données et sauvegarde** : `freshState()`, `migrate()` (complète les anciennes sauvegardes), `commit()` (enregistre après chaque changement), synchronisation.
6. **Règles du jeu** : `stageOf`, `skillLevel`, `moodOf`, `ensureDay`, `grant` (gain d'XP), `playerLevel`, etc.
7. **Sons** (`sfx`), **voix des créatures** (`VOICES`, `voice`), **musique** (`startMusic`, `playStep`), **chorale** (`chorusTap`, `answerPhrase`).
8. **Effets** : `confetti`, `ring`, `stars`, `coins`, `orbs`, `flyXp`, `speciesFx`, `ambientFx`, `shake`, `vflash`, `floatText`.
9. **Toasts** (notifications en haut), **Annuler**, **grands moments** (`evolveScene`, `openChest`).
10. **Actions de jeu** : `doAction`, `doMilestone`, `pet`.
11. **Feuilles modales** (éditeurs) : `openGoalEditor`, `openSkillEditor`, `openActionEditor`, `openMilestoneEditor`, `openShop`.
12. **Vues** : `viewHome`, `viewGoal`, `viewTrophies`, `viewSettings`, `viewWelcome`, puis `render()`.
13. **Clics** : un seul écouteur sur `document`, qui lit l'attribut `data-act` des boutons.
14. **Vie des créatures** : yeux qui suivent le doigt, comportements au repos.

**Principe d'affichage** : l'état est dans la variable `state`. Après chaque changement : `commit()` puis `render()`, qui reconstruit la vue courante en HTML.

---

## 3. Les données (`state`)

```js
{
  v: 1,
  started: true,                 // false = écran d'accueil du premier lancement
  goals: [{
    id, name, creature,          // creature = nom donné à la créature (facultatif)
    species, color,              // espèce et couleur de la créature
    xp, legendary, sadNeed,      // sadNeed = actions restantes pour consoler la créature
    createdAt, lastActive,       // lastActive = jour de la dernière action (AAAA-MM-JJ)
    acc: { head, face, neck },   // accessoires portés
    skills: [{ id, name, xp, parentId, unlockLevel, perks:[{level, name}] }],
    actions: [{ id, name, diff, skillId, target,      // target = objectif par semaine (0 = aucun)
                today:{d, n}, week:{k, n, done} }],   // compteurs du jour et de la semaine
    milestones: [{ id, name, size, skillId, done, doneAt, gained }]
  }],
  streak: { count, best, lastDay, jokers, jokerWeek },
  daily: { day, quests:[{id, text, target, key}], c:{actions, xp, hard, goals:[], pets, milestones, crits}, claimed },
  achievements: { idDuSucces: dateDeDeblocage },
  stats: { actions, crits, megas, milestones, evolutions, chests, pets, perks, xp, night, early, chorusBest, duets },
  log: [{ t, g, label, xp, crit }],   // journal, 150 entrées maximum
  days: { 'AAAA-MM-JJ': { n, xp } },  // activité par jour, 130 jours maximum
  coins: 40,                          // éclats (monnaie de la boutique)
  owned: ['paille', ...],             // accessoires achetés
  settings: { sound, music, musicVol },
  updatedAt                           // horodatage de la dernière modification
}
```

**Règle importante** : toute nouvelle donnée doit avoir une valeur par défaut dans `freshState()` **et** être complétée dans `migrate()`, pour que les sauvegardes existantes continuent de fonctionner.

---

## 4. Les règles du jeu

### Actions (gestes récurrents)

| Difficulté | XP |
|---|---|
| Facile | 10 |
| Moyen | 25 |
| Difficile | 50 |
| Épique | 100 |

- Valider = **1 touche**. Un bouton « Annuler » apparaît 5 secondes.
- **Coup critique** : 12 % de chance, XP ×2. **Méga critique** : 3 % de chance, XP ×5.
- **Bonus de série** : +2 % d'XP par jour consécutif, maximum +30 %.
- **Objectif par semaine** (facultatif) : quand il est atteint, bonus de 60 XP et 10 éclats. Le compteur repart chaque lundi. Le bouton affiche « 2/3 », puis « ✓ 3/3 ». Sans objectif, il affiche « ×2 » (nombre de fois aujourd'hui).
- Une action liée à une sous-compétence **verrouillée** ne peut pas être validée.

### Étapes (résultats concrets, validés une seule fois)

| Importance | XP | Éclats |
|---|---|---|
| Petite | 150 | 15 |
| Moyenne | 300 | 30 |
| Grande | 600 | 50 |
| Finale | 1000 | 100 |

- Validation en 2 touches (confirmation). L'étape finale rend la créature **légendaire**.
- Une étape faite peut être remise à « non faite » : l'XP gagnée est alors retirée.

### Évolution des créatures (selon l'XP de l'objectif)

| Forme | Condition |
|---|---|
| Œuf | 0 XP (des fissures apparaissent en approchant de 100) |
| Bébé | 100 XP |
| Jeune | 800 XP |
| Adulte | 3 000 XP |
| Légendaire | uniquement en validant l'étape finale |

### Compétences

- Niveau : il faut 40 XP pour le niveau 2, puis 20 XP de plus à chaque niveau (60, 80…).
- Une action donne son XP à sa compétence, et **50 %** à la compétence parente.
- **Sous-compétences** : se débloquent quand la compétence parente atteint un niveau choisi.
- **Capacités** : des paliers nommés qui se débloquent à certains niveaux.

### Humeur et pénalité

- 2 jours sans action sur un objectif : la créature s'inquiète.
- 3 jours : elle est **triste** (larmes, couleurs ternes). Gains −20 % sur cet objectif pendant 3 actions.
- 7 jours : elle **s'endort** (« Zzz »). Gains −20 % pendant 5 actions.
- On ne perd jamais d'XP.

### Série et jokers

- Une série compte les jours consécutifs avec au moins une action.
- **1 joker gratuit par semaine** (2 maximum en réserve). Un joker protège automatiquement la série si on rate un jour.

### Quête du jour

- 3 quêtes tirées chaque jour parmi 9 : valider 3 ou 5 actions, gagner 150 ou 300 XP, une action difficile ou épique, progresser sur 2 objectifs, câliner une créature, valider une étape, obtenir un critique.
- Les 3 réussies : un **coffre** s'ouvre. Il donne 50 à 200 XP à une créature au hasard, 10 à 25 éclats, et 35 % de chance d'un joker.

### Niveau de joueur

- Toute l'XP gagnée fait monter le niveau de joueur : 150 XP pour le niveau 2, puis 75 XP de plus à chaque niveau.
- **Titres** : Novice (1), Apprenti (3), Aventurier (5), Éleveur (8), Expert (12), Champion (16), Maître (20), Légende (30).
- Chaque niveau rapporte 20 éclats.

### Boutique et accessoires

- **Éclats** gagnés avec : coffres (10 à 25), étapes (15 à 100), succès (5), objectifs de la semaine (10), niveaux de joueur (20). Départ : 40 éclats.
- 10 accessoires, 3 emplacements par créature (un couvre-chef, des lunettes, un tour de cou) :

| Accessoire | Emplacement | Prix |
|---|---|---|
| Chapeau de paille | tête | 40 |
| Casquette | tête | 40 |
| Petit nœud | tête | 30 |
| Couronne de fleurs | tête | 60 |
| Casque audio | tête | 70 |
| Chapeau de magicien | tête | 90 |
| Lunettes rondes | visage | 35 |
| Lunettes de soleil | visage | 50 |
| Nœud papillon | cou | 30 |
| Écharpe | cou | 45 |

- Un accessoire acheté peut être porté par toutes les créatures. Les œufs ne peuvent rien porter.

### Succès (29)

Premier pas, Sur la lancée (10 actions), Habitude en marche (50), Centurion (100), Inarrêtable (500), Coup de chance (1er critique), Main chanceuse (25 critiques), MÉGA ! (méga critique), Cap franchi (1re étape), Bâtisseur (10 étapes), Éclosion, Ça pousse ! (Jeune), Tout grandit (Adulte), Légende vivante (Légendaire), Ça chauffe (série de 3), Semaine parfaite (7), Volonté de fer (30), Trésor ! (1er coffre), Chasseur de trésors (10 coffres), Spécialiste (compétence niv. 5), Maître (niv. 10), Nouvelle branche (capacité ou sous-compétence), Câlin, Meilleurs amis (50 câlins), Oiseau de nuit (action après 23 h), Lève-tôt (avant 7 h), Cinq mille (5 000 XP), Duo ! (une créature répond à ta mélodie), Maestro (chorale de 12 notes).

---

## 5. Les créatures

Cinq espèces originales (aucun personnage existant). Chacune a sa voix, sa façon de bouger, sa réaction au toucher et ses particules.

| Espèce | Description | Au repos | Au toucher | Voix | Particules |
|---|---|---|---|---|---|
| Flammèche | renard de braise | sautille, queue de flamme | vrille en l'air | jappements « kyu-kyu ! » + crépitements | braises |
| Nuvelin | chat-nuage | flotte | s'écrase puis s'envole | miaulement ondulant + ronronnement | bouffées de nuage (pluie si triste) |
| Pousseron | esprit des feuilles | se dandine | tourbillonne | marimba « pip-pop » | feuilles et pétales |
| Gouttelune | phoque des marées | gigote comme de la gelée | s'aplatit et rebondit | « bloup-bloup » | bulles et gouttes |
| Cristallou | golem de cristal | se balance lourdement | saut lourd, l'écran tremble | cloche cristalline + pas lourd | éclats brillants |

- **8 couleurs** au choix, et un nom personnalisé facultatif.
- **Animations permanentes** : respiration, clignement des yeux, yeux qui suivent le doigt ou la souris, regard autour, petits sauts, particules d'ambiance.
- **Expressions** : yeux rieurs (^ ^), bouche qui parle, yeux tristes avec sourcils et larme, yeux fermés quand elle dort.
- **Forme légendaire** : aura dorée, rayons qui tournent, ailes, couronne, étincelles, et écho féerique dans la voix.
- **La voix change avec la forme** : aiguë quand bébé, plus grave adulte, descendante quand triste, ronflement quand endormie. L'œuf fait « toc toc ».

---

## 6. Son et musique

- **Effets sonores** pour chaque événement : validation, critique, méga critique, niveau, évolution, coffre, succès, annulation.
- **Musique d'ambiance** générée en direct (86 BPM, accords Do maj7, La m7, Fa maj7, Sol) : nappe, basse, arpège, percussions douces et mélodie qui revient. Elle baisse pendant les grands moments.
- **Chorale** : toucher une créature lui fait chanter une note, avec son timbre, dans une gamme pentatonique, calée sur le rythme et les accords de la musique.
  - À partir de la 3e note, une autre créature visible harmonise parfois.
  - Après une pause, une autre créature **rejoue la mélodie** et la termine (« Duo ! »).
  - Les créatures dansent sur le rythme pendant la chorale. Les œufs jouent les percussions.
- **iPhone** : le code joue un son silencieux en boucle pour que le bouton silencieux ne coupe pas le son. Un bouton « Tester le son » dans Réglages affiche l'état du moteur audio.
- Réglages : effets sonores et musique séparés, volume de la musique.

---

## 7. Effets visuels

Confettis, ondes de choc, étoiles scintillantes, pièces qui tournent, **orbes d'XP qui volent jusqu'à la barre de progression** (qui se remplit à leur arrivée), flash coloré et tremblement de l'écran sur les critiques, reflet brillant sur les barres, lucioles en fond, notes de musique qui s'envolent.

**Animation d'évolution** : écran sombre, la créature clignote en silhouette blanche entre l'ancienne et la nouvelle forme, des lumières convergent, puis flash, ondes, étoiles, confettis, fanfare et premier cri de la nouvelle forme.

Si l'appareil demande moins d'animations (`prefers-reduced-motion`), les effets sont réduits.

---

## 8. Les écrans

- **Premier lancement** : trois œufs, et deux boutons : « Commencer avec mes 3 objectifs » (exemples modifiables) ou « Partir de zéro ».
- **Accueil** :
  - en-tête avec musique ♪, éclats 💎 (ouvre la boutique), série 🔥 et jokers 🛡 ;
  - barre de niveau de joueur ;
  - **le jardin** : toutes les créatures ensemble dans un paysage dont le ciel suit l'heure réelle (matin, jour, coucher de soleil, nuit étoilée) ;
  - quête du jour avec le coffre ;
  - une carte par objectif : créature, forme, progression, humeur et boutons d'action ;
  - « Nouvel objectif ».
- **Fiche d'un objectif** : grande créature, bouton « Habiller », chemin d'évolution (formes non atteintes en silhouette « ??? »), actions, étapes (la finale en doré), arbre des compétences avec niveaux, capacités et sous-compétences.
- **Succès** : niveau de joueur, chiffres clés, **calendrier d'activité** sur 12 semaines (cases de plus en plus vertes), grille des succès, journal des dernières actions.
- **Réglages** : état de la sauvegarde, effets sonores, musique et volume, « Tester le son », exporter et importer une sauvegarde, aide « Comment ça marche », et deux réinitialisations :
  - « Remettre ma progression à zéro » garde les objectifs, compétences, actions et étapes ;
  - « Tout effacer et recommencer » efface tout (il faut écrire EFFACER).
- **Barre du bas** : Accueil, Succès, Réglages.

**Style** : couleurs douces (lavande, rose, menthe, bleu ciel, doré), coins arrondis, mode clair et mode sombre automatiques, adapté au téléphone.

---

## 9. Sauvegarde et sécurité

- Les données restent **uniquement sur l'appareil** (`localStorage`). Il n'y a ni serveur ni compte.
- **Pas de synchronisation** entre téléphone et PC dans la version installable. Transfert manuel : Réglages › Exporter, puis Importer sur l'autre appareil (l'import remplace tout).
- Le dépôt GitHub est **public** : le code est visible, pas les données.
- Tout texte saisi par l'utilisateur est échappé avec `esc()` avant affichage.
- `navigator.storage.persist()` est demandé pour éviter que le navigateur efface la sauvegarde.

---

## 10. Comment mettre à jour l'app

1. Modifier `index.html` (et les autres fichiers si besoin).
2. Dans `sw.js`, **changer le numéro de version de `CACHE`** (ex. `eclosion-v8` → `eclosion-v9`), sinon les téléphones gardent l'ancienne version.
3. Envoyer les fichiers sur GitHub (branche `main`). Le site se met à jour en 1 à 2 minutes.
4. Sur le téléphone, fermer puis rouvrir l'app (parfois deux fois). Les données sont conservées.

**Précautions :**
- Ne jamais changer la clé de sauvegarde `eclosion.save.v1`, sinon l'utilisateur perd sa progression.
- Toujours ajouter les nouvelles données dans `freshState()` et `migrate()`.
- Tester sur un téléphone (largeur d'environ 400 px) et en mode sombre.
- Conseiller à l'utilisateur d'exporter une sauvegarde avant une grosse mise à jour.

---

## 11. Idées d'améliorations déjà évoquées

- Bilan de la semaine (chaque lundi : résumé de la progression et récompense).
- Nouvelles espèces de créatures.
- Synchronisation entre téléphone et PC (demanderait un service en ligne, par exemple un compte et une base de données).
- Rappels et notifications.
- Plus d'accessoires, ou des décors à acheter pour le jardin.

---

## 12. Message à copier dans un nouveau chat

> Je développe une app personnelle appelée Éclosion, une app de progression inspirée des RPG où mes objectifs de vie deviennent des créatures qui évoluent. Je ne sais pas coder : explique-moi simplement ce que tu fais et comment tester.
> Voici la documentation complète : https://github.com/sfnt656f64-tech/eclosion-/blob/main/DOCUMENTATION.md
> Et le code de l'app (un seul fichier) : https://raw.githubusercontent.com/sfnt656f64-tech/eclosion-/main/index.html
> L'app est en ligne ici : https://sfnt656f64-tech.github.io/eclosion-/
> Je voudrais l'améliorer : [écris ici ce que tu veux].
