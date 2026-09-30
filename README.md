# PancTrials — integrated v0.6.6

Private native SwiftUI iPhone app for finding **Recruiting** and **Not Yet Recruiting** U.S. clinical-trial candidates relevant to pancreatic cancer, including broad solid-tumor/basket studies when appropriate.

> PancTrials is a candidate-trial finder, not an eligibility engine. Biomarker/keyword text evidence is informational. The app does not state that a patient qualifies; confirm eligibility and site availability with the study team.

## What is integrated in this build

This build starts from the green v0.6.2 baseline and selectively merges the latest Agent 1 data/search work and Agent 2 UI-responsiveness work.

- Existing app-facing repository contract is unchanged:

```swift
protocol TrialRepository: Sendable {
    func searchTrials(states: Set<String>, marker: MarkerFilter) async throws -> [Trial]
}
```

- ClinicalTrials.gov API v2 remains the primary full-text source.
- NCI Clinical Trials Search / CTRP is implemented as a second structured discovery source and is enabled when an NCI developer API key is configured.
- WHO ICTRP was re-audited but is **not enabled** because the official web service currently requires controlled/partner access and its published conditions conflict with the local caching architecture. No WHO HTML scraping is present. The domain retains `.whoICTRP` for a future official adapter if access/terms become suitable.
- Per-source provenance, canonical NCT identity, conservative deduplication, NCI-to-ClinicalTrials.gov enrichment, site-level status filtering, biomarker safety classifications, and regression fixtures are preserved.
- ClinicalTrials.gov and NCI discovery fan out concurrently with bounded concurrency; pagination remains complete within each cursor/offset stream.
- Successful query/discovery caches accelerate repeated searches and state-only changes. Partial multi-source results are never cached as complete.
- State/marker UI changes retain 350 ms debounce, cancellation, and stale-generation protection.
- `Search these results` is strictly local and synchronous over a precomputed lightweight index containing only title, NCT ID, intervention/treatment names/aliases, and facility/site/city/state/ZIP text. It never calls `TrialRepository` and never scans eligibility/biomarker text while typing.
- The local-results control is an explicit SwiftUI `TextField` with stable accessibility identifier `localResultsSearchField`; the XCUITest targets that identifier rather than `.searchable` prompt text. The same test verifies intervention and site-name filtering without weakening the existing result assertions.
- UI-test scrolling now waits for an identified row/control to be both present **and hittable** before returning. This fixes the SwiftUI lazy-`List` case where an off-screen `NavigationLink` already exists in the accessibility tree but is not yet on-screen/tappable; row identifiers remain `trialRow_<NCTID>`.
- Keyword wording is now concise: one keyword has no redundant Any/All rule; 2+ keywords show `Match any keyword` or `Match all keywords`.
- The responsive marker categories remain **All / Marker match / Unclear / Marker not mentioned**.
- Duplicate SwiftUI identity risk is removed from interventions and other potentially repeated detail-screen arrays by using stable source-order presentation identities rather than non-unique display strings.
- Saved/Favorites, persisted filters, New tracking, the supplied PancTrials icon/branding, patient-friendly trial details, direct ClinicalTrials.gov links, and accessibility identifiers are preserved.
- Release builds always use `LiveTrialRepository`; mock trials are selected only by the explicit `UITEST_MOCK=1` launch environment in DEBUG UI-test runs.

## Source configuration

### ClinicalTrials.gov

Enabled by default, no API key required.

`https://clinicaltrials.gov/api/v2/studies`

The source performs high-recall pancreatic and broad solid-tumor discovery, retains complete pagination, and provides the full text used for eligibility/biomarker evidence.

### NCI CTRP

Implemented and wired into `LiveTrialRepository`, but disabled unless an API key exists. Obtain a key from the official NCI Clinical Trials Search developer portal and set the app target build setting:

```text
NCI_CLINICAL_TRIALS_API_KEY = <your key>
```

The project maps this to the generated Info.plist key `NCIClinicalTrialsAPIKey`. Do not commit a real key. Without a key the app remains fully functional with ClinicalTrials.gov only.

The public NCI feed does not contain biomarker data. Consequently, NCI-only missing marker text is treated as unknown due to source coverage, not as `NOT_FOUND`. When possible, an NCI record with an NCT ID is enriched from ClinicalTrials.gov before marker classification.

### WHO ICTRP

Not enabled in this build. The official WHO XML web service is access-controlled and its current published conditions are not compatible with silently embedding a cached native-app client. No HTML scraping was added. WHO bridge/NCT deduplication is regression-tested so an official adapter can be added later without duplicating trials.

## Performance profile

The execution environment used to integrate this project has no outbound DNS and no Xcode/iOS Simulator. Therefore the network measurements below use the app's real Swift search code with deterministic 50 ms HTTP transports, not internet latency. This isolates code-path performance and cache/concurrency behavior. Five repeated runs were taken after the final merge.

| Path | Final measured time |
|---|---:|
| Cold multi-source discovery code path | **203.60 ms mean** (203.32–203.85 ms) |
| Repeat identical state+marker search, warm merged cache | **0.089 ms mean** |
| State-only filter change with same marker, warm discovery cache | **0.0123 ms mean** |
| Marker change after warm base discovery (`KRAS G12D`) | **52.53 ms mean** |
| Build local `Search these results` index for 5,000 synthetic trials | **134.38 ms mean** |
| One local typing query across 5,000 trials | **9.88 ms mean** (9.17–11.03 ms) |

