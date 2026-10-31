# Validation, installation safety and fork maintenance

## 1. Current verification status

Completed during planning:

- Official source location confirmed via https://gadgetbridge.org/.
- Full default-branch history cloned from Codeberg, public repository created under `mrsid18`, `origin` and `upstream` configured.
- GitHub `master` verified at source SHA `edbea98cbe1377001c0b8154ac20cb930e1db7f6`.
- Relevant source inspected; no application code changed.
- Local Java command reports Temurin 25; a conventional Android SDK directory exists. This does **not** establish that the required JDK 21 toolchain or SDK 37 is installed.
- Android builds, tests, emulators, real watch behavior, backup/import compatibility and performance have **not** been validated.

The plans are intentionally gated on these unknowns rather than calling source inspection a successful device test.

## 2. Build baseline to establish in P0

Inspected configuration declares Java toolchain 21, bytecode target 17, min SDK 23, compile SDK 37, target SDK 35, Gradle wrapper 9.8.2, Android Gradle Plugin 9.4.1, and Kotlin catalog version 2.4.20. Use the pinned repository configuration; verify availability and compatibility rather than substituting versions speculatively.

Relevant sources: [app build](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/build.gradle), [dependency catalog](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/gradle/libs.versions.toml), [Gradle wrapper configuration](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/gradle/wrapper/gradle-wrapper.properties).

Proposed commands, **not executed in this planning task**:

```bash
# From the fork checkout. Establish baseline before UI edits.
git status --short
git submodule status
./gradlew --version
./gradlew :app:tasks --all

# Expected from current flavor names; confirm with tasks output first.
./gradlew :app:assembleMainlineDebug
./gradlew :app:testMainlineDebugUnitTest
./gradlew :app:lintMainlineDebug

# Cross-flavor regressions if shared code changes, after confirming tasks.
./gradlew :app:assembleBanglejsDebug :app:testBanglejsDebugUnitTest
```

Review submodule URLs and initialize pinned submodules only as required by the build. Do not assume an SDK directory implies all platforms/build tools/licenses are present. Record exact SDK/JDK selection and dependency failures.

**CI gotcha:** [Woodpecker test configuration](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/.woodpecker/run_tests.yml) calls older-looking `testMainDebugUnitTest` and `testBangleDebugUnitTest`, while the inspected build declares `mainline` and `banglejs`. Discover tasks rather than copy those names blindly. [Forgejo build configuration](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/.forgejo/workflows/build.yaml) is not a GitHub Actions workflow and its marker-file reporting does not itself establish a failing final job. The GitHub copy is not automatically CI-enabled.

When implementation starts, add a minimal GitHub workflow only after the baseline succeeds: fixed JDK/SDK, wrapper validation, discovered unit-test/build/lint tasks, failure propagation, no release-signing secrets on untrusted PRs. Do not upload health databases or raw logs as artifacts. Device tests need their own controlled environment and cannot be replaced by a green unit-test badge.

## 3. Automated test strategy

Existing starting points: [DashboardViewModelTest](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/test/java/nodomain/freeyourgadget/gadgetbridge/activities/dashboard/DashboardViewModelTest.kt) and [DashboardRepositoryImplTest](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/app/src/test/java/nodomain/freeyourgadget/gadgetbridge/activities/dashboard/data/DashboardRepositoryImplTest.kt). Read their test style before extending. JUnit, Robolectric, Mockito and coroutine-test are already cataloged; prefer those to a new test framework.

### Pure transformations and data contracts

| Fixture | Required assertion |
|---|---|
| Supported metric, no rows | No fabricated zero or current value |
| Valid zero with proven interval | Displays zero, not missing |
| Unknown capability / unsupported | Distinct state; no futile remedial action |
| Null/sentinel/invalid measurement | No physiological number or misleading score |
| Different devices on same day | Selected-device result; no silent double count |
| Seconds versus milliseconds | Correct provider bounds and displayed time |
| Midnight sample / cross-midnight workout | Explicit inclusion policy, no accidental loss or duplicate |
| Overnight sleep / naps | Matches documented sleep attribution |
| 23/25-hour DST day / timezone change | Correct intervals; no forced 24-hour assumption |
| Today partly recorded / completed past day | No unqualified full-day comparison |
| Sparse HRV history | Missing intervals remain visible as gaps |
| Vendor baseline unavailable | No invented baseline/status |
| Locale and units | Correct labels and conversions, no double conversion |

