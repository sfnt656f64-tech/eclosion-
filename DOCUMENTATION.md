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
    xp, legendary,
    comeback: { n, pct } | null, // bonus de retrouvailles restant
    lastComebackAt,              // jour du dernier bonus de retrouvailles
    metrics: [{ id, name, unit, dir, entries:[{t, v}], rewardWeek }],
    campaign: { start, weeks, final, woop, own, krs:[...] } | null, pastCampaigns: [...],
    paused: { since } | null,
    createdAt, lastActive,       // lastActive = jour de la dernière action (AAAA-MM-JJ)
    acc: { head, face, neck },   // accessoires portés
    skills: [{ id, name, xp, parentId, unlockLevel, perks:[{level, name}] }],
    actions: [{ id, name, diff, skillId, target,      // target = objectif par semaine (0 = aucun)
                today:{d, n}, week:{k, n, done} }],   // compteurs du jour et de la semaine
    milestones: [{ id, name, size, skillId, done, doneAt, gained }]
  }],
  streak: { count, best, lastDay, jokers, jokerWeek, repair:{need, got, until}|null, lastRepair, carry },
  plan: { date, items:[{ g, a, cue, done }], skipped } | null,
  week: { k, items:[...], vedette, ifthen, minimalMode } | null, weeks: [{ k, rate, n, note }], adjustLog: [...],
  daily: { day, quests:[{id, text, target, key}], c:{actions, xp, hard, goals:[], pets, milestones, crits}, claimed },
  achievements: { idDuSucces: dateDeDeblocage },
  stats: { actions, crits, megas, milestones, evolutions, chests, pets, perks, xp, night, early, chorusBest, duets, comebacks, repairs, promises, records },
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
- **Objectif par semaine** (facultatif) : quand il est atteint, bonus d'environ 60 XP (la base est 60, multipliée par les bonus de série en cours) et 10 éclats. Le compteur repart chaque lundi. Le bouton affiche « 2/3 », puis « ✓ 3/3 ». Sans objectif, il affiche « ×2 » (nombre de fois aujourd'hui).
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

### Humeur et retrouvailles (plus aucune pénalité)

- 2 jours sans action sur un objectif : la créature s'inquiète.
- 3 jours : elle **t'attend**, un peu triste (larmes, couleurs ternes).
- 7 jours : elle **s'endort** (« Zzz »).
- **Au retour, bonus de retrouvailles** : +30 % d'XP sur les 2 premières actions (après 3 à 6 jours), ou +50 % sur les 3 premières (après 7 jours ou plus). La créature saute de joie dès la première action.
- Le bonus est limité à **une fois tous les 14 jours par objectif** (champ `lastComebackAt`), pour qu'on n'ait pas intérêt à s'absenter exprès.
- On ne perd jamais d'XP, et aucun texte ne culpabilise (« je t'attendais », jamais « tu m'as abandonné »).
- L'ancien malus de −20 % (champ `sadNeed`) est supprimé ; `migrate()` remet `sadNeed` à 0.

### Série, jokers et réparation

- Une série compte les jours consécutifs avec au moins une action.
- **Bonus de série** : +2 % par jour, maximum +30 %. Il se calcule sur `max(count, carry)` (voir plus bas).
- **1 joker gratuit par semaine** (2 maximum par ce biais ; le coffre du jour peut en ajouter jusqu'à 3). Un joker protège automatiquement la série si on rate un jour.
- **Réparation sous 48 h** (« Rallume ta flamme ») : s'il ne reste pas assez de jokers et qu'il ne manque qu'un jour non couvert, la flamme devient une **braise** (pastille qui clignote et carte orange sur l'accueil). Il faut **2 actions, ou 1 action difficile ou épique** (une étape compte aussi double), le jour même ou le lendemain. La série est alors sauvée, avec animation de flamme. Au plus **une réparation tous les 7 jours** (`streak.lastRepair`).
- **Rupture douce** : si la série casse quand même, on garde la moitié du bonus : `carry = floor(min(série, 15) / 2)`. Une série de 20 jours (+30 %) repart donc à +14 %. `carry` disparaît dès que la nouvelle série le dépasse.
- Jamais de message « Tu as perdu ta série » : le texte affiché est « Nouvelle série, nouveau départ ».

### Couche Plan : campagne, semaine, jour

La cascade relie le quotidien à un résultat daté : **objectif final → campagne de 12 semaines → plan de la semaine → plan du jour → actions**. Principe non négociable : **l'app propose, la personne choisit et écrit avec ses mots** (aucun plan livré tout fait).

