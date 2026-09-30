# concert-setlist

A live-performance song viewer for a working band. Songs come from Google Drive; the band reads them on stage, moves between them together, and plays to a shared metronome.

Built for **Murphys Outlaws**, and shaped by that: every decision assumes a dark room, a phone on a mic stand, and no chance to fix anything mid-song.

**Live:** https://concert-setlist.insforge.site

---

## What it does

### Getting songs in

One **Import** menu on the songs page, three ways in:

- **Google Drive** — pick **PDFs, images (PNG/JPEG/HEIC), or Google Docs**. Text PDFs have their text extracted *with coordinates*, so the app knows where every line sits. A scan, an image-in-PDF, a Drive image, or a Google Doc (exported to PDF first) that has no text layer is **read by a vision model** and typeset into a real, editable chart.
- **Photo from the device** — a phone snap or scan (HEIC/PNG/JPEG); the AI reads it into a chart you check in a review card before saving. The photo itself is not kept.
- **Reads the structure off the chart** — sections, their labels, and the tempo markings written between them. Headings can be two words (`Jam Intro`) or name a player (`Solo drums`).
- **Reads the tempo off the page.** A BPM found by the parser stays *unconfirmed* until a person agrees — a number on a chart could be a year, a bar count or a track number. One click confirms it in the editor.
- **A single import opens straight into the editor** to check and adjust.
- **Delete** removes the song, its rendered pages and its original from storage, and says how many setlists lose it first.

### Preparing a song

- **A chord/beat-grid editor.** Each section is a row of framed **bars**, each split into its **beats** (from the time signature). Click a beat to type its chord, drag a chord to another beat, drag a bar's edge to resize it, and drag the bar row to line the chords up over the words. The words and headings are editable, and whole lines or sections can be removed (removing a section confirms first). A live PDF preview shows exactly how it prints and, rasterised, how it looks on stage.
- **Section lengths come from the grid** — each bar is a bar, so a section's length (and the whole song's `Total time`) is derived as you edit, and written back on save.
- **Tempo, time signature and confirmation live on the song**, not on a setlist. Changing the time signature re-grids every bar to the new beats-per-bar; one click confirms a parsed tempo.
- **Transpose** every chord up or down a semitone, or type a new key (Danish-aware: `H` is B natural) and the chords follow the interval.
- **Play through** steps a marker through the beats at tempo so you can watch it move over the words.
- **Tempo changes are detected and editable.** A song can change tempo *and* meter partway through; the leader can correct which bar the change lands on, since the chart rarely says.

### Building a set

- **Drag to reorder**, or use the arrows — HTML5 drag does not fire on touch, so both exist.
- **Mark encores.** They are timed separately, since they are the part you might not play.
- **Times the set** from bars × beats-per-bar ÷ BPM, summed section by section at each section's own tempo. A song that changes gear cannot be timed with one multiplication.

### On stage

- **Metronome** with an audible click, a swinging Maelzel pendulum or a pulsing disc, and two bars of count-in before bar 1 — counted in the meter the song *starts* in.
- **The click follows tempo and meter changes**, changing gear on the bar rather than at the next poll.
- **Mute without stopping.** Silencing the click never interrupts the pulse.
- **Marks position in the song** — current section, and the current line underlined on the page.
- **Syncs the band.** The leader changes song or starts the click; every follower's screen follows, tempo changes included.
- **Set stopwatch** — start, pause, reset. Shared, so a member joining at song four reads the same elapsed time as everyone else.
- **Says when the set is ending**: `Last of the set` before the encores, `Last song` when nothing follows.
- **Clear view** strips the screen to the metronome, the clock, and the way back.
- **Works offline.** Pages are cached by a service worker, because venue wifi cannot be trusted with a gig.

## Stack

Pinned deliberately. Read `AGENTS.md` before upgrading anything.

