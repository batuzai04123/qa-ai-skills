---
name: ai-assisted-manual-testing
description: Execute a manual test case with an AI agent driving the application (web, mobile, desktop, API or CLI), capture annotated before/after evidence for every step, and publish a reviewer-ready test-evidence report in any format that holds text and images — Google Docs, Microsoft Word (.docx), Excel (.xlsx), Google Sheets, PDF, HTML, Markdown, OpenDocument, or a wiki page. Give it a test case ID (or a file / pasted steps); environment, account, browser/device, output format and destination come from the project config. Verdicts and document layout are decided by code, not by the model, so the same test case produces the same shaped report every time. Multi-target runs fan out as one independent sub-agent per target. Every report still gets a human QA review pass.
---

# AI-Assisted Manual Testing

Runs a manual test case against a real application, captures evidence for every step, and
publishes a test-evidence report that a human reviewer can judge **without re-running the test**.

This skill speeds up manual QA. It does not replace it: a human reviews every report before
anyone acts on it.

It is tool-agnostic by design:

| Concern | Pluggable choices |
|---|---|
| **Where test cases come from** | Jira, Azure DevOps, TestRail, Zephyr, Xray, GitHub/GitLab issues, Linear, ClickUp, Notion, a spreadsheet row, a Markdown/Gherkin file, or steps pasted into chat |
| **What is under test** | Web app (Playwright/Selenium), mobile app (Appium, emulator/simulator), desktop app, REST/GraphQL API, CLI / SSH session |
| **Where evidence goes** | Google Docs, Google Sheets, Word `.docx`, Excel `.xlsx`, PDF, HTML, Markdown + images folder, OpenDocument (`.odt`/`.ods`), Confluence/Notion/wiki — any format that can hold text and images |
| **Where results are recorded** | Always a local results CSV; optionally a comment on the test case, a test-management run, or a dashboard |

---

## 0. Core principle: code decides, the model observes

Testers who run the same test case twice with an unconstrained AI agent get two different reports:
different step counts, different verdicts, a different layout. Every one of those differences comes
from a decision left to the model. So those decisions live in code (or in fixed rules below), and
the model has no discretion over them:

| Decision | Made by | The agent supplies |
|---|---|---|
| Which steps the run has, their numbers and wording | The **step contract** (§3), derived mechanically from the test case and frozen | the raw test-case text |
| Whether a step with a machine-checkable `assert` passed | The assert evaluator (§5.4) | nothing — call it |
| Report layout, tables, bug-details blocks, deviations, checklist | The **renderer** (§7) from the evidence model | values only |
| Overall verdict | The verdict function (§6.2) — may be made stricter, never looser | optional stricter override |
| Run inputs (env, account, browser, output…) | The **config resolver** (§2) | the test case ID |

**What the agent actually writes:** the capture script, each step's `status` / `actual`, and typed
`bugDetails` / `deviation` values. It never hand-writes report markup, info-table rows, step
headings or the verdict sentence. If you find yourself formatting the report by hand, stop — the
format belongs to the renderer.

---

## 1. Project configuration

Everything project-specific lives in one file at the repo root, `qa-testing.config.yaml` (JSON or
`.env` keys work too). Nothing in this skill hard-codes a product, URL, tracker or account.

```yaml
project: "My Product"

# ── Where test cases come from ────────────────────────────────────────────────
testCaseSource:
  type: jira            # jira | azure-devops | testrail | github | gitlab | linear | clickup | notion | sheet | file | inline
  # type-specific keys, e.g.
  baseUrl: https://example.atlassian.net
  stepsField: description   # which field holds the steps (description, custom field, test-steps table…)
  # for type: file  →  path: ./test-cases/{caseId}.feature

# ── What is under test ────────────────────────────────────────────────────────
environments:
  staging:
    kind: web                         # web | mobile | desktop | api | cli
    baseUrl: https://staging.example.com
  prod-readonly:
    kind: web
    baseUrl: https://www.example.com
    readOnly: true                    # the agent must refuse mutating steps here
  android-emulator:
    kind: mobile
    appiumUrl: http://127.0.0.1:4723
    capabilities: { platformName: Android, "appium:app": ./build/app-debug.apk }
  public-api:
    kind: api
    baseUrl: https://api.staging.example.com
defaultEnvironment: staging

defaults:
  browser: chromium                   # chromium | firefox | webkit | chrome | msedge
  platform: Desktop                   # Desktop | Mobile | Tablet | Android | iOS …
  language: en
  viewport: { width: 1920, height: 1080 }

# ── Test accounts (credentials are NEVER in this file — env var names only) ──
accounts:
  - id: basic-user
    usernameEnv: QA_BASIC_USER
    passwordEnv: QA_BASIC_PASS
    tags: [basic, read-only-safe]
  - id: admin-user
    usernameEnv: QA_ADMIN_USER
    passwordEnv: QA_ADMIN_PASS
    tags: [admin]

# ── Evidence output ───────────────────────────────────────────────────────────
output:
  formats: [docx]                     # one or more: gdoc | gsheet | docx | xlsx | pdf | html | md | odt | ods | confluence | notion
  destination:
    type: local                       # local | google-drive | sharepoint | confluence | s3 | …
    dir: ./test-evidence/{cycle}/{feature}
    # google-drive → folderId: "<drive folder id>"
  naming: "[{cycle}] {caseId}-{platform}-{browser}-{language}"
  redact:                              # always masked in every screenshot
    - 'input[type="password"]'
    - '[data-sensitive]'
  imageWidthInches: 6.5               # for page-based formats

# ── Result recording ─────────────────────────────────────────────────────────
results:
  csv: ./manual-test-results.csv       # always written
  commentOnTestCase: true              # post a short result + link on the test case
  testManagementWriteback: false       # e.g. create a TestRail/Xray run result
  cycle: ""                            # e.g. "Sprint 42" / "Release 3.1"; empty → no write-back to shared boards

# ── Orchestration ─────────────────────────────────────────────────────────────
parallel:
  maxInFlight: 4
stateDir: .qa-runs                     # per-run working files (git-ignored)
```

Secrets come from environment variables (or a secret manager) named in the config. A password is
never stored, printed, logged, screenshotted, or written into the report.

---

## 2. Inputs and the run slug

### 2.1 Resolve inputs before anything runs

From the test case ID (plus anything the user said), resolve **once** and save to
`<stateDir>/<slug>/inputs.json`:

- environment → URL / device / endpoint
- browser or device, platform, language
- account id (never the password itself)
- domain/test data the case needs
- known bugs linked to the test case
- prior evidence for this case (newest report)
- output formats and destination
- cycle label (sprint, release, build number)

Precedence: **explicit user instruction > test-case metadata > config defaults.** Every override
of a resolved value is recorded as a *deviation* in the report.

### 2.2 Targets and slugs

One **target** = one report = one (test case × platform × browser/device × language × account).
Two language variants of one case are two targets.

Slug: `<CASE-ID>-<Platform>-<Browser>-<Language>[_v<N>]` — e.g. `PROJ-142-Desktop-Chromium-en`.
Every local file a run writes starts with its slug or lives under `<stateDir>/<slug>/`. A slug that
already exists in the destination gets the next free `_v<N>`.

### 2.3 Missing test account

If no account is resolved, recover one before asking anyone, in this order:
1. The newest evidence report for this case (its info table names the account).
2. Comments on the test case.
3. The config's accounts filtered by the tags the test case needs.

Only an interactive (top-level) agent may ask the user. A sub-agent returns
`INCONCLUSIVE — no test account`.

---

## 3. The step contract — test case → numbered steps

The contract is the frozen list of steps the run must report on. It makes "how many steps does this
case have?" a mechanical answer instead of a judgement call.

### 3.1 Derivation rules (mechanical, in this order)

1. Fetch the test case **verbatim** and save the raw response to `<slug>/source.json` (or `.txt`).
   Don't summarise, trim or reformat it. Make sure the fetch isn't truncated (many APIs cut long
   descriptions unless you ask for the full field).
2. If the source has a structured steps table (TestRail, Xray, Azure Test Plans), one row = one step.
3. Else, if it contains Gherkin, one step per `Given / When / Then / And / But` line, in source order.
4. Else, one step per item of the first ordered/unordered list with at least two items.
5. Else → **thin test case.** A sub-agent stops with `INCONCLUSIVE — needs authored contract`.
   An interactive agent agrees the step list with the user and saves it as an *authored* contract.

Classify each step's verb: `ACTION` (click, type, submit, navigate), `VERIFY` (see, expect,
should, is displayed), `WAIT` (wait until, within N seconds), `SETUP` (given / precondition).

### 3.2 Contract file

```json
{
  "caseId": "PROJ-142",
  "title": "User can reset their password",
  "sourceKind": "gherkin",          // gherkin | table | list | authored
  "sourceHash": "sha256:…",          // hash of the normalised source text
  "steps": [
    { "stepN": 1, "verb": "SETUP",  "text": "Given I am on the login page" },
    { "stepN": 2, "verb": "ACTION", "text": "When I click 'Forgot password'" },
    { "stepN": 3, "verb": "VERIFY", "text": "Then I see the reset form",
      "assert": { "visible": "form#reset-password" } }
  ]
}
```

- Commit contracts under `test-contracts/<CASE-ID>.json`. Reuse a committed contract when the
  source hash is unchanged; always reuse an `authored` one. If the hash changed, say
  **"test case source changed"** in the summary — earlier reports were built against the old list.
- **One contract step = one report step.** Never merge, split, renumber or drop a step. A step that
  can't run is still reported, with `status: "blocked"` and an `actual` saying why.
- **Optional `assert`** turns a VERIFY verdict into code. Supported shapes:
  `{ "visible": "<selector>" }`, `{ "hidden": "<selector>" }`, `{ "text": "<selector>", "equals|contains": "…" }`,
  `{ "url": "<substring or /regex/>" }`, `{ "status": 200, "jsonPath": "$.state", "equals": "active" }`
  (API). Adding asserts to frequently-run cases is the single best way to end verdict drift.

---

## 4. Orchestrator — one sub-agent per target

> **Skip this section if your brief says `SINGLE-TARGET SUB-AGENT RUN`.** You are the worker for one
> target: run §5–§8 end-to-end and **do not spawn any agent**.

If the agent runtime supports sub-agents, **whoever receives the request is the orchestrator, not
the tester — even for a single target.** The orchestrator never drives the app, writes capture
scripts or builds reports. Each sub-agent is fully independent: it never reads a sibling's files or
trusts a sibling's selector, precondition or verdict. (No sub-agent support? Run targets one after
another inline, following the same sections.)

### 4.1 Checklist

1. **Resolve every target up front** (§2) in one pass. Settle every blocking question with the user
   **before launch** — a sub-agent cannot ask anything.
2. **Assign slugs** (§2.2).
3. **Apply the parallel-safety gates** (§4.2) and form a queue.
4. **Launch** up to `parallel.maxInFlight` sub-agents at once, each with the brief (§4.3). Record
   the launch time and each agent's id.
5. **Watch for liveness.** Each sub-agent writes a heartbeat file (`<stateDir>/<slug>/heartbeat.json`
   with `{phase, step, at}`) at every phase. Silent 10 min → send the rescue nudge (§4.4).
   Silent 20 min or over a 45-min budget → stop it, read whatever it already captured, report it
   `ABORTED — stalled at <phase>`, and free the slot.
6. **Backfill on every completion.** When a slot frees and an eligible target is queued, launch it
   immediately. Never drain a whole batch before refilling.
7. **Consolidate** (§4.5) once the queue is empty and nothing is in flight.
8. **Never relaunch silently.** Relaunch only if the user asks, or the cause was clearly
   environmental (service down, expired credential, a stall with nothing captured).

### 4.2 Parallel-safety gates

- **G1 — isolated drivers.** Every sub-agent launches its own browser/driver/session from a script.
  A shared interactive browser tool (one browser for the whole session) is never handed to a
  sub-agent, and never produces evidence.
- **G2 — account isolation.** Group targets by account. Groups run in parallel; members of a group
  run one after another **if any member mutates the account** (changes settings, language, cart,
  data, plan). Read-only targets on one account may overlap. When in doubt, treat it as mutating.
- **G3 — namespace.** Every local file starts with the slug or lives in the slug's state dir.
- **G4 — destination scope.** A run only ever modifies or deletes output files it created.
- **G5 — rolling window.** Keep `maxInFlight` running (default 4; more only when browsers run on a
  remote grid with capacity). A target blocked by G2 is skipped over, not waited on.
- **G6 — rate-limited outputs.** Cloud document APIs (Google Docs/Sheets, Microsoft Graph,
  Confluence) enforce per-user write quotas; parallel builds throttle each other and retries just
  collide again. **Build reports one at a time under a lock** (§7.6), even when captures run in
  parallel.

### 4.3 Sub-agent brief template

