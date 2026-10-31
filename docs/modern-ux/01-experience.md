# Experience specification: daily focus, trustworthy history

## 1. Product contract

The app answers four questions without requiring users to understand Bluetooth or sample providers:

1. **Today:** What has been recorded for me today and last night?
2. **Health:** How have my measurements changed over time?
3. **Activity:** What workouts did I record?
4. **Watch:** Is my watch connected, has data transferred, and where are its controls?

Whoop contributes hierarchy and a daily focal point, not its Recovery/Strain algorithms. Apple Health contributes legible measurements, favorites, and drill-down history, not iOS navigation or clinical implications. Use original artwork, Android Back, Material controls, and local storage.

### Launch scope

Today, reusable metric states, Health browsing, existing workout routes with refreshed presentation, and a grouped Watch landing screen. Balance 2 is the acceptance-test device; do not delete other devices or hardcode metric support from its model name. Existing screens remain reachable throughout rollout.

Out of scope: AI coach, chat, new backend, subscriptions, social feed, proprietary recovery reconstruction, new sensor protocols, firmware installation changes, custom map engine, Health Connect changes, or a wholesale settings rewrite.

## 2. Visual language

- Neutral canvas; lightly elevated cards; clear hierarchy instead of a wall of equally weighted gauges.
- System light/dark with equally complete contrast testing; no mandatory black/neon theme. Reuse current theme attributes where possible; do not globally recolor legacy screens in the first change.
- Proposed spacing scale: 4/8/12/16/24/32 dp. Screen gutters 16 dp; section separation 24 dp; cards approximately 20 dp radius, subject to existing widget sizing constraints.
- Type: approximately 28 sp screen title, 32 sp primary value, 16 sp body, 12–14 sp supporting labels. These are scalable roles, not fixed-height layouts. Use system fonts and tabular digits where available.
- One action accent; stable metric colors (sleep purple, movement blue/teal, heart rose). Color never carries status alone. Low steps are not an error.
- Common card grammar: metric name → date/interval → value + unit → supporting chart → quality/source note. Never show a large naked number.
- Motion only for state continuity, around 150–250 ms when enabled. Honor reduced/disabled animations. No chart animation on every sync event.
- Touch targets at least 48 dp; normal text contrast at least 4.5:1; 200% fonts without lost values/actions; TalkBack order follows visual hierarchy.
- Narrow phone: one column for large cards; two compact cards only when text fits. Wide layouts may use two columns. Large text may force one column. Do not force the existing fixed widget cells to clip multiline content.

## 3. Today

Illustrative hierarchy, not real measurements:

```text
Today                    Calendar
Balance 2 · Disconnected · Sync
Saved data available · Last transfer 08:42

LAST NIGHT
Sleep       7 h 20 min
Oct 30–31 · measured by Balance 2
[small sleep-stage chart, if available]

MOVEMENT SO FAR
Steps       4,120 / optional goal
[hourly bars with gaps for unknown coverage]

HEART
Latest recorded HR       68 bpm · 09:15
[never imply live measurement]

LATEST WORKOUT
Walk · Yesterday · 32 min       >

Recovery signals / pinned measurements
HRV, resting HR, etc. only if supported
```

### Required interactions

- Date picker, previous/next day, and explicit Today shortcut. Block future dates where no meaningful data exists.
- Visible Sync action, not pull-to-refresh alone. Feedback uses actual service status; never invent a percent complete.
- Tap a card to open the existing matching detail page with the selected device and date. Back preserves date, scroll, and device selection.
- Use existing widget reordering/customization rather than building a second layout editor. Apply the new preset only by explicit choice; preserve the old layout and its restore action.
- When viewing another day, call the heading by that date; movement is no longer labeled “so far.” Sleep uses the actual overnight interval, not a misleading calendar-day total label.
- Pinning, hiding, and reset to defaults use existing instance/layout persistence when feasible. No silent replacement of customized dashboards.

### First useful slice

Ship sleep duration, steps, recorded heart rate, a recent workout link, and device/sync affordances. No natural-language daily assessment is required. Add a factual summary later only if the observation semantics and freshness are trustworthy; never turn a missing HRV value into a green recovery verdict.

## 4. Health

The destination is a metric browser, not a duplicate Today grid. Start with supported favorites and grouped rows: Sleep, Heart, Movement, Other measurements. Search can wait until the catalog is difficult to scan. Unknown/unsupported metrics belong in the capability explanation, not a screen full of locked teasers.

### Detail template

1. Title and plain-language definition.
2. Selected device/source and period.
3. Latest observation, explicit units and timestamp; keep this distinct from period mean/min/max.
4. D/W/M period selector initially; longer/custom ranges after bounded-query performance is proven.
5. Chart with labeled axes, missing-data gaps, and selected-point readout.
6. Accessible textual summary and a navigable list/table alternative.
7. Comparison or vendor baseline, only when compatible and actually available.
8. Source, completeness, and “How this is measured” explanation.

MVP uses existing chart activities behind these routes. Replace details one metric at a time, beginning with Sleep, then Heart, then Steps/HRV. Do not make every chart redesign a prerequisite for Today.

### Data semantics

