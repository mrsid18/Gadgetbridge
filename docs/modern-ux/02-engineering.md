# Engineering and staged delivery

Status: proposal based on upstream commit `edbea98cbe1377001c0b8154ac20cb930e1db7f6`. No application changes are included in this planning branch.

## 1. Source map: reuse before replacing

Links are pinned to the inspected source snapshot, not future moving `master`.

| Existing module / file | Observed behavior | Consequence for the plan |
|---|---|---|
| [ControlCenterv2](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/activities/ControlCenterv2.java) | ViewPager2; dashboard/workouts/devices tabs; capability-filtered `main_screens`; `default_main_screen`; device and theme events; live HR TTL logic | Extend navigation incrementally; preserve preference keys, device fallback, permissions, and Back behavior |
| [DashboardFragment](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/activities/DashboardFragment.kt) | ViewBinding; grid binder; drag controller; date selection; new-data receiver; lifecycle cleanup | Existing screen can host a new presentation without replacing synchronization |
| [DashboardViewModel](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/activities/dashboard/DashboardViewModel.kt) | Injected repository and loader, saved date/columns, StateFlow, cancellable per-instance loads, cached values during same-day reload | Extend current tests and state flow; do not create a duplicate dashboard state owner |
| [DashboardUiState](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/activities/dashboard/DashboardUiState.kt) | Widget states are Loading / Ready(Any?) / Error | These states alone cannot express source, sample absence, partial data or freshness; add information narrowly where needed |
| [DashboardRepository](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/activities/dashboard/data/DashboardRepository.kt) | Layout, settings, notices, editing, snapshot/restore; nonblocking contract | This is NOT a general health-metric repository. Do not burden it with every health query |
| [DashboardWidgetLoader](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/activities/dashboard/data/DashboardWidgetLoader.kt) | Resolves each instance's config/devices and invokes widget loading on IO | This is the existing test seam for fixture-backed versus real loading |
| [GBWidget](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/widgets/GBWidget.kt) | Stable IDs, variants, capability check, suspend load, preview, create/bind View, click action | Add presentation variants before parallel metric implementations. Preserve IDs/config and launcher compatibility |
| [WidgetDataScope](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/widgets/WidgetDataScope.kt) | Read-only DB work on IO; daily totals; samples; workout summaries; per-widget device scope | Reuse queries, but inspect aggregation and time semantics before exposing new claims |
| [SleepWidget](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/widgets/impl/SleepWidget.kt) | Duration, stages, score and hypnogram depend on variant/data; chart route `sleep` | Most sleep display inputs already exist; do not write a second sleep algorithm |
| [HrvWidget](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/widgets/impl/HrvWidget.kt) | Nightly/weekly values and vendor baseline; returns null on caught errors; nullable numeric values can become zero; history graph filters missing points | Distinguish unavailable fields and errors before modern claims; preserve gaps rather than visually joining missing days |
| [AmazfitBalance2Coordinator](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/devices/huami/zeppos/watches/AmazfitBalance2Coordinator.java) | Inherits Zepp OS; declares GPX uploads, control center, to-do, hotspot, FTP, Bluetooth calling | Retain device integration; route controls by capabilities, not optimistic design mockups |
| [ZeppOsCoordinator](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/devices/huami/zeppos/ZeppOsCoordinator.java) | HR, stress, SpO2, sleep score/REM/awake declarations; HRV depends on display items; manual HR disabled; app management experimental | Source declarations are not hardware proof. Do not expose manual measurement or remove experimental gates |
| [App build](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/build.gradle) and [dependency catalog](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/gradle/libs.versions.toml) | Views/Material, lifecycle Kotlin, existing charts, GreenDAO; no Compose catalog setup found; Java toolchain 21, target bytecode 17, min SDK 23, compile SDK 37, target SDK 35; mainline/banglejs flavors | Preserve toolchain/dependencies initially; do not mix a UI migration with build modernization |

Other locations discovered for implementation-stage inspection: dashboard grid binder, widget registry/layout/defaults/migration, metric tile models/rendering, widget implementations for steps/heart/workouts, workout list ViewModel, device settings routes, theme resources, and their tests. Their existence is confirmed, but not every implementation/caller was audited. Read and trace each before editing.

## 2. Architecture decision

### Recommended: evolve Views and existing widgets

Keep the current activity and fragments, ViewBinding, Material styling, widget IDs/layout persistence, existing database and device services. New UI is presentation and carefully bounded metadata, not a replacement synchronization stack.

Flow to preserve:

```text
Watch / existing device support
    -> existing DB / existing providers
    -> WidgetDataScope + metric widget loadData
    -> DashboardWidgetLoader (real or fixture adapter)
    -> DashboardViewModel state
    -> DashboardFragment / widget views
    -> existing chart, workout and settings routes
```

**Seam:** `DashboardWidgetLoader` already admits a real adapter and fixture adapter for dashboard state tests. Use that seam. Keep metadata interpretation in a small module near the metric loaders; do not build a generic repository/event-bus/cache stack around existing functionality.

Where Today and a newly rewritten Health detail both need the same query, extract the real duplicated measurement logic into one deep module with a small query interface. Inputs should be explicit device identity, local date/range, timezone, and measurement kind; outputs should be immutable data/metadata, not Views or live DAO sessions. Do this only when the second caller exists. Until then, reuse existing widget data/routes rather than introduce a hypothetical abstraction.

### Compose alternative, explicitly not the default

If an isolated design trial demonstrates a material advantage, spike one metric card in a Fragment-owned ComposeView with lifecycle disposal and current ViewModel state. Prove compiler/toolchain compatibility, theme interoperability, TalkBack, min-SDK support, existing chart interop, and binary/startup cost. No approval of a full Compose migration is implied. Abandon the spike if it requires unrelated upgrades or duplicates the existing renderer. Do not deliver two maintained UI frameworks merely for fashion.

## 3. Metric and capability inventory

Create the following evidence table during P0 using the user's firmware and a private local test dataset. Fill with actual observations, not demo values:

| Metric | Source evidence now | Gate before shipping |
|---|---|---|
| Steps / movement | DailyTotals through WidgetDataScope; step capability filters | Recorded zero vs absent coverage; source-specific totals; goal semantics |
| Sleep duration/stages | SleepWidget and scope; Zepp OS REM/awake flags | Overnight attribution, nap behavior, sample coverage, actual Balance 2 stages |
| HR | Zepp OS HR flag; recorded sample flow; host live TTL behavior | Latest vs live, valid-value rules, source timestamp |
| Resting HR | Zepp OS heart-rate-stats flag | Inspect provider and confirm supplied resting statistic; never substitute minimum |
| HRV | HrvWidget summary provider; capability conditional on HRV/readiness display item | Actual paired capability, valid nightly/weekly fields, baseline provenance |
| Sleep score / stress / SpO2 | Zepp OS declarations and existing widget types | Actual firmware data, missing sentinel values, units/definitions |
| Workouts | Zepp OS summary parser / track provider; existing list/details | Actual sport fixture, missing route, interrupted transfer, date scope |
| Battery / transfer metadata | Existing device/state infrastructure; needs deeper inspection | Determine which observation/success timestamps are really persisted |
| Training readiness / load / body energy | Generic widget files exist | Not assumed supported by Balance 2; inspect coordinator and stored data first |

Record firmware, app commit, metric capability result, provider, sample units/timestamp unit, null/sentinel values, attribution interval, observed successful data, and known limitations. Keep private identifiers and health samples out of the public evidence document.

## 4. Data correctness issues to address narrowly

These are observed implementation semantics to account for, not a claim that all are upstream bugs:

1. `WidgetDataScope.stepsTotal()` sums across selected devices; sleep totals average over devices with sleep. The modern Balance 2 preset should scope every card explicitly to the user-selected device. Do not silently redefine legacy aggregate behavior.
2. Several totals return zero when no samples exist. If coverage cannot be proven, the modern view must show “No data received” or “Coverage unknown,” not confident zero. Carry presence evidence from existing providers where available; do not assume zero is missing either.
3. Activity query timestamps use epoch seconds; HRV summary queries multiply by 1000. Preserve provider-specific units and test conversion at the seam.
4. Day queries use system timezone and end-of-day bounds. Test exact midnight, DST, and timezone changes. Do not replace with fixed 86,400-second buckets.
5. Workout-summary query in the scope uses strict start/end comparisons and requires full containment. For a new day summary, decide whether to show workouts by start date or any overlap; test cross-midnight workouts before claiming completeness. Keep that change isolated from unrelated charts.
6. HrvWidget catches exceptions and returns null. A modern error state cannot be inferred reliably from that null. If changing the load contract, audit all callers and retain compatibility; never show a false “not recorded yet” for every DB error.
7. Existing HRV nullable fields default to zero and its graph can omit missing points while joining the remaining ones. New presentation should preserve field validity and render gaps; protect legacy/widget/launcher behavior with tests.
8. A connection event or generic new-data broadcast is not proof of successful full synchronization. Do not store “last successful sync” merely when refresh was requested. If no reliable success event exists, ship honest transfer status and individual observation timestamps first.
9. Scope functions use global application state and the system clock. Inject clock/zone into newly extracted pure transformations when needed for deterministic tests, but do not refactor the entire database infrastructure to make one screen testable.