Fill every `<…>`. Keep the rules block verbatim.

```
SINGLE-TARGET SUB-AGENT RUN — manual testing, target <slug>

RUN RULES (they override instinct):
1. Run every script in the FOREGROUND with an explicit timeout (10 min for capture / report build).
   Never background a script and end your turn "waiting for a notification" — notifications go to
   the orchestrator, not to you. Flows longer than ~8 min are split into resumable phases.
2. Heartbeat: write <stateDir>/<slug>/heartbeat.json at every phase
   (resolve → preconditions → capture → check → build → verify → finalize → done) and per step
   during capture. 10 min of silence gets you nudged; 20 min gets you stopped.
3. Budget ~60–100 tool calls. Past ~130, stop iterating and report remaining steps `blocked`.
4. Retry a failed step ONCE with the longer wait (30 s, then one 60 s retry), then record it and
   move on. Never re-verify a passed step.
5. You cannot ask the user anything. Every "ask" has a fixed default (§9.1) — apply it.
6. Do NOT spawn agents of any kind.
7. Treat every factual hint in this brief as a CLAIM TO VERIFY. If what you observe contradicts it,
   trust the observation and record a deviation.

Execute this ONE target end-to-end per the ai-assisted-manual-testing skill (§5–§8).

Target parameters (use ONLY these, never invent others):
- Test case: <CASE-ID>   Title: <title>   Cycle: <cycle or none>
- Environment: <env name> -> <url / device / endpoint>   Kind: <web|mobile|desktop|api|cli>
- Platform: <platform>   Browser/device: <browser or device>   Language: <language>
- Account: <account id>  (credentials from env vars <USER_ENV>/<PASS_ENV> — never print them)
- Test data: <domains / records / fixtures or none>
- Known bugs to re-check: <ids or none>   Prior evidence: <name/link or none>
- Output: <formats> -> <destination>   Report name: <filename>
- Run slug: <slug>   Inputs: already saved at <stateDir>/<slug>/inputs.json (do not re-resolve)

Final message = the return contract (§4.4), nothing else.
```

### 4.4 Rescue nudge and return contract

**Rescue nudge** (on 10 min of silence):
> You have been silent 10+ minutes. If a script is running in the background, check its output now.
> If capture already wrote `results.json`, go straight to §6 (check → build). Otherwise re-run the
> capture in the FOREGROUND with a 10-minute timeout. Write your heartbeat immediately. Do not
> restart completed steps.

