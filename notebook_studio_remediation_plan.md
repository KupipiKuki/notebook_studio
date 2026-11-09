# Notebook Studio — Remediation and Improvement Implementation Plan

Date: November 8, 2026  
Status: **Planning only — no application code changed or browser tests executed by this review**  
Repository: [KupipiKuki/notebook_studio](https://github.com/KupipiKuki/notebook_studio)  
Reviewed source: [`scripts/notebook-studio.html`](https://github.com/KupipiKuki/notebook_studio/blob/main/scripts/notebook-studio.html)  
Baseline branch: `main`  
Baseline commit: `8321e8c00ff917cc61c97964bfd6365610f15137`  
Baseline source blob SHA: `d0dddc013c760e3047907d3c7948cede9be85a5a`  
Baseline source length: approximately 3,212 lines / 111,401 GitHub blob bytes.  
Related plan: [Markdown Studio remediation plan](https://github.com/KupipiKuki/markdown_studio/blob/vibe/code_revisions/markdown_studio_remediation_plan.md).

## 1. Scope and engineering principles

Remediate current Notebook Studio behavior and improve reliability, interoperability, security, accessibility and maintainability **without replacing its single-file, locally runnable application**. Preserve the supported Markdown, Python/Pyodide, JavaScript, HTML, CSS, JSON, YAML, and SQL/sql.js functionality, multiple tabs, local browser persistence, File System Access API fallback, variable inspector, package installer, plot rendering, cross-studio transfers, command palette, imports, exports and demo notebook.

**Historical work already in `main`:** PRs #1–#5 introduced Markdown sanitization, collapsed Markdown editing, persisted themes, SQL, Pyodide enhancements, variable inspection, inline plots, native file I/O, Studio Bus, command palette and examples. Do not implement these again; test them and fix defects in their current implementations. The older `stage1` branch is already an ancestor of `main` (seven commits behind, no commits ahead at review). The two other project names in the user's suite should not be inferred from the repeated Notebook Studio URL; interoperability should be validated against the actual connected peer repositories.

Engineering rules:
- Preserve an immutable copy and SHA-256 of the baseline before edits; record actual runtime and dependencies.
- Use a separate review/implementation branch; keep changes tightly scoped and do not overwrite saved user notebooks.
- Automatically correct localized minor defects found during testing, tracking them as D-series findings; escalate architectural changes, data format migrations, feature expansions, or incompatible security tradeoffs.
- Assign every finding a test, before/after evidence, change reference and disposition. Do not mark suspected conditions as reproduced until independently verified.
- No gate passes on syntax-only checks, simulated tests substituted for browser checks, or environments that were not tested. Avoid any claim of Windows validation without running on Windows.
- Never silently change or execute imported/transferred code. Keep destructive operations reversible or explicitly confirmed.

## 2. Source findings register

Evidence levels: **source-confirmed** = implementation path directly visible; **risk/hypothesis** = likely consequence requiring runtime proof. Links use the reviewed `main` commit, so line references remain stable.

| ID | Priority | Finding / current source evidence | Verification target |
| --- | --- | --- | --- |
| N01 | P0 | **JavaScript cells execute in app's page realm:** `executeJavaScript` uses `new Function(...)` with `window.nbVars` ([L2279–2317](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2279-L2317)). User/transfer/imported notebook code can access same-origin storage and application globals. Source-confirmed boundary, not a claim that malicious execution has occurred. | Controlled test of access to app storage and DOM; design explicit run/trust UX and sandbox/worker isolation tradeoffs. |
| N02 | P0 | **CSS code injected into the top-level UI:** `executeCss` builds a `<style>` from raw cell source inside `outputEl.innerHTML` ([L2365–2374](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2365-L2374)). Unlike HTML, CSS runs in the app document. | Verify that an intentionally hostile fixture cannot alter or obscure app controls after isolation. |
| N03 | P0 | **Saved output HTML is reapplied unsanitized:** `renderCells` assigns `cell.output` to `outputBox.innerHTML` ([L1893–1901](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1893-L1901)); `exportHtml` embeds stored output HTML ([L2537–2552](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2537-L2552)). The Markdown sanitizer is conditional on DOMPurify being defined ([L2415–2420](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2415-L2420)). Source-confirmed sinks and fail-open path; exploitability requires testing. | Untrusted file fixture, event-handler content, sanitizer unavailable, exported-file reopen. |
| N04 | P0 | **Notebook close may discard unsaved source edits:** `closeNotebook` removes the notebook before `saveState` snapshots only the active notebook ([L1821–1832](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1821-L1832), [L1695–1725](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1695-L1725)). Switching first snapshots current editors, but closing an inactive tab or closing without confirming disk synchronization warrants separate checks. | Type, close active/inactive, reopen; verify no unexpected data loss and clear close/save semantics. |
| N05 | P0 | **Active edits not guaranteed in exports:** CodeMirror `change` only updates a dirty indicator ([L1935–1937](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1935-L1937)); export HTML, Markdown, Jupyter and JSON read `cell.content` directly ([L2537–2595](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2537-L2595)). `saveState` first copies visible editors into model, but export handlers do not call it. | Edit without running/saving, export each format and compare exact text. |
| N06 | P1 | **Async execution can target stale DOM/state:** `runCell` awaits engines without generation or notebook identity checks, then updates captured instance and shared model ([L2092–2153](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2092-L2153)). | Switch tabs, delete cell, run twice, finish out of order. Only newest applicable result appears. |
| N07 | P1 | **Pyodide stdout routing uses one mutable global:** `activeExecutingCellOutput` ([L1588–1598](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1588-L1598)) is set prior to awaiting runtime load and cleared in a later `finally` ([L2156–2182](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2156-L2182)). Python execution and inspector both call the shared Python runtime asynchronously. | Overlapping Python/inspector operations, output routing, error recovery, global state isolation. |
| N08 | P1 | **File formats need validation and round trips:** Native and fallback import duplicate conversion logic ([L2484–2527](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2484-L2527), [L2896–2922](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2896-L2922)). `parsed.cells` is accepted without schema/type normalization. | Valid, malformed, large and legacy `.nb` / `.json` / `.ipynb` fixtures; block prototype surprises and unsupported language values. |
| N09 | P1 | **Jupyter interoperability incomplete:** Jupyter exporter writes `outputs: []` for every cell and labels notebook language Python ([L2556–2575](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2556-L2575)). Imported `.ipynb` code cells default to Python without inspecting language metadata. | Notebook-format schema validation, import/export round trip, retention of supported outputs and language metadata with explicitly documented limitations. |
| N10 | P1 | **Cross-studio transfer at-least-once/duplicate handling is unsafe:** messages are appended to a ten-item localStorage queue and broadcast ([L3007–3022](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L3007-L3022)); receipt removes queue items and inserts cells without durable processed-ID tracking ([L3030–3056](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L3030-L3056)). A success toast is issued before delivery acknowledgment. | Duplicate broadcast plus focus, two peer tabs, retry after crash, queue overflow, untrusted payload validation, receipt acknowledgment. |
| N11 | P1 | **Persistence failures are not propagated:** `saveState` catches storage exceptions, logs/toasts, but returns no failure to callers ([L1695–1730](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1695-L1730)); `loadState` falls back to starter notebook after parse errors ([L1733–1750](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1733-L1750)). | Quota exhausted, corrupt storage, no overwrite of recoverable original, accurate saved/unsaved status, backup/restore. |
| N12 | P1 | **Generated HTML export may embed active content:** `exportHtml` concatenates raw `cell.output` ([L2537–2552](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2537-L2552)). Source code is escaped; rendered output needs an independent policy. | Export/reopen malicious or interactive test output; safe offline readability and accurate user guidance. |
| N13 | P2 | **Markdown formatting needs shared coverage:** Markdown uses Marked and conditional DOMPurify but lacks the Markdown Studio diagram/math lifecycle and regression fixtures ([L2415–2440](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2415-L2440)). No Mermaid parity is implied to exist today. | Test Markdown/code fences, task lists, images and sanitization; assess Mermaid/MathJax only as separately approved feature work. |
| N14 | P2 | **Error-log handlers interpolate text into inline JS:** [L1514–1528](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1514-L1528). This is fragile for quotes, backslashes and special characters; the current globals `window.copyErrorText` / `window.dismissError` exist, so do not carry over Markdown Studio R04's missing-scope diagnosis blindly. | Delegated event handlers, exact clipboard content, no inline script/attribute injection. |
| N15 | P2 | **Dependency and launch portability:** CDN-hosted CodeMirror, Marked, DOMPurify, Pyodide, sql.js and fonts ([L8–33](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L8-L33), [L1577–1589](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1577-L1589)). Single-file HTML does not equal offline execution. | `file://` vs `http://localhost`, CDN-denied mode, explicit failures, pinned versions, non-destructive dependency update policy. |
| N16 | P2 | **No committed automated test harness is present in repository tree:** reviewed `main` contains `LICENSE`, `README.md` and `scripts/notebook-studio.html` only. PRs document syntax checks, but do not establish full browser regression. | Add Node/static checks and Playwright fixture tests plus CI, preserving single-file release distribution. |
| N17 | P2 | **Clipboard, picker, and save error UX is inconsistent:** per-cell Copy calls `navigator.clipboard.writeText` directly ([L1964–1979](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1964-L1979)); native file save often falls back to download ([L2932–2974](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2932-L2974)). | Denied clipboard/picker permissions, missing APIs, cancellation vs errors, accurate UI feedback. |
| N18 | P2 | **Rendering and lifecycle scale risks:** `renderCells` rebuilds all cell DOM and CodeMirror instances ([L1847–2015](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L1847-L2015)); serializing output HTML and plot images can strain localStorage. | Repeat 50 switches/rerenders; cell-count/load-time and memory tests; preserve input focus and scroll. |
| N19 | P2 | **SQL state scope/documentation:** one in-memory `sqlJsDb` shared by notebooks ([L2206–2244](https://github.com/KupipiKuki/notebook_studio/blob/8321e8c/scripts/notebook-studio.html#L2206-L2244)); on reload this DB is recreated. | Explicitly decide app-wide vs notebook-local DB, persistence/export semantics, and test isolation. |

New defects identified during execution use `D001`, `D002`, etc. Severity is a prioritization of **potential impact**, not a claim of a proven exploit. Every diagnosis must be confirmed or revised during Phase 0.

## 3. Phase 0 — Immutable baseline, coverage and controlled reproduction

- [ ] Verify `main` commit/blob; preserve source byte-for-byte, compute SHA-256, record dependency URLs and versions. Do not mistake Git blob SHA for SHA-256.
- [ ] Inventory every toolbar item, shortcut, modal, tab control, cell action, status indicator, run mode, import/export operation, and Studio Bus behavior.
- [ ] Establish baseline static syntax validation and Playwright browser harness. Include screenshots, console output and exact source/data assertions.
- [ ] Create synthetic fixtures: a multi-tab notebook with uncommitted edits, malformed/large/legacy `.nb`, Jupyter 4.x formats, Markdown with potentially unsafe HTML, hostile CSS/JS (local test only), Python output and plots, SQL side effects, and queued duplicate/oversized transfers.
- [ ] Test Chrome and Edge on Windows where available; test Firefox as secondary. Record OS, version, launch origin, test source hash, viewport, network conditions.
- [ ] Ensure no real user data, stored notebooks or connected applications are harmed by tests.

**Gate G0:** baseline hash and control inventory recorded; each N-series issue has evidence or an explicit unverified status; environment constraints and fixtures documented. Baseline remains unchanged.

## 4. Phase 1 — Data integrity and persistence safeguards (N04, N05, N08, N11)

- [ ] Centralize editor-to-model synchronization and invoke it before any tab close/rebuild, file write, export or transfer. Separate model capture, persisted save and native disk save outcomes.
- [ ] Add reliable dirty tracking, close confirmation or recoverable close behavior, and explicit status for browser-only vs disk-saved changes.
- [ ] Ensure save failures never clear the dirty indicator or silently replace the last good saved data; offer recovery/export.
- [ ] Normalize and validate imported `.nb` / JSON models; enforce bounded size/cell count, stable supported language and cell IDs, reject malformed data without altering current notebooks.
- [ ] Remove duplicated native/fallback import transformation logic; retain formats and normal legacy notebooks.
- [ ] Introduce schema versioning and backward-compatible migration only after sample fixtures; do not destructively rewrite saved documents.
- [ ] Test private/incognito, storage denied, quota exhausted, malformed JSON, open tab and native-picker cancellation.

**Gate G1:** fresh edits survive every save/export/switch path; close behavior is predictable; corrupted/quota-limited storage does not destroy recoverable notebooks; import errors never partially mutate active state. Capture byte-exact before/after test data.

## 5. Phase 2 — Trust boundaries and safe execution (N01, N02, N03, N12, N14)

- [ ] Document trust model: editor-authored scripts vs downloaded/imported/transferred notebooks, raw HTML/CSS preview vs sanitized Markdown, and export trust levels.
- [ ] Stop CSS cells from installing stylesheet rules in the host UI. Use a constrained preview container or isolated iframe with suitable sandbox and content security controls.
- [ ] Isolate JavaScript execution from the host document (sandboxed unique-origin iframe or worker depending on DOM needs) with a narrowly defined message API; preserve console/output capture and the intended persistent `nbVars` functionality or document a negotiated compatibility change.
- [ ] Harden `renderCells` output restoration. Prefer structured render records or explicitly sanitized output; never assume source marked safe once remains safe when reopened or exported.
- [ ] Make Markdown sanitizer **fail closed** if sanitizer not loaded, with safe text and actionable error.
- [ ] Replace error-log inline `onclick` string injection with delegated events and element `textContent`; preserve Copy/Dismiss.
- [ ] Test exported HTML separately from live preview. Choose static, non-executable output by default; warn if an interactive export is optional.
- [ ] Restrict/bound iframe resize message handling using `event.source`, expected type and numeric limits; examine message origin constraints under opaque-origin sandbox.
- [ ] Run crafted but non-destructive malicious fixtures and confirm UI/storage cannot be manipulated through import/render/preview by default.

**Gate G2:** hostile fixtures cannot execute in trusted host page through passive render/reopen; deliberate JS execution is clearly user-authorized and isolated; CSS stays inside preview; safe output and safe export retain expected appearance. No security guarantee is declared until runtime validation.

## 6. Phase 3 — Deterministic runtimes and cell lifecycle (N06, N07, N18, N19)

- [ ] Introduce per-cell run generations and stable notebook/cell identity checks; prevent old results from overwriting newer output after edits, switches, rerenders, or deletes.
- [ ] Serialize or coordinate Python Pyodide work and stdout/error capture; prevent competing inspector or package operations from corrupting stream attribution.
- [ ] Define run-all policy: sequential execution in selected notebook snapshot, deterministic failure continuation/stop, cancellation where runtime permits, visible completion status and no false completion on failed cell.
- [ ] Make variable inspector output not race notebook execution. Preserve run timing/cell counters when valid.
- [ ] Add lifecycle cleanup for CodeMirror, DOM listeners, iframes, plot elements; consider granular updates rather than wholesale rebuilds when feasible without rewrite.
- [ ] Document sql.js DB scope and cleanup; do not reset or persist existing session data without a scope decision.
- [ ] Add timeouts/limits where technically feasible, distinguishing cooperative cancellation from termination of an isolated runtime.

**Gate G3:** rapid run/re-run, concurrent Python, tab switches, cell deletion and 50 render cycles cannot misattribute outputs or grow retained detached listeners unboundedly. Pass exact execution traces and retention checks.

## 7. Phase 4 — Import/export and cross-studio compatibility (N05, N08, N09, N10, N12, N13, N17)

- [ ] Define `.nb` schema and version. Preserve source and meaningful metadata across JSON, Markdown, HTML, and Jupyter export.
- [ ] Validate generated Jupyter format with a real notebook schema validator; preserve supported text/PNG outputs and mark unsupported polyglot semantics explicitly instead of silently claiming Python equivalence.
- [ ] Ensure Markdown fences adapt to user content (including embedded triple backticks); verify exact-text round trips.
- [ ] Normalize safe output state before exports; prevent stale or running cells from yielding inconsistent snapshots. Reopen HTML exports in a test browser.
- [ ] Create versioned Studio Bus payload contract: `schemaVersion`, unique message ID, sender/receiver, language normalization, content size limits, acknowledgment, de-duplication and queue expiry. Plan same-origin behavior explicitly.
- [ ] Reconcile BroadcastChannel and localStorage queue handling across peers; don't display delivered status without acknowledgment. Support older messages during migration without duplicate replay.
- [ ] Compare protocols with the current `markdown_studio` and actual Notepad Studio code/repositories before deploying shared changes; avoid one-sided protocol drift.
- [ ] Add failure handling for clipboard/download/file pickers and clear errors for incompatible formats.

**Gate G4:** format fixture round trips preserve all documented fields, Jupyter validation passes with documented polyglot limitations, cross-studio messages are delivered at most once per receiver across focus/broadcast/restart cases, and fail/retry states are accurate.

## 8. Phase 5 — UI polish, accessibility, dependency resilience (N13, N15, N16–N19)

- [ ] Audit keyboard focus, screen-reader labels, tab and cell actions, status announcements, dialog focus trap/restoration, color contrast and narrow layouts.
- [ ] Test theme switching, Markdown collapse/edit, inspector, package installer, HTML preview sizing, long cell output and large plots.
- [ ] Ensure safe CSS/HTML renders do not contaminate surrounding notebook; maintain readable view after theme/width changes and export.
- [ ] Detect missing CDN components and fail gracefully. Record exact versions/risks and evaluate integrity/CSP versus `file://` portability; no blanket dependency upgrade.
- [ ] Add CI for JS parse, import/export fixture checks, browser-smoke functions and accessibility checks. The source distribution stays one HTML file; tests and evidence can live outside it.
- [ ] Decide whether Mermaid/MathJax rendering parity is an explicit feature milestone only after reviewing `markdown_studio` fixes; do not bundle new rendering features into defect remediation without separate approval.
- [ ] Complete inventory with every button, menu, keyboard path and modal tested, including denied API permissions and browser fallback.

**Gate G5:** full documented control matrix passes or each unavailable case is blocked with environment detail; accessibility and viewports have no severe regressions; no false offline compatibility claims.

## 9. Phase 6 — Frozen candidate, audit and handoff

- [ ] Re-run Gates G0–G5 on one frozen candidate; do not substitute earlier passing builds for final verification.
- [ ] Compare immutable baseline to final, review diff for lost features and unrelated reformatting, record final SHA-256 and actual browser matrix.
- [ ] Generate issue-by-issue audit, behavioral diff, raw unified patch, test evidence index and checksums.
- [ ] Validate patch applicability to baseline and byte-for-byte reconstruction of final source.
- [ ] Document safe rollback preserving prior saved notebooks and disk files. Do not clear localStorage as a routine rollback procedure.
- [ ] Distinguish `fixed`, `disproved`, `deferred (approved)`, `failed`, `blocked` and `not run`; never treat blocked as passed.

**Gate G6:** no unresolved P0 source risks, all required functional and security gates passed in tested environments, regression tests and audit trace each fix, and all deliverables resolve to their evidence. A blocked required check prevents an unqualified completion claim.

## 10. Shared suite alignment with Markdown Studio

This is a **cross-project checklist**, not an assertion that the same bug exists in both applications:

| Markdown Studio plan theme | Notebook Studio analogue | Integration rule |
| --- | --- | --- |
| R01–R04 diagram source and action fidelity | Cell source, output, action handlers and UI rehydration (N03–N05, N14) | Always retain canonical editable source independently of ephemeral rendered DOM. |
| R05/R15 parser mapping and stale rendering | Notebook identity, run generation and source snapshot (N06) | Stable IDs and latest-request guards before output writes. |
| R06–R08 visual/theming conflicts | CSS isolation, Markdown formatting, output sizing (N02, N13) | Keep document styles separate from application chrome. |
| R09 listener leaks | Cell rerender lifecycle and iframe handlers (N18) | Teardown and repeat-cycle tests. |
| R10–R12 editing command correctness | Cell insertion, selection, duplicate source transformation (N05, N08) | Exact output assertions for every command. |
| R13/R14 export readiness and failure paths | Native save, export, Jupyter, HTML and clipboard (N05, N09, N12, N17) | Export a synchronized, well-defined snapshot and report failures truthfully. |
| G0–G6 evidence-driven workflow | Gates G0–G6 in this plan | Reuse test ID convention, fixture inventory, browser matrix, audit and patch verification. |

Coordinate shared protocols and design language, but avoid importing Markdown Studio's Mermaid-specific bugs or fixes without observing a corresponding Notebook Studio issue.

## 11. Deliverable structure and evidence standards

Suggested structure (test harness files are allowed even though released app remains single HTML file):

```text
notebook_studio_remediation/
  baseline/notebook-studio.original.html
  final/notebook-studio.html
  tests/fixtures/
  tests/e2e/
  reports/notebook_studio_remediation_audit.md
  reports/notebook_studio_code_diff_report.md
  reports/notebook_studio.patch
  evidence/index.md
  evidence/phase-0/ ... phase-6/
  evidence/checksums.sha256
```

Each check should have: test ID, N/D finding IDs, source/commit SHA-256, browser/OS/origin/viewport, fixture, exact steps, expected/actual, pass/fail/blocked, timestamps, screenshots/logs and retest references. Include security tests only with synthetic content and a disposable browser profile.

The final report must enumerate every N-series and D-series item, root cause, severity, code changes, tests, evidence and disposition; list minor auto-fixes separately; provide control/browser coverage; distinguish visually verified behavior from hypotheses; include rollback, dependency changes, hashes, per-change diff stats and patch-reconstruction proof. Never fabricate screenshots, test statuses, source hashes or patch statistics.

## 12. First implementation checkpoint

Start with **Phase 0**, then fix **Phase 1 data-integrity** and immediate P0 passive-render/security vulnerabilities from Phase 2 as the earliest functional change batch. Show the updated source, source hash, G0/G1 outcome, security test status, before/after fixtures and initial audit entries. Keep JavaScript execution isolation as a separately reviewed compatibility decision if it changes the notebook's intentional JS behavior; do not simply remove JS execution.

## 13. Explicit review boundaries

This is a **source inspection and implementation plan**; the reviewer inspected the source on `main`, the GitHub branch ancestry, PR summaries and Markdown Studio's published plan. No application execution, dependency-security audit, browser reproduction, visual check, actual patch, or regression result was conducted in this planning pass. Do not interpret any numbered finding as proof of an exploit or completed remediation.

---

## Architecture Addendum A — Server-centered Unified Studio Suite (November 8, 2026)

**Decision status:** Preferred strategic direction selected by the project owner; implementation architecture remains **conditional on a measured architecture decision gate (AG0)**. This addendum is planning-only and does not authorize an immediate rewrite, monorepo conversion, or removal of standalone builds. It supplements rather than replaces all existing remediation findings, phase gates, source baselines, or in-progress Markdown Studio work.

**Target:** One local-first Studio Launcher hosting Markdown Studio, Notebook Studio, and Notepad Studio as modular applications, with shared document/workspace services and an authenticated loopback backend. The default technical candidate is **Node.js/TypeScript** for launcher, API, orchestration, frontend build tooling and integration; **Python as an isolated optional worker** for Python execution and related computation. The final technology selection must be benchmarked against Python-first, static modular, and existing single-file options at AG0.

### A1. Architecture decision phase — AG0 (before irreversible refactoring)

- [ ] Capture current behavior, baseline versions, startup times, file round-trip fidelity, memory, dependency load, offline behavior, feature coverage, code duplication and security boundaries for all three apps. Use the existing G0 evidence standard.
- [ ] Compare four concrete alternatives: (1) existing single-file apps with targeted fixes; (2) multi-file static apps with optional local server; (3) Python-first local service; (4) Node-first unified launcher with optional Python execution workers. Compare deployment/installation, Windows portability, offline startup, testability, API/file access, execution isolation, performance, upgrade/rollback, maintenance and packaging.
- [ ] Build a **disposable, minimal proof of concept** outside production application paths: launch three unmodified app views, transfer a synthetic document, reopen it, save to an approved workspace, and shut down cleanly. This PoC requires separate implementation approval; it is not part of this documentation-only update.
- [ ] Specify trust model: loopback-only binding, per-session random authentication secret or equivalent, origin/CSRF protections, strict CORS and WebSocket origin checks, no unrestricted filesystem endpoint, path traversal/symlink defenses, approved workspace roots, subprocess sandbox/resource limits, safe update mechanism, explicit consent before code execution.
- [ ] Specify compatibility contract: document IDs, content types, language metadata, encodings/line endings, versioned messages, at-most-once insertion with durable deduplication, acknowledgments, conflict handling, autosave/recovery, file permissions and import/export semantics.
- [ ] Decide distribution: developer Node runtime vs packaged local launcher, optional Python runtime, fully offline dependency bundling, standalone HTML fallback and whether a monorepo is warranted. Do not make a monorepo a prerequisite.
- [ ] Produce an architecture decision record (ADR) with a weighted scorecard, cost estimates, migration risk register, rejected alternatives, ownership and explicit go/no-go approval.

**Gate AG0:** Written ADR approved; measured baseline and PoC requirements recorded; no P0 data-integrity/security issue deferred solely for architectural migration. If the preferred candidate fails measurable constraints, revisit the decision rather than forcing migration.

### A2. Proposed target architecture and boundaries

```text
Studio Launcher (local browser / optional packaged desktop shell)
  ├── Markdown Studio UI (modular editor and preview)
  ├── Notebook Studio UI (cells, execution and outputs)
  └── Notepad Studio UI (text, search, diff and macros)
            │
  Shared client SDK / versioned API contract
            │ authenticated loopback HTTP + WebSocket
  Local Studio Service (Node.js/TypeScript candidate)
  ├── Workspace + document service (atomic writes, backups, recovery)
  ├── File access broker (explicit workspace scope and permissions)
  ├── Transfer/event broker (ack, dedupe, replay, delivery status)
  ├── Settings, themes, search and structured diagnostics
  └── Execution supervisor (process isolation and cancellation)
            ├── Python worker (optional, controlled)
            └── other sandboxed runtime workers (only if needed)
```

Keep app-specific editing and rendering in separate modules. Avoid sharing incompatible editor internals merely to reduce duplicate code. A server alone does **not** make arbitrary notebook code safe: runtime isolation and OS permissions require independent verification. Preserve browser-side editing responsiveness and avoid requiring network/cloud accounts.

### A3. Shared-suite roadmap and migration gates

| Gate | Work package | Required evidence / exit |
| --- | --- | --- |
| **AG0** | Architecture decision and alternatives scorecard | Approved ADR, baseline metrics, portability/security acceptance criteria, migration budget and rollback plan. |
| **MG1** | Contract-first foundation | Versioned schemas for documents, workspace, transfers, errors and capabilities; fixture-based compatibility tests across all three current apps. |
| **MG2** | Modular extraction | Extract CSS/JS/assets and introduce tests/build pipeline **without behavior changes**; source maps, pinned dependencies, functional parity, no loss of standalone release path until explicitly approved. |
| **MG3** | Local service and launcher | Loopback binding, authentication, origin/CSRF checks, safe filesystem broker, service startup/shutdown, health checks, offline packaging and no arbitrary path access; integrated smoke tests. |
| **MG4** | Incremental application integration | Migrate **one app at a time** behind feature flags/adapters, with legacy and new modes tested against the same fixtures; no all-at-once rewrite. |
| **MG5** | Shared workspace and Studio Bus replacement | Atomic file operations, crash recovery, versioned transfers, durable acknowledgments/dedupe, conflict behavior, old-message compatibility and multi-instance tests. |
| **MG6** | Execution, security and resilience | Notebook execution isolation, resource limits, process lifecycle, permission-denied cases, restart/recovery, large-file performance, threat-model test evidence. |
| **MG7** | Release, rollback and distribution | Windows-focused packaging/installer or documented local setup, clean uninstall, offline startup, signed/reproducible artifacts where applicable, upgrade/migration/rollback and preserved user data. |

**Migration order:** stabilize existing high-priority save/security paths; agree shared contracts; extract reusable modules; introduce launcher/service in parallel; migrate Notepad Studio as a lower-execution-risk pilot **only after AG0**, then Markdown Studio, then Notebook Studio execution components (or revise based on measured complexity). Existing Markdown Studio remediation can continue in its current branch; cherry-pick or adapt proven fixes rather than discarding them. No app is considered migrated until old/new exact-output regression fixtures pass.

### A4. Repository strategy and deliverables

**Default interim strategy:** keep all three repositories and their existing remediation branches; establish a fourth shared-suite/launcher repository only after AG0 approval. A monorepo remains an evaluated option, not a foregone conclusion. Publish shared contracts as a versioned package with compatibility matrix, semantic versioning, and CI testing all three applications. Do not create new repositories, modify app code or change default branches as part of this addendum.

Deliverables: `studio_suite_architecture_adr.md`, `studio_suite_migration_roadmap.md`, API/document schema specification, threat model, compatibility matrix, baseline and benchmark report, proof-of-concept test plan, rollback and data-migration plan, packaging decision, and gate evidence index.

**Cross-plan rule:** The existing Markdown G0–G6, Notebook G0–G6 and Notepad G0–G6 remain mandatory for the legacy application work. AG0/MG1–MG7 are additional **suite-wide** gates, not substitutes for defect closure. Any migration changing user data or execution trust boundaries requires explicit approval and a reversible path. Do not claim that an architectural gate has passed merely because this document defines it.