Use synthetic records with no real identifiers. Fix clock/zone in new transformation tests. Test source-derived values separately from formatting.

### Dashboard state tests

- Real fixture adapter through the existing loader seam; view tests do not mock raw SQL internals.
- Loading → ready, loading → error, retry → ready.
- Same-day refresh retains existing content; date/device switch cannot flash a prior day's values as current.
- Rapid date switching and late results do not overwrite the active selection.
- View destruction/recreation and process state restoration do not duplicate listeners or lose selection.
- Widget removal during load is safe; mode changes preserve layout snapshots.
- Connection success alone cannot freshen metric timestamps.
- Permission failure affects only dependent functions.

### Presentation/navigation tests

- First-slice card route carries device/date; Back restores date/scroll.
- Modern/legacy preference mapping is reversible and idempotent.
- Unknown/hidden/duplicate navigation keys, no paired device, unsupported device, removed default device.
- Common renderer changes retain widget IDs, settings and launcher compatibility.
- UI fixtures cover each state in the experience specification; screenshots are useful but not substitutes for semantic assertions.

## 4. Hardware acceptance matrix

Record app commit, firmware, phone/Android version, selected device and test results locally. The public report contains only sanitized conclusions.

| Scenario | Pass condition |
|---|---|
| Existing pairing and initial sync | Same successful behavior as baseline; no surprise reset/auth-key request |
| Repeated sync with historical data | No duplicated records or changing totals caused by UI |
| Bluetooth off/disconnected | Stored history usable; clear reversible reconnect action |
| Interrupt sync / watch out of range | Saved values remain; partial/error copy honest; retry works |
| Background/foreground and process death | No new service loop, battery drain or lost UI state |
| Battery restriction / permission denial | Specific guidance; unrelated history remains usable |
| Two nights of wear and sync | Sleep date/stage/duration agree with verified underlying data |
| Workout with and without GPS | Correct list/detail; missing route not a total failure |
| Notification/alarm settings round trip | Existing control behavior unchanged |
| Multiple devices / device removed | Source stays explicit; no accidental aggregation |
| Switch legacy/modern repeatedly | No layout loss, lost settings, or duplicated widgets |
| App upgrade under test signing key | Settings/history retained on disposable installation |

Firmware updates, factory resets, account switching and experimental app-management changes are not part of routine redesign validation. They require separate explicit approval because they can disrupt a working watch.

## 5. Accessibility and usability gates

Test a narrow phone, normal phone and wide layout; minimum supported Android via emulator where feasible, plus the user's actual Android phone. Add RTL and long translated strings. Include light/dark, 200% fonts, TalkBack, keyboard/switch navigation and animations disabled.

Core tasks:

1. Find last night's sleep and describe which interval it belongs to.
2. Tell whether HR is live, recorded today, or old.
3. Explain missing HRV after a successful transfer without concluding that it is zero/good.
4. Open an earlier workout and return to the same list/date.
5. Sync a disconnected watch without deleting data.
6. Find notification settings and return after denying a permission.
7. Switch back to the legacy UI and recover the prior dashboard.

First review can be with the owner; before public release, aim for 6–8 varied participants plus dedicated accessibility checking. Proposed gate: at least 80% unassisted completion of core navigation tasks and **zero unresolved critical misunderstandings** about missing/stale values, data deletion or medical meaning. This is a design target, not a result already obtained.

## 6. Performance and privacy gates

Set device/dataset-specific budgets in P0. Initial targets, not measurements:

- Cached Today usable within 1 second of destination entry on the named reference phone.
- Sync action provides visible acknowledgment within 100 ms; transfer duration remains hardware-dependent.
- No DB work on main thread and no new periodic polling/background worker for the redesign.
- No material regression from baseline in cold launch, scroll jank, peak memory or battery use; investigate over 10% regression on repeatable runs rather than hiding it with a new cache.
- W/M chart requests bounded to the selected range; test a realistic multi-year database before longer-range browsing.

Audit the built mainline merged manifest and dependencies: no newly introduced Internet permission, analytics or account requirement. Existing other flavors may differ. Check actual logs/crash exports for health values, device identifiers, routes and credentials. No sensitive screenshots or dumps in public CI. Do not claim the entire app was privacy-audited merely because the UI adds no network code.

## 7. Installation, data and signing safety