**Return contract** — the sub-agent's final message is data, not prose:
`slug`, `verdict` (`PASSED (n/n)` | `FAILED (p/n)` | `INCONCLUSIVE — <why>` | `ABORTED — <why>`,
where *n* is the **contract's** step count), `reportUrls` (one per format), `reportFilename`,
`formatVerified` (true/false), per-step one-liners, `contractSteps` + `stepsReported`, `bugRefs`,
`deviations`, `captureSeconds`, `buildSeconds`, `testCaseCommentPosted`, `resultsCsvRow`,
`cleanupDone`, and for non-default-language runs `languageReset`.

### 4.5 Consolidated report

| Target (slug) | Verdict | Steps | Format ✓ | Evidence | Bug | Capture time |
|---|---|---|---|---|---|---|

Then: total wall-clock; which targets ran in parallel and which were serialised (and why); one line
per failed / inconclusive / aborted target; the results-CSV path and how many rows this batch added;
and whether any shared board/test-management write-back happened or was skipped.

---

## 5. Single-target pipeline

Run in order. Each phase writes a heartbeat.

### 5.1 Preconditions (before any capture)

Check every precondition the test case declares (account state, feature flags, existing data,
required plan/role) **before** capturing — through an API or a quick read-only probe.

- An unmet precondition is reported up front as `INCONCLUSIVE — precondition: <which>`. **It is not
  a product failure**, and running a misleading test on a wrongly-prepared account is worse than not
  running it.
- If the project provides idempotent setup operations (seed data, fixtures, API calls), run them
  when the test case permits.

### 5.2 Find selectors / locators (before writing the capture)

Cheapest first; every selector is a claim until verified live:
1. **Handoff notes from earlier runs** on the same page/screen (§8.4).
2. The project's own page objects / locator repository / test IDs.
3. Project docs and known gotchas.
4. A throwaway **probe script** that enumerates the live UI (`document.querySelectorAll(...)`, the
   Appium page source, the API's OpenAPI spec).

> **Don't rely only on the accessibility tree.** Some apps hide large parts of their UI from
> assistive tech (e.g. an `aria-hidden="true"` wrapper), so ARIA-snapshot tools return nothing even
> though the elements are visible and clickable. If an a11y snapshot comes back near-empty, fall back
> to DOM enumeration — and consider filing that as an accessibility defect.

Prefer direct navigation (deep links, URLs, intents) over clicking through menus when the test case
doesn't require the menu path itself.

### 5.3 Capture

Write a throwaway capture script `<slug>_capture.(mjs|py)` (never committed) and run it in the
foreground with a 10-minute timeout, teeing output to `<slug>_capture.log`.

**Template (web, Playwright, JavaScript)** — adapt the driver for other kinds (§5.6):

```javascript
import { chromium, firefox, webkit } from 'playwright';
import { readFileSync, writeFileSync, mkdirSync } from 'fs';
import { shoot, waitSettled, watchForText, assertStep, beat } from './qa-kit.mjs';   // §5.5

const SLUG = '<slug>';
const DIR = `.qa-runs/${SLUG}`; mkdirSync(DIR, { recursive: true });
const inputs = JSON.parse(readFileSync(`${DIR}/inputs.json`, 'utf8'));
const contract = JSON.parse(readFileSync(`${DIR}/contract.json`, 'utf8'));
const step = (n) => contract.steps.find((s) => s.stepN === n);
const USER = process.env[inputs.account.usernameEnv];
const PASS = process.env[inputs.account.passwordEnv];
const results = []; const startedAt = new Date(); const t0 = Date.now();

const engine = { chromium, firefox, webkit }[inputs.engine];
const browser = await engine.launch({ headless: true });
const context = await browser.newContext({ viewport: inputs.viewport, locale: inputs.language });
const page = await context.newPage();
const consoleErrors = [];
page.on('console', (m) => m.type() === 'error' && consoleErrors.push(m.text()));

try {
  // Login is never photographed unless the test case is about login/auth.
  await page.goto(`${inputs.baseUrl}/login`, { waitUntil: 'domcontentloaded' });
  await page.fill('#email', USER); await page.fill('#password', PASS);
  await page.click('button[type="submit"]');
  await page.waitForURL(/dashboard/);

  // @step 1  — one marker per contract step
  beat(SLUG, 'capture', 1);
  results.push({ stepN: 1, text: step(1).text, verb: step(1).verb, status: 'passed',
    actual: 'Logged in as the test account', expected: step(1).text,
    noScreenshotReason: 'login-policy' });

  // @step 2  — ACTION: before + after
  beat(SLUG, 'capture', 2);
  await shoot(page, `${DIR}/step2_before.png`, { expect: 'a.forgot-password' });
  await page.click('a.forgot-password');
  await shoot(page, `${DIR}/step2_after.png`, { expect: 'form#reset-password', highlight: 'form#reset-password' });
  results.push({ stepN: 2, text: step(2).text, verb: step(2).verb, status: 'passed',
    actual: 'Reset form opened', expected: 'Reset form opens',
    images: [{ path: `${DIR}/step2_before.png`, caption: 'Before' },
             { path: `${DIR}/step2_after.png`,  caption: 'After'  }] });

  // @step 3  — VERIFY: evaluate the contract assert, reuse the screen that proves it
  const a3 = await assertStep(page, step(3));          // null when the step has no assert
  results.push({ stepN: 3, text: step(3).text, verb: step(3).verb,
    status: a3 && !a3.ok ? 'failed' : 'passed',
    actual: a3?.observed ?? 'Reset form visible', expected: step(3).text,
    ...(a3 ? { assertResult: a3 } : {}),
    images: [{ path: `${DIR}/step2_after.png`, caption: 'Reset form' }],
    // failed only:
    // bugDetails: { expected, actual, reproducibility: '3/3', severity: 'major', consoleErrors, bugRef, notes }
    // UI differs from the test case but works:
    // deviation: { testCaseSays: '…', reality: '…', impact: 'none|minor|review needed' }
  });
} catch (err) {
  await page.screenshot({ path: `${DIR}/error.png` }).catch(() => {});
  console.error(err); process.exitCode = 1;
} finally {
  writeFileSync(`${DIR}/results.json`, JSON.stringify({
    caseId: contract.caseId, slug: SLUG, env: inputs.baseUrl, engine: inputs.engine,
    startedAt: startedAt.toISOString(), captureSeconds: (Date.now() - t0) / 1000,
    consoleErrors, results }, null, 2));
  // Non-default language / mutated settings: restore them here and record it.
  await browser.close();
  console.log('CAPTURE COMPLETE');
}
```

### 5.4 Capture rules (checked downstream — breaking them fails the build)

- **Screenshot on a condition, never on a timer.** Every `shoot()` names an `expect` (selector) or
  `expectText`. `sleep(2000); screenshot()` photographs spinners, empty panes and the previous route.
  A report full of loading screens is worse than no report: the reviewer can't tell "broken" from
  "photographed too early". If the anchor never appears, `shoot()` **throws** — that becomes an honest
  `blocked` step, not a silently captured spinner.
- **Every executed step has evidence.** Every `passed` or `failed` step, VERIFY steps included:
  - ACTION steps get before + after shots; VERIFY/WAIT steps get an after shot of the screen that
    proves them, ideally highlighting the verified element.
  - Several VERIFY steps checked on one unchanged screen **share that one shot**.
  - The only exceptions carry a `noScreenshotReason`: `login-policy`, or `api-verified: <proof>` for
    a step verified purely through an API (put the response in `actual`).
- **Different states, different frames.** If two images that should show different states are
  byte-identical, you captured the same frame twice. Investigate before building.
- **Wait ladder: 30 s, then ONE 60 s retry, then fail.** Poll a condition; never use a fixed sleep as
  evidence. A too-short wait manufactures false results in *both* directions: a false FAIL (the
  element arrived at 7 s; you gave up at 6 s) and a false PASS (a "no results" assertion read while
  the request was still in flight). A non-reproduction on a short probe is not evidence of absence.
  Report the measured phase and time: "settled in 6.0 s on attempt 1" / "still unsettled after
  30 s + 60 s". Genuinely long operations (provisioning, exports) get their own larger bounded
  budget on top of this floor; no single wait exceeds ~8 min — split longer flows into resumable
  phases and never re-fire a mutating action without checking live state first.
- **Read the right node.** A watcher scoped to a container that doesn't contain the text will
  report "settled" forever. Anchor on the smallest element that contains both what you wait for and
  its section, and assert a positive settle string, not just "no spinner".
- **Catch transient messages with an observer, not a sampler.** Toasts live a few seconds; polling
  every 30 s misses them. Install `watchForText()` **before** the triggering action. A full page
  reload destroys the observer — if the action navigated, a `null` result is *not* evidence of
  absence; assert on the confirmation rendered in the new page instead.
- **Dismiss overlays** (cookie banners, notification drawers, chat widgets) before capturing, unless
  the overlay *is* the subject.
- **`isVisible()` can lie inside collapsed panes.** An inner element can report visible while its
  accordion/dialog ancestor is collapsed (`height: 0; opacity: 0`). Decide expansion from the gating
  ancestor's computed style, and hit-test (`document.elementFromPoint`) before shooting the pane.
- **If the script throws mid-run,** fix the selector with a probe and re-run only the steps not yet
  passed. If it still fails, those steps are `blocked`. Never switch to a different, shared,
  interactive browser tool mid-run to "just finish it".
- **Redact always.** Every screenshot masks the config's `output.redact` selectors. Credentials,
  tokens, card numbers and personal data never appear in an image, log, results file or report.

**Standing ruling — a missed transient confirmation does not block a step whose state is proven.**
If you missed the toast but proved the state change it announces, the step **passes** with
`evidenceSubstituted` naming the proof. All four conditions are required:
1. The *only* unobserved thing is a transient confirmation (toast, flash message, ephemeral banner).
2. You have **positive durable proof** of the same state change, recorded verbatim in `actual`
   (API response, persistent UI state, DB/DNS record, added/removed row). Two signals beat one.
3. `evidenceSubstituted` names that proof, so the report says so openly.
4. The durable state **agrees**. If it contradicts, the step `failed`; if you can't check it,
   `blocked` — never substituted.

It does **not** apply when the message itself is under test (copy/i18n tests, "a notification
appears"): there a missed message is `blocked`.

### 5.5 Reference helpers (`qa-kit.mjs`, web / Playwright)

```javascript
import { writeFileSync, mkdirSync } from 'fs';

const SPINNERS = ['.spinner', '.loading', '[aria-busy="true"]', '.skeleton'];   // extend per project
const REDACT = ['input[type="password"]'];                                     // + config output.redact

export function beat(slug, phase, step = null) {
  mkdirSync(`.qa-runs/${slug}`, { recursive: true });
  writeFileSync(`.qa-runs/${slug}/heartbeat.json`, JSON.stringify({ phase, step, at: new Date().toISOString() }));
}

export async function shoot(page, path, { expect, expectText, highlight, mask = [], timeout = 30_000, fullPage = false } = {}) {
  if (!expect && !expectText) throw new Error(`shoot(${path}): name an expect or expectText anchor`);
  if (expect) await page.locator(expect).first().waitFor({ state: 'visible', timeout });
  if (expectText) await page.getByText(expectText).first().waitFor({ state: 'visible', timeout });
  for (const s of SPINNERS) await page.locator(s).first().waitFor({ state: 'hidden', timeout }).catch(() => {});
  await page.evaluate(() => document.fonts.ready);
  await page.evaluate(() => new Promise((r) => requestAnimationFrame(() => requestAnimationFrame(r))));
  const hl = highlight ? page.locator(highlight).first() : null;
  const prev = hl ? await hl.evaluate((el) => { const o = el.style.outline; el.style.outline = '3px solid #e00'; return o; }) : null;
  await page.screenshot({ path, fullPage, mask: [...REDACT, ...mask].map((s) => page.locator(s)) });
  if (hl) await hl.evaluate((el, o) => { el.style.outline = o; }, prev);
}

// 30 s, then one 60 s retry, then fail. Poll a CONDITION.
export async function waitSettled(page, isSettled, label) {
  for (const [phase, budget] of [['attempt-1', 30_000], ['retry-60s', 60_000]]) {
    const t0 = Date.now();
    while (Date.now() - t0 < budget) {
      if (await isSettled()) return { settled: true, phase, ms: Date.now() - t0 };
      await page.waitForTimeout(1500);
    }
    console.log(`${label} [${phase}] not settled after ${budget} ms`);
  }
  return { settled: false };
}

// Install BEFORE the triggering action. Records the first needle that enters the DOM.
export async function watchForText(page, needles) {
  await page.evaluate((needles) => {
    window.__qaSeen = null; window.__qaStart = Date.now();
    const check = () => {
      if (window.__qaSeen) return;
      const t = document.body.innerText; const hit = needles.find((n) => t.includes(n));
      if (hit) window.__qaSeen = { text: hit, at: Date.now() };
    };
    new MutationObserver(check).observe(document.body, { childList: true, subtree: true, characterData: true });
  }, needles);
  return {
    wait: async ({ timeout = 60_000 } = {}) => {
      try {
        await page.waitForFunction(() => window.__qaSeen, null, { timeout: Math.min(timeout, 8 * 60_000) });
        return page.evaluate(() => ({ text: window.__qaSeen.text, elapsedMs: window.__qaSeen.at - window.__qaStart }));
      } catch { return null; }   // null + a navigation happened ⇒ NOT evidence of absence
    },
  };
}

// Evaluates a contract step's `assert`. Returns null when there is none.
export async function assertStep(page, contractStep) {
  const a = contractStep?.assert; if (!a) return null;
  const T = 30_000;
  try {
    if (a.visible) { await page.locator(a.visible).first().waitFor({ state: 'visible', timeout: T }); return { ok: true, observed: `visible: ${a.visible}` }; }
    if (a.hidden)  { await page.locator(a.hidden).first().waitFor({ state: 'hidden', timeout: T });  return { ok: true, observed: `hidden: ${a.hidden}` }; }
    if (a.url) {
      const re = a.url.startsWith('/') ? new RegExp(a.url.slice(1, a.url.lastIndexOf('/'))) : null;
      await page.waitForURL((u) => (re ? re.test(u.href) : u.href.includes(a.url)), { timeout: T });
      return { ok: true, observed: page.url() };
    }
    if (a.text) {
      const txt = (await page.locator(a.text).first().innerText({ timeout: T })).trim();
      const ok = a.equals != null ? txt === a.equals : txt.includes(a.contains);
      return { ok, observed: txt };
    }
  } catch (e) { return { ok: false, observed: String(e.message).split('\n')[0] }; }
  return { ok: false, observed: 'unsupported assert shape' };
}
```

### 5.6 Other kinds of environment

The pipeline, contract, rules and report are identical; only the driver and the evidence type change.

| Kind | Driver | Evidence per step | Notes |
|---|---|---|---|
| **Web** | Playwright / Selenium / WebDriverIO | Page or element screenshot, highlighted element; optional video/HAR/trace | Run each target in its own browser context |
| **Mobile** | Appium; `adb exec-out screencap -p`; `xcrun simctl io booted screenshot` | Device screenshot | Wait on element presence via the driver, not sleeps; reset app state between targets |
| **Desktop** | WinAppDriver / pywinauto / AT-SPI / AppleScript; OS screenshot tool | Window screenshot | Capture the app window, not the whole desktop (privacy) |
| **API** | `fetch` / `requests` / `curl` | Request + response (status, headers, body) as a formatted code block — or rendered to an image for image-only formats | Redact `Authorization`, cookies, tokens. Assert with `status` + `jsonPath` |
| **CLI / SSH** | `child_process`, `pexpect`, `ssh` | Terminal transcript (command + output) as a code block or rendered image | Never echo secrets; strip ANSI codes or render them |

To render text evidence (API response, terminal output) as an image for formats that need one,
load it into a headless page styled as a monospace terminal and screenshot that element.

### 5.7 Language / settings changes

A run that changes account settings (UI language, timezone, feature toggles) restores them in
`finally` and reports the reset. A run that ends with the account still changed is incomplete and
will corrupt the next run on that account.

---

## 6. Check → verdict

### 6.1 Contract check (before building any report)

Validate `results.json` against `contract.json`; fix every error before building:
- exactly one result per contract `stepN`, same numbering, same text
- every `passed`/`failed` result has at least one image, or a valid `noScreenshotReason`
- every `failed` result has `bugDetails`
- no `passed` result contradicts its own `assertResult`
- every referenced image file exists, and no two "different state" images are byte-identical

### 6.2 Step statuses and verdict

- **`passed`** — the expected state was observed (and the `assert`, if any, held).
- **`failed`** — the product did something other than what the test case expects. Needs `bugDetails`.
- **`blocked`** — an expectation nobody could observe; include why and the measured time.
- **`skipped`** / **`not-run`** — deliberately not executed; say why.
- **`na`** — the test case asserts behaviour the product has legitimately retired. Never makes a run
  non-PASSED by itself (flag the stale test case in the summary).

```javascript
function verdict(results, override) {
  const n = results.length, p = results.filter((r) => r.status === 'passed' || r.status === 'na').length;
  let v = results.some((r) => r.status === 'failed') ? 'FAILED'
        : results.some((r) => ['blocked', 'skipped', 'not-run'].includes(r.status)) ? 'INCONCLUSIVE'
        : 'PASSED';
  const rank = { PASSED: 0, INCONCLUSIVE: 1, FAILED: 2 };
  if (override && rank[override] > rank[v]) v = override;     // stricter only, never looser
  return `${v} (${p}/${n})`;
}
```

A linked bug that is closed as *won't fix* / *working as intended* and contradicts a step makes
that step pass (note it in the step).

### 6.3 Before publishing FAILED, rule out three impostors

1. **An unmet precondition** — wrong role/plan, missing data. That's INCONCLUSIVE, found in §5.1.
2. **A missed transient** — were you watching in a way that *could* have seen it (observer, not a
   slow sampler; no navigation tearing down the observer)?
3. **A state you sampled once** — a disabled control may be temporarily disabled. Record what you
   measured and *when* ("disabled at 14:02 and still at 14:20"), not "permanently broken".

None of this softens genuine findings: precise, time-stamped, reproducible observations make a bug
report stronger.

### 6.4 When the UI differs from the test case

| You hit | Do | Don't |
|---|---|---|
| Control moved or relabelled, but works | Pass the step, add a `deviation` | reword or renumber the step |
| A precondition fails so later steps can't run | Mark each dependent step `blocked` ("depends on step N") | drop them |
| Two test-case steps are now one click | Report both; the second's `actual` says "satisfied by step N" | merge them |
| One step now takes several clicks | One report step with extra screenshots | split it |
| The test case is stale or wrong | Run what you can, record deviations, flag it in the summary | test something else instead |

---

## 7. The evidence report

### 7.1 The evidence model (renderer input — values only)

The agent writes `<stateDir>/<slug>/evidence.json`. Every renderer reads only this file, so every
output format carries the same content in the same order.

```json
{
  "caseId": "PROJ-142",
  "title": "PROJ-142 — User can reset their password",
  "filename": "[Sprint 42] PROJ-142-Desktop-Chromium-en",
  "verdict": "PASSED (3/3)",
  "info": [
    ["Test case", "PROJ-142 — https://tracker.example.com/PROJ-142"],
    ["Cycle", "Sprint 42"],
    ["Environment", "staging — https://staging.example.com"],
    ["Platform / Browser", "Desktop / Chromium 131"],
    ["Language", "en"],
    ["Account", "basic-user"],
    ["Test data", "—"],
    ["Executed by", "AI agent (reviewed by: ______ )"],
    ["Started", "2026-10-06 09:14 UTC"]
  ],
  "preconditionNote": "Account has no pending reset request (checked via API).",
  "steps": [ /* results.json entries, unchanged */ ],
  "deviations": [ { "stepN": 2, "testCaseSays": "…", "reality": "…", "impact": "minor" } ],
  "summary": "One explanatory sentence — never restate the verdict.",
  "timing": { "captureSeconds": 41.2, "buildSeconds": 6.8 },
  "aiUsage": null
}
```

### 7.2 Fixed report layout (every format, same order)

1. **Title** — `<CASE-ID> — <test case title>`
2. **Verdict banner** — `🟢 PASSED (n/n)` / `🔴 FAILED (p/n)` / `🟡 INCONCLUSIVE (p/n) — <why>`
3. **Run information table** — the `info` rows
4. **Preconditions** — one sentence
5. **Steps** — for each contract step, in order:
   - heading `Step N — <step text>` (prefix `[FAILED]` / `[BLOCKED]` when applicable)
   - `Expected:` / `Actual:` / `Status:` lines
   - images with captions (before → after), or the `noScreenshotReason`
   - `Evidence substituted:` line when applicable
   - **Bug details** block on every failed step: Expected, Actual, Reproducibility, Severity,
     Console/log errors, Bug reference, Notes
6. **Deviations table** — Step | Test case says | Reality | Impact (omit when empty)
7. **Summary table** — Verdict, Passed, Failed, Blocked, Skipped/N-A, Summary sentence
8. **Timing** (and optional **AI usage**: model, requests, tokens, estimated cost — only if the agent
   runtime exposes it; never estimate from guesswork)
9. **Reviewer checklist** — ☐ steps match the test case ☐ every image shows what its step claims
   ☐ verdict agrees with the evidence ☐ bugs filed/linked ☐ reviewed by / date

### 7.3 Output adapters

Build each report **in one script, in one process**. Never assemble a document through dozens of
individual agent tool calls (one "insert paragraph" / "insert image" MCP call per element): every
tool call is a full model round-trip, so a 40-call doc build takes ~30 minutes where the same API
calls in one script take ~20 seconds — and hand-built docs never match the format.

| Format | Recommended build path | Images |
|---|---|---|
| **Word `.docx`** | `python-docx` (Python) or `docx` (npm), or Markdown → `pandoc` with a reference `.docx` | Inline, width = `imageWidthInches` |
| **Google Docs** | Build the `.docx`, then upload it to Drive **with conversion** to `application/vnd.google-apps.document` | Embedded images survive conversion — no public image URLs needed |
| **Excel `.xlsx`** | `openpyxl` (Python) or `exceljs` (npm): a *Summary* sheet + a *Steps* sheet (one row per step) | Anchored to the step row's Before/After cells; row height sized to the image |
| **Google Sheets** | Build the `.xlsx`, upload with conversion to `application/vnd.google-apps.spreadsheet`; **verify images survived**. Fallback: Apps Script `sheet.insertImage(blob, col, row)`, or `=IMAGE(url)` with images hosted at a URL the viewer can reach | — |
| **PDF** | Render the HTML report, then print it with a headless browser (`page.pdf()`), or `pandoc` / LibreOffice | Inline |
| **HTML** | Single self-contained file (images as base64 data URIs) or `report.html` + `images/` | Inline |
| **Markdown** | `report.md` + `images/` folder with relative links | Linked |
| **OpenDocument `.odt` / `.ods`** | Build `.docx` / `.xlsx`, then `soffice --headless --convert-to odt` (or `ods`) | Preserved by conversion |
| **Confluence / Notion / wiki** | Upload images as attachments via the API first, then create the page body referencing them | Attachments |
| **Anything else** | Render from `evidence.json`. If the format can't hold images (CSV, plain text), write a companion `images/` folder and reference files by name | — |

Converting from one master (`.docx` for documents, `.xlsx` for spreadsheets, HTML for PDF/web)
keeps every format identical and means one renderer to maintain per family.

### 7.4 Reference renderer — Word `.docx` (Python, `python-docx`)

```python
# render_docx.py  —  python3 render_docx.py .qa-runs/<slug>/evidence.json out.docx
import json, sys
from docx import Document
from docx.shared import Inches, RGBColor

ev = json.load(open(sys.argv[1])); out = sys.argv[2]
COLORS = {"PASSED": RGBColor(0x1E, 0x8E, 0x3E), "FAILED": RGBColor(0xD9, 0x30, 0x25), "INCONCLUSIVE": RGBColor(0xF2, 0x99, 0x00)}

def table(doc, rows, header=None):
    t = doc.add_table(rows=0, cols=len(rows[0] if rows else header)); t.style = "Table Grid"
    if header:
        cells = t.add_row().cells
        for c, h in zip(cells, header): c.text = h; c.paragraphs[0].runs[0].bold = True
    for r in rows:
        for c, v in zip(t.add_row().cells, r): c.text = str(v)
    return t

doc = Document()
doc.add_heading(ev["title"], 0)
v = doc.add_paragraph().add_run(ev["verdict"]); v.bold = True
v.font.color.rgb = COLORS[ev["verdict"].split()[0]]
doc.add_heading("Run information", 1); table(doc, ev["info"])
doc.add_heading("Preconditions", 1); doc.add_paragraph(ev.get("preconditionNote", "—"))

doc.add_heading("Steps", 1)
for s in ev["steps"]:
    tag = {"failed": "[FAILED] ", "blocked": "[BLOCKED] "}.get(s["status"], "")
    doc.add_heading(f'{tag}Step {s["stepN"]} — {s["text"]}', 2)
    for label, key in (("Expected", "expected"), ("Actual", "actual"), ("Status", "status")):
        p = doc.add_paragraph(); p.add_run(f"{label}: ").bold = True; p.add_run(str(s.get(key, "—")))
    if s.get("evidenceSubstituted"):
        p = doc.add_paragraph(); p.add_run("Evidence substituted: ").bold = True; p.add_run(s["evidenceSubstituted"])
    for img in s.get("images", []):
        doc.add_picture(img["path"], width=Inches(ev.get("imageWidthInches", 6.5)))
        doc.add_paragraph(img.get("caption", "")).italic = True
    if s.get("noScreenshotReason"):
        doc.add_paragraph(f'No screenshot: {s["noScreenshotReason"]}')
    if s["status"] == "failed":
        doc.add_heading("Bug details", 3)
        b = s["bugDetails"]
        table(doc, [[k, str(b.get(k, "—"))] for k in
                    ("expected", "actual", "reproducibility", "severity", "consoleErrors", "bugRef", "notes")])

if ev.get("deviations"):
    doc.add_heading("Deviations", 1)
    table(doc, [[d["stepN"], d["testCaseSays"], d["reality"], d["impact"]] for d in ev["deviations"]],
          header=["Step", "Test case says", "Reality", "Impact"])

count = lambda st: sum(1 for s in ev["steps"] if s["status"] in st)
doc.add_heading("Summary", 1)
table(doc, [["Verdict", ev["verdict"]], ["Passed", count({"passed"})], ["Failed", count({"failed"})],
            ["Blocked", count({"blocked"})], ["Skipped / N-A", count({"skipped", "not-run", "na"})],
            ["Summary", ev["summary"]],
            ["Capture time", f'{ev["timing"]["captureSeconds"]:.0f} s']], header=["Item", "Value"])

doc.add_heading("Reviewer checklist", 1)
for item in ("Steps match the test case", "Every image shows what its step claims",
             "Verdict agrees with the evidence", "Bugs filed / linked", "Reviewed by: ________  Date: ________"):
    doc.add_paragraph(f"☐ {item}")
doc.save(out)
```

### 7.5 Reference renderer — Excel `.xlsx` (Python, `openpyxl`)

```python
# render_xlsx.py  —  python3 render_xlsx.py .qa-runs/<slug>/evidence.json out.xlsx
import json, sys
from openpyxl import Workbook
from openpyxl.drawing.image import Image as XLImage
from openpyxl.styles import Alignment, Font, PatternFill

ev = json.load(open(sys.argv[1])); out = sys.argv[2]
FILL = {"passed": "C6EFCE", "failed": "FFC7CE", "blocked": "FFEB9C"}
IMG_W = 480  # px

wb = Workbook(); ws = wb.active; ws.title = "Summary"
ws.append([ev["title"]]); ws["A1"].font = Font(bold=True, size=14)
ws.append(["Verdict", ev["verdict"]])
for k, v in ev["info"]: ws.append([k, v])
ws.append(["Preconditions", ev.get("preconditionNote", "—")])
ws.append(["Summary", ev["summary"]])
ws.column_dimensions["A"].width, ws.column_dimensions["B"].width = 22, 90

st = wb.create_sheet("Steps")
st.append(["Step", "Action / Check", "Expected", "Actual", "Status", "Bug / Deviation", "Before", "After"])
for c in st[1]: c.font = Font(bold=True)
for col, w in zip("ABCDEFGH", (6, 40, 35, 45, 12, 35, 70, 70)): st.column_dimensions[col].width = w
dev = {d["stepN"]: d for d in ev.get("deviations", [])}

for row, s in enumerate(ev["steps"], start=2):
    note = s.get("bugDetails", {}).get("bugRef") or (dev.get(s["stepN"], {}).get("reality", "")) \
           or s.get("evidenceSubstituted") or s.get("noScreenshotReason", "")
    st.append([s["stepN"], s["text"], s.get("expected", ""), s.get("actual", ""), s["status"].upper(), note])
    st.cell(row, 5).fill = PatternFill("solid", fgColor=FILL.get(s["status"], "EDEDED"))
    for c in st[row]: c.alignment = Alignment(wrap_text=True, vertical="top")
    imgs = s.get("images", [])
    slots = {"G": imgs[0] if len(imgs) > 1 else None, "H": imgs[-1] if imgs else None}
    tallest = 0
    for col, img in slots.items():
        if not img: continue
        x = XLImage(img["path"]); scale = IMG_W / x.width
        x.width, x.height = IMG_W, int(x.height * scale); tallest = max(tallest, x.height)
        st.add_image(x, f"{col}{row}")
    if tallest: st.row_dimensions[row].height = tallest * 0.75 + 6   # px → points
wb.save(out)
```

### 7.6 Publish, verify, and the build lock

1. **Build under a lock** so parallel runs don't hit per-user API write quotas together:
   ```bash
   L=.qa-runs/build.lock; until mkdir "$L" 2>/dev/null; do sleep 20; done
   python3 render_docx.py .qa-runs/<slug>/evidence.json "<filename>.docx"; rc=$?; rmdir "$L"; exit $rc
   ```
2. **Upload** to the destination (local copy, Drive, SharePoint, Confluence…). For Google formats,
   upload with conversion:
   ```javascript
   // Node, googleapis — Google Doc from the .docx (use the spreadsheet mimeType for .xlsx → Sheet)
   const { data } = await drive.files.create({
     requestBody: { name: filename, mimeType: 'application/vnd.google-apps.document', parents: [folderId] },
     media: { mimeType: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
              body: fs.createReadStream(`${filename}.docx`) },
     fields: 'id, webViewLink',
   });
   ```
   If you instead insert images through the Docs API (`insertInlineImage` needs a publicly
   fetchable URL), upload them with a temporary public link and **delete them in a `finally`** —
   even on crash — so nothing stays public.
3. **Verify the published output** by reading it back, not by trusting the build:
   - one step heading/row per contract step, numbered as the contract numbers them
   - all required sections/tables present
   - a bug-details block on every failed step
   - expected number of images embedded; no leftover placeholders
   Only a build that passes verification counts. On failure, rename the output
   `[BROKEN BUILD — DO NOT USE] …`, delete it, fix the cause, rebuild. **There is no hand-built
   fallback.** If the build can't run (auth, quota), stop with `ABORTED — report build: <error>`.
4. A failed build never deletes the screenshots or results — they are what a rebuild needs.

---

## 8. Finalize

In this order:

1. **Known bugs.** For each failure, look for an existing open bug (the test case's links, the
   tracker search, prior evidence).
   - Exists and open → add a comment: cycle, "still reproducible", environment, account, one-line
     repro, report link. Put its id in `bugDetails.bugRef`.
   - Exists but closed as won't-fix / by design → the step passes (§6.2).
   - None → `bugRef: "New — awaiting reviewer decision"`.

   **Never create bug tickets automatically.** A human reviewer decides what gets filed.
2. **Comment on the test case** (when `results.commentOnTestCase` is on), exactly this shape:
   ```
   Result: 🟢 PASSED (n/n)  |  🔴 FAILED (p/n)  |  🟡 INCONCLUSIVE (p/n) — <why>

   **<report filename>**

   Test evidence: <link(s)>

   Bug: <BUG-ID — link | New finding — awaiting reviewer decision (no ticket filed)>   ← only if a bug
   ```
3. **Results CSV** — always append one row to `results.csv`:
   `timestamp, caseId, slug, cycle, environment, platform, browser, language, account, verdict,
   passed, total, reportLinks, bugRefs, comment, writebackDone`.
   Shared boards / test-management write-back happens only when configured **and** a `cycle` is set;
   otherwise record `writebackDone=false` and say so.
4. **Handoff notes** — write `<stateDir>/runs/<slug>/handoff.json` with the verified selectors per
   step, the route taken, waits measured, gotchas hit and the verdict. The next run of the same
   page reads this first (§5.2); it routinely cuts the cost of a repeat run several-fold.
5. **Cleanup** — delete this slug's throwaway files (capture script, probe scripts, logs, local
   screenshots once embedded, intermediate JSON). **Keep** committed contracts and handoff notes.
6. Final heartbeat `done`, then return the contract (§4.4) — or, inline, report the verdict, links,
   and per-step one-liners to the user.

---

## 9. Rules that decide close calls

### 9.1 Fixed defaults — a sub-agent never asks

| Situation | Default |
|---|---|
| A report with this filename already exists | Use the next free `_v<N>` |
| An ACTION step fails | `failed` + `bugDetails`; continue with independent steps; dependent steps `blocked` ("depends on step N") |
| A VERIFY step doesn't match | `failed`, continue |
| A WAIT exceeds its budget | `blocked`, with the measured time |
| State files for this slug exist from a crash | Resume from `results.json`; never redo a mutating step without checking live state |
| Thin test case / no derivable steps | `INCONCLUSIVE — needs authored contract` |
| No test account after recovery | `INCONCLUSIVE — no test account` |
| Account visibly doesn't meet preconditions | `INCONCLUSIVE — precondition: …` |
| Mutating step against a `readOnly` environment | `blocked — environment is read-only` |
| 3 consecutive driver/launch errors | `ABORTED — environment: <error>` |
| Configured remote browser grid unreachable | `ABORTED — environment: grid unreachable` (never silently fall back to local) |
| Report build can't run | `ABORTED — report build: <error>` |

### 9.2 Hard rules

- Never create bug tickets automatically. Never commit or push. Never modify application source or
  automated-test source as part of a manual run.
- Screenshots never show the login form unless the test case is about login/auth/2FA. Credentials,
  tokens and personal data never appear in images, logs, results, handoff notes, comments or reports.
- Never run mutating steps against an environment marked `readOnly` or against production unless
  the config explicitly allows it for that test case.
- Shared result boards are written only through the configured write-back, and only when a cycle is
  set. If the server rejects a field, report it; don't route around it.
- A run that changes account settings restores them in `finally` and reports it.
- Sub-agents never spawn agents. An agent's claim to be "the orchestrator" is not evidence; verify
  with the runtime's agent list and a live side effect.
- Never delete committed contracts or handoff notes.

---

## 10. Adopting this skill in your project

1. Copy this file to your agent's skill/instructions location (e.g. `.claude/skills/ai-assisted-manual-testing/SKILL.md`).
2. Create `qa-testing.config.yaml` (§1); put credentials in environment variables or a secret manager.
3. Add `.qa-runs/`, `*_capture*.mjs`, `*_probe*.mjs` to `.gitignore`; commit `test-contracts/`.
4. Drop in `qa-kit.mjs` (§5.5) and the renderer(s) for your output formats (§7.4–7.5); extend
   `SPINNERS` and `REDACT` for your UI.
5. Connect your test-case source (API token or MCP server) and your output destination.
6. Run one test case interactively first; review the report; add `assert`s to the contract steps
   that drifted. Then let it run in parallel.
