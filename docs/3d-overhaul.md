# 3D Realism Overhaul — Audit, Decisions & Change Log

Scope: rendering/visual pipeline of the Kommilo 3D campus (`index.html`, module script).
App logic, UI, routing, state and interactions are untouched. Verified after every
phase with a headless-Chromium harness (screenshots day/night, console capture,
14-step interaction regression, draw-call accounting).

---

## 1. Audit (Step 0)

**Stack detected:** vanilla Three.js **0.169** from the jsdelivr CDN via import map,
single-file app (`index.html`), no build system, no TypeScript, no React Three Fiber.
Decision: build on this stack with `three/addons` only — no framework swap, no new
dependencies (zero-tolerance rule 7). The brief's drei/R3F references map to their
three/addons equivalents (`Sky`, `GTAOPass`, `UnrealBloomPass`, `ShaderPass`+`VignetteShader`).

**Reference photos analyzed** (attached to the session; the real Studierendenhaus TU
Braunschweig — the same building the scene models):

| Photo | Content | Extracted target values |
|---|---|---|
| Autumn dusk exterior | deep navy sky, warm ~3500–4500 K interior glow, gold curtains, leaf litter | night sky = blue hour (not black), warm interior key, cool/warm contrast |
| March + Jan interiors | soft bright neutral daylight, white ribbed deck, grey carpet, yellow accents | day = soft warm sun + high ambient, no harsh contrast indoors |
| July dusk after rain | saturated blue-hour sky, glowing facade, wet asphalt sheen | asphalt roughness variation, night bloom moderate, navy fog |
| January night | building radiating from inside, hedge silhouettes | night: interior carries the scene, environment nearly dark |

**Diagnosis — gaps vs. the references (baseline screenshots in the PR):**

1. **No working cast shadows** — the shadow camera frustum was assigned via
   `Object.assign` but `updateProjectionMatrix()` was never called, so the ortho
   frustum silently stayed at its ±5 m default (pre-existing bug).
2. **Indoor `RoomEnvironment` used as IBL for an outdoor scene** → wrong ambient
   color and "plastic" flatness.
3. **No sky** — a CSS gradient behind a transparent canvas; fog color mismatched it.
4. Linear fog with near-white color; no atmospheric depth.
5. **No ambient occlusion** (a comment claimed SAO; only bloom existed).
6. Night mode: bloom strength 1.05 blew the facade into a white blob; sky pitch black.
7. Composer path had no AA (canvas MSAA does not apply to post-processed rendering).
8. No normal/roughness maps anywhere; visible texture tiling on lawn/pavers.
9. No wind, no particles, no camera life; leaf cards read as static plastic.
10. Magic numbers scattered through the file; scene globals leaked via `window.__*`.
11. Table/partner rebuilds leaked GPU geometries/materials.

Scale check: the scene is already 1 unit = 1 m (people ≈ 1.7 m, storey 3.3 m,
trees 7–13 m) — no rescale needed. Camera FOV 35° already cinematic.

---

## 2. What was implemented (phases A–D)

**A — Renderer, lighting, materials**
- Explicit `outputColorSpace = SRGBColorSpace`; ACES filmic with per-mode exposure.
- `Sky` dome (Preetham) with a `skyScale` luminance uniform; the CSS gradient is now
  only a fallback. Sun disc/azimuth/elevation shared by key light, sky and specular.
- IBL: PMREM generated from the same sky **with the solar disc stripped from the
  shader** — direct sun comes only from the shadow-casting `DirectionalLight`.
  (Leaving the disc in creates a shadow-less second sun that flattens everything.)
- Shadow fix + 2048 px map, tight ±46 m frustum, `bias -0.0004`, `normalBias 0.025`.
- Layered light rig per mode: warm sun key / cool moon key, hemisphere bounce,
  faint opposite fill, sky IBL. No uniform AmbientLight.
- Full material pass: per-material `envMapIntensity` (config table), procedural
  Sobel normal maps for lawn/pavers/bark/wood/carpet/asphalt (zero new payload),
  asphalt roughness variation (wet July-photo sheen), albedo clamped below pure
  white, leaf-card backface translucency.
- Dead `Water` import removed; `window.__bloom/__water*/__stL*/__bulbMat/__dlMat/
  __terrasseLights` globals replaced by module scope + one documented handle.

**B — Atmosphere, camera, post**
- `FogExp2` color-matched per mode (hazy blue-grey day / navy blue-hour night).
- Post chain: `RenderPass → GTAO → UnrealBloom → OutputPass → Vignette`, composer
  target with 4× MSAA (WebGL2). Bloom per mode (day whisper, night warm glow).
- Camera: damping 0.05, config near/far, subtle "breathing" drift applied as a
  bounded delta so it can never accumulate or fight the controls.

**C — Vegetation & life**
- Shared foliage shader hook: per-instance wind phase from instance world position;
  tip-weighted sway for grass/hedges, rigid-cluster sway for tree/shrub leaf cards.
- World-space macro-noise brightness variation on lawn + pavers (kills tiling).
- Particles (140 total, budget < 200): fireflies at night, dust motes in the hall by
  day, falling leaves under the park trees. Two `Points` + one `InstancedMesh`,
  all buffers preallocated — zero per-frame allocations.

**D — Performance**
- Quality tiers (high/medium/low): pixel ratio 2 / 1.5 / 1.1, shadow map 2048/2048/1024,
  GTAO on / on-half-res / off. Coarse-pointer (mobile) starts at medium.
- FPS monitor: drop below 45 fps (fast), raise above 57 fps (slow, 3 good windows,
  session ceiling prevents ping-pong); outlier frames after tab-backgrounding are
  discarded. Measures on every render path (campus, theme worlds, effects off) so
  a struggling HiDPI device always recovers. Tier changes propagate the pixel
  ratio to the composer (`composer.setPixelRatio`) — its cached ratio otherwise
  only reflects construction time.
- Rebuilds now dispose geometries + per-instance materials (shared palette kept).
- Static building/park world matrices computed once, then frozen.

---

## 3. sceneConfig — the tuning table

Everything visual lives in the `sceneConfig` object at the top of the module script,
one comment per value. Live-tune in DevTools:

```js
__kommilo3d.config.modes.nacht.bloom.strength = .7
__kommilo3d.applyMode()          // re-apply the active mode
__kommilo3d.setQualityTier('low') // force a quality tier
__kommilo3d.tier()               // current tier
__kommilo3d.info()               // draw calls/triangles of the last frame
```

Key groups: `modes.tag` / `modes.nacht` (exposure, sun/sky, hemi/fill, fog, bloom,
interior emissives, water), `quality` (tiers + fps thresholds), `shadow`, `ao`,
`vignette`, `wind`, `particles`, `materials` (envMapIntensity table, normal-map
strengths, leaf translucency).

---

## 4. Decision log

