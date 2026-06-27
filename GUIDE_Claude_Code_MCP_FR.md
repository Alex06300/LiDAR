# Guide pas à pas — Construire ton jeu « LiDAR » avec Claude Code + le MCP d'Unreal 5.8

Ce guide t'explique **de A à Z** comment faire construire ton jeu LiDAR par **Claude Code**
(lancé sur **ton PC**), qui pilote l'éditeur Unreal via le **MCP**. Tu le lis en parallèle du
fichier **`LIDAR_GAME_SPEC.md`** (le cahier des charges technique, déjà placé à la racine du projet).

> **Comment ça marche, en une phrase :** le MCP d'Unreal est un petit serveur qui tourne *dans
> l'éditeur* sur `http://127.0.0.1:8000`. Un agent lancé **sur la même machine** (ici Claude Code)
> s'y connecte et peut spawn des acteurs, créer des Blueprints/Niagara/matériaux, compiler, etc.
> *(C'est pour ça que je ne peux pas le piloter depuis Cowork : mon environnement est isolé et
> n'atteint pas ton `127.0.0.1`. Toi, depuis ton PC, tu le peux.)*

**Durée estimée :** 30–45 min de mise en place, puis tu construis jalon par jalon.

---

## Sommaire
0. [Prérequis](#0-prérequis)
1. [Installer Claude Code](#1-installer-claude-code)
2. [Démarrer le serveur MCP dans Unreal](#2-démarrer-le-serveur-mcp-dans-unreal)
3. [Installer le plugin Epic « Unreal Skills » pour Claude Code](#3-installer-le-plugin-epic-unreal-skills-pour-claude-code)
4. [Lancer Claude Code et vérifier la connexion](#4-lancer-claude-code-et-vérifier-la-connexion)
5. [Construire le jeu — les prompts à coller](#5-construire-le-jeu--les-prompts-à-coller)
6. [Si le MCP bloque sur une étape (plans B)](#6-si-le-mcp-bloque-sur-une-étape-plans-b)
7. [Dépannage](#7-dépannage)
8. [Sécurité — bonnes pratiques](#8-sécurité--bonnes-pratiques)
9. [Aide-mémoire des commandes](#9-aide-mémoire-des-commandes)

---

## 0. Prérequis

- **Windows 10/11** + ton projet `LIDAR_MCP` ouvrable dans **Unreal Engine 5.8**.
- Le plugin **ModelContextProtocol** est **déjà activé** dans ton projet ✅ (vérifié).
- Un compte **Claude Pro ou Max** (Claude Code **n'est pas inclus dans le plan gratuit**).
- **Git for Windows** (recommandé) : <https://git-scm.com/downloads/win>
  Il fournit *Git Bash*, nécessaire au plugin Epic et pratique pour Claude Code.

---

## 1. Installer Claude Code

Ouvre **PowerShell** (menu Démarrer → tape `PowerShell`). Ton invite affiche `PS C:\...>`.

```powershell
irm https://claude.ai/install.ps1 | iex
```

**Ferme puis rouvre** PowerShell, et vérifie :

```powershell
claude --version
claude doctor
```

> `claude doctor` te dit si tout est OK (binaire, dépendances, mises à jour). L'installeur natif
> se met à jour tout seul ensuite.

Première fois : tape `claude`, ça ouvre le navigateur pour te **connecter** à ton compte Anthropic.
(Si `claude` n'est pas reconnu, ferme/rouvre le terminal ; sinon voir [Dépannage](#7-dépannage).)

---

## 2. Démarrer le serveur MCP dans Unreal

1. Ouvre ton projet **`LIDAR_MCP`** dans Unreal 5.8.
2. **Active le démarrage auto** du serveur (recommandé) :
   *Edit → Editor Preferences → (groupe General) → **Model Context Protocol** → coche
   **Auto Start Server**.* Il démarrera à chaque ouverture de l'éditeur sur
   `http://127.0.0.1:8000/mcp`.
   *Alternative ponctuelle :* ouvre la **console** (touche `²` ou `` ` `` en haut à gauche du clavier)
   et tape :
   ```
   ModelContextProtocol.StartServer
   ```
3. Vérifie dans l'**Output Log** (Window → Output Log) une ligne indiquant que le serveur écoute
   sur `127.0.0.1:8000`.
4. Le fichier `.mcp.json` est **déjà présent** à la racine du projet ✅. Si jamais tu changes le port,
   régénère-le avec :
   ```
   ModelContextProtocol.GenerateClientConfig ClaudeCode
   ```

---

## 3. Installer le plugin Epic « Unreal Skills » pour Claude Code

Ce plugin officiel d'Epic donne à Claude Code la compétence **`unreal-mcp`** et l'accès à
**30+ toolsets** (Acteurs, **Blueprints** — créer/éditer des graphes, ajouter des nœuds, connecter
des pins, compiler —, **Matériaux**, **Niagara**, UMG, tests d'automatisation, etc.). C'est lui qui
rend la construction du jeu réellement possible via MCP.

Lance d'abord Claude Code **dans le dossier du projet** (voir l'étape 4 pour la commande `cd`),
puis, **dans Claude Code**, tape :

```
/plugin marketplace add EpicGames/unreal-engine-skills-for-claude-code-plugin
```

puis :

```
/plugin install
```

et **choisis `unreal-engine-skills-for-claude-code` dans la liste** (l'installation interactive
évite toute erreur de nommage).

> **Plan B si le raccourci GitHub ne marche pas :** clone le dépôt puis ajoute-le par chemin local :
> ```powershell
> git clone https://github.com/EpicGames/unreal-engine-skills-for-claude-code-plugin C:\Tools\unreal-skills
> ```
> Dans Claude Code :
> ```
> /plugin marketplace add C:\Tools\unreal-skills
> /plugin install
> ```
> (le nom du « marketplace » est celui du dossier ; passe par `/plugin install` interactif pour
> sélectionner le bon plugin).

> ⚠️ Le petit *hook* de contexte du plugin est un script **bash** → installe **Git for Windows**
> (étape 0) pour qu'il s'exécute. Sans bash, les **outils MCP fonctionnent quand même**, tu perds
> juste une note de contexte automatique.

---

## 4. Lancer Claude Code et vérifier la connexion

1. Ouvre **PowerShell** (ou Git Bash) et place-toi dans le dossier du projet :
   ```powershell
   cd "D:\UnrealEngine STUFF\LIDAR_MCP"
   claude
   ```
   ⚠️ **Important :** lance toujours `claude` **depuis ce dossier** (c'est là qu'est `.mcp.json`).
2. Dans Claude Code, tape :
   ```
   /mcp
   ```
   Tu dois voir **`unreal-mcp`** listé comme **connecté**. ✅
3. Test rapide (l'éditeur doit être ouvert et le serveur démarré) :
   ```
   Liste tous les acteurs de la map actuellement ouverte dans l'éditeur.
   ```
   S'il te répond avec la liste des acteurs → tout est branché, tu peux construire.

*(Rien ne s'affiche dans `/mcp` ? → [Dépannage](#7-dépannage).)*

---

## 5. Construire le jeu — les prompts à coller

Principe : on **pointe Claude vers le cahier des charges** (`LIDAR_GAME_SPEC.md`) et on avance
**un jalon à la fois**, en testant en *Play* (PIE) après chaque étape. Colle les prompts ci-dessous
**dans l'ordre**, un par un. (Claude comprend très bien le français.)

> 💾 **Avant de commencer :** fais une copie de sauvegarde du dossier projet **ou** un commit Git.
> Le MCP modifie les assets en direct.

### Prompt 0 — Cadrage
```
Lis le fichier LIDAR_GAME_SPEC.md à la racine de ce projet Unreal.
Résume-moi le plan en jalons M1 à M8, confirme que tu es connecté au MCP unreal-mcp,
puis attends ma validation avant de commencer M1.
N'avance que d'un jalon à la fois et demande-moi de tester en Play après chacun.
```

### Prompt 1 — M1 : scène de test sombre
```
Implémente le jalon M1 du spec : crée une petite map de test "L_LidarTest" avec quelques
murs, cubes et objets pour avoir des surfaces à scanner. Désactive les lumières,
règle l'auto-exposition en manuel (Post Process Volume non borné), et désactive Lumen
(Dynamic GI = None, Reflections = None). Objectif : en Play, l'écran est quasi noir et je
peux me déplacer. Dis-moi quoi vérifier, puis attends ma validation.
```

### Prompt 2 — M2 : personnage LiDAR + entrées
```
Jalon M2 : réutilise le personnage First Person, retire la logique d'arme/projectile.
Ajoute les Input Actions IA_LidarBurst (clic gauche), IA_LidarScan (clic droit),
IA_Aperture (molette) et mappe-les dans l'Input Mapping Context. Ajoute les variables du
paragraphe 7 du spec sur le personnage (ou un composant LidarComponent). Pour l'instant,
fais juste un print à l'écran quand je clique gauche / droit / molette. Attends ma validation.
```

### Prompt 3 — M3 : rendu des points (ISM + matériau émissif)
```
Jalon M3 : crée le matériau M_Point (Unlit, Emissive), un petit mesh de point (SM_PointDot,
sphère bas-poly), et l'acteur BP_PointCloud avec un Instanced Static Mesh + une fonction
AddPoint(Location, Hit) qui ajoute une instance. Pour tester, appelle AddPoint sur quelques
positions au BeginPlay. Objectif : voir des points lumineux dans le noir. Attends ma validation.
```

### Prompt 4 — M4 : clic gauche (rafale en disque)
```
Jalon M4 : implémente la rafale du clic gauche selon le paragraphe 3.1 du spec
(déprojection de RaysPerBurst points tirés dans un disque de rayon ApertureRadiusPx autour
du centre écran, LineTraceSingleByChannel, AddPoint sur chaque impact). Objectif : un clic
gauche peint un disque de points sur ce que je vise. Attends ma validation.
```

### Prompt 5 — M5 : molette (ouverture)
```
Jalon M5 : implémente le réglage de l'ouverture à la molette (paragraphe 3.2) : ApertureRadiusPx
augmente/diminue avec la molette, borné entre ApertureMin et ApertureMax. Objectif : la molette
change visiblement la taille du disque peint par la rafale. Attends ma validation.
```

### Prompt 6 — M6 : clic droit (scan plein écran)
```
Jalon M6 : implémente le scan plein écran du clic droit (paragraphe 3.3) : une ligne horizontale
balaie l'écran de haut en bas sur ScanDurationSeconds, en déprojetant toute la largeur à chaque
frame (1 ligne/frame). Objectif : un clic droit déclenche un balayage qui peint toute la vue.
Attends ma validation.
```

### Prompt 7 — M7 : perf & couleurs
```
Jalon M7 : ajoute le plafond MaxPoints avec buffer circulaire (recycle la plus ancienne instance),
la couleur des points par distance (ou par normale) via PerInstanceCustomData, et colore en rouge
les impacts sur les acteurs ayant le tag Enemy. Vérifie que le FPS reste stable après beaucoup de
points. Attends ma validation.
```

### Prompt 8 — M8 : finitions & build
```
Jalon M8 : ajoute un viseur (cercle dont le rayon suit ApertureRadiusPx), un petit son optionnel
au tir, puis package un build Windows autonome. Donne-moi les étapes pour lancer l'exécutable.
```

> Après chaque jalon : **teste en Play**, dis à Claude ce qui marche / ne marche pas, puis passe au
> prompt suivant. N'hésite pas à demander des réglages (« plus de points », « points plus gros »,
> « scan plus lent »…) : tout est paramétré (§7 du spec).

---

## 6. Si le MCP bloque sur une étape (plans B)

Le MCP d'Unreal est **expérimental** : il se peut qu'un outil échoue ou ne couvre pas une action
précise (surtout sur des graphes Blueprint complexes). Dans ce cas, demande explicitement :

```
Si un outil MCP échoue, ne bloque pas : propose-moi soit une implémentation en C++
(écris les fichiers Source, configure le module, compile), soit la procédure manuelle exacte
nœud par nœud à faire moi-même dans l'éditeur.
```

- **Voie C++** : Claude Code peut écrire les fichiers `Source/…` et compiler. Cela nécessite
  **Visual Studio 2022** avec la charge de travail **« Développement de jeux avec C++ »**.
  Tu m'as dit ne pas savoir si tu l'as → pour vérifier : menu Démarrer → cherche *Visual Studio
  Installer* ; s'il n'existe pas, installe **Visual Studio 2022 Community** (gratuit) puis, dans
  l'installeur, coche **« Game development with C++ »** (inclut le toolchain pour Unreal). Tu peux
  aussi demander à Claude Code de te guider pas à pas pour l'installation.
- **Voie manuelle** : Claude te donne la liste exacte des nœuds Blueprint à créer/relier ; tu les
  poses toi-même dans l'éditeur. Plus long, mais imparable.

> Pour la plupart des jalons (ISM + Blueprint), le MCP suffit **sans** C++. Garde le C++ comme
> filet de sécurité.

---

## 7. Dépannage

- **`/mcp` ne montre rien / `unreal-mcp` absent**
  → l'éditeur Unreal est-il **ouvert** ? le serveur est-il **démarré** (étape 2, regarde l'Output
  Log) ? as-tu lancé `claude` **depuis** `D:\UnrealEngine STUFF\LIDAR_MCP` (là où est `.mcp.json`) ?
  Relance `claude` après que l'éditeur soit complètement chargé.
- **Port 8000 déjà utilisé** → dans la console Unreal : `ModelContextProtocol.StartServer 9001`,
  puis `ModelContextProtocol.GenerateClientConfig ClaudeCode`, puis relance Claude Code.
- **`claude` non reconnu** → ferme/rouvre le terminal ; sinon relance l'install (étape 1) et lis
  `claude doctor`.
- **Le hook bash du plugin échoue (Windows)** → installe **Git for Windows**, ou ignore : les outils
  MCP marchent quand même.
- **L'écran n'est pas noir / on voit dans le noir** → l'auto-exposition n'est pas en *Manual*
  (voir §5 du spec) : demande à Claude de régler le Post Process Volume (Metering Mode = Manual).
- **Ça rame** → réduis `RaysPerBurst`, baisse `MaxPoints`, vérifie que Lumen est bien **désactivé**
  et que `M_Point` est bien **Unlit**.
- **Outils de Claude Code à valider sans cesse** → c'est **normal et voulu** (sécurité). Évite
  `--dangerously-skip-permissions` avec ce plugin (voir §8).

---

## 8. Sécurité — bonnes pratiques

(Recommandations d'Epic pour ce plugin.)

- Le MCP donne à Claude un **accès en direct** à l'éditeur : il peut **créer, modifier, déplacer ou
  supprimer** des assets. **Sauvegarde et commit Git avant** une longue session, et **relis les
  changements**.
- `localhost` n'est **pas** une frontière de confiance : ne fais pas tourner le serveur MCP sur une
  machine partagée, ne l'expose jamais hors de `127.0.0.1`.
- **Évite `--dangerously-skip-permissions`** tant que ce plugin est chargé (il permet d'exécuter du
  Python arbitraire dans l'éditeur sans confirmation). Garde les validations activées.

---

## 9. Aide-mémoire des commandes

**PowerShell (ton PC)**
```powershell
irm https://claude.ai/install.ps1 | iex      # installer Claude Code
claude --version                              # vérifier
claude doctor                                 # diagnostic
cd "D:\UnrealEngine STUFF\LIDAR_MCP"          # se placer dans le projet
claude                                         # lancer Claude Code
```

**Dans Claude Code**
```
/plugin marketplace add EpicGames/unreal-engine-skills-for-claude-code-plugin
/plugin install        # choisir unreal-engine-skills-for-claude-code
/mcp                   # vérifier que unreal-mcp est connecté
```

**Console Unreal (touche ² ou `)**
```
ModelContextProtocol.StartServer            # démarrer le serveur (ou Auto Start dans les prefs)
ModelContextProtocol.StartServer 9001       # sur un autre port
ModelContextProtocol.GenerateClientConfig ClaudeCode   # régénérer .mcp.json
ModelContextProtocol.RefreshTools           # recharger les outils
ModelContextProtocol.StopServer             # arrêter
```

---

Bon dev ! Commence par les **étapes 1 à 4** (mise en place), puis déroule les **prompts du §5** un
par un. Le fichier `LIDAR_GAME_SPEC.md` contient tout le détail technique que Claude Code suivra.