- Sleep: separate asleep duration from time in bed and awake time; expose stage unavailability without discarding duration. Label overnight attribution and naps using existing sleep rules until explicitly changed and tested.
- Heart rate: “latest recorded,” not “current,” unless a live observation really exists. Resting heart rate is not the minimum sample of the day.
- HRV: label nightly and weekly values separately; never compare incompatible statistics or mix devices. Device-reported baseline remains device-reported, not an app-derived diagnosis.
- Steps: show zero only if coverage/source establishes it. A default numeric zero from an aggregate is not proof the watch recorded zero.
- SpO2/stress: units and watch-specific definitions must be explicit; no medical alerts added by this redesign.
- Charts must not visually interpolate across absent days as if there were measurements. If existing charts do this, fix the specific renderer with regression tests before claiming gap-aware behavior.
- Compare completed days, or explicitly matched partial periods. Do not compare today's morning steps with an entire previous day as if equivalent.

## 5. Activity

Retain existing workout identity, loading, parsers, export, and detail routes. Modernize list spacing and summary hierarchy first.

- Period selector and simple reversible activity-type filter.
- List row: sport, date/time, duration, one relevant secondary measure.
- Header: session count and total workout time; sum distance only for compatible activities and label the scope.
- Detail: sport/date → duration/distance or sport-appropriate measures → HR/laps when available → route when available → existing actions.
- A missing route is not a failed workout. Empty periods say “No recorded workouts in this period,” not “You did no exercise.”
- No new manual editing, live controls, deduplication algorithm, or export format is introduced by the UI plan. Test existing duplicate handling and report problems as separately scoped work.

## 6. Watch

A grouped landing screen that links to existing controls rather than reimplementing them:

- Device identity and switcher for multi-device users.
- Connection, battery with observation time if available, transfer state, and last successful transfer if genuinely recorded.
- Sync now and expandable known error; offer reversible recovery first.
- Notifications; health recording; display/apps/watch faces; alarms/calendar; device maintenance. Show only supported routes and preserve experimental gating.
- Advanced links preserve all existing settings, debug, backup/export/import, and device management access.
- Separate Disconnect, Unpair, Reset, and Delete data. Do not flatten them into “Remove.” Keep existing confirmations and accurately describe consequences.

Bluetooth permission denial must not block stored charts. Notification access denial affects notifications, not unrelated sleep history. “Connected” never means “all data is current.” Do not recommend clearing storage to recover a routine sync failure.

## 7. Shared state contract

Model independent dimensions rather than one mutually exclusive status enum:

| Dimension | Required states |
|---|---|
| Capability | Unknown / supported / unsupported |
| Observation | Loading / absent / present / invalid / read error |
| Quality | Unknown coverage / partial / verified sufficient for the displayed claim |
| Freshness | Observation time known/unknown; current-claim freshness only where cadence is established |
| Transport | Disconnected / connecting / connected / transferring / failed, as actually exposed |

Each present metric carries source identity, units, observation timestamp or interval, and aggregation definition. Last successful transfer is separate metadata. Some upstream loaders erase these distinctions; implementation must not invent information to fill the gaps.

| Situation | Presentation | Action |
|---|---|---|
| Capability unknown | Support not yet confirmed | Capability detail; no promise |
| Unsupported | Omit from default cards; explanation in Watch | No futile Sync CTA |
| Supported, never received | No sleep data received yet | Verified recording/setup guidance only |
| Empty period | No observations in this period | Choose another period |
| Refreshing with cached values | Retain values + Updating | No blocking full-screen spinner |
| Read/transfer failure | Saved data remains available where valid | Retry; known cause only |
| Partial samples | Values plus Incomplete data | Explain affected interval |
| Old observation shown on Today | Last measured [absolute date/time] | Never relabel as today/live |
| Historical complete record | Its historical date, no stale warning merely for age | Normal browsing |
| Invalid/sentinel value | Reading unavailable | Never display as zero or physiological reading |
| Verified zero | 0 with unit and interval | No absence styling |

Freshness thresholds are metric-specific and must follow verified measurement cadence. Until known, show measurement time without claiming “up to date.” A successful sync cannot mark all cards fresh. Unknown timestamps are disclosed. Never use `previewData()` as a production fallback.

## 8. Recovery and strain policy

Launch with **Recovery signals**, not a composite Recovery score: independently valid sleep, resting HR, and HRV where supported. No “push harder,” “safe to train,” medical clearance, or diagnostic advice.

A vendor score may be displayed only after proving that this device/firmware supplies it through the existing implementation; retain its original name, observation time, and “Reported by watch” attribution. A generic widget named Training Readiness does not establish Balance 2 support.

No new baseline is needed for MVP. A later local trend/baseline feature needs its own written definition, minimum valid sample count, compatible measurement protocol, missing-data rules, formula version, tests, and clearly non-clinical language. Do not bake arbitrary 14/28-day thresholds or weighted 0–100 scores into this redesign.

## 9. Privacy and accessibility acceptance

- No new network permission, telemetry, cloud service, or account requirement for the mainline experience.
- No health values, raw routes, auth keys, device addresses, or user databases in public issues, screenshots, fixtures, or CI artifacts.
- Preserve existing optional integration consent. Do not silently turn on Health Connect or alter export behavior.
- Every chart has a text alternative; TalkBack announces metric, value/unit, interval, and quality together.
- Full navigation and Sync work without gestures; keyboard/switch focus and Android Back behave normally.
- Locale-aware dates, decimal separators, metric/imperial units, RTL, and long translations are exercised.
- All specified data states have synthetic fixtures and reviewed copy. Absence/staleness must never be mistaken for zero or good health.

## 10. Design review gate

Review light and dark Today with populated, empty, stale, and failed-sync fixtures before expanding the design. Ask the user to find last night's sleep, identify whether a displayed HR is live, open yesterday's workout, and locate notification settings. Pass only when the date/source distinction is understood and legacy settings remain discoverable. Iterate on hierarchy before investing in animation or rebuilding charts.