| | |
|---|---|
| Next.js | 15.5, App Router, `src/` |
| React | 19 |
| Tailwind CSS | 3.4 — **not** v4 |
| TypeScript | 5.9 |
| Backend | [InsForge](https://insforge.dev) — Postgres + RLS, auth, storage |
| PDF | `pdfjs-dist` 6.1 (extraction + rasterisation), `jspdf` (typeset charts) |
| Vision | OpenRouter → `google/gemini-2.5-flash`, reads scans/images into charts |
| Email | Resend |

## Running locally

```bash
npm install
npm run dev          # http://localhost:3000
npm test             # unit tests, node:test
npx tsc --noEmit     # types
```

`npm run build` writes to `.next` and breaks a running dev server — stop `dev` first.

Tests import with explicit `.ts` extensions and relative paths: `node --test` resolves neither the `@/` alias nor extensionless imports.

### Environment

`.env.local` — gitignored, never committed:

```
NEXT_PUBLIC_INSFORGE_URL=
NEXT_PUBLIC_INSFORGE_ANON_KEY=
NEXT_PUBLIC_APP_URL=
INSFORGE_API_KEY=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
NEXT_PUBLIC_GOOGLE_PICKER_API_KEY=
OPENROUTER_API_KEY=
RESEND_API_KEY=
RESEND_FROM_EMAIL=
```

`OPENROUTER_API_KEY` (server-only) powers the vision transcription of scans and images. It must also be set in the deployment env (`deployments env set`).

The CLI reads `.insforge/project.json` separately. `.env.local`, `.mcp.json` and `.insforge/` are all gitignored.

## Database

Migrations live in `migrations/` and apply in filename order:

```bash
npx @insforge/cli db migrations new <name>
npx @insforge/cli db migrations up --all
```

**A deploy does not run migrations.** Code expecting an unapplied column fails at runtime, not at build.

Every table is RLS-protected. Two traps worth knowing before writing queries:

- **`.insert().select()` fails under RLS** when band membership is granted by an `AFTER INSERT` trigger — `RETURNING` is checked before the trigger has run. Generate the UUID client-side, insert without `.select()`, then read back by primary key.
- **RLS decides which rows you _may_ see, not which row you _mean_.** A `.limit(1)` with no user filter returns an arbitrary band member — which is how every device once believed it was the leader.

## Deploying

```bash
npx @insforge/cli deployments deploy
```

Google OAuth redirect URLs must be registered in InsForge auth settings for **every** domain. A new domain silently breaks sign-in until it is added.

## Layout

```
src/app/stage/[setlistId]/   the performance view — metronome, overlays, sync
src/app/songs/               import, tempo, section tagging
src/app/setlists/            set building, encores, durations
src/lib/charts/              PDF text → lines → sections → tempo map → position  (pure, tested)
src/lib/stage/               sync, clock skew, set clock                        (pure, tested)
src/lib/metronome*.ts        drift-corrected scheduling, Web Audio click
migrations/                  SQL, applied in filename order
context/                     architecture, UI tokens and rules, decisions
```

The maths lives in `src/lib/**` as pure functions with unit tests, so "is the marker on the right line" can be answered without opening a browser at a rehearsal.

## Four ideas that explain most of the code

**Two planes.** The *preparation plane* (setlists, import, settings) is themed and uses 44px touch targets. The *performance plane* (`/stage`) is always dark, uses 56px targets, renders outside the site shell, and carries no navigation that can be hit by accident. They are not themes of one another — see `context/ui-tokens.md`.

**Shared anchors, not shared events.** Devices never send each other "start now". The leader publishes an *instant*, corrected for clock skew, and each device derives its own beat grid from it. A follower joining mid-song lands in phase rather than restarting the bar. The same idea drives the count-in and the set clock, which is why neither needs a message of its own.

**A song is a tempo map, not a tempo.** Everything used to divide elapsed time by one BPM, which is wrong the moment a song changes gear — one of these spends three sections at 80 in 3/4 between stretches of 175 in 4/4, and a single division is 42 seconds out before the last chorus. The map is a list of constant-tempo segments built from the sections, and it drives the marker, the click and the duration alike. It is *derived* on each device rather than broadcast, so a tempo change cannot arrive late or out of order on a follower.

**The songs are Danish.** Chords use `H` for B natural, and `Outtro` is spelled with two t's. Both broke parsers written against English conventions; assume Danish when reading anything musical out of these files.

## Working on this

`AGENTS.md` is the contract — read `context/` in the order it lists before implementing. `context/progress-tracker.md` and `context/ui-registry.md` are updated after every feature.

### Not yet verified on hardware

Two-device sync, offline playback, the count-in and the tempo-map handover are unit-tested and reasoned through, but have never been run on two phones in a room or on a device in airplane mode.

Real songs broke code that passed synthetic tests over and over during development — a chord with a marking merged onto it, a heading naming an instrument, a tempo written inside brackets. Treat passing tests as necessary and not sufficient for anything touching sync, audio or PDF text, and check a change against the band's actual charts before believing it.
