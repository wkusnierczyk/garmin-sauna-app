# Sauna App. Design

Status: Draft  
Version: 0.1  
Date: 2026-10-10  
Owner: Wacek  
Repository: https://github.com/wkusnierczyk/garmin-sauna-app  
Watch simulator: [simulations/watch.html](simulations/watch.html), published at https://claude.ai/artifact/Qb7C17vU1umjzQ9bRGrT41  
Companion simulator: [simulations/companion.html](simulations/companion.html), published at https://claude.ai/artifact/SpRYheRUkiU91DbDsfbcgy

The product name is open. "Sauna" is used as a working title throughout. See [Risks and open questions](#20-risks-and-open-questions).

## Contents

1. [Overview](#1-overview)
2. [Goals and non-goals](#2-goals-and-non-goals)
3. [System components](#3-system-components)
4. [Deployment modes](#4-deployment-modes)
5. [Watch app](#5-watch-app)
6. [Heart rate pipeline](#6-heart-rate-pipeline)
7. [Warnings and alerts](#7-warnings-and-alerts)
8. [Connect IQ settings](#8-connect-iq-settings)
9. [Program model](#9-program-model)
10. [Program sources and ownership](#10-program-sources-and-ownership)
11. [Program validation](#11-program-validation)
12. [Watch and backend sync](#12-watch-and-backend-sync)
13. [Session data](#13-session-data)
14. [Companion web app](#14-companion-web-app)
15. [AI insights and suggestions](#15-ai-insights-and-suggestions)
16. [Backend and deployment](#16-backend-and-deployment)
17. [Privacy and data protection](#17-privacy-and-data-protection)
18. [Device temperature limits](#18-device-temperature-limits)
19. [Testing and validation](#19-testing-and-validation)
20. [Risks and open questions](#20-risks-and-open-questions)
21. [Milestones](#21-milestones)
22. [References](#22-references)

## 1. Overview

A Garmin Connect IQ app for sauna, cold plunge and contrast sessions, plus a companion that turns the recorded sessions into history, statistics, insights and programs.

The watch app works on its own. It tracks sessions as they happen (Tracker mode) or guides the user through a planned sequence of steps (Program mode). It shows time and heart rate in a form readable through a fogged screen without glasses, warns when heart rate crosses a user-chosen level, and records the session as a Garmin activity.

The companion adds what a watch cannot do well: browsing history, detailed charts, building and sharing programs, and AI-generated insights and program suggestions.

The watch behaviour in this document was developed in the watch simulator, and the companion layout in the companion simulator. Both are single HTML files in `design/simulations/`; open them in a browser, no build step. **This document is authoritative.** When it is silent, match the simulators.

## 2. Goals and non-goals

### 2.1 Goals

- A watch app that is useful with no phone app, no account and no backend.
- Glanceable in the sauna: large monospaced time, large heart rate, colour-coded zones, vibration for anything important.
- Two ways to run a session: Tracker (log phases as they happen) and Program (follow a plan).
- A program library with four sources: built-in, community, user-made and AI-suggested, all in one model.
- A companion that lets the user choose which deployment they want: everything on their own devices, or an account-based web app with sync, sharing and AI.
- Self-hosting first. The backend runs on the owner's workstation in Kubernetes before any cloud hosting.

### 2.2 Non-goals for v1

- Medical claims of any kind. The app is a wellness and logging tool (§17.4).
- The native phone app. It follows the web app (§21).
- The community service. Sharing starts with share codes (§10.4).
- Automatic phase detection on the watch. It is designed for (§5.6) but ships after manual phases.
- Garmin Health API or Activity API integration. It needs approval from Garmin's developer program (§13.4).

## 3. System components

```
                         ┌────────────────────────────────────────────────┐
                         │  Backend (Kubernetes, owner's workstation)     │
 ┌──────────────┐  HTTPS │  ┌─────────┐  ┌──────────┐  ┌──────────────┐  │
 │  Watch app   │◄──────►│  │   API   │──│ Postgres │  │  AI worker   │──┼──► AI provider API
 │ (Connect IQ) │ via GCM│  └────┬────┘  └──────────┘  └──────────────┘  │
 └──────┬───────┘  proxy │       │                                        │
        │ settings       │  ┌────┴────────┐                               │
 ┌──────┴───────┐        │  │  Web app    │◄──── browser (phone, tablet,  │
 │ Garmin       │        │  │  (static)   │       laptop)                 │
 │ Connect app  │        │  └─────────────┘                               │
 └──────────────┘        └────────────────────────────────────────────────┘

 Later: native phone app (Personal mode, talks to the watch directly)
        community service (shared program catalogue)
```

| Component | Role | Needs the backend |
|---|---|---|
| Watch app | Runs sessions, records activities, holds up to 8 programs, queues uploads | No |
| Connect IQ settings | Simple per-user settings, edited in the Garmin Connect phone app (§8) | No |
| Web app | History, charts, program library and builder, insights, settings | Yes |
| Backend API | Accounts, watch pairing, session ingest, program sync, validation | Yes |
| AI worker | Insights and suggested programs, off the request path | Yes |
| Native phone app (later) | Personal mode: history and programs on the phone, no account | No |
| Community service (later) | Shared catalogue of programs | Yes |

"GCM proxy" means Garmin Connect Mobile: a watch app's `Communications.makeWebRequest` goes through the paired phone, so the backend must be reachable from the phone's network (§16.3).

## 4. Deployment modes

The user decides how much infrastructure they want. Both modes share one data model (§9) and one program file format, so a user can move between them.

| | Watch only | Personal (later) | Connected |
|---|---|---|---|
| What runs | Watch app | Watch app + native phone app | Watch app + web app + backend |
| Account | None | None | Yes |
| Programs on the watch | Built-in set, chosen in Connect IQ settings | Any, pushed from the phone | Any, synced from the backend |
| History | Garmin Connect only | On the phone | In the web app |
| AI insights | No | No, unless the user brings an API key (open question) | Yes |
| Sharing | No | Share codes (file or QR) | Share codes, later community |
| Who maintains a service | Nobody | Nobody | Owner of the backend |

Order of delivery: Watch only (M1), Connected (M2 to M4), Personal (M5).

## 5. Watch app

### 5.1 Screens and flow

```
Watch face ─START─► Activity list ─START on Sauna─► Mode list
                                                     │
                     ┌───────────────────────────────┴──────────────┐
                     ▼                                              ▼
              Tracker ready                                   Program list
                     │ START                                        │ START
                     │                                       Program detail
                     │                                              │ START
                     └──────────────────► Running ◄─────────────────┘
                                          │  START
                                          ▼
                                 Paused: Resume, Save, Discard
                                          │ Save              │ Discard
                                          ▼                   ▼
                                   Saved summary      Confirm ─► Watch face
                                          │ START or BACK
                                          ▼
                                     Watch face
```

When a program's last step ends, the pause menu opens as "Complete" with Save and Discard only.

### 5.2 Buttons

All actions fire on `onKey`, after release. UP waits for release so that a long press can become `onMenu` without also scrolling.

| Screen | START | BACK | UP | DOWN |
|---|---|---|---|---|
| Lists | Select | Back | Previous (hold: settings) | Next |
| Program detail | Begin | Programs | Scroll steps | Scroll steps |
| Running | Pause | Lap: next phase (Tracker), skip step (Program) | Data page | Data page |
| Paused | Select | Resume (not after Complete) | Previous | Next |
| Saved summary | Done | Done | | |

Lists wrap around at both ends.

### 5.3 Modes

- **Tracker.** Phases cycle Sauna, Cold, Rest. BACK marks a phase change and records a lap. There is no planned length.
- **Program.** Steps advance automatically at their planned length, with a vibration and a banner naming the next step. BACK skips to the next step early. Plan adaptation (stretching or shortening a step from heart rate) is designed later.

### 5.4 Running screen

From the top: phase label in the phase colour (heat orange, cold blue, rest grey), step detail or lap number, the large timer, round and step counter (Program) or session total (Tracker), the heart rate block, and the next step.

- **Timer:** remaining time in Program, elapsed phase time in Tracker. Monospaced, so digits never shift.
- **Ring:** the outer ring is the step's progress, coloured by heart rate zone over time (§6.4). In Program one turn is the planned step. In Tracker one turn is a reference length (heat 15 min, cold 3 min, rest 15 min) until the phase runs over, then the ring rescales to the elapsed time.
- **Heart rate block:** heart icon, value and zone badge ("Z4"), all in the zone colour. The block is a fixed-width box, sized for three digits and the badge, centred once. Content is right-aligned in it, so only the heart icon moves when the value gains a digit. The badge slot is reserved below zone 1.
- **Second data page (UP or DOWN):** session totals for total time, heat, cold, rest and max heart rate. No zone strip here, because it collides with the ring.

### 5.5 Saved summary

One header line, "Saved: <program name>", then five rows: time, rounds (or heat laps in Tracker; replaced by warning count and time when a warning fired), heat time, cold time, average and max heart rate. Labels get whatever width the value leaves, so they never overlap. Below the rows, the zone strip, spanning the same margins as the rows.

Zone strip options (a setting, §8):

| Option | Shows |
|---|---|
| Time per zone (default) | Time in each zone, sorted by zone, like a histogram laid end to end |
| Timeline | Zones in time order, start to end, with a notch at each phase change |
| Both | Timeline above time per zone |

The timeline keeps every zone change for the session. Cap it, or merge segments shorter than a few seconds, so a long session cannot grow it without limit.

### 5.6 Wear profiles and phase boundaries

| | Watch on wrist | Watch off wrist with chest strap |
|---|---|---|
| Heart rate | Wrist optical or strap | Strap only, drops out under water |
| Phase boundaries | Manual now; automatic later from the temperature slope and motion | Manual; later prompts from the strap's heart rate pattern |
| Ambient temperature | Crude estimate from the internal sensor, or stated | Stated, or an external sensor |
| Warning | Device temperature warning (§18) | None needed |

Wrist detection uses the accelerometer: a watch on a bench has a fixed gravity vector and almost no variance; on a wrist there is constant micro-motion. Re-evaluate continuously, because users take the watch off mid-session. Record the detection mode per segment in the FIT file.

While the strap is under water in the cold phase, the watch shows "--" and "no signal", and the ring segment is drawn in the no-signal grey.

### 5.7 Fonts

Ticking numbers (clock, timers, heart rate, totals) use a custom monospaced bitmap font. Labels use a condensed proportional font. Both ship with the app as custom fonts, with sizes per display resolution.

### 5.8 Built-in programs on the watch

The watch app bundles the built-in program library (§10.1). In Watch-only mode the user picks which built-ins appear on the watch in Connect IQ settings. In Connected mode the backend's selection replaces this (§12.3).

## 6. Heart rate pipeline

### 6.1 Stages

```
sensor (1 Hz) ──► raw ──► smoothed (τ = 6 s) ──► displayed integer ──► displayed zone
                    └───► fast estimate (τ = 2 s) ──► warning (§7)
```

| Stage | Rule | Why |
|---|---|---|
| Smoothing | Exponential, τ = 6 s | Rises and falls gradually instead of jittering |
| Displayed integer | Changes only when the smoothed value is 0.8 bpm away from it | Stops toggling between neighbouring values |
| Displayed zone | Drops a zone only 2 bpm below that zone's lower cutoff | Stops the ring striping and alerts repeating at a boundary |
| Fast estimate | Exponential, τ = 2 s | The warning should not wait for the display smoothing |

The 6 s smoothing delays the cold-shock spike by a few seconds. That is acceptable for display only.

### 6.2 Zones

Zone cutoffs come from the user's own Garmin zones via `UserProfile.getHeartRateZones()`, so any changes made in Garmin Connect apply. Without them, use Garmin's default: percentages of max heart rate.

| Zone | Range (% of max HR) | Colour |
|---|---|---|
| Below 1 | under 50 | white text, dark grey ring |
| 1 | 50 to 60 | grey |
| 2 | 60 to 70 | blue |
| 3 | 70 to 80 | green |
| 4 | 80 to 90 | orange |
| 5 | 90 to 100 | red |

These are exercise zones. In the sauna, heart rate rises from heat load, not effort, so a zone does not mean the same strain as on a run. A sauna-specific override of the cutoffs is a later setting (§8).

### 6.3 Colour is not the only channel

Green, orange and red are the hardest colours to tell apart with red-green colour blindness. The zone number is always shown in the badge, and warnings also vibrate.

### 6.4 Zone ring

Each stretch of the ring takes the colour of the displayed zone at that time. A new segment starts whenever the displayed zone changes; the hysteresis in §6.1 keeps the number of segments small. A white tick marks the current position.

## 7. Warnings and alerts

### 7.1 Heart rate warning

| Setting | Threshold |
|---|---|
| Off | None |
| Zone 5 floor (default) | Lower cutoff of zone 5 |
| Max HR | Max heart rate (the zone 5 ceiling) |
| Custom | A bpm value the user enters |

- **On** when the fast estimate reaches the threshold. **Off** when it falls 2 bpm below.
- While on: vibration every 2.5 s, the screen flashes red at 2 Hz (under the 3 flashes per second photosensitivity limit), and a "HIGH HEART RATE" badge replaces the step detail line.
- The saved summary records how many times the warning fired and for how long.

### 7.2 Other alerts

Designed, not yet simulated: phase too long (Tracker), strap signal lost, device temperature near its rated limit (§18).

### 7.3 Wording

Alerts state what was measured. They do not tell the user what to do medically. "HIGH HEART RATE" is acceptable; "Leave the sauna now, you are in danger" is not (§17.4).

## 8. Connect IQ settings

Edited in the Garmin Connect phone app, defined in `settings.xml` and `properties.xml`, read at start and in `onSettingsChanged()`. These settings work in every deployment mode.

| Key | Values | Default |
|---|---|---|
| wear | wrist, strap | wrist |
| zoneSource | watch zones, custom | watch zones |
| maxHr | number (custom zones only) | from user profile |
| warnAt | off, z5, max, custom | z5 |
| warnBpm | number | 165 |
| zoneStrip | perZone, timeline, both | perZone |
| defaultSaunaType | finnish, finnishLoyly, steam, infrared | finnish |
| defaultSaunaTemp | number, °C | 85 |
| recordAs | generic, cardio | generic |
| builtins | one toggle per built-in program (Watch-only mode) | first 5 on |
| pairingCode | read-only text, shown when pairing (Connected mode) | |

Programs are not edited here. A program is a list of structured steps, and the settings UI is not built for editing lists of objects.

`recordAs`: depending on the sport type, Garmin may count an HR-heavy sauna session toward training load. Generic is the default; cardio is an explicit opt-in. Verify how Connect IQ sessions affect training load.

## 9. Program model

### 9.1 Schema

```json
{
  "id": "p_8f2c1a",
  "schema": 1,
  "rev": 4,
  "name": "Heat ladder",
  "origin": "mine",
  "copiedFrom": { "id": "b_heat_ladder", "rev": 2, "origin": "builtin" },
  "author": null,
  "createdAt": "2026-10-10T17:20:00Z",
  "updatedAt": "2026-10-12T08:02:11Z",
  "steps": [
    { "kind": "heat", "label": "Sauna", "dur": 480, "round": 1,
      "sauna": { "type": "finnishLoyly", "tempC": 86 } },
    { "kind": "cold", "label": "Plunge", "dur": 120, "round": 1,
      "water": { "type": "plunge", "tempC": 6 } },
    { "kind": "rest", "label": "Rest", "dur": 600, "round": 1 }
  ],
  "tags": ["contrast", "evening"]
}
```

- `kind` is one of `heat`, `cold`, `rest`. Only `kind` and `dur` drive the watch; the rest is display and validation.
- `dur` is in seconds. `round` groups steps for display and summaries.
- `origin` is one of `builtin`, `community`, `mine`, `suggested`.
- `rev` increases on every saved change. Sync compares ids and revisions (§12).
- `copiedFrom` links a copy to the program it came from, with that program's revision at copy time, so the app can say "the original has changed".

### 9.2 Watch form

The watch receives a compact form: id, rev, name, and steps as `[kind, dur, round, label, detail]`, where detail is a short display string such as "86°C · löyly". A program is well under 1 KB.

## 10. Program sources and ownership

| | Built-in | Community | Mine | Suggested |
|---|---|---|---|---|
| Comes from | Shipped with the app | Shared by other users | Built by the user, from scratch or from another program | Generated from history, profile and current guidance |
| Editing | Edit makes a linked copy | Edit makes a linked copy | In place | Accept first, then it is the user's |
| Updates from origin | With app updates | When the author revises it | No | No |
| Can go on the watch | Yes | Yes | Yes | Only after acceptance |

### 10.1 Built-in

A curated set of about ten programs covering classic Finnish rounds, quick contrast, a gentle first-timer session, a heat ladder, infrared and steam. Read-only. They ship with the watch app and the web app, versioned with releases.

### 10.2 Mine

- **New** starts from an empty program or a template.
- **Edit** changes the program in place and increases `rev`.
- **Clone** makes a new program with a new id and `copiedFrom` set.
- **Edit on a built-in or community program** creates a linked copy in Mine and opens it. Editing the original in place would force the next update to either overwrite the user's changes or stop updating that program.
- When the original of a copy gets a newer revision, the program shows "Original updated" with the option to view the differences. Nothing changes automatically.

### 10.3 Suggested

Generated by the AI worker (§15). A suggestion is a draft: it can be viewed, accepted or dismissed, but not put on the watch. Accepting it moves it to Mine with `origin: mine` and keeps a note of where it came from. A program should not reach the watch without a person having looked at it.

### 10.4 Community

**Phase 1, share codes.** Any program can be exported as a share link, a file or a QR code containing the program JSON. Importing one creates a program in Mine with the original author recorded. No catalogue, moderation or accounts for strangers are needed.

**Phase 2, community service.** A shared catalogue with browsing, search, ratings and reporting. Its programs appear under Community and follow the same model. It needs moderation, a reporting path and terms for shared content.

### 10.5 On the watch

- The user ticks which programs go on the watch, up to **8**. Storage is not the limit; the limit is how many programs can be scrolled through comfortably on a round screen with buttons.
- The user orders them; the watch list follows that order.
- Suggested programs cannot be ticked until accepted. A program that fails validation (§11) cannot be ticked.

## 11. Program validation

Every program from every source passes the same validator before it can be saved to the watch selection. Rules have two levels: **block** (cannot go on the watch) and **warn** (allowed, with a visible note).

The numbers below are **placeholders** for review. They are not medical advice, and they need checking against the literature and, ideally, a clinician before release.

| Rule | Warn | Block |
|---|---|---|
| Heat step length | over 20 min | under 1 min or over 30 min |
| Sauna temperature | over 100 °C | over 110 °C |
| Cold step length | over 5 min | over 10 min |
| Cold water temperature | under 4 °C | under 0 °C |
| Rounds | more than 6 | more than 10 |
| Total session | over 2 h | over 3 h |
| Order | heat directly after heat with no rest or cold between | |
| Empty | | no heat step, or no steps at all |

The validator runs in the web app as the user edits, and again on the backend before sync. The watch trusts what it receives.

## 12. Watch and backend sync

### 12.1 Pairing

1. In Connected mode the watch calls `POST /api/watch/pair/start` and shows a 6-character code (also visible in Connect IQ settings).
2. The user enters the code in the web app under Settings, Watch.
3. The backend binds the watch to the account and returns a device token on the watch's next poll. The watch stores it in `Application.Storage`.

`Communications.makeOAuthRequest` is an alternative; the code flow is simpler and needs no browser on the watch side.

### 12.2 Endpoints

| Method and path | Purpose |
|---|---|
| `POST /api/watch/pair/start` | Get a pairing code |
| `GET /api/watch/pair/status` | Poll for the device token |
| `GET /api/watch/programs/manifest` | Ordered list of `{id, rev}` selected for the watch |
| `GET /api/watch/programs/{id}` | Compact program (§9.2) |
| `POST /api/watch/sessions` | Upload a session summary and trace (§13.2) |
| `GET /api/watch/settings` | Optional server-side settings, if any move off Connect IQ settings |

### 12.3 Program sync

On app start and after saving a session, the watch fetches the manifest, downloads programs whose id is new or whose revision changed, deletes programs no longer listed, and stores the order. If the backend is unreachable, the watch keeps what it has.

### 12.4 Upload queue

`makeWebRequest` only works while the phone is connected, and after a sauna the phone is often in a locker. The watch stores each finished session in `Application.Storage` and retries on later starts and connections. Each session carries a client-generated UUID, and the backend treats uploads as idempotent, so a retry after a lost response cannot create a duplicate. Cap the queue (for example 20 sessions) and drop the oldest trace first, keeping the summary.

### 12.5 Transport

The backend must be served over HTTPS with a publicly trusted certificate, reachable from the phone's network. Requests through the phone proxy have practical size limits; keep uploads to a few kilobytes and verify the limits for target devices in M0.

## 13. Session data

### 13.1 On the watch (FIT)

`ActivityRecording.createSession`, a lap at every phase boundary, and FIT developer fields for: phase kind, plan step, ambient temperature and its source, device temperature, detection mode per segment, displayed zone.

### 13.2 Upload

| Part | Content | Size |
|---|---|---|
| Summary | UUID, start time, mode, program id and rev, totals per kind, rounds, avg and max HR, warning count and time, time per zone, wear profile | about 0.5 KB |
| Laps | per lap: kind, duration, avg and max HR, ambient value and source | about 40 bytes each |
| Trace | every 10 s: HR, displayed zone, phase index | an hour is 360 points, a few KB |

### 13.3 Ambient values

Every ambient value carries its source: `stated` (user or saved sauna), `estimated` (internal sensor), or `sensor` (external). Summaries and the AI weight values by source. The user can correct values after the session in the web app; anything derived from them is recomputed.

### 13.4 Full-resolution data, later

Garmin's Activity API can deliver the recorded FIT file, including developer fields, to the backend. It needs approval from Garmin's developer program. When available, it replaces the 10 s trace for detailed charts.

## 14. Companion web app

### 14.1 Views

| View | Content |
|---|---|
| Overview | This week (sessions, heat and cold time), last session with its chart, what is on the watch, latest insight, pending suggestions |
| Sessions | List with date, program, length, max HR and zone strip. Detail: HR chart with phase bands, zone cutoff lines and the warning threshold; per-lap table; ambient values with their source; correction of ambient values |
| Programs | Library filtered by source (All, On watch, Built-in, Mine, Community, Suggested); watch selection with checkboxes, the 8-program limit and ordering; program detail with timeline, steps and validation |
| Builder | Name, steps grouped by round, add, duplicate round, reorder, remove; live validation; Save increases `rev` |
| Insights | AI insights with evidence level and citations; generate a suggested program |
| Settings | Deployment mode, watch pairing, heart rate zones, warning level, saved saunas, data and privacy (AI consent, export, delete) |

### 14.2 Layout

One adaptive layout, not separate mobile and desktop sites.

| Width | Navigation | Lists with detail |
|---|---|---|
| Phone, under 720 px | Bottom tab bar | One pane; detail opens as its own view with a back button |
| Tablet, 720 to 1100 px | Top tab bar | Two panes side by side where space allows |
| Laptop, over 1100 px | Left sidebar | Two panes; the builder gets the wider pane |

### 14.3 Visual language

The web app shares the watch's vocabulary: the same zone colours, heat orange and cold blue, monospaced numbers, and the zone strip. A session on the web reads like a larger version of what the watch showed.

## 15. AI insights and suggestions

### 15.1 Pipeline

1. A job runs after each uploaded session, and on request.
2. The worker builds a compact context: profile (age band, max HR, resting HR if known), aggregates over recent weeks, the last few sessions as summaries, and the user's stated goals.
3. Retrieval selects passages from a curated literature corpus (§15.2).
4. The model returns structured JSON: insights (text, evidence level, citation ids) and, when asked, a suggested program in the §9 schema.
5. Suggested programs go through the validator (§11). Failing suggestions are discarded or regenerated, never shown.
6. Results are stored and shown in the web app.

The API key lives only on the backend. Per-user rate limits and a monthly cost cap apply.

### 15.2 Evidence corpus

A curated, versioned set of sources with citations. The evidence base for sauna and cold exposure is thin compared with exercise: much of it is observational cohort data (notably the Finnish KIHD cohort), plus small intervention trials. Each insight states its evidence level (observational, small trial, review), and the wording stays hedged.

### 15.3 Data minimisation

The model receives summaries and aggregates, not raw traces, and no names or identifiers. AI processing needs the user's explicit consent (§17.2), and it can be switched off.

## 16. Backend and deployment

### 16.1 Components

| Component | Notes |
|---|---|
| API service | Accounts, pairing, ingest, programs, validation, sync |
| Web app | Static build served by the same ingress |
| Postgres | Users, devices, sessions, laps, traces, programs, insights |
| AI worker | Queue consumer for insight and suggestion jobs |
| Backups | Nightly database dump to a second disk or off-site storage |

Stack (language and framework) is open.

### 16.2 Hosting on the workstation

- A local Kubernetes distribution (for example k3s) on the owner's workstation.
- Manifests or a Helm chart kept in the repository, so the same deployment can move to a cloud cluster later.
- Secrets (database password, AI key, session signing key) as Kubernetes secrets, never in the repository.

### 16.3 Reaching the workstation from phones

The watch talks to the backend through the user's phone, from whatever network the phone is on, so the backend needs a public HTTPS address. Options: a tunnel service that terminates TLS (for example Cloudflare Tunnel), or a public DNS name with a Let's Encrypt certificate and a port forward. A tunnel avoids opening ports at home.

When the workstation is off, the watch keeps its programs and queues uploads (§12.4), so downtime delays sync but loses nothing.

### 16.4 Moving to the cloud

The same manifests deploy to a managed cluster. The triggers for moving are other users depending on the service, or the community service going live.

## 17. Privacy and data protection

### 17.1 Category of data

Heart rate and session data are health data. Under the GDPR (Article 9) and the Swiss revised Federal Act on Data Protection, health data is a special or sensitive category, with stricter rules on consent and processing.

### 17.2 Rules

- Explicit consent at sign-up for storing session data, and separate consent for AI processing.
- Export of all data (JSON, plus FIT where available) and full account deletion from Settings.
- The AI provider is a processor; check its data processing terms and retention before enabling AI for anyone other than the owner.
- No analytics or third-party trackers in the web app.

### 17.3 Personal mode

Personal mode keeps all data on the user's devices. That is the privacy-maximal option and a reason to build it (§4).

### 17.4 Wellness, not medical

No diagnoses, no treatment claims, no instructions framed as medical advice, in alerts, insights, program descriptions or the store listing. Get a wording review before release. This is general guidance, not legal advice.

## 18. Device temperature limits

Garmin owner's manuals list an operating range, commonly -20 to 60 °C, lower on some models (the Forerunner 735XT is rated to 50 °C). Sauna air is typically 70 to 100 °C. Many users, the owner included, wear their watches in saunas without problems: skin cools the case and exposure is short. The likely damage is cumulative (seals and adhesive), not sudden.

- The app supports wearing the watch in the sauna, with an explicit, referenced, device-specific warning on first use of the on-wrist profile, and the same statement in the store listing.
- A table maps device part numbers to their rated maximum, from each manual's Specifications page.
- A live guard reads `Sensor.Info.temperature` and alerts as it approaches the rated maximum.

The internal sensor is a poor thermometer for the sauna air (it reads a blend of skin and air, with errors amplified about five times when inverted) but a good boundary detector: its slope jumps on entering or leaving. Details in the earlier design notes.

## 19. Testing and validation

- **Unit tests** for the heart rate pipeline (smoothing, step threshold, hysteresis), zone computation, warning on and off, the validator rules, program revision and copy logic, and sync manifest diffing.
- **Simulator parity:** for a fixed seed and scripted inputs, watch behaviour matches the watch simulator.
- **Hardware probes (M0):** `makeWebRequest` size limits, behaviour without a phone, internal temperature in a real sauna, strap dropout under water, wrist detection signal.
- **Sync tests:** retries with lost responses must not duplicate sessions; manifest changes must add, update, remove and reorder correctly.
- **Web app:** layout at 375, 768 and 1280 px; keyboard navigation; both colour themes.

## 20. Risks and open questions

| Item | Status | Notes |
|---|---|---|
| Product name | Open | "Nordic" clashes with Nordic Semiconductor in the Garmin ecosystem (the EmberDip rename). Check trademarks before naming. |
| Validator limits | Open | Placeholder numbers in §11 need review. |
| makeWebRequest limits | M0 | Payload size and behaviour through the phone proxy on target devices. |
| Health metrics in Connect IQ | M0 | Which `SensorHistory` and profile values exist on target devices. |
| Training load side effects | M1 | Effect of the recorded sport type on Garmin's training load. |
| Workstation availability | Accepted | Downtime delays sync only (§16.3). |
| Garmin Activity or Health API | Later | Needs approval; full FIT data depends on it. |
| AI cost and provider terms | M3 | Per-user caps; processor terms before opening AI to others. |
| Community moderation | Later | Needed before the community service, not for share codes. |
| Personal mode AI | Open | Whether a user may bring their own API key in the native app. |
| Backend stack | Open | Language, framework and auth library. |

## 21. Milestones

| Milestone | Scope | Done when |
|---|---|---|
| M0. Probes | Hardware and transport probes (§19) | Limits and sensor behaviour recorded for target devices |
| M1. Watch v1 | Tracker and Program modes, built-in programs, HR pipeline, zones, warning, summary, Connect IQ settings, FIT fields | Usable on the owner's watch in a real sauna, Watch-only mode |
| M2. Backend and web app | Accounts, pairing, upload queue and ingest, sessions views, program library, builder, validator, sync | A program built on the web runs on the watch, and the session appears on the web |
| M3. AI | Insights and suggested programs, evidence corpus, consent | Suggestions pass validation and cite sources |
| M4. Sharing | Share codes; later the community service | A program moves between two accounts via a share code |
| M5. Native app | Personal mode via the Connect IQ Mobile SDK | Watch and phone work with no backend |

## 22. References

- [EmberDip (formerly Nordic) showcase thread, Garmin forums](https://forums.garmin.com/developer/connect-iq/f/showcase/445304/app-nordic-free-sauna-cold-plunge-contrast-session-timer)
- [Garmin owner's manual Specifications page, 60 °C example](https://www8.garmin.com/manuals/webhelp/GUID-5D183A14-BB43-4A9B-B441-5F824214CE40/EN-GB/GUID-0ED1C47A-727F-4481-9911-BAF1FBC40166.html)
- [Garmin forum. 735XT in sauna (50 °C rating)](https://forums.garmin.com/sports-fitness/sports-fitness/f/forerunner-735xt/123512/735xt-in-sauna-good-or-bad-idea)
- [Garmin forum. Fenix 6 in sauna (48 °C internal reading at 85 °C)](https://forums.garmin.com/outdoor-recreation/outdoor-recreation/f/fenix-6-series/239785/fenix6-sauna-spa-wellness)
- Earlier design notes: [considerations.md](considerations.md) (temperature model, boundary detection, EmberDip comparison)
