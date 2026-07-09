# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Klasapps — a collection of standalone Dutch-language educational web apps for primary school
students (rekenen, taal, Frans, WO, ...), plus one teacher dashboard. There is no build system,
package manager, framework, or test suite. Every app is a single self-contained `.html` file
(HTML + inline `<style>` + inline `<script>`) that pulls Tailwind and Firebase from CDNs. There
is nothing to install, build, lint, or run — open the `.html` file directly in a browser, or
serve the directory with any static file server, to develop and test.

## Doelgroep & pedagogische richtlijnen

Deze apps zijn gemaakt voor leerlingen van 11-13 jaar in het buitengewoon GO! onderwijs in Vlaanderen
(type autisme), die functioneren op een niveau van ongeveer het 3e-4e leerjaar lager onderwijs.

Richtlijnen voor elke nieuwe of aangepaste app:
- Gebruik korte, eenvoudige zinnen. Vermijd figuurlijke taal, dubbele ontkenningen
  en abstracte instructies.
- Beperk visuele prikkels: geen felle/contrasterende kleurencombinaties, geen snelle
  of onvoorspelbare animaties, geen te veel gelijktijdige UI-elementen op één scherm.
- Feedback moet voorspelbaar en consistent zijn (zelfde stijl/plek/timing bij elke
  oefening), geen verrassingseffecten.
- Instructies mogen herhaald/herlezen worden — geen tijdsdruk tenzij expliciet
  gevraagd door de leerkracht.
- Voorkeur voor duidelijke, grote klikgebieden en weinig scrolwerk per scherm.

* baseer je voor de inhoud/structuur/volgorde op de de meest recente leerplannen van het GO! en de minimumdoelen van het Vlaamse lager onderwijs

Over TEACHER_PWD: dit is een bewuste, eenvoudige keuze voor een kleinschalige
klasomgeving, geen bug. Niet vervangen door complexere auth zonder dit expliciet
te bespreken met de leerkracht.

## Architecture

### Per-app structure
Each exercise app follows the same shape: a login/name-select screen, an exercise/quiz UI driven
by inline vanilla JS, and a Firebase Firestore write at the end of a session to persist the
student's score. There's no shared JS file or component library — logic is duplicated
per-app rather than imported, so when fixing a bug that likely affects multiple apps (e.g. the
login flow or score-upload logic), check whether the same pattern needs the same fix in other
`.html` files.

### Firebase / Firestore integration
Most apps connect to the same Firestore project (`klaswebsite-database`) via the Firebase v9
*compat* SDK loaded from `gstatic.com` (`firebase-app-compat.js` + `firebase-firestore-compat.js`).
The config object (`apiKey`, `authDomain`, `projectId`, ...) is copy-pasted into each file rather
than shared.

Common per-app Firestore variables/pattern:
- `_db` — the Firestore handle.
- `_currentUser` / `_isGuest` — set by the login screen; guest mode ("Gast") skips saving scores.
- `COL_NAME` — the Firestore collection name this app writes results to. **Naming convention
  matters**: a `wo_` prefix means subject WO, `taal_` means Taal, `fr_`/`ll_`/`seo_`/`ned_`/`nl_`
  have similar meaning — the dashboard uses this prefix to auto-classify the collection into a
  subject ("vak") when no explicit mapping exists. Look at `COL_NAME` in existing apps
  (e.g. `spreekwoorden.html`, `delenvandeplant.html`) before inventing a new collection name.
- `leerlingen` collection — the roster of students (`naam`, `weergavenaam`, ...), read on login
  to resolve a student ID.
- `toets_registry` collection — each app self-registers here (`{collection, vak, lastSeen}`) on
  first score write, which is how [dashboard.html](dashboard.html) auto-discovers new
  quiz/collection apps without needing code changes.

`00firestore_debug.html` is a standalone admin tool (not linked from other apps) for inspecting
and bulk-deleting documents in a given collection directly — useful as a reference for raw
Firestore read/delete calls, and as a manual way to clear test data.

### dashboard.html
The teacher-facing app. On load it:
1. Reads `leerlingen` for the student roster (seeding a default roster via
   `initieelLeerlingenAanmaken` if empty).
2. Reads `toets_registry` to auto-discover every collection any app has ever written to, and the
   `vak` (subject) each is tagged with.
3. Queries every discovered collection to build `allRecords`, normalizing disparate per-app score
   document shapes into a common shape via `normRecord()` (fields like `_pct`, `_score`, `_total`,
   `_leerlingId`, `_tijd`, `_mode`, ...). Since apps aren't fully consistent about field names
   (e.g. `score`/`total` vs `totalQ`, `hintsGebruikt` vs `hintsUsed`), `normRecord()` is the single
   place that reconciles those differences — extend it, don't special-case elsewhere.
4. Resolves each collection to a subject ("vak") via `colVak()`, in priority order: explicit
   `vak` field on the document → `toets_registry` vak info → `COLLECTION_VAK_FALLBACK` table →
   collection-name prefix → default `'Wiskunde'`.

The dashboard renders an overview tab, a per-class/per-student view, a matrix view (drag-to-reorder
rows/columns), and a student-management tab, plus a simple hardcoded teacher password
(`TEACHER_PWD`) gate — a deliberate choice, see "Doelgroep & pedagogische richtlijnen" above.
Firestore security rules (not present in this repo) are the actual access boundary in production.

### Adding a new exercise app
When creating a new app, follow the existing convention rather than introducing a new pattern:
copy the Firebase compat script tags + `initializeApp` config from an existing app, pick a
`COL_NAME` with the correct subject prefix (or add an explicit entry to
`COLLECTION_VAK_FALLBACK` in dashboard.html), and write to `toets_registry` on first save so the
dashboard auto-discovers it.