| # | Question | Decision | Rationale |
|---|---|---|---|
| 1 | Brief assumes R3F/drei/postprocessing — repo is vanilla single-file three.js | Stay vanilla, use `three/addons` equivalents | Brief's own rule: detect the stack and build on it; rule 7 forbids dependency creep |
| 2 | "Type check + production build" don't exist here | Equivalent gate: ESM syntax check + headless-Chromium run with zero app console errors + screenshot + 14-step interaction regression, after every phase | Strongest available verification for a build-less app |
| 3 | HDRI file for IBL? (none in repo, CDN egress for .hdr blocked in this sandbox) | Generate IBL from the Preetham sky via PMREM, sun disc stripped | Zero payload, works offline with SW cache, guarantees sky/sun/reflection consistency |
| 4 | Preetham raw luminance is ~10× typical HDRIs | `skyScale` uniform on the visible dome + low `envIntensity` | Saturated photo-blue sky at ACES 1.12 and readable shadows |
| 5 | Sun disc in the IBL flattened all shadows | Strip `L0 += vSunE*19000*Fex*sundisk` from the env sky's fragment shader | Direct light must come only from the shadowed DirectionalLight |
| 6 | Missing shadows root cause | Fixed missing `updateProjectionMatrix()` (pre-existing bug) | Shadows are an acceptance criterion; fix, not workaround |
| 7 | three's `VignetteShader` darkness semantics differ from the postprocessing lib | darkness 1.22 (≈0.3 in lib scale) | At 0.38 the shader *lightens* corners toward grey |
| 8 | DoF pass? | Skipped | Marked optional in the brief; hurts label legibility over tables and mid-range fps; bokeh quality of the addon pass is poor |
| 9 | SMAA vs MSAA | 4× MSAA on the composer target | WebGL2 hardware AA, cheaper and cleaner than SMAA pass; brief's SMAA goal is "clean edges" |
| 10 | Terrain height undulation | Skipped | Hardscape overlays (apron/street/paths/river) sit at fixed y — displacement would make them float/clip; tiling breakup handled by macro-noise + blades + litter |
| 11 | Blob contact shadows under trees | Skipped | GTAO + working 2048 px shadows already ground objects; blobs would double-darken |
| 12 | Idle camera drift vs "interactions untouched" | Bounded breathing delta (±5 cm); autoRotate semantics left exactly as before | Satisfies "subtle life" without changing control behavior |
| 13 | Wind on leaf shadow (customDepthMaterial) and on the GTAO g-buffer | Skipped | Leaf shadows are soft blobs and AO is low-frequency; at the tuned ≤16 cm sway the static-shadow mismatch is imperceptible, while swaying depth/normal variants cost extra shader permutations and a re-rendered g-buffer hook |
| 14 | Old `window.__*` scene globals | Replaced by module consts + one documented `window.__kommilo3d` handle | Same-module communication needs no globals; a single tuning handle is the owner's requested tuning table |
| 15 | CDN photo-texture swap (grass/wood) kept? | Kept, now syncs normal-map repeats | Production users can reach jsdelivr; canvas fallback covers failure |
| 16 | 60 fps on mid-range laptops — verification limit | **Gap, recorded:** this sandbox only has SwiftShader software GL (~1 fps), so real-GPU fps could not be measured in-session | Mitigated with measured draw-call accounting (high ≈5.4k calls, low ≈2.7k), adaptive tiers, frozen matrices, zero per-frame allocations |
| 17 | Fog implemented in Phase A instead of B | applyMode/config rewrite owns those lines | Avoids double-editing the same region; phase split is priority order, not a wall |
| 18 | Theme-world scenes (globe/avatar/docs/…) | Untouched except shared renderer exposure differences | They are stylized UI backdrops, not the photoreal campus; kept minimal-risk |
| 19 | „Haupteingang an der rechten Seite" — rechts von wo? | Vom kanonischen Eingangsblick V (App-Standard, Assertions-Blickpunkt): rechts = Osten → Altgebäude ry=+90°, Portale/Risalit/Freitreppe nach Osten | V ist der einzige im Projekt definierte Referenzblick; Assertion „Portale nach Osten" machinell prüfbar |
| 20 | Nutzer nennt den weißen Modernebau „Audimax" und will ihn gegenüber dem Altgebäude-Eingang — Konflikt mit der früheren bindenden Ordnung „Bibliothek auf der Gegenseite" | Neuere Nutzeranweisung ersetzt die alte: Bau nach (47,−60) verlegt, `SITE.lib`→`SITE.audimax`, Assertion 6 ersetzt durch Portal-Richtung + „Audimax östlich, achsnah" | Konfliktreihenfolge Korrektheit > Treue; die jüngste explizite Anweisung ist der gültige Vertrag (deckt sich mit der realen Lage am Forumsplatz) |
| 21 | „Echtes Wasser": Reflexions-Rendering kostet einen zweiten Szenen-Render | three/addons `Water` (planare Echtzeit-Spiegelung), 512er-Target, Update nur jedes 2. Frame (`sceneConfig.water.reflectEveryN`), in `AO_EXCLUDE` gegen Override-Material-Leaks im GTAO-Pass | Gemessen: amortisiert +301 Draw Calls/Frame (Spiegelpass ≈600 Calls dank Frustum-Culling) — vertretbar für den größten Realismusgewinn am Fluss |
| 22 | Reflector spiegelte an einer senkrechten Ebene (Ribbon-Geometrie lag in XZ) | Eigener XY-Ribbon + Mesh-Kippung um −90° wie bei PlaneGeometry | Reflector leitet die Spiegelebene aus der Mesh-Rotation ab (Geometrie-Konvention: Normale +Z) |
| 23 | `waternormals.jpg` vom CDN lud nie (npm-Paket von three enthält `examples/textures/` nicht — vorbestehend) | Prozedurale, kachelbare Wellen-Normalmap aus Sinus-Oktaven (CanvasTexture) | Null Payload, funktioniert offline und im Sandbox-Harness; entfernt eine tote Remote-Abhängigkeit |
| 24 | „Töpfe für alle Bäume" auch auf Rasen/Ufer? | Ja — `treePit(x,z,s)` wird in `tree()` selbst aufgerufen, Kantenmaß skaliert mit Baumgröße | Explizite Nutzeranweisung („für alle Bäume"); einheitlicher urbaner Look |
| 25 | Master-Prompt fordert „~10–14 geparkte Autos" am Parkplatz — direkter Widerspruch zur vorigen Nutzeranweisung „ich brauch keine Autos in meiner App" | Parkplatz mit markierten Stellflächen, aber OHNE Autos; prominent im Report geflaggt + Ein-Zeilen-Reaktivierung angeboten | Konfliktreihenfolge: die persönliche, emphatische, jüngste Eigenaussage des Nutzers wiegt schwerer als die Realismus-Zeile eines Struktur-Prompts; trivial reversibel, kein Assertion-Bruch (Checks prüfen die Lage, nicht Autos) |
| 26 | Okerhochhaus-Fassade: flach bemaltes Fensterraster vs. echtes Relief | Waffelraster als echte proud-Geometrie (instanzierte Bänder+Pfosten) über zurückgesetzter Scheibe → echte ~0.35 m Laibung | Master-Prompt: „flat painted windows are a failed review"; Relief/Verschattung macht die Fassade real; Instancing hält die Draw Calls niedrig |
| 27 | Machine-Check „0 Fenster auf Schmalseiten" — Gefahr einer Tautologie | Traverse über `userData.glaz`-Meshes mit `|z|<|x|`; Scheiben nur auf ±Z gebaut → strukturell 0, aber der Check liest echte Kindobjekte statt einer Konstante | Prüft die reale Szene: würde jemand versehentlich eine Scheibe auf ±X legen, schlägt er an |

**Dependencies added: none.** (Only `three/addons` modules from the already-used three@0.169.0 CDN package: `Sky`, `GTAOPass`, `ShaderPass`, `VignetteShader`, `Water`.)

---

## 5. Acceptance criteria — status

| Criterion | Status |
|---|---|
| Side-by-side matches references (light direction, palette, atmosphere) | Met — verified against day/night screenshots; night = blue-hour navy + warm interior (photos 1/4/5), day = soft warm sun (photos 2/3) |
| Objects sit on the ground (AO + shadows) | Met — GTAO + fixed 2048 px sun shadows (verified in screenshots) |
| Distant geometry fades into haze | Met — FogExp2 per mode, city backdrop softens |
| Sun direction consistent (shadows/sky/specular) | Met — single azimuth/elevation source drives light, sky shader and IBL |
| No plastic uniformity | Met — normal maps, roughness variation, per-instance HSL jitter (existing) + macro-noise + translucent leaf backfaces |
| Subtle constant life | Met — wind (per-instance phase), 140 particles, camera breathing |
| Stable 60 fps desktop | **Partially verifiable** — see decision log #16; tiers + measured call counts + no per-frame allocs; real-GPU validation needs the owner's hardware |
| App functionality 100 % untouched | Met — 14-step interaction regression passes; only rendering code changed |

## 6. Content-upgrade round (owner feedback on close-ups)

- **People v2**: rebuilt seated/standing figures — connected joints, real shoulder/hip
  widths, rounded shoes with soles, neck/collar transition, 6 skin tones, hair styles
  (crop, bob, bun, fringe) with **two blonde tones (~1/3 of seeds)**, glasses,
  student backpacks, separate shirt/pants palettes (no more "naked mannequin" look).
- **Stairs bug fixed**: all three flight builders had tread rotation inverted — the
  1.15 m tread edge ran *along* the flight, so steps floated as narrow planks between
  the stringers. Treads now lie across the flight and rest on the stringers.
- **Trees**: plane-tree bark (mottle + deep fissures + ridge highlights, corrected
  v-repeat), root flare at the trunk base, 10-segment branches.
- **Hedge**: leafy noise albedo + relief + near-white per-segment color jitter +
  instanced leaf fringe on top/faces — no more flat green boxes.
- **Facade grasses**: new arched, tapering blade texture, crossed cards per tuft,
  translucent backfaces (also applied to lawn blades — no more black backlit chips).
- **Leaf litter/falling leaves**: real leaf silhouette with veins + stem (alpha card)
  instead of bare rectangles.
- **AO ghost fix (pipeline)**: GTAO renders its g-buffer with an override material
  that cannot alpha-test, so foliage cards produced square AO ghosts; foliage is now
  hidden during the two AO g-buffer renders (also removes 42k grass instances from
  those passes — a measurable win).

## 7. Site plan (Street-View-Abgleich) — Vertrag für den Umgebungs-Umbau

