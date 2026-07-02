# we learn together — Veröffentlichen in 5 Minuten (GitHub Pages)

## Schritt 1 — Repository anlegen
1. github.com → oben rechts **+** → **New repository**
2. Name: `we-learn-together` · **Public** · „Create repository"

## Schritt 2 — Dateien hochladen
1. Im neuen Repo: **„uploading an existing file"** anklicken
2. ALLE Dateien aus diesem Ordner hineinziehen:
   `index.html`, `manifest.webmanifest`, `sw.js`, `icon-192.png`, `icon-512.png`
3. Unten **„Commit changes"**

## Schritt 3 — Pages aktivieren
1. Repo → **Settings** → links **Pages**
2. Source: **Deploy from a branch** · Branch: **main** · Ordner: **/ (root)** → **Save**
3. Nach ~1 Minute steht oben die Live-URL:
   `https://DEINNAME.github.io/we-learn-together/`

Fertig — die App läuft öffentlich, mit HTTPS, und ist als App installierbar
(Handy: „Zum Startbildschirm hinzufügen" · Desktop-Chrome: Installieren-Symbol in der Adressleiste).

## Schritt 4 (empfohlen) — Gratis-Domain aus dem Student Pack
1. education.github.com/pack → Angebot **Namecheap** (1 Jahr .me gratis) oder **Name.com**
2. Domain sichern, z. B. `welearntogether.me`
3. Repo → Settings → Pages → **Custom domain** eintragen → bei Namecheap einen
   CNAME-Eintrag `www` → `DEINNAME.github.io` setzen → „Enforce HTTPS" aktivieren

## Schritt 5 (wichtig) — EmailJS absichern
EmailJS-Dashboard → **Account → Security** → erlaubte Domain(s) eintragen
(deine Pages-/Custom-Domain). Dann kann niemand fremdes dein Mail-Kontingent benutzen.

## Ehrliche Hinweise für den echten Launch
- Die App speichert Daten aktuell **pro Gerät** (Demo-Modus). Gemeinsame Konten,
  echte Gruppen zwischen Nutzern und echte 0,25-€-Zahlungen brauchen das Backend
  (Supabase + Stripe) — das bauen wir als nächsten Schritt ein.
- Play Store / App Store später möglich: Die PWA lässt sich mit TWA (Android, einmalig
  25 $ Entwicklerkonto) bzw. Capacitor (iOS, 99 $/Jahr) verpacken.