## 5. Delivery backlog

Effort ranges assume one experienced Android developer, existing code builds, and timely access to a Balance 2. They exclude waiting for user review, firmware investigations, public branding decisions, and upstream changes. Each item is independently reviewable; do not combine all milestones into one PR.

### P0 — baseline, safety and evidence (2–4 engineering days)

**Depends on:** none.

- Compare installed app version to source snapshot; preview upstream dashboard configuration before adding code.
- Discover actual Gradle tasks, establish baseline build/test/lint; record existing failures separately.
- Read scoped callers and tests from source map; map exact workout/detail/settings routes and extras.
- Inventory Balance 2 metric support with firmware and a private local backup; validate backup/restore on a disposable installation, never the only copy.
- Decide separate application identity and stable signing strategy for alpha; audit authority/action strings before changing it.
- Record cached Today timing and query count on a named reference phone with realistic synthetic history.

**Outputs:** baseline receipt, capability evidence matrix, build recipe, safety checklist, screenshot inventory held privately if it contains health data.

**Exit:** existing functionality reproducible; no data-safety unknown blocks alpha installation; supported first-slice metrics identified. If build fails, resolve the concrete baseline problem before promising dates.

### P1 — visual contract and synthetic preview (2–3 days)

**Depends on:** P0 build/source facts; can sketch fixture-only designs before hardware evidence completes.

**Write scope:** new scoped resources/presentation styles; development-only fixtures; no protocol or schema changes.

- Implement two theme previews for one Today layout, not two independent products.
- Specify card typography, source/date labels, chart alternatives, state copy, touch targets and large-text behavior.
- Use existing widget preview mechanism or test-only fixtures; never default production data to previews.
- User reviews populated, empty, disconnected and failed-refresh examples.

**Tests:** screenshots or reviewed captures at narrow/normal width, light/dark, 100/200% text; TalkBack semantics checks.

**Exit:** user approves hierarchy; no clipped values/actions; stale and missing data cannot be read as healthy/zero.

### P2 — trustworthy Today vertical slice (4–7 days)

**Depends on:** P0 safety/data evidence and P1 design approval.

**Write scope:** dashboard fragment/binder/state/viewmodel as needed; scoped widget variants/default preset; first-slice loaders only; tests/resources. Avoid changes to device protocol and DB schema.

- Add opt-in modern presentation preference with unchanged legacy default during alpha.
- Preserve existing instance IDs/config; take a layout snapshot before applying the proposed preset. Distinguish this preference from upstream's own “restore classic design” action.
- Scope preset widgets to an explicit selected device. Handle device removed/unavailable without selecting a different source silently.
- Render sleep, steps, recorded HR and recent workout; show unknown provenance/coverage honestly where upstream lacks it.
- Add measurement metadata and validity narrowly for these loaders; propagate real loading errors rather than masking them as absence.
- Reuse existing date navigation, cancellation, new-data refresh and card detail routes. Avoid duplicate reload listeners or DB queries on main thread.
- Expose existing Sync through the current host/service path; retain cached values during refresh and failure.

**Tests:** ViewModel cancellation/date/device races, loading/ready/error transitions, fixture absence vs zero, source scoping, Back/date preservation, live-vs-recorded HR, rotation/process recreation, widget/launcher regression where common renderer changes.

**Exit:** real Balance 2 observations render with parity against existing screens for the same date/device; retry and legacy toggle work; no new account/network/schema requirement. This is the first useful alpha.

### P3 — navigation and Health browsing (3–5 days)

**Depends on:** P2.

**Write scope:** ControlCenterv2 tab mapping/pager, navigation resources/preferences, new Health landing fragment and tests.

- Keep stored `dashboard`, `workouts`, `devices` keys stable; change display labels only in modern mode to Today, Activity, Watch. Introduce a `health` key without rewriting old preferences silently.
- Modern mode's four-tab configuration must preserve the old `main_screens` ordering/hidden choices for rollback. Define a reversible opt-in mapping rather than reuse a destructive migration.
- Handle no devices, unsupported activity device, removed device, invalid default tab, and hidden tabs with a reachable Devices/Watch fallback.
- Health landing presents capability-filtered metric rows and pinned existing metrics; deep-link existing detail routes first.
- Preserve original chart screen access and drawer/settings routes; test saved pager identity and return state.