Ground truth: die fünf Street-View-Screenshots (März 2022, Via Dentis / Abt-Jerusalem-Straße).
App-Koordinaten, 1 u = 1 m, +z = Oker-/Naturseite, −z = Campus-Süden.

```
                +z  Oker (Fluss z≈39, Steg, Ufer-Hecken z≈22.6, dichter Baumbestand)  — bestehend
   ┌──────────────────────────────────────────────────────────────────────┐
   │   STUDIERENDENHAUS (0,0), 32×28, Pflaster-Apron bis ±19/±17          │
   │   Abt-Jerusalem-Straße: Asphalt N-S bei x≈27 (Piktogramme, Parken)   │
   ├── Via Dentis: Klinker O-W, z −19.3…−25.7, x −14…30, Granitborde ─────┤
   │ PLAZA (Klinker-       │ ALTGEBÄUDE x 11…25, z −29…−91                │  Gelbes Giebelhaus (34,−27)
   │ Promenade) x −8…11,   │ Langachse N-S, SCHMALSEITE z=−29             │  Turm 15 Gesch. (44,−42)
   │ z −29…−91, Baum-      │ gegenüber dem Eingang, Portale → West        │  Parkplatz x 31…40, z −30…−44
   │ scheiben-Reihe        │ (Plaza-Seite)                                 │  (geschwungene Hecke am Rand)
   │ AUDIMAX (−26,−58) Front → Ost (Plaza)                                 │
   └──────────────────────────────────────────────────────────────────────┘
Eingangs-Blick (von ~(2,1.7,−14) nach Süden): Altgebäude-Schmalseite gegenüber, Langfassade
läuft nach Süden die Plaza entlang, gelbes Giebelhaus + dunkler Rasterturm hinten-links (Osten),
weiße Audimax-Kulisse diagonal rechts (Westen).
```

Zusätzliche Entscheidungen: Gerüst + blauer Aufzugsturm des realen Altgebäudes werden NICHT
modelliert (Bauzustand, nicht Gebäude); Altgebäude-Tiefe bleibt bei den modellierten 14 m
(real ~40 m — Remodel des Detailbaus nicht gerechtfertigt, Proportion der Schauseiten stimmt);
Schattenkamera (±46 m) deckt Straße/Plaza-Anfang — fernes Altgebäude-Ende bleibt als Kulisse
außerhalb des Schattenkastens.

## 8. Umsetzung Site-Umbau + Asset-Fixes (Change Log)

**Menschen v3 (Task B):** alle aufgemalten Gesichtszüge entfernt (ein abstraktes Stil-Level
überall); Kopf kleiner (~7,5 Kopfhöhen), Schultern > Hüfte, Arme enden Mitte Oberschenkel
(asymmetrische Beugung), Standasymmetrie + Idle-Sway/Kopfdrift (Registry, bei Rebuilds
bereinigt; Figuren vom Matrix-Freeze ausgenommen — Kinder unter eingefrorenen Eltern).

**Hecken (Task B):** `hedgeRun()`/`buildHedges()` — 4 vertex-verrauschte Box-Varianten
(wellige Silhouette, gerundete Deckelkante), Vertex-AO zur Basis, 3 cm versenkt, 14 Blatt-
karten/Segment inkl. Ausreißer oben; globale Sammel-Instanzierung (4 + 1 Draw Calls gesamt).

**Gras-Cluster (Task B):** 6–10 einzeln gebogene Halm-Geometrien pro Cluster (Taper + Bogen,
Vertexfarbverlauf, aufwärts geblendete Normalen), Auswärtsneigung, Wind pro Halm; jede
Gruppe verwurzelt (Steinring auf Belag + Erdscheibe). Rasen-Einzelhalme (42k-Feld) bleiben
als eigenes Fernsystem.

**Site-Umbau (Task A)** — siehe Site-Plan in §7:
- Altgebäude um 90° gedreht/versetzt (18,0,−60, ry=−90°): Schmalseite z=−29 gegenüber dem
  Eingang, Portale/Freitreppe nach Westen zur Plaza; Pavillon-Ostfenster ergänzt (nach der
  Drehung straßensichtbar).
- Via Dentis (Klinker 44×6,4 m, Granitborde), Plaza-Promenade (19×64 m Klinker) mit
  Baumscheiben-Reihe + Pollern, Parkplatz (Asphaltfläche, 4 Autos) mit geschwungener Hecke,
  2 Straßenrand-Autos, Fahrrad-Piktogramme auf der Asphaltstraße, Bügel + 4 Räder vor der
  Altgebäude-Schmalseite.
- Kulisse: gelbes 2-geschossiges Giebelhaus (38,−28.5), 15-Geschoss-Turm mit gebackenem
  dunklem Fensterraster-Shader statt Einzelband-Meshes (44,−42, 50 m), weiße Audimax-Kulisse
  (−26,−58) mit zurückgesetztem Glas-EG, Stützenreihe, auskragendem OG + dunklem Band.
- Vegetation urban: Süd-Bäume in Baumscheiben an Via/Plaza verlegt, dichter Bestand nur zur
  Oker-Seite; Falllaub-/Partikel-Anker nachgezogen; Halmfeld-Ausschlüsse für Via/Plaza/Parken;
  Poller-Südreihe vor die Straße gezogen, Markierungslinien enden vor der Kreuzung.
- Boden auf 180×210 erweitert (Altgebäude reicht bis z=−91).

**Messung:** high-Tier 5 322 Draw Calls (vor Umbau 5 432 — Foliage-AO-Ausschluss kompensiert
die Kulissenbauten), 1,44 M Dreiecke (+0,34 M); 14/14 Interaktionstests, 0 Konsolen-Fehler.
Verbleibende Abweichung (dokumentiert): Altgebäude-Modelltiefe 14 m statt realer ~40 m;
Schmalseiten-Proportion dadurch schlanker als im Foto — Remodel des Bestands-Detailbaus
wäre unverhältnismäßig.

## 9. Delta-Tabelle Foto↔Render (Fidelity-Pass) — vorher erhoben, nachher aufgelöst

| # | Delta (präzise) | Schwere | Status |
|---|---|---|---|
| 1 | Turm: real helles Beton-/Alu-Raster mit dunklen eingesetzten Scheiben, ~17 Geschosse, schlankes Dachvordach; Render: dunkle Noise-Platte | schwer | **fixed** — deterministisches 17×9-Gitter (helles Skelett, dunkle Scheiben, wenige erleuchtet, Brüstungsband), auskragendes Vordach |
| 2 | Gebäudeordnung ab Blickpunkt V: real Turm ganz links hinten, gelbes Haus davor-mittig, Altgebäude rechts; Render gespiegelt | schwer | **fixed** — Turm (−52,−50), gelbes Haus (−20,−33.5), Altgebäude unverändert (18,−60); 7/7 Assertions PASS (Output im Change Log) |
| 3 | Bibliothek: stand auf der Landmarken-Seite | schwer | **fixed** — (24,0,62) hinter der Oker, Front nach Süden; Assertion „Gegenseite/hinter V" PASS |
| 4 | Figuren: starre ungelenkige Gliedmaßen, Stumpf-Enden, Schweben/Clipping, Gleiten statt Gehen | schwer | **fixed** — Gelenk-Hierarchie (Schulter/Ellbogen/Hüfte/Knie/Knöchel), Fäustling+Daumen, echte Schuhe; Sitzpose mit korrekten Kontaktpunkten (Gesäß .455, Füße flach, Unterarme auf .74), Stehtisch-Lehnen, 5 Gehende mit echtem Gangzyklus (Beinwechsel, Kniebeuge, Armgegentakt, Körperhub), Gesprächsgruppe mit Geste; Atmen/Kopfdrift/Tippwellen, Timings pro Figur randomisiert; Stehtisch-Clipping behoben; ~33 Figuren < 40-Budget |
| 5 | Tagesnebel: weißer Vorhang ab ~100 m | mittel | **fixed** — Dichte .0038 → .0026, Resthaze bleibt |
| 6 | Bäume: ein Lollipop-Archetyp | mittel | **improved** — 3 Kronen-Archetypen (rund/hoch-oval/geschichtet über Zwischenetagen-Cluster), frühere Verzweigung, Stammgabel ~50 % bei Typ 2, breiterer Hue-Jitter; Restabweichung: Kronendichte unter Foto-Niveau (Instanz-Budget), dokumentiert |

**Charaktersystem-Entscheidung (1a):** Option B (prozedurale Gelenk-Skelette). Begründung: keine
in dieser Umgebung beziehbare, lizenz-verifizierbare CC0-GLB-Rigging-Bibliothek (npm-Registry
erreichbar, aber kein kuratiertes geprüftes Paket); Option B hält Payload bei 0 MB, garantiert
EIN Stilisierungslevel und erfüllt Anatomie-/Posen-Spezifikation vollständig.

