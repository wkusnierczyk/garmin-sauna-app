# Garmin Sauna App. Design Notes

**Status:** draft, captured from an initial design discussion  
**Date:** 2026-10-08  
**Repo:** https://github.com/wkusnierczyk/garmin-sauna-app

## Contents

- [Overview](#overview)
- [Feature requirements](#feature-requirements)
- [Reference app EmberDip](#reference-app-emberdip)
- [Device temperature limits](#device-temperature-limits)
- [Deployment profiles](#deployment-profiles)
- [Wrist detection](#wrist-detection)
- [Boundary detection](#boundary-detection)
- [Ambient conditions](#ambient-conditions)
- [Health and profile data](#health-and-profile-data)
- [Advising and alerts](#advising-and-alerts)
- [Recording](#recording)
- [Session summaries and evidence layer](#session-summaries-and-evidence-layer)
- [Architecture](#architecture)
- [Staging](#staging)
- [Open questions](#open-questions)
- [References](#references)

## Overview

A Connect IQ app for sauna, cold plunge and contrast sessions. It goes beyond a timer and logger. It tracks sessions passively or guides them actively, adapts in real time, uses the user's health data, and produces evidence-based summaries after the session.

The existing app closest to this idea, EmberDip (formerly Nordic), covers only a small subset of these features. See [Reference app EmberDip](#reference-app-emberdip).

## Feature requirements

- **Passive mode.** Track the activities as they happen.
- **Active mode.** Build a schedule of activities and guide the user through it.
- **Advising.**
  - In passive mode, give feedback on the activities and suggest changes in real time.
  - In active mode, adapt the plan based on what the app observes over the session.
- **User profile.** Connect to the user's health and performance data and let the user set general preferences.
- **Boundary conditions.** The user marks activity changes manually, or the app detects them automatically (movement, temperature, HR).
- **Ambient conditions.** The user states the environment (temperature, humidity, sauna type), or the app detects it automatically, at least temperature.
- **Customizable alerting.** Examples include a phase lasting too long or HR going too high.
- **Post-session summaries.** Extensive performance and health summaries, including AI + RAG evidence-based insights and suggestions.
- **Recording.** Save as a generic session or as cardio, whichever fits.

More features may emerge during development.

## Reference app EmberDip

Formerly Nordic. It was renamed after a forum reply pointed out the clash with Nordic Semiconductor, which Garmin's SDK samples use. It is a free sauna, cold plunge and contrast-session timer.

What it does:

- Sauna, cold plunge, and "Nordic Cycle" (sauna, transition, plunge, rest) with round counts and per-phase totals.
- Manual phase changes only, no preset routine.
- Saves sessions as generic Garmin activities, with a lap at each phase boundary.
- Optional ANT+ chest strap HR, with an indicator when readings are missing. No Bluetooth strap support and no backfill of stored HR.
- Pause/resume, discard confirmation, and recovery of a checkpointed summary after an interruption.
- Requires keeping the watch outside the sauna. Timings come from button presses, not from measured exposure.

Coverage of our requirements: it covers passive mode, manual boundaries and generic recording. Everything else is missing: active plans, advising, profile data, auto-detection, ambient conditions, custom alerts, rich summaries and cardio recording.

## Device temperature limits

**Facts.**

- Garmin owner's manuals list an operating temperature range in the Specifications section. It is commonly -20 to 60°C. Some models are lower, for example the Forerunner 735XT at 50°C.
- Sauna air is typically 70–100°C.
- No Garmin support article specific to saunas was found. The per-device manual is the authoritative reference.

**Field experience.**

- Many users, including the author, routinely wear Garmin watches in saunas without problems.
- One forum report: the internal sensor of a Fenix 6 peaked at 48°C in an 85°C sauna.
- Mechanism: the case is cooled by skin (about 40°C in the sauna), the arm sits low where the air is cooler, and exposure is short. The watch likely runs near its rated limit, not far above it.
- The likely failure mode is cumulative: repeated heat cycles degrade seals and adhesive, especially when going from sauna straight to plunge. This shows up later as water ingress, not as sudden failure. Years without problems mean the risk is moderate, not zero.

**Decision.** The app supports wearing the watch in the sauna, with an explicit, referenced, device-specific warning:

- Ship a table mapping device to rated operating maximum, keyed on `DeviceSettings.partNumber` and sourced from each manual's Specifications page.
- On first use of the on-wrist profile, show an acknowledgment screen. Example wording: "This watch is rated to 60°C (owner's manual, Specifications). Sauna air is 70–100°C. Wearing it is at your own risk; heat damage to seals is cumulative." Open the manual page on the phone via `Communications.openWebPage`.
- Put the same statement and links in the store listing.
- **Live device-temperature guard.** Monitor `Sensor.Info.temperature` during the session and alert when it approaches the device's rated maximum, with a configurable margin. Show device temperature as a trace in the summary.

## Deployment profiles

| | Watch on wrist | Watch off wrist (outside sauna) + strap |
|---|---|---|
| HR | Wrist optical (degrades with heat and sweat), or strap | Strap only; drops out underwater or out of range |
| Boundary detection | Automatic: temperature slope + motion | No automatic detection; HR-based prompts only |
| Ambient temperature | Crude estimate from internal sensor, or user input / external sensor | User input or external sensor only |
| Warning | Explicit referenced warning + live guard | None needed |

**Decision.** Both profiles are supported. In the off-wrist profile, automatic detection is turned off.

## Wrist detection

- Don't rely on HR presence alone. With a paired strap, `Sensor.Info.heartRate` can come from the strap.
- Primary signal is the accelerometer. A watch lying on a bench has a fixed gravity vector and almost no variance. On a wrist there is constant micro-motion (tremor, breathing, pulse) even when the user sits still.
- Secondary signal: whether the optical sensor itself is producing a reading.
- **Re-evaluate continuously.** Don't lock the profile at session start. Users take the watch off mid-session, before a plunge or to lie down. Switch per segment, and record the detection mode for each segment in the FIT file so summaries know which boundaries were detected and which were entered manually.

## Boundary detection

- **Manual.** Button press, always available.
- **On wrist, automatic.**
  - The rate of change of the internal temperature sensor jumps on entering or leaving the sauna. This makes it a good boundary detector even though it's a poor thermometer.
  - Motion adds signal around transitions: walking, immersion.
- **Off wrist with strap, prompts.** HR rises steadily in the heat. Cold immersion causes a sharp cold-shock spike followed by slowing. These patterns aren't reliable enough to switch phases automatically. Use them to suggest a change ("Looks like you left the sauna, switch to rest?") that the user confirms with one button.

## Ambient conditions

**Temperature from the internal sensor gives only a crude estimate.** Show it as a suggestion that the user corrects upfront or after the session.

The mechanism follows from a blend model:

```
T_watch ≈ α·T_air + (1 − α)·T_skin
T_air   ≈ (T_watch − (1 − α)·T_skin) / α
```

- On the wrist, α is small. From the Fenix 6 data point (48°C reading at 85°C air, skin around 39°C), α ≈ 0.2.
- Errors are therefore amplified by roughly 1/α ≈ 5×. A 2°C error in the sensor or the skin estimate becomes about 10°C of error in air temperature.
- α also varies with arm position, strap tightness, sweat and löyly.

Ways to improve the estimate:

- **Asymptote, not current value.** The sensor lags, with a time constant of minutes. Fit an exponential to the rise after the entry boundary and extrapolate the plateau.
- **Range, not point.** Display something like "Estimated ~75–95°C, tap to set".
- **Calibrate from user corrections.** Each user-entered true temperature is a data point to fit α. Store α per user, device and saved sauna.

Input paths:

- **Upfront.** Saved saunas, for example "home sauna, 85°C, Finnish dry", with preset values. This covers most sessions with no input at all.
- **After the session.** Correct values in the phone or web summary, then recompute everything derived from temperature.
- **External sensor.** ANT+ Tempe, or a BLE thermo-hygrometer via the Connect IQ BLE API. Check each sensor's own heat rating.

Data handling:

- Keep the raw sensor trace in the FIT file so values can always be recomputed.
- Tag each ambient value with its source: sensor-estimated, user-stated, or external sensor. Summaries and the AI layer weight values by source.

Values the sensor can't estimate:

- **Humidity.** User input or preset only.
- **Sauna type.** Must be explicit. It is a required field per session or saved sauna, because it breaks the temperature model:
  - Infrared: air at maybe 45–60°C, but radiant heat warms the watch directly. The blend model doesn't apply.
  - Steam room: about 45°C at 100% humidity, which gives a very different sensor signature.
  - Finnish dry, or Finnish with löyly.

## Health and profile data

Available on the watch, via Connect IQ:

- `UserProfile`: age, weight, resting HR, HR zones.
- `SensorHistory`: HR, stress, Body Battery, SpO2 and temperature, depending on the device.
- RR intervals during the session for HRV, via the sensor data listener.

Likely not exposed to Connect IQ: Training Readiness, HRV status, sleep score. **Verify against the current SDK.** If these are needed, they come from the Garmin Health API on a backend, which requires approval from Garmin's developer program.

User preferences include: default protocols, alert thresholds, saved saunas, sauna types, and the deployment profile.

## Advising and alerts

- Real-time advising, in both modes, runs on the watch as deterministic rules parameterized by the user profile.
  - App memory on the watch is tight.
  - An LLM round-trip through the phone proxy is too slow and too failure-prone during a session.
- In active mode, "adapting the plan" means adjusting the duration or target of the next phase based on in-session HR response, against thresholds from the profile.
- Alerts use `Attention.vibrate` and `Attention.playTone`. Conditions include:
  - phase too long
  - HR above threshold
  - device temperature near its rated limit
  - strap signal lost
- Framing: individualized real-time advice ("your HR is too high, get out") moves toward health claims. In the EU and Switzerland, keep it clearly in the wellness category, and word alerts and summaries accordingly.

## Recording

- `ActivityRecording.createSession` with a lap at each phase boundary.
- FIT developer fields for: phase, plan step, ambient temperature with its source tag, device temperature, and detection mode per segment.
- **Generic by default; cardio by explicit opt-in.** Depending on the sport type, Garmin may count an HR-heavy sauna session toward training load and fitness estimates. For example, 15 minutes at 130 bpm could look like training. Verify how CIQ-recorded sessions affect these metrics.

## Session summaries and evidence layer

- At session end, the watch posts a compact summary via `Communications.makeWebRequest`, which is proxied through Garmin Connect Mobile, to our backend.
- The backend performs analysis and RAG over the sauna and cold-exposure literature.
- Evidence caveat: the evidence base is thin compared with exercise. Much of it is observational cohort data, mostly the Finnish KIHD studies, plus small intervention trials. Summaries should be appropriately hedged, and the corpus curation should reflect this.

## Architecture

Three tiers:

1. **Watch.** State machine, active plans, rule-based advisor, alerts, wrist detection, boundary detection, temperature estimation, live device-temperature guard, recording with FIT fields.
2. **Phone and web.** Plan authoring (CIQ app settings are too limited for building schedules), saved saunas, preferences, post-session corrections, summary display.
3. **Backend.** Session ingestion, recomputation after corrections, α calibration, AI + RAG summaries, optional Garmin Health API integration.

## Staging

1. **On-watch v1.** Passive and active modes, plans, alerts, rule-based advisor, FIT fields, manual boundaries plus strap HR prompts, device-temperature warning and guard, wrist detection.
2. **Auto-detection.** Temperature slope and motion boundaries, ambient estimation with calibration, external sensor support.
3. **Backend.** Summaries, evidence layer, post-hoc corrections with recomputation, Health API integration.

Stage 1 alone already goes well beyond EmberDip.

## Open questions

- Which Connect IQ health metrics are actually exposed on target devices in the current SDK?
- Garmin Health API access: is it needed for v3, and is approval realistic?
- Effect of the chosen sport type on Garmin training load for CIQ-recorded sessions.
- Target device list and how to populate the per-device operating-temperature table.
- Wellness vs. medical claims: wording review for alerts and summaries.
- Exact on-wrist classifier thresholds (accelerometer variance), validated on real sessions.

## References

- [EmberDip (formerly Nordic) showcase thread, Garmin forums](https://forums.garmin.com/developer/connect-iq/f/showcase/445304/app-nordic-free-sauna-cold-plunge-contrast-session-timer)
- [Garmin owner's manual Specifications page, 60°C example](https://www8.garmin.com/manuals/webhelp/GUID-5D183A14-BB43-4A9B-B441-5F824214CE40/EN-GB/GUID-0ED1C47A-727F-4481-9911-BAF1FBC40166.html)
- [Garmin forum. 735XT in sauna (50°C rating)](https://forums.garmin.com/sports-fitness/sports-fitness/f/forerunner-735xt/123512/735xt-in-sauna-good-or-bad-idea)
- [Garmin forum. Fenix 6 in sauna (48°C internal reading at 85°C)](https://forums.garmin.com/outdoor-recreation/outdoor-recreation/f/fenix-6-series/239785/fenix6-sauna-spa-wellness)