**Tests:** preference mapping/idempotence, invalid/duplicate keys, hidden/default tabs, capability changes, mode switching, process recreation, Back/accessibility focus.

**Exit:** four distinct destinations on Balance 2; other devices retain usable navigation; no lost settings or duplicated tabs.

### P4 — reusable Health detail, Sleep first (4–7 days)

**Depends on:** P3 and metric validity work in P2.

**Write scope:** new or adapted detail rendering/transformations, relevant existing chart routes, metric tests. No wholesale chart-library replacement.

- Start with Sleep duration/stages, D/W/M navigation, clear overnight attribution, source/quality detail and accessible text summary.
- Reuse existing sleep calculations. Extract only repeated query logic shared with Today into a deep module; document inclusive/exclusive bounds and units.
- Add heart-rate detail next; add HRV only after its conditional capability and actual fields are validated.
- Preserve chart gaps, units, valid comparison periods and known baseline provenance. Do not derive new baselines/scores here.
- Bound queries and chart resolution; measure history browsing before introducing caches. Preserve raw data for detail selection.

**Tests:** overnight/nap/DST/timezone fixtures, zero/null/invalid data, known vs unknown coverage, sparse chart gaps, long ranges, text alternatives, locale units.

**Exit:** redesigned details agree with source values and do not silently alter existing analysis; unconverted metrics still route to legacy details.

### P5 — Activity and Watch polish (3–5 days)

**Depends on:** P3; can run alongside P4 with disjoint write scopes.

**Write scope:** workout presentation only and a Watch landing/router; existing settings/device command paths remain authoritative.

- Improve workout list hierarchy; reuse identity, filtering, list state, parsing and detail actions.
- Confirm midnight inclusion policy for Today/recent workout; add separate focused query fix only if required.
- Group Watch navigation into supported settings categories; display real connection/transfer states and observation timestamps where known.
- Preserve experimental flags and confirmations. Do not offer manual HR measurement while upstream marks it unsupported.
- Keep advanced device/settings, backup and troubleshooting reachable; no account or new background polling.

**Tests:** workout with/without route, partial sync, device switch/removal, notification permission denial, busy transfer, failed retry, settings round trip, disconnected history.

**Exit:** core daily tasks are clearer, with unchanged device behavior and no unreachable legacy controls.

### P6 — hardening and opt-in beta (3–5 days plus observation time)

**Depends on:** P4 and P5.

- Run the full validation matrix, min-SDK/current-target checks, accessibility and performance measurements.
- Validate backup/restore and signed upgrade path on disposable installs; rehearse rollback to the old UI without data migration.
- Exercise normal watch use across at least two overnight sync cycles; prefer a week-long personal alpha before broader release.
- Audit permissions/network/logs and final upstream delta; document unsupported features rather than adding speculative fallbacks.
- Keep opt-in until acceptance passes; change default only in a separate reviewed release change.

**Exit:** no critical data-loss, connection, date/source, or accessibility defects; explicit known-issues list; reproducible tagged artifact and corresponding source.

### Planning range

P0–P6 totals roughly **21–36 engineering days**, with the first useful alpha after P0–P2 (roughly 8–14 days). These are contingent planning ranges, not measured estimates or commitments. Prefer shipping P2 for feedback over completing the entire roadmap before the user sees anything.

## 6. Verification surfaces and dependency discipline

Before each implementation PR, enumerate callers/tests/resources/config affected by the exact edit. In particular:

- Widget interface/rendering changes can affect launcher widgets and every device's cards; keep compatibility or test all consumers.
- Navigation changes reach preference arrays, menu/ID resources, pager identity, default selection and saved state—not just labels.
- Theme changes reach legacy screens; scope styles until visual regression checks exist.
- Loader metadata changes reach null/error handling and fixture adapters; do not infer metadata in the View.
- No changes to generated entities by hand. A true storage requirement needs a separate schema/migration proposal and backup/rollback testing.
- Shared task work may parallelize P4 and P5 only after agreed route/state contracts; each contributor owns disjoint files. Lead reviews the combined diff and runs final gates.

## 7. Deferred work and decision triggers

- Compose: only after a bounded experiment demonstrates concrete maintenance/UX value.
- New local baseline/index: only after a separately reviewed methodology and missing-data contract.
- New sync timestamps in storage: only if existing events cannot support honest status; a separate narrowly scoped persistence change.
- More devices: retain compatibility now; expand hardware certification later.
- Generalized health repository: only when multiple actual callers duplicate nontrivial query/validation logic.
- New dependency: only when the existing library/platform cannot meet a documented acceptance criterion.