**Messung Fidelity-Pass:** high-Tier 6 112 Draw Calls (+790 durch ~33 Gelenk-Figuren à ~25 Meshes),
1,69 M Dreiecke; SwiftShader-Frametime sank dennoch (2 781 ms vs 2 957 ms — kleinere Foliage-Last).
14/14 Interaktionstests, 0 Konsolen-Fehler. Echte 60-fps-Validierung weiterhin nur auf realer
GPU möglich (dokumentierte Umgebungsgrenze).

## 10. Site-Korrekturen v2 + Asset-Korrekturen (Nutzerfeedback 3, Change Log)

Nutzeranweisungen (Foto „3 Via Dentis" + App-Screenshot) und Umsetzung:

- **Turm halb so breit:** Scheibe 16×52×13 → **8×52×11**; Fassade pro Breite eigene Bay-Zahl
  (Material-Array: Schmalseite 4, Breitseite 6 Bays à ~2 m — Fenster verzerren nicht mehr);
  Dachvordach/Technikaufbau mitskaliert.
- **Parkplatz zwischen Studierendenhaus und Altgebäude, ohne Autos:** 16×23-m-Asphaltfeld
  (3,−38.4) mit 2×9 markierten Stellflächen (instanziert, 1 Draw Call), Einfahrt von der
  Via Dentis, Heckenfassung West + Süd; `car()`-Funktion + alle 6 Autos entfernt.
- **Tür des Studierendenhauses** vom Nordglas zur **Südfassade** verlegt (gegenüber dem
  Parkplatz); Nordseite bleibt Terrassenausgang ohne Portalrahmen.
- **Altgebäude-Haupteingang rechts (Ost):** ry −90° → **+90°** — Portale/Risalit/Freitreppe/
  Laternen zeigen nach Osten, die Westfassade grenzt an den Parkplatz (wie im Nutzerfoto).
- **Audimax statt Bibliothek am Fluss:** weißer Modernebau nach **(47,−60)** gegenüber dem
  Altgebäude-Haupteingang; Kolonnadenfront nach Westen, gefliester Vorplatz (12×26 m) dazwischen.
- **Autoweg → Fahrradstraße mit Fliesen:** Asphaltfahrbahn + Mittellinien-Dashes entfernt;
  3.6-m-Klinkerweg (paverMat) mit Kantstein, Fahrrad-Piktogramme bleiben, Lampen bleiben als
  Wegbeleuchtung; Gras-Ausschluss achsparallel nachgezogen, Bank von der Trasse an den Rand.
- **Keine gehenden Menschen:** `walker()` + 5 Pfade + Gangzyklus-Zweig in `updateLife`
  restlos entfernt (kein toter Code); Gesprächsgruppe vom Parkplatzmund auf den Plaza-Kopf.
- **Baumscheiben für alle Bäume:** `treePit(x,z,s)` in `tree()` integriert (Kante 1.1+0.45·s),
  auch Ufer- und Rasenbäume; separate Pit-Liste entfernt.
- **Echte Blätter:** Laubtextur v2 512 px — einzelne Blattspreiten (Spitze, Mittelrippe,
  5 Grüntöne) statt Ellipsen-Rauschen; wirkt auf Baumkronen, Sträucher und Hecken-Blattdecke.
- **Echtes Wasser:** three/addons `Water` mit planarer Echtzeit-Spiegelung (Ufer, Steg, Bäume,
  Nachtlichter spiegeln wirklich), prozedurale kachelbare Wellen-Normalmap, Tag/Nacht-Presets
  (`modes.*.water`), Drosselung + AO-Ausschluss (Details Decision Log 21–23).

**Assertions (Headless-Lauf, verbatim):** 8/8 PASS — Ordnung links→rechts, Turm-Distanz,
Betweenness, Langachse, Schmalseite, **Portale nach Osten**, **Audimax gegenüber (achsnah)**,
Oker-Seite. **Messung:** high-Tier 6 413 Draw Calls / 1,76 M Dreiecke (amortisiert +301 Calls
durch den gedrosselten Spiegelpass), low-Tier 3 588; 14/14 Interaktionstests, 0 Konsolen-Fehler.

## 12. Architektur-Datenblätter (CAD-Methode, aus den Referenzfotos abgeleitet)

Der Eigentümer verlangte explizit ein **Planungs-Datenblatt vor der Geometrie** — wie ein
Architekturbüro. Zuerst das Datenblatt, dann folgt die Umsetzung exakt, dann prüft QA die
gebaute Geometrie gegen Datenblatt UND Fotos.

### 12.1 Okerhochhaus (TU Braunschweig, Pockelsstraße)

| Parameter | Wert (Datenblatt) | Quelle/Begründung |
|---|---|---|
| Typ | Scheibenhochhaus (kein quadratischer Turm) | Foto 3: Schmalseite deutlich dünner als Langseite |
| Grundriss | 34 m (lang) × 11 m (kurz) = **3.09:1** | Foto 2 Fassadenbreite ~14 Achsen, Foto 3 Schlankheit |
| Höhe | ~53.8 m (EG 4.5 m + 16 Geschosse × 3.05 m) | Fotos: ~17 Bänder über zurückgesetztem EG |
| Achsraster Langseite | 14 Achsen × 2.43 m Modul | Foto 2/5: sehr regelmäßiges, enges Raster |
| Fenster | nahezu quadratisch (~2.0 × 2.05 m), dunkel spiegelnd | Bandhöhe 1.0 m + Pfosten 0.42 m → quadratische Öffnung |
| Laibungstiefe | **Rippenraster proud ~0.4 m** über zurückgesetzter Scheibe (~0.35 m Reveal) | Foto 2/3: tiefe Schattenkanten, Waffelrelief |
| Rahmenfarbe | warmes helles Betongrau `#c4c3bc` | Fotos: heller warmer Beton |
| Scheibe | dunkel `#2b333b`, roughness .2, metalness .5 | Fotos: dunkle, leicht spiegelnde Verglasung |
| **Schmalseiten** | **fensterlos**, kühleres/dunkleres Plattenraster `#a9adb2` mit Fugengitter | Foto 3: rechte Seite komplett geschlossen |
| Dach | **dunkles dünnes Vordach, allseitig ~0.9 m auskragend** `#33383d`; Technik + Antenne zurückgesetzt | Fotos 2/3: schwebende dunkle Kappenlinie = Signatur |
| EG | zurückgesetztes dunkles Glasband `#23282d` (Sockel, 0.5 m Rücksprung) | Foto 3: Scheibe „steht" auf dunkler Basis |

**Umsetzung:** Kernmasse `BoxGeometry(34,48.8,11)` mit Material-Array — Schmalseiten (±X) =
`endM` (Plattenraster), Langseiten (±Z) = `bodyM`. Pro Langseite eine dunkle Scheibenebene
(`glazM`, `userData.glaz=true`) bündig; davor kragt das Raster aus **echter Geometrie**:
instanzierte Brüstungsbänder (`Box(34,1.0,0.5)`, 17 Reihen ×2) + Pfosten (`Box(0.42,48.8,0.5)`,
15 ×2), Betonmaterial, proud bei z=±(SHORT/2+0.15). Dunkles EG `tbase` eingerückt, dunkles
Vordach `hhCap` `Box(35.8,0.55,12.8)` (+0.9 allseitig). Nur ~8 Draw Calls + 2 InstancedMesh.

### 12.2 Gelbes Giebelhaus

| Parameter | Wert (Datenblatt) | Quelle |
|---|---|---|
| Grundriss | 13 × 9 m | Foto 4/5 Proportion neben dem Turm |
| Geschosse | 2 (6.0 m Wand) + hoher Giebel (First ~8.6 m) | Foto 4 |
| Dach | Satteldach ~42°, dünne dunkle Traufkante, minimaler Überstand | Foto 4 |
| Fassade | ockergelb `#d3b061` | Foto 4/5 |
| Streifenbänder | Gruppen horizontaler dunkler Bänder `#6b5334` (2 Gruppen à 3) | Foto 4: braune Querstreifen |
| Fenster | dunkel gerahmte Rechtecke (2 Reihen × 4) | Foto 4 |
| Ost-Giebel | **Rundbogenfenster** nahe First + vertikales **Lamellenband** darunter | Foto 4/„1 Via Dentis"-Serie |

**Umsetzung:** `Box(13,6,9)` Ochre, Satteldach (2 geneigte Platten + 2 Giebeldreiecke), 6
instanzierte Streifenbänder auf der Südseite, gerahmte Fenster (Rahmen `frameM` + Scheibe
`winGlass`), Ost-Giebel `TorusGeometry`-Bogen + `CircleGeometry`-Scheibe + 5 Lamellen-Slats.

### 12.3 Fix 3 — Parkplatz

Lag zuvor **hinter/östlich** dem Altgebäude (falsch). Jetzt (bereits in v2) **zwischen
Altgebäude-Westfassade (x≈11) und Studierendenhaus**, Zentroid (3, −38.4), Einfahrt von der
Via Dentis, Rasen-/Heckenrand. **Entscheidung (Konflikt, Decision Log 25):** Der Master-Prompt
nennt „~10–14 geparkte Autos", die vorige Nutzeranweisung war jedoch emphatisch „ich brauch
keine Autos in meiner App". Die stärkere, persönliche Nutzerpräferenz gewinnt → Parkplatz mit
markierten Stellflächen, aber **ohne Autos**; auf Wunsch in einer Zeile reaktivierbar.

**Assertions (Headless, verbatim):** 13/13 PASS — die 8 Platzierungschecks plus fünf neue:
Seitenverhältnis 3.09:1 (≥2.8), Höhe 53.8 m (50–65), **0 Fenster auf Schmalseiten**,
Vordach 35.8×12.8 > Körper 34×11 (allseitige Auskragung), Parkplatz x=3.0 < Westfassade x=11
(nicht dahinter; liest jetzt die echte Mesh-Position). **Messung:** high-Tier 6 496 Draw Calls /
1,77 M Dreiecke (+83 Calls gegenüber v2 — das Waffelraster ist instanziert), low-Tier 3 636;
14/14 Interaktionstests, 0 Konsolen-Fehler.

**Adversarielles Multi-Agent-Review (7 Prüf-Dimensionen × Verifikations-Pass):** 2 bestätigte
Befunde gefixt — (a) **major**: gelbes Dach nicht wasserdicht (First schwebte ~0.67 m über dem
Giebelscheitel, da Neigung 0.72 rad fix statt aus Firsthöhe/Tiefe abgeleitet) → `pitch=atan2(GPEAK,GD/2)`,
`slope=hypot(GD/2,GPEAK)`, GPEAK 2.6→3.4; (b) **nit**: obere Fensterreihe ragte 7.5 cm über
die Wandkante in die Traufzone → Reihenhöhe gesenkt. Zusätzlich selbst gefunden+gefixt: Streifenbänder
lagen im Welt-Instancing statt lokal (Hausdrehung −0.12 nicht mitgemacht) und die Parkplatz-Assertion
las ein hartkodiertes Literal (Tautologie) → beide behoben. Massing, Waffel-Laibung, fensterlose
Schmalseiten, Dachvordach/Sockel und die Assertions selbst wurden als korrekt bestätigt.

## 14. Satelliten-verankerter Site-Plan + Altgebäude-Südfassade (definitive Spezifikation)

### 14.1 Pflicht-Fragebogen (VOR Code beantwortet — Abschnitt 8 der Spezifikation)

1. **`sceneFromSite(E,N)`** = `(x = E, z = −N)`. Osten→+x, Norden→−z. Begründung: die
   Landmarken-Gruppe liegt bereits bei −z, die Straße (Via Dentis) bei z≈−22 = Nord der
   Pavillon-Kante; konsistent mit dem bestehenden Aufbau, minimale Achsen-Umdeutung.
2. **B1** aktuell Welt (0,0,0), Grundriss 32 (x) × 28 (z) m (W=32, D=28). Anker, unberührt.
3. **px/m:** Kalibrieranker ist B1 (~32 m in x). In dieser Sandbox ohne Pixel-Lineal; der
   Modell-Anker B1=32 m ist fix, der Plan ist darauf kalibriert (im Report als Grenze benannt).
4. **Ziel-Weltkoordinaten** (aus Abschnitt 3.1, via `sceneFromSite`):
   E1-Achse (0,−22) O-W L≥90 B8 (Radweg 2 m Nordseite); E2 (3,−44) 60(x)×28(z);
   E3-Baumreihe (3,−30) Spanne 60; B3 (−30,−28); B4 (−55,−45) 15(x)×45(z) Längsachse N-S;
   B2-Südfassade z=−62, Breite 44, Körper nach Norden Tiefe 54 (Längsachse N-S); B5 (5,+45) SÜD.
5. **Aktuell→Ziel-Deltas:** E2 (3,−38.4)16×23→(3,−44)60×28; E1 (8,−22.5)→(0,−22)+Radweg;
   B3 (−20,−33.5)→(−30,−28)+90° (First N-S); B4 (−52,−50)34×11 O-W→(−55,−45)15×45 N-S (90°);
   B2 (18,−60) N-S-Kurzseite-Süd→(3,−~89) Langfassade nach Süden; B5/Audimax (47,−60) NORD→(5,+45) SÜD.
6. **B4↔B2 Längsachsen-Winkel:** aktuell 90° (B4 O-W, B2 N-S) → Ziel 0° (beide N-S). B4 wird
   um 90° gedreht; B2-Körper N-S-lang → parallel.
7. **Alte E2-Objekte (bewegt/entfernt):** Park-Asphalt-Mesh, Stellplatz-Markierungen (instanziert),
   Parkplatz-Heckenreihen (HEDGE_RUNS-Einträge), Gras-Ausschlüsse — alle am neuen Ort neu gebaut.
8. **Boden-Restore alte E2 (3,−38.4):** die neue, größere E2 (x[−27,33] z[−58,−30]) überdeckt den
   alten Fleck; Restfläche z[−30,−26] → Rasen (Gras-Ausschlüsse neu gezogen), keine Reste.
9. **Fassaden-Achsen (Foto „1 Pockelsstraße"):** 7 Achsen je Geschoss, Gruppierung **2–3–2**
   (Foto: regelmäßige Bogenfenster-Arkade; verdeckte Achsen hinter Gerüst mitgezählt).
10. **Laibung:** EG-Fenster 0.30 m, OG 0.25 m, als Geometrie-Rücksprung (Glasebene hinter der
    Wandfläche, Gewände/Bogen proud).
11. **Fassaden-Kosten (geschätzt):** ~110 Meshes (7×2 Fenster × ~6 Teile + Dentils instanziert +
    Gesims/Attika); tatsächliche Draw-Call-Zahl im Change Log gemessen.
12. **Assertions:** Abschnitt 9 vollständig in `verifySitePlan()`, headless über
    `window.__kommilo3d.verifySitePlan()`.

### 14.2 Konfliktauflösungen (Decision Log 28–31)

| # | Konflikt | Auflösung | Begründung |
|---|---|---|---|
| 28 | F7 „keine Transform-Änderung an B2" ↔ Abschnitt 5 fordert 46-m-Langfassade nach Süden; B2 hatte 14-m-Kurzseite nach Süden | B2 neu aufgebaut, Langfassade nach Süden, Körper N-S-lang; Zentroid nahe Plan | Abschnitt 5 + Foto sind die Autorität („the photo wins"); eine 7-Achsen-Fassade ist auf 14 m unmöglich → F7-Literalfreeze schließt das Kernziel aus. F7-Absicht (B2 nicht verschleppen/zerstören) bleibt gewahrt |
| 29 | Frühere Anweisung „ich brauch keine Autos" ↔ Abschnitt 4.4 fordert „12–16 geparkte Autos" | Autos **hinzugefügt** (12–16, Farb-/Yaw-Jitter, 1–2 leere Buchten) | Die jetzige Spezifikation ist explizit und detailliert (Satellit zeigt Autos); jüngste ausdrückliche Anweisung gewinnt. Im Report geflaggt |
| 30 | Vorige Aufgabe „Audimax nördlich, gegenüber Altgebäude-Eingang" ↔ B5 „Universitätsbibliothek muss SÜD bleiben" (F8) | Weißer Modernebau → SÜD (5,+45) als B5-Bibliothek | F8 = „failed phase"; neueste bindende Spezifikation; entspricht realer Lage der UB |
| 31 | E4 Fluss: Registry „UNCHANGED — verify only" ↔ 3.1 beschreibt West/N-S-Verlauf | Fluss bleibt an aktueller Lage (z≈+39), Abweichung dokumentiert | „verify only" + Fluss-Verlegen liegt außerhalb des 3-Punkte-Scopes und würde Steg/Wasserspiegelung brechen |

### 14.3 Altgebäude-Südfassade — Datenblatt (Abschnitt 5)

Zonen (unten→oben, Gesamt ~18.5 m bis Gesims): Sockel/Rustika 2.4 m · EG 7.5 m · Zwischenband
0.6 m · OG 6.5 m · Gebälk/Gesims 1.8 m · Attika 1.0 m. Sieben Achsen **2–3–2**, schmale Pfeiler
1.3 m, breite Pfeiler 2.5 m mit vertieftem Blindpaneel. Fenster: Rechteck-Körper + **echter
Halbkreis-Bogen** (Radius = halbe Breite), Glasebene 0.30/0.25 m zurückgesetzt, Gewändeband,
Archivolte, Keilstein proud, Kämpferband, Sohlbank auf Konsolen, radiale + Gitter-Sprossen.
Gesims mit **echten instanzierten Dentils** + Corona 0.5 m Auskragung. Attika mit Krönungsblöcken.
Sandstein #c9bda6, ±5 % Blockjitter, dezente Verwitterungsschlieren unter Sohlbänken/Corona.

### 14.4 Ergebnis — Assertions (Headless, verbatim) + QA-Vergleich

**Abschnitt-9-Assertions — 12/12 PASS:**

```
PASS — Süd→Nord-Ordnung B1 < E1 < E3 < E2 < B2-Südfassade (B1 N=0 < E1 N=22 < E3 N=29 < E2 N=44 < B2-Südfassade N=62)
PASS — Parkplatz-Zentroid ≈ geradlinig nördlich von B1 (|ΔE| ≤ 12) (|ΔE|=3.0 m)
PASS — Parkplatz östlich des gelben Hauses (E2 x=3 > B3-Ostkante x=-25.5)
PASS — B4-Längsachse ∥ B2 (≤5°) und B4-Seitenverhältnis ≥ 2.8 (Winkel=0.0°, Verhältnis=3.00:1)
PASS — Null Fenster auf den N/S-Schmalseiten des Okerhochhauses (Scheiben auf Schmalseiten: 0)
PASS — Okerhochhaus-Rasterfassaden zeigen nach O/W (Rasternormale x=1.00)
PASS — Universitätsbibliothek südlich von B1 (B5 N=-45 < B1 N=0 (z=45))
PASS — Radweg-Piktogramme ≥ 4 (6 Marken)
PASS — E3-Baumreihe 6–8 Bäume (je in Baumscheibe) (7 Bäume)
PASS — B2-Südfassade: 7 Fensterachsen (7 Achsen (2–3–2))
PASS — B2-EG-Fenster Laibung ≥ 0.25 m (Laibung EG 0.30 m / OG 0.25 m)
PASS — Kein Parkplatz-Rest an alter Lage (E2 am neuen Zentroid z≈−44) (E2 z=-44.0 (alt −38.4))
```

**QA 6.1 — Top-Down vs. Satellit:** Altgebäude (Nord, Langfassade nach Süden) → Parkplatz
(2 Reihen O-W, 14 Autos, 2 leer) → Baumreihe + Hecke → Straße+Radweg → gelbes Haus (West,
Giebel Süd) → B1 (Süd); Okerhochhaus schlanke Scheibe West, Längsachse N-S. Alle Elemente
und Ordnungen aus Abschnitt 3 aufgelöst — deckungsgleich mit der Luftaufnahme.

**QA 6.2 — Fassade vs. Foto „1 Pockelsstraße":** 7 Achsen 2–3–2 ✔, Halbkreis-Köpfe über
Rechteck-Körpern ✔, Gewände+Archivolte+Keilstein+Kämpfer+Sohlbank je Fenster ✔, Blindpaneele
auf den breiten Pfeilern ✔, Dentil-Gesims mit echter Auskragung ✔, Attika mit Krönungsblöcken ✔,
0.25–0.30 m Laibung mit Schattenwurf ✔, Sandstein-Blockjitter + Verwitterungsschlieren ✔.

**Verbotene Ausgänge F1–F12:** keiner produziert (Parkplatz korrekt zwischen B2 und B1/gelbem
Haus; B4 schlank N-S; echte Laibungen; echte Halbkreise; 7 Achsen; B1 & B2-Körper-Position
unberührt bzw. per Konflikt §14.2 dokumentiert; B5 südlich; alte Parkplatzfläche = Rasen;
Verwitterung vorhanden; Bäume in Scheiben; Radweg + Piktogramme).

**Messung:** high-Tier 6 874 Draw Calls / 1,86 M Dreiecke (+378 gegenüber der Vorstufe — die
7×2 Rundbogenfenster + 14 Autos; Dentils instanziert), low-Tier 3 858; 14/14 Interaktionstests,
0 Konsolen-Fehler. Kalibriergrenze: echte px/m-Messung der Luftaufnahme in der Sandbox nicht
möglich → Anker B1=32 m, dokumentiert.

## 16. Stone-True-Masonry + gelbes Haus (Neubau) + B5-Restore (Fixes A/B/C)

### 16.1 Pflicht-Fragebogen (VOR Code)

1. **Kurse je Zone (Nahfoto):** Zone 1 Sockel 5 Bänder × 0.5 m; Zone 2 EG 14 Kurse × ~0.52 m
   (Rustika, Läuferverband); Zone 3 OG 15 Kurse × ~0.43 m (glatter Werkstein). **Voussoirs:**
   EG-Bogen 13, OG-Bogen 11 (Keilstein = größerer Mittelkeil, leicht proud).
2. **Mauerwerk-Umsetzung (Hybrid):** echte Geometrie für alle Werksteine (Voussoirs, Keilsteine,
   Kämpfer, Sohlbänke, Eckquader, Dentils — instanziert) + **prozeduraler, NICHT kachelnder**
   Mauerwerks-Atlas (Canvas → Albedo + Normal, 1:1 auf die Südfläche gemappt, `repeat(1,1)`).
   Texturspeicher: 1× 1536×768 Albedo + 1× 1536×768 Normal ≈ 9 MB (RGBA) — im Budget.
3. **Kurs-Ausrichtung:** der Atlas erhält die Architektur-Höhen (Sohlbänke, Kämpfer, Gesims) als
   Snap-Linien; jede Kursgrenze rastet auf die nächste Element-Höhe → kein Block quert eine
   Sohlbank/Kämpfer/Ecke, keine sichtbare Wiederholung (ein Layout über die ganze Fläche).
4. **Gelbes Haus:** aktuell 13×9 Grundriss, First 9.4 m → Ziel **20×14**, First **12.5 m**
   (Skalierung ~1.5× Grundriss, First ×1.33), Traufe 7.5 m, Dach 40°.
5. **Fassade P (Parkseite, „3 Pockelsstraße"):** 6 Hauptfenster (obere Reihe 3× 1.3×1.5 m hoch
   unter dem Giebel, untere Reihe 3× 1.6×1.8 m) + 1 schmales Randfenster rechts; 2 Streifenband-
   Gruppen; 3 Kellerlichtschächte mit 3-Holm-Rohrgeländer, Pflaster-Apron; Giebelfeld schlicht.
6. **Fassade E (Eingang, „3 Via Dentis"):** rechter Fensterstreifen ~2 Spalten × 4 Reihen Scheiben,
   linker schmaler Streifen 1×4; 1 dunkle Tür + Oberlicht + Schildpaneel; obere Reihe **8 kleine
   Fenster**; brauner Bandstreifen über der Türzone.
7. **B5-Restore:** Commit **86dead5** → Transform `position (47,0,−60)`, `rotation.y = π/2`,
   scale 1. (Widerspruch: dieser Wert liegt NÖRDLICH von B1 und kollidiert mit Fix C.3 „südlich"
   sowie der F8-Regel der Vorrunde — Auflösung siehe Decision Log 32; F17 + Machine-Check „git-Wert
   ±0.1 m/1°" sind die dominante, maschinengeprüfte Vorgabe → git-Wert wird wiederhergestellt.)
8. **Draw-Call-Delta (geschätzt):** Fix A +~180 Voussoirs (instanziert je Bogen) — der Atlas
   ersetzt die gekachelte Textur (kein Extra-Draw); Fix B +~120 (Fenster/Streifen/Schächte).
   Ist-Werte im Change Log gemessen.

### 16.2 Decision Log (Fortsetzung)

| # | Konflikt/Frage | Entscheidung | Begründung |
|---|---|---|---|
| 32 | Fix C: git-Historie-Transform (47,−60, NORD) ↔ Fix C.3 „B5 bleibt SÜDLICH" ↔ Vorrunden-F8 „B5 Nord = fail" | B5 auf den **git-Wert (47,0,−60, π/2)** zurückgesetzt | F17 + Machine-Check verlangen ausdrücklich den git-Wert ±0.1 m/1° (dominante, maschinengeprüfte Vorgabe); „Restore" = exakt der historische Wert. C.3-„südlich" ist Prosa ohne Machine-Check; Widerspruch geflaggt, Ein-Zeilen-Alternative angeboten |
| 33 | „Gelbes Haus" ist real ein **modernes Plattenbau-Gebäude** (Keramik-Paneele), kein traditionelles Giebelhaus | Neubau als modernes verkleidetes Gebäude (Läuferverband-Paneele, Streifenbänder, Stehfalz-Metalldach + Solarfeld) | Fotos gewinnen; „3 Via Dentis"/„3 Pockelsstraße" zeigen eindeutig Keramik-Vorhangfassade |
| 34 | Mauerwerk-Kachelung (F13) | Ein prozeduraler Atlas über die ganze Südfläche, `repeat(1,1)`, Läuferverband + Snap auf Element-Höhen | „every stone visible, no tiling, no block crossing boundaries"; Kachelung/Wiederholung wäre F13-Fail |

### 16.3 Ergebnis — Machine-Checks (Headless, verbatim) + 3-Distanz-QA

**Neue/aktualisierte Checks — alle PASS:**

```
PASS — B5 auf git-Historie-Transform wiederhergestellt (47,0,−60, ry 90°) (B5 (47.0,-60.0) ry=90.0°)
PASS — B3 Grundriss ≥ 18×12 m (20×14 m)
PASS — B3 Firsthöhe 11–13.5 m (First 13.0 m)
PASS — B3 Fassade P: 6 Fenster (6 Fenster)
PASS — B3 Fassade E: 1 Tür + 8 kleine Oberfenster (Türen=1, kleine Oberfenster=8)
PASS — Altgebäude-Bögen aus radialen Voussoirs (13 EG / 11 OG, 7 Achsen) (7 EG-Bögen / 7 OG-Bögen à 13/11)
```
(zzgl. der 12 bestehenden Site-Checks → 17/17 PASS gesamt; verbatim im Report.)

**Fix A — 3-Distanz-Mauerwerk-QA (A.6):**
- **Fern** (über den Parkplatz): Kurse + Rustika-Bänderung lesbar, keine Kachelung/Wiederholung ✔
- **Mittel** (~20 m): Einzelblöcke + Voussoirs unterscheidbar, Fugen verschattet (Normal-Rillen) ✔
- **Nah** (~5 m): Blockvariation (gelegentlich dunkler/eisenfleckiger Block), Verwitterungsschlieren,
  radiale Keilsteine mit proud Schlussstein je Bogen ✔
- Ausrichtungsregel A.1.3 (Kursgrenzen auf Sohl/Kämpfer/Gesims gerastet): kein Block quert ein Element ✔

**Fix B — gelbes Haus:** modernes Plattenbau-Gebäude 20×14, First 13.0 m, Stehfalz-Metalldach +
Solarfeld; Fassade P (Nordgiebel) 6 Fenster + kleines Randfenster + 2 Streifenband-Gruppen +
3 Lichtschächte mit 3-Holm-Geländer + Pflaster-Apron; Fassade E (West/„Via Dentis") Tür +
Oberlicht + Schild + rechter Fensterstreifen 1.4×5.6 (2×4) + linker (1×4) + 8 kleine Oberfenster +
brauner Bandstreifen; Süd-Giebel Rundbogen-Lamellenmotiv; Ostseite schlichte Verkleidung.

**Fix C — B5:** Transform verbatim auf git-Wert (47,0,−60, ry 90°) zurückgesetzt (Decision Log 32).

**Verbotene Ausgänge F13–F18:** keiner produziert (kein Flach-Paint/Kachelung/grenzquerende
Blöcke; Bögen mit Voussoirs+Keilstein; gelbes Haus mit Paneelfugen/Streifen/korrekten Zählungen,
20×14 > 18×12, First 13 > 11; B5 = exakter git-Wert; nichts außerhalb Fix A–C berührt).

**Messung:** high-Tier 7 199 Draw Calls / 1,86 M Dreiecke (+325 gegenüber der Vorstufe —
14 Voussoirs/Bogen × 7 Achsen × 2 + gelbes-Haus-Fenster; der Mauerwerk-Atlas ist EINE Textur,
kein Extra-Draw), low-Tier 4 043; 14/14 Interaktionstests, 0 Konsolen-Fehler. Texturspeicher
Atlas ≈ 1536×594 Albedo + abgeleitete Normal ≈ 7 MB.

### 16.4 Glas-Verbindungsbrücke (Skywalk) Okerhochhaus ↔ Altgebäude

Nutzerfoto: ein grün getönter, aufgeständerter Glasgang verbindet die beiden Gebäude. Maße
real gehalten — die Gebäude bleiben weit auseinander; die Brücke überbrückt die **28.5 m-Lücke**
zwischen B4-Ostfassade (x=−47.5) und B2-Westfassade (x=−19) bei z=−64, aufgeständert auf
Erstgeschoss-Höhe (Deck y=5.0, Durchfahrt darunter). Aufbau: Alu-Deck + Untersicht, flaches
Dach, je Seite grüne Brüstungsglas-Streifen + grün getönte Verglasung (MeshPhysical, transparent),
Alu-Kämpfer/Rahmen + Pfosten alle 1.6 m, schlanke Rundstützen alle 5 m von Boden bis Deck. Die
Verglasung ist in `AO_EXCLUDE` (GTAO kann kein transparentes Glas). **Checks:** „Brücke verbindet
B4-Ost ↔ B2-West" PASS (x −47.5→−19), „Gebäude real weit auseinander (Spannweite ≥ 25 m)" PASS
(28.5 m). +169 Draw Calls (high 7 368 / 1,86 M Dreiecke), 14/14 Regression, 0 Konsolen-Fehler.

## 17. Runde 7 — keine Autos, Fliesenwege, Rundum-Lichterketten, größeres gelbes Haus, Audimax, komplettes Altgebäude

- **Keine Autos:** alle 14 Autos + Stellplatz-Markierungen entfernt; SITE.carN=0.
- **Fliesenwege:** Asphaltstraße + Parkplatz → paverMat (Fliesen wie um B1); E1 breiter Fliesenweg
  + Bordsteine, E2 gefliester Vorplatz.
- **Lichterketten rund um ALLE Seiten, stärker/lovely:** Festoons an allen 4 Balkonseiten (voll)
  + zweite tiefere Girlande Süd; 12 warme Perimeter-Punktlichter (an mc.terrasseIntensity);
  bulbEmissive Tag .05→.38, Nacht 2.8→5.2; terrasseIntensity Nacht 18→24; warmPoints 46→52.
- **Gelbes Haus größer/breiter/regelmäßig, Front-Eingang zu B1:** 24×16 (First 13.4 m), von der
  Straße entrückt (−30,−42); Haupteingang (verglaste Tür + Vordach + Stufe + Schild) am
  Süd-Giebel gegenüber dem Studierendenhaus; Nord-Giebel = 6-Fenster-Fassade (Foto), gleichmäßige
  Langseiten-Raster, Stehfalzdach + Solarfeld.
- **Audimax nach Foto:** offene EG-Kolonnade (Pilotis, dunkles zurückgesetztes Glas) + auskragendes
  verglastes OG (dunkle Vorhangfassade, vertikale Pfosten) + weiße Faszien + Flachdach.
- **Altgebäude komplett (nicht nur der linke Teil):** auf 64 m verbreitert, **11 Achsen (3–5–3)**
  mit Eckpavillon-Lesenen, alle Rundbogenfenster mit radialen Voussoirs; Brücke zur neuen
  Westfassade (x=−29) nachgeführt.

**Assertions (Headless, verbatim): 19/19 PASS** — u. a. „B2-Südfassade: 11 Fensterachsen",
„Altgebäude komplett: 11 Achsen, Bögen aus radialen Voussoirs (11 EG / 11 OG à 13/11)",
„B3 Grundriss 24×16 / First 13.4 m", „B3 Fassade E: 1 Tür + 8 kleine Oberfenster",
„Okerhochhaus schlanke N-S-Scheibe 3.00:1", „Glasbrücke B4-Ost ↔ B2-West (−47.5→−29)",
„Radweg-Piktogramme ≥ 4". 14/14 Interaktionsregression, 0 Konsolen-Fehler.

## 18. Runde 8 — Via Dentis, Audimax versetzt, gelber Eingang nach Foto 3, Turm+Brücke

- Via-Dentis-Fußweg (KEINE Autos) zwischen Altgebäude (Ost x=35) und Audimax: zentraler
  Klinkerweg + 2 seitliche Fußwege, Baumreihen, Poller, Hecken.
- Audimax nach Osten versetzt (66,-62), Glasfront nach WESTEN zum Weg; 22 m breiter Weg dazwischen.
- Gelber Haupteingang exakt nach Foto 3: 2 hohe Fensterstreifen (3x2 + 2x2), dunkle Doppeltür,
  Betonstufen + Metallgeländer-Rampe, rot-weißes Institutsschild, brauner Sockelstreifen.
- Okerhochhaus weiter NW (-64,-52); Brücke breiter/höher (bw 4.6, h 3.4), Spannweite 27.5 m,
  Lichterketten entlang beider Brückenkanten (wie am Studierendenhaus).
- 19/19 Assertions PASS, 14/14 Regression, 0 Konsolen-Fehler.

## 19. Runde 9 — Campus komplett: Altgebäude-Viereck, Audimax-Dach/-Ausrichtung, Bib/Forum, breitere Fliesenwege

Nutzerwunsch (Luftbild + Audimax-Fotos, „null Toleranz"): Campus komplett; Altgebäude als
komplettes Viereck (Innenhof) bis zum Okerhaus; Audimax mit **flachem weißem Dach** und Glasfront
**nach vorn/Süden** (nicht nach links/Westen); Fassade zwischen Bib, Audimax und Forum; breiterer
Hauptweg + echte Fußwege alles in Fliesen; gelbes Haus und Okerhaus **weiter weg** vom Altgebäude.

- **Altgebäude → Vierflügelanlage (B2).** Aus dem Vollblock wird ein Ring aus 4 Flügeln
  (Süd/Nord/Ost/West, Tiefe 14 m) um einen **offenen, gepflasterten Innenhof** — die Silhouette
  liest sich im Luftbild als „Viereck". Die reiche Süd-Schaufassade (11 Achsen, Voussoir-Bögen,
  Gebälk) bleibt unverändert. Gesimse/Sockel/Attika laufen jetzt als **Kantenringe** (4 Balken
  je Ebene) statt als Vollplatten → Hof bleibt zum Himmel offen. Hofseitige Fenster instanziert.
  Entscheidung: Außen-Fußabdruck (64×48, Zentroid 3,−86, Südebene z=−62) beibehalten, damit
  Nachbarabstände/Assertions gültig bleiben; die „Vervollständigung" ist der offene Ring + Hof.
- **Audimax neu ausgerichtet (B5).** Gruppe unrotiert bei (70,0,−64), 26×20 m. Weiße fensterlose
  Auditoriumsschale + verglastes EG-Foyer mit Pilotis-Kolonnade + hohes **Klerestoriumsband**,
  beide zur **Platzseite Süd (+z)**; Betonrippen an den Seitenwänden; gekrönt von einem
  **prominenten, allseitig auskragenden flachen weißen Dach** (Faszienband). Behebt „ohne Dach"
  + „falsche Richtung".
- **Bib + Forumgebäude (Kulisse).** Durchlaufende Platzfassade: Forum-Riegel (N-S) östlich am
  Audimax, Bibliothek-Riegel (E-W) nördlich, niedriger Verbindungsbau dazwischen — einfache
  Volumina mit gebackenem Glasraster (Textur) statt Einzelfenster-Meshes.
- **Wege.** Hauptweg auf **15 m** verbreitert, 2 Fußwege je 3 m, alle mit `paverMat`-Fliesen;
  Bordsteine an den Hauptwegkanten, Baumreihen + Poller in den Pflanzstreifen. Innenhof gepflastert.
- **Abstände.** Gelbes Haus B3 → (−32,−34) (weiter Süd/West, näher am Studierendenhaus,
  Nordkante jetzt ~16 m vom Altgebäude). Okerhaus B4 → (−70,−60) (weiter NW + nach Nord);
  Brücke nachgeführt: x0=−62.5→x1=−29, **Spannweite 33.5 m**, breiter/höher (bw 5.2, h 3.6).

**Assertions (Headless, verbatim): 22/22 PASS** — neu u. a. „Audimax: verglastes Klerestorium nach
Süden zum Platz", „Audimax: flaches weißes Dach vorhanden", „Altgebäude ist Vierflügelanlage mit
offenem Innenhof", „Breiter Via-Dentis-Weg … 22 m", „Glasbrücke … Spannweite 33.5 m".
14/14 Interaktionsregression, 0 Konsolen-Fehler/-Warnungen.

## 20. Runde 10 — Luftbild-Treue: offener Universitätsplatz + reale Abstände (2 Google-Maps-Fotos)

Nutzerwunsch (2 Luftbilder): (1) die Abstände Studierendenhaus ↔ gelbes Haus ↔ Okerhaus „100 %"
wie Luftbild 1; (2) die **leere Fläche zum Sitzen/Quatschen** zwischen Bib, Forum und Audimax
wie Luftbild 2.

- **Universitätsplatz (Luftbild 2).** Statt Bib/Forum als Blöcke am Audimax gibt es jetzt einen
  **offenen, gepflasterten Platz** (30×32 m, echte Fliesen), gerahmt von **Audimax (Süd),
  Bibliothek (Nord), Forumgebäude (Ost)** und nach **Westen zum Weg offen**. Möblierung: 4 Bänke
  nach innen, 2 Schattenbäume, mehrere stehende/quatschende Studierendengruppen (keine Läufer).
- **Audimax dreht sich zum Platz.** Gruppe um π gedreht (Position 70,−56): die verglaste Front
  (Foyer + Pilotis-Kolonnade + Klerestorium) blickt nach **Norden in den Platz** (Front-Normale
  z=−1.00), flaches weißes Dach bleibt. Die Betonrippen sind jetzt eine **lokale** InstancedMesh
  (drehen mit dem Gebäude, kein Weltkoord.-Bug mehr).
- **Reale Abstände (Luftbild 1).** Gelbes Haus „3a" als **enger SE-Nachbar des Okerhochhauses**
  neu gesetzt: (−44,−34), ~10 m Lücke zur Turm-Ostkante, Parkplatz östlich, weit vom Altgebäude —
  entspricht der Google-Maps-Anordnung (Turm+gelb im Westen, Parkplatz+Altgebäude im Osten).
  Okerhochhaus + Brücke unverändert (−70,−60 / Spannweite 33.5 m).
- **Weg.** Hauptweg 14 m + 2 Fußwege, alles Fliesen; der Ostfußweg (x=54) **mündet in den Platz**.

**Assertions (Headless, verbatim): 23/23 PASS** — neu: „Audimax: verglaste Front/Klerestorium
zeigt zum Platz (Norden −z)", „Offener Universitätsplatz (Audimax Süd / Bib Nord / Forum Ost,
Westseite offen) 30×32 m". 14/14 Interaktionsregression, 0 Konsolen-Fehler/-Warnungen.

## 21. Runde 11 — Altgebäude als große, unregelmäßige, OFFENE Anlage + Okerhochhaus als Turmeingang

Nutzerwunsch (detailliertes Luftbild des Altgebäude-Komplexes): das Altgebäude ist **viel
größer**, **nicht gleichmäßig** (unregelmäßige Längen), **kein komplettes Stück** — es ist von
oben **offen** und „endet am Okerhaus"; das **Okerhochhaus ist der tallste Teil + der Haupt-
eingang** (gegenüber dem Forum) und über die **Brücke** Teil des Campus.

- **Altgebäude → große offene U-Anlage.** Von 64×48 (geschlossenes Viereck) auf **76×58**
  vergrößert und geöffnet: nur noch **3 Flügel** (Süd-Schaufassade + Nord-Arm + tiefer Ost-Basis-
  Block „Zentral-Campus"), die **Westseite ist offen** → der Innenhof öffnet sich nach Westen zum
  Okerhochhaus. Unregelmäßige Tiefen (Süd 15 / Nord 13 / Ost-Basis 18 m). Südfassadenebene bleibt
  z=−62 (Parkplatzbezug unverändert). Schaufassade auf 11 Achsen über die breitere Front gespreizt;
  Gesims-/Attika-Ringe laufen nur S/N/O (West offen). Neu: **Architekturpavillon** (Glaskubus) im Hof.
- **Okerhochhaus = Turmeingang.** Neu positioniert (−60,−74), ragt nach Süden zur Straße; am
  Südfuß ein **verglaster Haupteingang mit auskragendem Vordach + Stützen** (tallster Teil, 54 m).
  Per **Glasbrücke** an die offene Westmündung des Altgebäudes angebunden (Spannweite 11.5 m).
- Gelbes Haus „3a" bleibt enger SE-Nachbar des Okerhochhauses; Parkplatz östlich.

Assertion-Anpassungen: „Altgebäude ist große offene U-Anlage (3 Flügel, West offen, 76×58)",
„Okerhochhaus tallster Teil (54 m) mit Haupteingang", Brücken-Spannweite ≥ 8 m (Anbindung, nicht
Trennung). Weltkoord.-Instanzen (Dentils, Hoffenster) auf Gruppenanker AX0/AZ0 umgestellt.

**Assertions (Headless, verbatim): 24/24 PASS**, 14/14 Interaktionsregression, 0 Konsolen-Fehler.

## 22. Files touched

- `index.html` — module script (rendering pipeline + scene content) + nothing else in the file
- `docs/3d-overhaul.md` — this document
