# LIDAR_GAME_SPEC.md — Spécification technique du jeu « LiDAR »

> Ce fichier est le **cahier des charges** que Claude Code (connecté au MCP d'Unreal) doit lire
> et implémenter, **un jalon à la fois**. Il est aussi lisible par un humain.
> Projet : `LIDAR_MCP` — Unreal Engine 5.8 — modèle *First Person* (Blueprint) — Enhanced Input.

---

## 1. Vision du jeu

Le joueur évolue dans une scène **plongée dans le noir total**. Il ne voit rien… jusqu'à ce
qu'il « tire » des faisceaux : chaque faisceau qui touche une surface y dépose un **petit point
lumineux**. En accumulant les points, le joueur reconstruit visuellement le monde autour de lui,
comme un **nuage de points LiDAR** (ambiance proche du jeu *Scanner Sombre*).

Contraintes : **léger** (60 FPS sur GPU moyen), lisible pour un débutant, réglable facilement
depuis l'éditeur.

---

## 2. Contrôles (résumé)

| Entrée | Action | Effet |
|---|---|---|
| **Clic gauche** | Rafale ponctuelle | Émet `N` faisceaux dans un **disque** (l'« ouverture ») centré sur le viseur. Chaque impact dépose un point. |
| **Molette** | Réglage de l'ouverture | Agrandit / réduit le rayon du disque de tir (taille du faisceau). Valeur bornée. |
| **Clic droit** | Scan plein écran | Une **ligne horizontale** balaie l'écran **de haut en bas** (~1,5 s) en couvrant toute la largeur. Chaque impact dépose un point. |
| ZQSD + souris | Déplacement / regard | Déjà fourni par le modèle First Person. |

---

## 3. Technique cœur : déprojection écran → rayon → trace

Toute la logique de tir repose sur **une seule** technique simple, qui colle exactement à la
demande (« ouverture » = taille d'un disque à l'écran ; « scan » = tout le moniteur) :

1. Prendre une coordonnée **écran** `(x, y)` en pixels.
2. La convertir en rayon monde avec **`DeprojectScreenToWorld`** (nœud du PlayerController) →
   on obtient une `WorldLocation` (origine) et une `WorldDirection`.
3. Lancer **`LineTraceSingleByChannel`** depuis `WorldLocation` le long de
   `WorldDirection * MaxRange`.
4. Si `bHit` : appeler `AddPoint(Hit.Location, Hit)` (voir §4).

> Avantage : pas de maths de cône à gérer. L'« ouverture » devient un simple rayon de disque en
> pixels, et le « scan plein écran » devient un balayage de lignes de pixels. Intuitif et exact.

### 3.1 Clic gauche — rafale dans un disque
Pour `i` de 1 à `RaysPerBurst` :
- tirage uniforme dans un disque de rayon `ApertureRadiusPx` autour du **centre écran**
  (`cx = ViewportX/2`, `cy = ViewportY/2`) :
  - `r = ApertureRadiusPx * sqrt(random01())`  ← le `sqrt` garantit une répartition uniforme
  - `a = 2 * PI * random01()`
  - `x = cx + r*cos(a)` ; `y = cy + r*sin(a)`
- déprojeter `(x, y)` → trace → si impact, `AddPoint`.

### 3.2 Molette — réglage de l'ouverture
- `ApertureRadiusPx = Clamp(ApertureRadiusPx + WheelAxis * ApertureStep, ApertureMin, ApertureMax)`.
- Optionnel : afficher un cercle de visée (HUD/UMG) dont le rayon = `ApertureRadiusPx`.

### 3.3 Clic droit — scan plein écran (haut → bas)
- Au déclenchement : `bScanning = true`, `ScanY = 0`, démarrer un timer/Tick.
- À chaque frame tant que `bScanning` :
  - `ScanY += (ViewportY / ScanDurationSeconds) * DeltaTime`  (avance la ligne vers le bas)
  - pour `x` de 0 à `ViewportX` par pas de `ScanColStepPx` :
    - déprojeter `(x, ScanY)` → trace → si impact, `AddPoint`.
  - si `ScanY >= ViewportY` : `bScanning = false` (fin du scan).
- Étaler le travail **sur plusieurs frames** (1 ligne par frame) garde le coût CPU/GPU stable.

---

## 4. Rendu des points (point critique pour les performances)

### 4.1 Solution recommandée — Instanced Static Mesh (ISM)
Un acteur **`BP_PointCloud`** contenant **un** composant `InstancedStaticMeshComponent` :
- **Mesh** : une toute petite sphère bas-poly (≈ 12–42 tris) ou un quad. La sphère évite d'avoir
  à orienter chaque point vers la caméra.
- **Matériau** `M_Point` : **Unlit**, sortie **Emissive Color** uniquement (aucun coût d'éclairage).
- `AddPoint(Location, Hit)` = `AddInstance` d'une transform à `Location`, échelle ≈ `PointSize`
  (par défaut 1,5 cm).

**Plafond + recyclage (anti-fuite mémoire) :** variable `MaxPoints` (par défaut 300 000).
Quand on atteint le plafond, **réutiliser** la plus ancienne instance
(`UpdateInstanceTransform` sur l'index `NextIndex`, en buffer circulaire) au lieu d'en ajouter.
Mémoire et FPS restent bornés même après des heures de jeu.

### 4.2 Couleur (optionnel mais joli)
- Utiliser **`PerInstanceCustomData`** (1 à 3 floats par instance) lu par `M_Point` pour teinter :
  - par **distance** (proche = chaud, loin = froid), ou
  - par **normale** de la surface (aide à percevoir le relief), ou
  - **rouge** si l'impact touche un acteur avec le tag `Enemy` (repérer les ennemis dans le noir).

### 4.3 Alternative haute densité — Niagara (optimisation)
Pour des **millions** de points, remplacer l'ISM par un système **Niagara** GPU persistant
alimenté par un tableau de positions (Niagara Data Channel / data interface « Array »).
Plus performant à très grande échelle, mais plus complexe à autoriser. **Ne pas commencer par là** :
implémenter d'abord l'ISM, puis basculer en Niagara seulement si nécessaire.

---

## 5. Scène & éclairage (rester léger ET sombre)

Le monde doit être **noir** : ce sont les points émissifs qui éclairent. Réglages :
- **Supprimer / désactiver** les lumières du niveau (ou les passer très bas).
- **Désactiver l'auto-exposition** : ajouter un **Post Process Volume** (cocher *Infinite Extent /
  Unbound*) → *Exposure* → **Metering Mode = Manual** (ou `Min EV100 = Max EV100`). Sinon la caméra
  « s'adapte » et finit par voir dans le noir, ce qui casse tout le concept.
- **Désactiver Lumen** (inutile ici, gros gain de perf) : *Project Settings → Engine → Rendering* :
  - *Dynamic Global Illumination Method* = **None**
  - *Reflection Method* = **None**
- Garder les meshes d'environnement avec un **matériau noir simple** ; la **collision reste active**
  (les traces touchent quand même la géométrie).
- Anti-aliasing léger (TSR ou FXAA). Pas de besoin de fog volumétrique, etc.

---

## 6. Entrées (Enhanced Input)

Le modèle First Person fournit déjà un *Input Mapping Context* (`IMC_Default`) et des *Input Actions*.
Ajouter / mapper :
- `IA_LidarBurst` (Digital/bool) → **Souris bouton gauche** (on peut réutiliser l'`IA_Fire` existant).
- `IA_LidarScan` (Digital/bool) → **Souris bouton droit**.
- `IA_Aperture` (Axis1D/float) → **Molette souris** (Mouse Wheel Axis).
Désactiver / retirer la logique d'arme (tir de projectile) du personnage du template.

---

## 7. Paramètres exposés (à mettre en variables *Instance Editable* du composant LiDAR)

| Variable | Défaut | Rôle |
|---|---:|---|
| `RaysPerBurst` | 400 | Faisceaux par clic gauche |
| `ApertureRadiusPx` | 120 | Rayon du disque de tir (px) |
| `ApertureMin` / `ApertureMax` | 8 / 500 | Bornes de la molette |
| `ApertureStep` | 20 | Pas de la molette |
| `MaxRange` | 8000 | Portée des faisceaux (cm ≈ 80 m) |
| `ScanColStepPx` | 6 | Pas horizontal du scan (px) |
| `ScanDurationSeconds` | 1.5 | Durée du scan plein écran |
| `MaxPoints` | 300000 | Plafond du nuage (buffer circulaire) |
| `PointSize` | 1.5 | Taille d'un point (cm) |
| `EnemyTag` | `Enemy` | Tag des acteurs colorés en rouge |

---

## 8. Budget performance (cibles)

- ≥ 60 FPS sur GPU milieu de gamme.
- `RaysPerBurst` ≤ 600 ; scan ≤ ~30 000 rayons **étalés** sur la durée (≈ 1 ligne/frame).
- Nuage ≤ `MaxPoints` (buffer circulaire).
- Matériau **Unlit**, **Lumen désactivé**, **exposition fixe**, collision **simple** sur l'environnement.
- `LineTraceSingleByChannel` (pas de multi-trace) sur un canal dédié (Visibility ou canal custom `Lidar`).

---

## 9. Jalons d'implémentation (à faire DANS L'ORDRE, valider chacun avant le suivant)

- **M1 — Scène de test sombre.** Une petite map avec quelques murs/cubes/objets ; lumières
  désactivées ; auto-exposition manuelle ; Lumen off.
  *Validation :* en *Play*, l'écran est quasi noir, on se déplace, rien n'est visible.

- **M2 — Personnage LiDAR.** Réutiliser le perso First Person, retirer l'arme. Ajouter le
  composant/variables LiDAR (§7) et les Input Actions (§6).
  *Validation :* on se déplace ; aucun tir d'arme ; les entrées clic G/D/molette sont reçues (log).

- **M3 — Rendu des points.** Créer `M_Point` (Unlit/Emissive), le petit mesh, et `BP_PointCloud`
  (ISM) avec une fonction `AddPoint(Location, Hit)`.
  *Validation :* appeler `AddPoint` manuellement (ex. à `BeginPlay`) crée un point visible.

- **M4 — Clic gauche (rafale disque).** Implémenter §3.1.
  *Validation :* clic gauche « peint » un disque de points sur les surfaces visées.

- **M5 — Molette (ouverture).** Implémenter §3.2.
  *Validation :* la molette élargit/réduit visiblement la zone peinte par la rafale.

- **M6 — Clic droit (scan plein écran).** Implémenter §3.3.
  *Validation :* une ligne balaie l'écran de haut en bas et peint toute la vue.

- **M7 — Perf & couleurs.** Plafond + buffer circulaire (§4.1) ; couleur par distance/normale ;
  points rouges sur les acteurs `Enemy` (§4.2).
  *Validation :* FPS stable après des dizaines de milliers de points ; ennemis distinguables.

- **M8 — Finitions.** Viseur (cercle = ouverture), petit son optionnel, et build packagé.
  *Validation :* exécutable autonome qui tourne correctement.

---

## 10. Conventions d'assets

- Tout sous `/Content/Lidar/` : `BP_PointCloud`, `M_Point`, `SM_PointDot`, `IA_LidarBurst`,
  `IA_LidarScan`, `IA_Aperture`, `L_LidarTest` (map).
- Canal de collision : *Visibility* (ou créer un canal de trace `Lidar` dédié dans les Project Settings).
- **Avant chaque jalon : sauvegarder le projet et commit Git** (le MCP modifie des assets en direct).

---

## 11. Consigne à l'agent (Claude Code)

> Si un outil MCP échoue ou ne couvre pas une étape (le MCP est *expérimental*), **ne bloque pas** :
> propose soit (a) l'implémentation **en C++** (écris les fichiers `Source/…`, configure le module,
> compile), soit (b) la **procédure manuelle exacte** (nœud par nœud) à réaliser dans l'éditeur.
> Avance jalon par jalon, et **demande une validation** après chaque jalon.