**Campagne de 12 semaines** (section « Campagne » de la fiche d'un objectif, assistant en 4 étapes) :
1. Objectif final : un résultat constatable, une date (par défaut la fin de la campagne) et « pourquoi ça compte pour moi ».
2. Contraste mental (WOOP) : meilleur résultat, obstacle intérieur (« Si… »), parade (« …alors je »).
3. 2 à 3 **résultats clés en plage** (« 40 à 60 »), chacun de type **processus** (dépend de moi) ou **résultat** (dépend en partie des autres). Règle imposée : au moins 1 processus, au plus 1 résultat. Un résultat clé peut suivre automatiquement une **mesure** existante, sinon on le met à jour à la main (+1, +5 ou valeur).
4. Question de propriété : « De 1 à 5, à quel point c'est le tien ? ». À 3 ou moins, l'app propose de reformuler (on peut valider quand même).
- La campagne commence le lundi de la semaine en cours (si on la crée du lundi au mercredi) ou le lundi suivant, et dure 12 semaines. Lancer une campagne : 50 XP.
- **Bilan de campagne** à la fin : statut de chaque résultat clé (atteint, dépassé, pas encore) et leçon retenue. Récompense : 100 XP, +600 XP par résultat clé atteint, +300 XP s'il est dépassé, +30 éclats. La campagne est archivée (`pastCampaigns`).
- Données : `goal.campaign = {start, weeks, final:{text, date, why}, woop:{best, obstacle, plan}, own, krs:[{id, name, type, min, max, unit, metricId, value}]}`.

**Bilan du lundi et plan de la semaine** (carte sur l'accueil) :
1. Regarder la semaine passée : prévu et fait, sans rouge, et une question facultative « Qu'est-ce qui a aidé, ou gêné ? ».
2. Calibrer : taux moyen des 4 dernières semaines, avec la cible de 70 à 80 % (au-dessus de 90 %, l'app suggère de monter d'un cran ; sous 60 %, d'alléger).
3. Choisir **1 à 3 engagements par objectif** (10 au maximum) parmi ses actions ou avec ses mots, chacun avec une **plage** et une **version minimale**. Les choix de la semaine passée sont pré-cochés.
4. Choisir sa **créature vedette** (+15 % d'XP toute la semaine) et un plan de secours « Si…, alors… ».
- Un engagement lié à une action se compte tout seul quand on la valide ; un engagement libre se compte avec le bouton +.
- Récompenses : 25 XP pour la créature vedette à chaque bilan ; semaine tenue à 70 % : 20 éclats (100 % : 30).
- **Semaine minimale** : un bouton fait compter chaque engagement dès sa version minimale (objectif 1 fois).
- **Ajuster** (jamais « abandonner ») : change la plage d'un engagement ou d'un résultat clé, avec une raison en un tap, sans aucune perte. Après 3 ajustements à la baisse en 30 jours, l'app suggère simplement de prendre moins d'engagements.
- Données : `state.week = {k, items:[{id, g, actionId, text, min, max, minimal, count}], vedette, ifthen:{si, alors}, minimalMode}`, `state.weeks` (historique des taux, 52 semaines), `state.adjustLog`.

### Mettre en pause (hibernation)

- Sur un objectif (bouton crayon) : « Mettre en pause ». La créature dort paisiblement, sans tristesse ni bonus de retrouvailles, et rien n'est perdu.
- Un objectif en pause s'affiche en carte compacte à la fin de l'accueil, avec un bandeau sur sa fiche. Il sort du bilan du lundi, du plan du jour et de la quête « Progresser sur 2 objectifs ». Ses engagements de la semaine en cours et du plan du jour sont retirés.
- « Réveiller » (ou valider une de ses actions) le réactive ; son compteur d'inactivité repart d'aujourd'hui.
- Donnée : `goal.paused = {since} | null`.

### Plan du jour (« Mes coups du jour »)

- Carte sur l'accueil : chaque jour, on choisit **1 à 3 actions**, et si on veut le moment déclencheur, sous la forme « Après ___ » (suggestions : mon café, le déjeuner, la salle…). C'est une intention de mise en œuvre « si… alors… ».
- Une action du plan validée (depuis la carte ou ailleurs) est une **promesse tenue** : **+25 % d'XP** et un tampon « Tenue ». Les 3 promesses tenues : +10 éclats.
- Aucune pénalité si une promesse n'est pas tenue : le plan disparaît le lendemain. Bouton « Pas aujourd'hui » pour ne pas en faire.
- Donnée : `state.plan = {date, items:[{g, a, cue, done}], skipped}`.

### Mesures (records réels)

- Dans chaque objectif, section **Mesures** : on crée ses indicateurs (nom, unité, sens « plus haut = mieux » ou « plus bas = mieux ») et on note une valeur de temps en temps. Les virgules sont acceptées (« 62,5 »).
- Chaque mesure affiche sa dernière valeur, son record, une mini-courbe et l'historique (avec suppression d'une saisie en cas d'erreur).
- La première saisie est le point de départ. **Battre son record** : animation « RECORD ! », la créature danse, **75 XP et 10 éclats**, au plus une fois par semaine et par mesure (`rewardWeek`) ; les records suivants de la semaine sont fêtés sans XP.
- Les objectifs d'exemple ont des mesures toutes prêtes (candidatures, entretiens, charge, tour de bras, aisance à l'oral, prises de parole).
- Donnée : `goal.metrics = [{id, name, unit, dir, entries:[{t, v}], rewardWeek}]`.

### Quête du jour

- 3 quêtes tirées chaque jour parmi 9 : valider 3 ou 5 actions, gagner 150 ou 300 XP, une action difficile ou épique, progresser sur 2 objectifs, **tenir 2 promesses du plan du jour** (remplace l'ancienne quête « câliner une créature », qui récompensait le temps passé dans l'app), valider une étape, obtenir un critique.
- Les 3 réussies : un **coffre** s'ouvre. Il donne 50 à 200 XP à une créature au hasard (hors légendaires), 10 à 25 éclats, et 35 % de chance d'un joker (seulement si on en a moins de 3).

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

### Succès (40)

Premier pas, Sur la lancée (10 actions), Habitude en marche (50), Centurion (100), Inarrêtable (500), Coup de chance (1er critique), Main chanceuse (25 critiques), MÉGA ! (méga critique), Cap franchi (1re étape), Bâtisseur (10 étapes), Éclosion, Ça pousse ! (Jeune), Tout grandit (Adulte), Légende vivante (Légendaire), Ça chauffe (série de 3), Semaine parfaite (7), Volonté de fer (30), Trésor ! (1er coffre), Chasseur de trésors (10 coffres), Spécialiste (compétence niv. 5), Maître (niv. 10), Nouvelle branche (capacité ou sous-compétence), Câlin, Meilleurs amis (50 câlins), Oiseau de nuit (action après 23 h), Lève-tôt (avant 7 h), Cinq mille (5 000 XP), Duo ! (une créature répond à ta mélodie), Maestro (chorale de 12 notes), Retrouvailles (revenir après une absence), Flamme rallumée (réparer sa série), Parole donnée (1re promesse tenue), Parole d'or (25 promesses), Preuve du réel (1er record), En progrès (10 records), Cap sur 12 semaines (lancer une campagne), Le point du lundi (1er bilan), Pilote (4 bilans), Semaine tenue (70 % des engagements), Résultat clé (en atteindre un).

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

**Règle n° 1 (version 14) : l'app doit rester simple.** Son cœur, c'est les créatures et les actions validées en une touche. L'utilisateur a trouvé que les couches de planification (versions 9 à 13) rendaient l'app compliquée et moins motivante. Elles sont donc rangées derrière un réglage, **« Outils de planification », désactivé par défaut** (Réglages). Toute nouvelle fonction issue d'un rapport ou d'une idée doit être **facultative et désactivée par défaut**, ne jamais ajouter de bloc sur l'accueil sans l'accord explicite de l'utilisateur, et être validée par lui **avant** d'être codée.

**Ce que l'utilisateur veut (ses mots)** : « une sorte de check-list simple pour atteindre mes objectifs ». Les objectifs lui plaisent ; les compétences et les étapes lui semblent « bof ». Il veut ce qui donne le plus envie de compléter ses tâches.

**Check-list du jour (version 16, mode simple)** : chaque action d'un objectif est une ligne avec une case, bordée de la couleur de sa difficulté. La case se coche quand l'action est faite aujourd'hui ; on peut la refaire (badge ×2, ×3…). Chaque objectif affiche « Aujourd'hui : 2/4 » avec une barre. Quand toutes ses actions sont cochées (au moins 2 actions) : **journée complète**, +25 XP, 5 éclats et une célébration, une fois par jour et par objectif (`goal.perfectDay`, `stats.perfectDays`, succès « Journée complète » et « Sans faute »). Les cases repartent à zéro chaque jour (`action.today`).

**Journée complète** : ne compte que les actions **du jour** (sans objectif par semaine) ; les actions prévues quelques fois par semaine (séance, appel réseau…) sont en bonus et ne bloquent pas (`dailyActions`).

**Bonnes actions de check-list (version 17)** : un verbe, faisable en moins de 30 minutes, dont on sait exactement quand c'est fini, et qui ne dépend pas des autres. Les objectifs d'exemple utilisent désormais ces actions (CDI : candidature ciblée + message, relances, message LinkedIn, réponse STAR, appel réseau 1/semaine ; muscu : séance 3/semaine, protéines, coucher avant minuit, marche ; parler : s'enregistrer 2 min, réécouter, demande claire, avis en réunion 2/semaine). Une carte sur l'accueil (et un bouton dans Réglages, « Actions proposées ») propose de remplacer les anciennes actions par celles-ci, objectif par objectif, en décochant celles qu'on ne veut pas ; l'XP, la créature et les compétences sont gardées (`ACTION_SETS`, `state.actionsUpgrade`).

**Objectifs concrets** : un bon titre d'objectif dit **quoi** et **pour quand**, et on sait sans hésiter s'il est atteint. L'utilisateur écrit ses propres titres dans l'app (bouton crayon). **Ne jamais écrire les objectifs réels de l'utilisateur dans le code ou la documentation : le dépôt est public.** Les objectifs d'exemple sont génériques et sans date (« Décrocher un nouvel emploi », « Retrouver une routine sportive », « Être à l'aise en entretien »). La proposition d'actions est versionnée (`UPGRADE_V`) : quand elle change, elle est reproposée, mais seulement pour les objectifs dont les actions diffèrent (`setIsApplied`).

**Compétences et étapes : désactivées par défaut** (réglage « Compétences et étapes », `settings.skillsSteps`). Cachées : sections Compétences et Étapes, sélecteur de compétence dans l'éditeur d'action, verrouillage des sous-compétences, notifications de niveau de compétence, quête « Valider une étape » et 5 succès liés. L'XP des compétences continue de s'accumuler en silence (rien n'est effacé). Pour la forme légendaire, la fiche affiche un bouton **« J'ai atteint mon objectif ! »** qui valide l'étape finale (créée si besoin).

**Mode simple (par défaut)** :
- Accueil : en-tête (éclats et série), le jardin, une carte avec les quêtes du jour (et la braise à rallumer si besoin), puis les objectifs avec **toutes** leurs actions.
- Fiche d'un objectif : la créature avec son chemin d'évolution, puis Actions, Étapes et Compétences, sans onglets.
- Restent actifs car ils ne prennent pas de place : pas de pénalité, bonus de retrouvailles, réparation de série, mise en pause, objectifs par semaine sur les actions.
- Cachés (données gardées, rien n'est effacé) : campagne, bilan du lundi et engagements, plan du jour, mesures, et les 9 succès qui en dépendent. La quête « Tenir 2 promesses » n'est jamais tirée, et elle est remplacée si elle avait déjà été tirée dans la journée (`fixDailyQuests`).

**Avec les outils de planification** : l'affichage décrit ci-dessous (carte « Aujourd'hui », 3 actions par carte, onglets dans la fiche).

- **Premier lancement** : trois œufs, et deux boutons : « Commencer avec mes 3 objectifs » (exemples modifiables) ou « Partir de zéro ».
- **Accueil**, dans cet ordre :
  - en-tête avec deux pastilles seulement : éclats 💎 (ouvre la boutique) et série 🔥 (les jokers sont dans son infobulle). La musique se règle dans Réglages, le niveau de joueur est dans Succès ;
  - **le jardin** : toutes les créatures ensemble dans un paysage dont le ciel suit l'heure réelle ;
  - **une seule carte « Aujourd'hui »** en trois parties : braise à rallumer (si besoin) et plan du jour, quêtes du jour avec un petit coffre, puis « Ma semaine » en une ligne (taux et barre), dépliable avec « Voir » (préférence gardée sur l'appareil). Le bilan du lundi y apparaît en une ligne quand il est dû ;
  - « Mes objectifs » : une carte par objectif avec **3 actions au maximum** (celles de la semaine et du plan d'abord, puis les plus utilisées) et un bouton « +N autres » vers la fiche. Les objectifs en pause sont en carte compacte à la fin ;
  - « Nouvel objectif ».
- **Fiche d'un objectif** : la créature, sa forme, sa progression et le bouton « Habiller », puis **trois onglets** (collés en haut quand on fait défiler) :
  - **Actions** : les actions et les étapes (la finale en doré) ;
  - **Progrès** : la campagne, les mesures et le chemin d'évolution (formes non atteintes en silhouette « ??? ») ;
  - **Compétences** : l'arbre des compétences avec niveaux, capacités et sous-compétences.
  L'onglet ouvert reste le même quand on valide une action ; il revient sur « Actions » quand on ouvre un autre objectif.
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
2. Dans `sw.js`, **changer le numéro de version de `CACHE`** (actuellement `eclosion-v19` ; passer à `eclosion-v20`, etc.), sinon les téléphones gardent l'ancienne version.
3. Envoyer les fichiers sur GitHub (branche `main`). Le site se met à jour en 1 à 2 minutes.
4. Sur le téléphone, fermer puis rouvrir l'app (parfois deux fois). Les données sont conservées.

**Précautions :**
- Ne jamais changer la clé de sauvegarde `eclosion.save.v1`, sinon l'utilisateur perd sa progression.
- Toujours ajouter les nouvelles données dans `freshState()` et `migrate()`.
- Tester sur un téléphone (largeur d'environ 400 px) et en mode sombre.
- Conseiller à l'utilisateur d'exporter une sauvegarde avant une grosse mise à jour.

---

## 11. Fondements et prochaines étapes

Les évolutions de la version 9 suivent un rapport d'analyse fondé sur la recherche en motivation :
- **Pack rattrapage** (retrouvailles et réparation de série) : la possibilité de réparer une série atténue la démotivation liée à sa rupture (Silverman & Barasch, 2023), et la bienveillance envers soi après un échec augmente la motivation à s'améliorer (Breines & Chen, 2012).
- **Plan du jour** : les intentions de mise en œuvre « si… alors… » ont un effet moyen à fort sur l'atteinte des objectifs (Gollwitzer & Sheeran, 2006).
- **Mesures** : suivre ses progrès améliore l'atteinte des objectifs, surtout quand ils sont notés (Harkin et al., 2016).

La version 10 ajoute la **couche Plan**, d'après un second rapport : objectifs précis et ambitieux (Locke & Latham), sous-objectifs proches (Bandura & Schunk, 1981), objectifs en plage (Scott & Nowlis, 2013), contraste mental WOOP (Oettingen ; méta-analyse Wang et al., 2021), engagement et propriété de l'objectif (Klein et al., 1999 ; Chi et al., 2026, préprint), calibrage contre l'erreur de planification (Buehler et al., 1994).

À faire ensuite d'après ce rapport : chapitres mensuels avec « boss » relié au résultat du mois, revue mensuelle, mues de fin de campagne, carte de chasse branchée sur les engagements, plafond de l'XP de planification.

Autres idées du premier rapport, pas encore faites : critiques proportionnels à la difficulté, rendements décroissants après 3 fois la même action dans la journée, coffre attribué à la créature de son choix, carte de chasse hebdomadaire, bilan du lundi avec créature vedette, jours de repos déclarés (utile pour la musculation), mues intermédiaires entre Jeune et Adulte, boss mensuel, partage du bilan, chroniques narratives, rappels par fichier agenda (.ics).

## 12. Idées d'améliorations déjà évoquées

- Bilan de la semaine (chaque lundi : résumé de la progression et récompense).
- Nouvelles espèces de créatures.
- Synchronisation entre téléphone et PC (demanderait un service en ligne, par exemple un compte et une base de données).
- Rappels et notifications.
- Plus d'accessoires, ou des décors à acheter pour le jardin.

---

## 13. Message à copier dans un nouveau chat

> Je développe une app personnelle appelée Éclosion, une app de progression inspirée des RPG où mes objectifs de vie deviennent des créatures qui évoluent. Je ne sais pas coder : explique-moi simplement ce que tu fais et comment tester.
> Voici la documentation complète : https://github.com/sfnt656f64-tech/eclosion-/blob/main/DOCUMENTATION.md
> Et le code de l'app (un seul fichier) : https://raw.githubusercontent.com/sfnt656f64-tech/eclosion-/main/index.html
> L'app est en ligne ici : https://sfnt656f64-tech.github.io/eclosion-/
> Je voudrais l'améliorer : [écris ici ce que tu veux].