**Never uninstall the user's working app as the first test step.** A self-signed APK normally cannot update an F-Droid/upstream-signed installation with the same app ID. Changing the ID creates a separate sandbox; it does not automatically copy data, pairing credentials or permissions.

Proposed alpha route:

1. Inspect existing backup/import paths and identify exactly what is included: DB, preferences, auth material, files and version compatibility. Do not promise that a backup contains everything before inspection.
2. Make an explicit user-approved backup; verify integrity and restore on a disposable install before touching the working installation. Store backup outside Git, protected as sensitive health data.
3. Use a separately identified alpha build. Audit provider authorities, receiver permissions, action names, redirect schemes, external paths and integration references for hardcoded application IDs. A suffix alone may not isolate every interaction.
4. Import only through existing supported paths after compatibility is proven. Do not copy SQLite files between app sandboxes by an improvised production migration.
5. Grant required Android permissions independently. If testing actual watch sync, stop the other app's watch connection to prevent competing clients; do not assume two apps can own the device reliably.
6. Keep the original installation and verified backup intact. If alpha fails, stop it and reconnect the original app; no factory reset is part of this rollback.
7. A production signing key is created/backed up securely before any release intended for future upgrades. Never commit the key or credentials. Do not silently transition debug users to a differently signed package with an expectation of in-place update.

First milestones require no schema migration. UI rollback is a preference change, not a database downgrade. If a later change requires a schema update, stop and design its migration/restore strategy separately; compatibility is not guaranteed by shared ancestry.

## 8. License, attribution and identity

Preserve [LICENSE](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/LICENSE), source notices and existing third-party/artwork terms. The project states AGPLv3 and individual source headers commonly specify version 3 or later; distribution must follow the actual applicable terms. Make corresponding source for distributed modified binaries available as required, including build instructions and modifications. A tagged source commit matching each release is the operational default.

Review [artwork licensing](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/LICENSE.artwork) before reusing branding. Public fork releases should have a distinct name/icon and a clear “based on Gadgetbridge; not an official release” statement. Do not use Whoop/Apple/Amazfit assets or imply affiliation. Inspiration is interaction hierarchy, not a branded clone. Obtain legal advice if distribution obligations are unclear; this plan is not a legal opinion.

Upstream [contribution policy](https://github.com/mrsid18/Gadgetbridge/blob/edbea98cbe1377001c0b8154ac20cb930e1db7f6/CONTRIBUTING.md) requires clean-room behavior and disclosure of AI-assisted contributions with human responsibility. The fork plans are AI-assisted. Any upstream submission must be understood/tested by its human author and disclosed appropriately.

## 9. Fork/upstream workflow

- Keep `upstream` pointing to Codeberg and `origin` to the user's GitHub copy.
- Leave upstream baseline history intact; work on short feature branches. Avoid bulk reformatting, package renames or unrelated cleanup.
- Fetch upstream regularly; review changes touching dashboard/data/device support before merging into the integration branch.
- Use a normal reviewed merge for shared branches; no automatic force-push or rewriting published history.
- Resolve upstream conflicts with tests around affected widget/data/navigation interfaces. Rerun build, unit tests, lint and the relevant hardware subset.
- Contribute narrow generic fixes upstream where appropriate; keep product-specific presets/branding isolated. Do not assume upstream will accept the redesign.
- Tag tested source and record firmware/Android coverage and known issues for every APK release. GitHub releases/Actions are new fork infrastructure, not something inherited from Codeberg automatically.

## 10. Release checklist

- [ ] Baseline and changed-scope automated gates pass; pre-existing failures documented separately.
- [ ] Real Balance 2 capability matrix complete for every advertised metric.
- [ ] No fabricated recovery/strain score, no preview values in production.
- [ ] Date, units, provenance, missing-data and partial-sync semantics pass fixtures.
- [ ] Modern-to-legacy rollback preserves customization and history.
- [ ] Backup/restore and signing/upgrade path rehearsed on disposable installs.
- [ ] Accessibility and critical usability misunderstandings resolved.
- [ ] Performance compared to baseline; no new polling/network behavior.
- [ ] Hardware sync/notification/workout subset passes and overnight observation complete.
- [ ] Source/license/attribution/branding reviewed; no credentials or health data committed.
- [ ] Release artifact matches source tag; known limitations and rollback steps published.

Unmet safety/correctness gates block widening distribution. A good-looking screenshot is not sufficient evidence to replace a working daily-use app.
