# Balance 2 modern UX — implementation plan

Status: proposed; source-inspected, not implemented or hardware-validated.

## Decision in one minute

Build **one hybrid experience**: Whoop-inspired daily focus, Apple Health-inspired readable metrics and history, and native Android behavior. Preserve Gadgetbridge's local-first device integration. Do not copy either brand's assets, proprietary scores, or exact interface.

**Start with a polished Today screen on the existing Android Views/widget architecture.** Current upstream already has Kotlin ViewModels, state flows, configurable widgets, charts, and data loaders. Compose is not required to make this modern; a broad rewrite would increase risk without proving the experience.

The first milestone is real sleep, movement, and heart-rate data, explicit missing-data states, working detail navigation, and safe return to the existing dashboard. No Bluetooth rewrite, database migration, account, cloud service, invented recovery score, or firmware work.

## Read in order

1. [Experience specification](https://github.com/mrsid18/Gadgetbridge/blob/docs/balance2-modern-ux-plan/docs/modern-ux/01-experience.md): visual direction, screen behavior, data-state rules, and scope.
2. [Engineering and delivery plan](https://github.com/mrsid18/Gadgetbridge/blob/docs/balance2-modern-ux-plan/docs/modern-ux/02-engineering.md): source map, implementation seams, dependency-ordered tasks, estimates, and acceptance criteria.
3. [Validation and release plan](https://github.com/mrsid18/Gadgetbridge/blob/docs/balance2-modern-ux-plan/docs/modern-ux/03-validation-release.md): test matrix, build gates, data safety, signing, upstream maintenance, and release checklist.

Source links are pinned to the inspected commit; plan links point to the planning branch.

## Repository and source baseline

- Public independent fork: https://github.com/mrsid18/Gadgetbridge
- Official source and local `upstream`: https://codeberg.org/Freeyourgadget/Gadgetbridge.git
- Local `origin`: `git@github.com:mrsid18/Gadgetbridge.git`
- Checkout: `/Users/shmehta/Projects/Gadgetbridge`
- Planning branch: `docs/balance2-modern-ux-plan`
- Source snapshot: `edbea98cbe1377001c0b8154ac20cb930e1db7f6`, upstream `master`.
- This is a Git-history-preserving cross-host copy, **not** a GitHub-native fork-network relationship. The initial GitHub default branch contains the upstream snapshot; these proposals live on the planning branch.
- Only the default branch was initially pushed. Other upstream branches, tags, issues, releases, and CI secrets were not mirrored. Local submodules were not initialized.

Browser links: [experience](https://github.com/mrsid18/Gadgetbridge/blob/docs/balance2-modern-ux-plan/docs/modern-ux/01-experience.md), [engineering](https://github.com/mrsid18/Gadgetbridge/blob/docs/balance2-modern-ux-plan/docs/modern-ux/02-engineering.md), [validation](https://github.com/mrsid18/Gadgetbridge/blob/docs/balance2-modern-ux-plan/docs/modern-ux/03-validation-release.md).

## Evidence and assumptions

Inspected the current build configuration, contribution policy, navigation host, dashboard state/loader interfaces, widget loading and aggregation, Balance 2 coordinator, and relevant Zepp OS capability declarations. See the source map in the engineering plan.

The user reports that Gadgetbridge works with their Balance 2. No user's database, watch firmware version, Android version, device screen, or installed Gadgetbridge version was inspected. Source capability declarations do not prove that a specific watch/firmware actually supplies a metric. Screens and numerical examples in this plan are specifications, not screenshots of running software.

The checkout is newer than some older Gadgetbridge descriptions: it already includes a substantial configurable dashboard. Before coding, compare the user's installed version with this source snapshot; an upstream update or existing widget configuration may satisfy part of the request without new code.

## Decisions to confirm during milestone 0

Defaults are provided so work can proceed without a long design questionnaire:

| Decision | Proposed default | What could change it |
|---|---|---|
| Product direction | One hybrid, not two competing skins | User rejects the first-screen prototype |
| Theme | System light/dark; neutral surfaces, restrained accents | Explicit preference after reviewing both |
| Primary device | User-selected Balance 2 | Multi-device requirements; never select by model alone |
| Initial UI technology | Existing Views, ViewBinding, Kotlin, Material | A later bounded spike proves Compose has concrete value |
| Scores | Observations and existing verified watch scores only | Separate, reviewed methodology project |
| First test build | Separate app identity; no replacement of installed app | Only after backup/import/signing validation |
| Distribution | Personal alpha first | Public release after license, branding, safety and support gates |

## Deliverable and non-deliverable

This change supplies a fork and detailed plans only. It does not change the application, install an APK, access health data, enable integrations, or alter the watch. Android builds and hardware tests have not been run. Estimates are engineering effort ranges, not delivery promises.
