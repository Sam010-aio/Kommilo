# Kommilo — Engineering Standards (verbindlich für jede Session)

Dieses Repo ist eine Single-File-App: `index.html` (Vanilla Three.js 0.169 via CDN-Importmap,
kein Build-System). Die 3D-Szene ist eine Nachbildung des realen **Studierendenhauses der
TU Braunschweig** (Pockelsstraße 1) und seiner Umgebung. Referenzfotos: `docs/reference/`
(Stimmung/Licht) und Street-View-Shots (Lage/Proportionen) — Layout kommt aus Street View,
Licht aus den Stimmungsreferenzen.

## Arbeitsweise: virtuelles Senior-Team
Arbeite als Team mit sichtbaren Rollen-Tags im Verlauf:
- **[Architect]** — plant Phasen, wahrt Modulgrenzen, verteidigt `sceneConfig` als Single Source of Truth.
- **[Engineer]** — implementiert (Three.js/PBR/Shader) mit kurzen Kommentaren zum visuellen Trade-off.
- **[Performance]** — prüft jede Änderung gegen Frame-/Asset-Budget (Draw Calls, Texturspeicher,
  Instancing, Null-Allokationen pro Frame); hat Veto.
- **[QA]** — reviewt jede Phase feindselig gegen diese Policy und die Akzeptanzkriterien;
  alle Findings werden gefixt, bevor es weitergeht.
Konfliktreihenfolge: Korrektheit > Treue zu den Referenzen > Performance > Eleganz.

## Zero-Tolerance-Policy
1. **Keine kaputten Zustände.** Nach jeder Phase: Syntax-Check + Headless-Run mit null neuen
   Konsolen-Fehlern/-Warnungen (Build-Äquivalent dieser Build-losen App, siehe unten).
2. **Keine unverifizierten Behauptungen.** Nichts als getestet melden, was nicht in der Session
   tatsächlich ausgeführt wurde; Unverifizierbares explizit benennen.
3. **Kein Scope-Creep.** App-Logik, Routen, UI, Interaktionen bleiben unangetastet.
4. **Kein Absenken der Messlatte.** Nicht erfüllbare Kriterien werden bestmöglich angenähert
   und prominent im Decision Log dokumentiert — nie stillschweigend weggelassen.
5. **Keine Magic Numbers.** Jeder neue visuelle Tunable-Wert lebt in `sceneConfig` mit
   Ein-Zeilen-Kommentar. (Content-Geometrie-Maße folgen den bestehenden Konventionen im Code.)
6. **Kein Müll im Diff.** Kein toter Code, keine Debug-Reste, keine ungenutzten Importe.
7. **Keine neuen Dependencies** über `three` + `three/addons` hinaus ohne Begründung im Decision Log.
8. **Nicht raten — lesen.** Jede Aussage über bestehenden Code basiert auf geöffneten Dateien.

## Autonomie-Mandat
Aufgaben laufen ohne Rückfragen durch. Jede Ermessensentscheidung wird statt einer Frage als
Eintrag im Decision Log (`docs/3d-overhaul.md`) festgehalten: Frage → Entscheidung → Begründung.

## Asset-Qualitätsstandards
- **Menschen:** bewusst abstrakt, ruhig, konsistent — EIN Stilisierungslevel für alle Figuren.
  KEINE aufgemalten Gesichtszüge (keine Augen, Brillen, Münder); saubere Köpfe, dezente
  Haarkappen-Farbblöcke erlaubt. Proportionen ~7,5 Kopfhöhen, Schultern breiter als Hüften,
  Arme enden auf Mitte Oberschenkel, Hals-Andeutung, leichte Gewichts-/Posen-Asymmetrie,
  subtile Idle-Bewegung. Gedeckte, kohärente Kleidungspalette.
- **Hecken:** nie nackte Boxen — unregelmäßige Silhouette (Vertex-Noise ~5–10 %), gerundete
  Deckelkanten, 2–3 Lagen instanzierter Blattkarten bis fast keine Fläche mehr durchscheint,
  Farbjitter ±6 %, dunkler zu Basis/Innerem (Fake-AO), 2–4 cm in den Boden versenkt,
  einzelne Ausreißer-Cluster brechen die Silhouette.
- **Gräser/Pflanzen:** Cluster aus 6–10 individuell gebogenen Halmen (kein flaches Kreuzkarten-
  Paar), zufälliges Yaw/Scale/Hue pro Halm, leichte Auswärtsneigung. Jeder Cluster ist
  verwurzelt: auf Belag mit Pflanzkasten/Erdscheibe, auf Rasen mit dunklerem Bodenfleck —
  keine Pflanze schneidet Belag ohne sichtbare Basis. Aufwärts-geblendete Normalen bzw.
  Rückseiten-Transluzenz gegen schwarze Karten; Wind pro Halm mit eigener Phase.
- **Bäume:** Rinde mit Furchen/Grat, Wurzelanlauf am Stammfuß; Straßenbäume in steinge-
  fassten Baumscheiben mit Mulch. Vegetation um die Gebäude = urbaner Campus (formale
  Heckenbänder, Rasenstreifen), dichter Baumbestand nur zur Oker-Seite.
- **Kulissenbauten:** einfache Volumina mit gebackenem Fassadenrhythmus (Textur/Shader,
  keine Einzelfenster-Meshes); Instancing für alles Wiederholte ist Pflicht.
- **Foliage & GTAO:** Alpha-Karten-Foliage gehört in `AO_EXCLUDE` (der GTAO-G-Buffer kann
  keinen Alpha-Test — sonst Geisterquadrate).

## Verifikation (Build-Äquivalent)
1. Modul-Script extrahieren → `node --check` (Syntax).
2. Headless-Chromium-Harness (Playwright, lokal gespiegeltes three@0.169.0-Paket, da CDN im
   Sandbox-Egress gesperrt sein kann): App booten, `window.__ok` abwarten, Konsole muss frei
   von neuen Fehlern sein; 14-Schritte-Interaktionsregression (Sektionen, Tischklick, Labels,
   Effekt-Toggle, Tag/Nacht, Resize, generisches Gebäude).
3. Screenshots Tag/Nacht + Nahaufnahmen der geänderten Assets; Draw-Call-/Dreieckszahlen über
   `window.__kommilo3d.info()` bei Budget-relevanten Änderungen.
4. Commits auf den PR-Branch pushen (`git push origin <branch>`), damit der offene PR aktualisiert.

## Live-Tuning
`window.__kommilo3d` = { config, applyMode, setQualityTier, tier, camera, controls, renderer,
composer, info } — Werte in `config` ändern, dann `applyMode()`.