The cold profile generated 13 ClinicalTrials.gov query requests and 16 NCI query requests. The repeated and state-only searches generated **zero additional source requests**. The warm marker change issued only 3 additional ClinicalTrials.gov marker-alias requests; NCI stayed at 16 because its public feed lacks biomarker data.

The local typing profile is intentionally a 5,000-row stress test, much larger than typical visible pancreatic-trial result sets. Query normalization is performed once per keystroke, not once per row. `Search these results` remains entirely local with no debounce/network request.

Agent 1's before/after deterministic audit measured the earlier cold path at about 0.806 s versus about 0.204 s after bounded query concurrency, and 1,000-candidate local post-processing at 23.77 s versus 1.18 s after indexed deduplication and normalization optimizations.

True carrier/Wi-Fi live timings should still be measured in Xcode/Simulator/device because WAN latency, NCI availability, pagination depth, and result volume vary.

## Filters and result behavior

Persisted filters/settings:

- U.S. states
- Any/specific marker
- sex
- arbitrary keyword phrases
- keyword Any/All mode
- selected marker category
- Show only new

Specific-marker categories:

- **All**
- **Marker match** — only `REQUIRED` or `ALLOWED_COMPATIBLE`
- **Unclear** — `MENTIONED_UNCLEAR`
- **Marker not mentioned** — `NOT_FOUND` only after sufficient/full registry text was searched

`EXCLUDED_CONTRADICTORY` remains excluded from positive candidate results. “Marker not mentioned” never means that a trial accepts that marker.

## New-trial tracking

The previous successful refresh's NCT-ID set is stored locally. After a later successful refresh, newly appearing IDs receive the New treatment. Failed, cancelled, or incomplete refreshes do not replace the prior successful baseline.

## Saved/Favorites

The existing favorite store is preserved. Saved IDs persist independently of current filters and New tracking.

## Xcode setup / install on a personal iPhone

1. Unzip the project and open `PancreaticTrials.xcodeproj` in a current Xcode that supports iOS 17+.
2. Select the **PancreaticTrials** target → **Signing & Capabilities**.
3. Keep **Automatically manage signing** enabled.
4. Change `com.example.PancreaticTrials` to a unique bundle identifier.
5. Choose your Apple account / Team.
6. Connect and trust the iPhone; select it as the run destination.
7. If requested, enable **Settings → Privacy & Security → Developer Mode** on the iPhone and restart it.
8. Press `Command-R` to build/install/run.
9. For NCI coverage, configure `NCI_CLINICAL_TRIALS_API_KEY` locally in Xcode/xcconfig. ClinicalTrials.gov needs no key.

A free Personal Team is sufficient for direct Xcode installation but development provisioning is short-lived and normally requires periodic rebuilding/re-signing. A paid Apple Developer Program membership is the straightforward option if you do not want the Personal Team reinstall cadence.

The production app does not require access to the Mac Documents folder. Asset tests running under iOS validate compiled app resources rather than reading the source tree.

## Automated validation

Run the portable core suite from Terminal:

```bash
swift test
```

Final integrated Linux/Swift 6.2.1 result: **68/68 tests passed**.

Coverage includes:

- ClinicalTrials.gov API v2 decoding/pagination/status/state/site filtering
- NCI mapping, configuration, enrichment, provenance, and failure isolation
- multi-source conservative deduplication
- biomarker alias/context safety and regression fixtures
- WHO-style NCT/bridge dedup regression without enabling a WHO network client
- sex/keyword/marker-category behavior
- persisted settings and New tracking
- successful cache reuse and non-caching of partial source failures
- bounded CT.gov/NCI query concurrency and cancellation
- large-candidate performance regression guard
- strictly local result-search index and field scope
- duplicate intervention presentation identities
- keyword wording
- icon/AppLogo asset wiring

The Xcode target also contains UI/hosted tests that require macOS/Xcode/iOS Simulator and therefore cannot execute in this Linux integration environment. Run `Command-U` before treating the simulator/device suite as confirmed.

## Runtime-console classification

No production `print`, `debugPrint`, `NSLog`, `Logger`, `os_log`, `fatalError`, or custom warning emissions are present in the app source, and the portable suite completes without app-generated runtime warnings.

Simulator messages such as CoreHaptics library-file notices, UIKit keyboard/prediction constraint warnings, or simulator service messages are Apple framework/Simulator noise unless accompanied by a PancTrials stack trace or reproducible UI failure. An earlier app-originated `No symbol named 'dna' found` warning is not present in the current source; the current marker row uses a supported `atom` symbol.

Because this environment cannot launch iOS Simulator, the final simulator console must still be checked once in Xcode. Any new message that names a PancTrials source file, invalid SF Symbol, SwiftUI duplicate ID, decoding failure, or concurrency violation should be treated as an app warning and fixed rather than ignored.

## Important files

- `SOURCE_AND_PERFORMANCE_AUDIT.md` — Agent 1 source/performance audit
- `AGENT3_INTEGRATION_NOTE_A1.md` — data-layer integration handoff
- `AGENT3_UI_RESPONSIVENESS_NOTE.md` — UI optimization handoff
- `QA_CHECKLIST.md` — final verification matrix
