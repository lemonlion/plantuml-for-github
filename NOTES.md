# Option B evidence rig (2026-09-05)

Prototype + measurements backing drafts/issue-option-b.md. All scenarios run
against engine v1.2026.8beta1-0e4f452 (the stock fork tag Kronikol pins),
Chromium headless via playwright-core.

## The prototype is the real change, not a simulation

TeaVM JSBody blocks survive minification verbatim, so `rig/make-patched.js`
splices the exact proposed `loadOnce` replacement (PLANTUML_STDLIB_LOADER
hook + PLANTUML_STDLIB_BASE prefix) into the stock build, plus the
window -> globalThis swaps on the stdlib globals (8 sites total, same
pattern getTheme already uses). Engine grows 3,944,248 -> 3,944,577 bytes.
`rig/make-json.js` generates c4.json by executing the published c4.min.js
in an isolated vm and serializing the globals it populates - mechanically
identical to what a JsonBuilder sibling in plantuml-stdlib would emit
natively ({info, files, json}; 37 files, C4 v2.13.0).

## Scenario results

| # | scenario | result |
|---|----------|--------|
| 1 | stock engine, page on localhost, C4 include | FAILS: 404 on `/c4.min.js` at the page's own origin, "Fatal parsing error" on the include line (1-before-stock.png). This is every npm/CDN consumer today. |
| 2 | patched engine + `PLANTUML_STDLIB_BASE=https://plantuml.github.io/plantuml/js-plantuml/` | renders correctly; 200 from the project site; ~890-900 ms total cold, network fetch included (2-after-baseurl.png) |
| 3 | patched engine + `PLANTUML_STDLIB_LOADER` fetching local `c4.json` | renders correctly from pure data, no script execution; ~850-990 ms (3-after-jsonhook.png) |
| 4 | overhead control: stdlib-free sequence diagram, 5 reps | stock median 147-154 ms vs patched 147-148 ms: no measurable overhead when the knobs are unset |
| 5 | Web Worker: patched engine + hook, Kronikol's mock-DOM worker host | renders C4 in a worker in 909 ms (svg 3,319 bytes). Stock engine in the identical host: render never completes (20 s timeout) - the script-tag loader has no path in a worker |

## Wire cost of the JSON format

- c4.min.js from plantuml.github.io, gzipped on the wire: 24,755 bytes
- c4.json gzip -9: 23,360 bytes (raw 186,098)

The data format is marginally SMALLER than the minified JS. Also measured:
the project site serves `Cache-Control: max-age=600` (the unversioned-URL
caching point in the draft).

## Addendum (2026-09-05, same day): the real thing, not the prototype

Both upstream changes are now written and verified locally:

- plantuml clone c:\code\plantuml branch stdlib-loader: TeaVmScriptLoader
  hook + base + globalThis, tools/browser-test/check-stdlib-loader.js (15
  assertions) + README section. `gradlew :plantuml-mit:npmPackage -Pci`
  BUILT SUCCESSFULLY on this machine and the resulting engine passes all
  15 checks; the current engine fails the 7 new-behaviour ones (the check
  discriminates). The real build renders the issue-10 C4 sample via base
  URL (996 ms median of 5), JSON hook (826 ms) and inside a Web Worker
  (896 ms median of 3): 2-after-baseurl-realbuild.png,
  3-after-jsonhook-realbuild.png.
- plantuml-stdlib clone c:\code\plantuml-stdlib branch json-bundles:
  JsonBuilder + MainJs wiring + gradle copy step + JsonBuilderTest (8
  tests green; the repo's first tests, needed the Gradle 9 JUnit
  platform-launcher dep). Full runJs generated all 34 .json bundles;
  generated c4.json vs published-bundle data: same 37 keys, 34/37 files
  byte-identical, 3 diffs + info = the stdlib C4 2.13.0 -> 2.14.0 update.
- Benchmarks (30-rep in-page): warm render median 11 ms / p95 13 on
  stock, prototype-patched AND the real branch build. Chart:
  perf-chart.png (dataviz-skill palette, validated).
- PR bodies: drafts/pr-body-plantuml-loader.md,
  drafts/pr-body-stdlib-json.md (placeholders for issue/PR numbers).

## Addendum 2 (2026-09-05): the globalThis swaps proven, not assumed

The earlier worker demo aliased `self.window = self` (the Kronikol host's
normal setup), which made the loader's window-to-globalThis swaps
unfalsifiable. `rig/run-worker-strict.js` closes that: after the host sets
up, `window` is REPLACED with a measurement-only stub `{ document }` so no
PlantUML global is reachable through it, and the hook writes stdlib data to
globalThis only. Results:

- branch build (globalThis readers): C4 renders (svg 3,319 bytes);
- control, `engine-hookonly.js` (the loadOnce hook spliced in but getRaw /
  isLoaded still reading window): the hook runs and the data lands on
  globalThis, yet the include fails with "cannot include <C4/..." because
  the readers look at window.

Positive and negative cases both behave, so the swaps are a demonstrated
capability: a worker host does not need to mirror PlantUML data globals
onto its window alias. (The window stub itself is still needed by the
engine's canvas text measurement, which reads window.document; making
measurement worker-native via OffscreenCanvas would be a separate, larger
engine change and would still leave SVG output construction needing the
DOM mock.)

## Reproducing

1. Copy an engine build to `rig/engine-stock.js` (e.g. Kronikol
   tools/render-bench/core-1.2026.8beta1-0e4f452.js) and viz-global.js
   next to it; `node make-patched.js`; `node make-json.js` (reads the
   extension repo's Chrome/c4.min.js).
2. `node server.js` (port 8137).
3. `node run-evidence.js` (page scenarios + screenshots to out/),
   `node run-worker2.js` (worker + control). Both runners have two
   consts at the top (playwright-core path, Chromium path) to adjust.

`rig/plantuml-worker-host.js` is Kronikol's worker host (mock DOM +
OffscreenCanvas text measurement), reused unmodified: the host proves a
worker can supply everything else the engine needs - the stdlib loader is
the one piece only the engine can open up.
