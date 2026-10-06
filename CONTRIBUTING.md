# Contributing to QA AI Skills

Thanks for helping make AI-assisted QA more reliable. This guide covers how to propose a change,
how to **update an existing skill**, and how to **add a new one**.

If anything here is unclear, open an issue. A confusing contributing guide is a bug too.

---

## Contents

- [Ways to contribute](#ways-to-contribute)
- [Ground rules](#ground-rules)
- [The contribution workflow (fork → pull request)](#the-contribution-workflow-fork--pull-request)
- [Updating an existing skill](#updating-an-existing-skill)
- [Adding a new skill](#adding-a-new-skill)
- [How to write a good skill](#how-to-write-a-good-skill)
- [Testing your skill before you open a PR](#testing-your-skill-before-you-open-a-pr)
- [Sanitisation checklist](#sanitisation-checklist)
- [Pull request checklist](#pull-request-checklist)
- [Commit messages](#commit-messages)
- [How review works](#how-review-works)
- [Licensing of contributions](#licensing-of-contributions)

---

## Ways to contribute

| You want to… | Do this |
|---|---|
| Report a skill giving wrong or inconsistent results | Open an issue with the **bug** template. Include the test case (sanitised), what the agent did, and what it should have done. |
| Fix a typo, broken link or unclear sentence | Open a PR directly — no issue needed. |
| Improve an existing skill (new rule, new output format, new integration) | Open an issue first if it changes behaviour, then a PR. See [Updating an existing skill](#updating-an-existing-skill). |
| Add a new skill | Open a **skill proposal** issue first, then a PR. See [Adding a new skill](#adding-a-new-skill). |
| Build one of the 🗺️ Planned skills in the README | Comment on (or open) its issue so nobody duplicates the work, then follow [Adding a new skill](#adding-a-new-skill). |
| Share a real-world run | Add a sanitised sample report to `examples/` — real outputs are the most useful feedback we get. |

---

## Ground rules

1. **Tool-agnostic and company-agnostic.** No company names, internal URLs, hostnames, account IDs,
   ticket IDs from private trackers, or product-specific selectors in a skill. Anything
   project-specific belongs in the user's config file.
2. **No secrets, ever.** Not in skills, examples, screenshots, logs or commit history. Credentials
   are always referenced by environment-variable *name*.
3. **Humans make the consequential calls.** A skill may draft a bug, propose a fix or prepare a
   board update. It must not file, merge, delete or publish shared records on its own unless the
   user's config explicitly enables it.
4. **Follow the [design principles](README.md#design-principles)** in the README — especially
   *code decides, the model observes*.
5. **Be kind.** This project follows the [Code of Conduct](CODE_OF_CONDUCT.md).

---

## The contribution workflow (fork → pull request)

Only maintainers can push to this repository. Everyone else contributes from a fork.

### 1. Fork and clone

Click **Fork** on GitHub, then:

```bash
git clone https://github.com/<your-username>/<this-repo>.git
cd <this-repo>
git remote add upstream https://github.com/<owner>/<this-repo>.git
```

### 2. Create a branch

Never work on `main`, even in your fork. Name branches by type:

| Prefix | For |
|---|---|
| `feat/<skill>-<short-desc>` | New skill or new capability |
| `fix/<skill>-<short-desc>` | Wrong behaviour in a skill |
| `docs/<short-desc>` | README, CONTRIBUTING, examples |
| `chore/<short-desc>` | Repo housekeeping |

```bash
git switch -c feat/ai-assisted-manual-testing-notion-output
```

### 3. Make your change, then test it

See [Testing your skill](#testing-your-skill-before-you-open-a-pr). Untested changes to skill
behaviour won't be merged.

### 4. Keep your branch up to date

```bash
git fetch upstream
git rebase upstream/main
```

### 5. Push to your fork and open a pull request

```bash
git push -u origin feat/ai-assisted-manual-testing-notion-output
```

Open the PR against `<owner>/<this-repo>:main`. Fill in the PR template completely — the checklist
is how reviewers know the change was tested and sanitised.

### 6. Respond to review

Push follow-up commits to the same branch; the PR updates automatically. Maintainers squash-merge,
so don't worry about a tidy commit history inside the PR.

---

## Updating an existing skill

### What counts as what

| Change type | Examples | Needs an issue first? | Version bump |
|---|---|---|---|
| **Patch** | Typo, clearer wording, fixed snippet bug, broken link | No | `x.y.Z` |
| **Minor** | New output format, new test-case source, new optional config key, new rule that only *adds* safety | Recommended | `x.Y.0` |
| **Breaking** | Renamed/removed config key, changed report layout, changed verdict rules, changed file names other tools depend on | **Yes** | `X.0.0` |

### Steps

1. **Read the whole `SKILL.md`** before editing. Rules in one section often depend on another
   (e.g. a capture rule that the contract check enforces later).
2. **Make the smallest change that solves the problem.** Don't reformat sections you aren't
   changing — it makes review much harder.
3. **Keep the section structure.** Agents and users navigate by section numbers; if you add a
   section, append it or renumber *and* fix every cross-reference (`§5.4` etc.).
4. **When you add a rule, add the reason.** A rule with no "why" gets deleted by the next person
   who finds it inconvenient. One sentence of real-world failure is enough:
   > *Never screenshot on a timer — a fixed 2 s sleep photographed a loading spinner and the
   > reviewer couldn't tell "broken" from "too early".*
5. **Update everything the change touches:** the `description` in the frontmatter (if what the
   skill does changed), the config schema, reference snippets, the README catalog row, and the
   skill's `CHANGELOG.md`.
6. **Never loosen a safety rule silently.** Changes that make verdicts more lenient, remove
   redaction, or allow writes to shared systems need an issue and explicit maintainer sign-off.

### Skill changelog

Each skill folder has a `CHANGELOG.md`. Add an entry at the top under `Unreleased`:

```markdown
## Unreleased
### Added
- Notion output adapter (§7.3).
### Fixed
- Word renderer: image captions now render in italics.
```

Maintainers turn `Unreleased` into a version number when they merge.

---

## Adding a new skill

### 1. Propose it

Open a **skill proposal** issue that answers:

- **What QA problem does it solve?** Who runs it, and how often?
- **What goes in and what comes out?** (e.g. "a test-case ID in → a Word report out")
- **Which decisions must be identical every run?** Those need to be code or fixed rules.
- **What could go wrong if an agent runs it unattended?** (writes to shared systems, deletes data,
  leaks credentials). How does the skill prevent that?
- **Is it tool-agnostic?** What will the user configure instead of hard-coding?

A maintainer will confirm the name and scope before you invest in writing it.

### 2. Create the folder

```
skills/<skill-name>/
├── SKILL.md            # required — the skill
├── CHANGELOG.md        # required — starts with "## Unreleased / ### Added - Initial version"
├── references/         # optional — deep-dive docs the skill loads only when needed
└── scripts/            # optional — helpers the skill runs (renderers, checkers, parsers)
```

**Naming:** lowercase, hyphenated, verb-or-noun phrase describing the job, no company or vendor
names — `test-data-seeder`, `file-bug`, `investigate-failed-tests`.

### 3. Start from the template

```markdown
---
name: <skill-name>
description: <One paragraph. What it does, what it takes as input, what it produces, and the
  phrases a user might say that should trigger it ("seed an account", "why did these tests fail").
  The agent decides whether to load the skill from this text alone — make it specific.>
---

# <Human-readable name>

<Two or three sentences: what this skill does, and what a human still has to do.>

## 0. Core principle
<Which decisions are made by code / fixed rules, and what the agent supplies. A table works well.>

## 1. Configuration
<The config keys this skill reads, with a YAML example and defaults. Secrets by env-var name only.>

## 2. Inputs
<What the user gives; how everything else is resolved; precedence of overrides.>

## 3. Pipeline
<Numbered phases. Each one says what to run and what output proves it finished.>

## 4. Rules that decide close calls
### 4.1 Fixed defaults
| Situation | Default |
|---|---|
| <every situation where an unattended agent would otherwise have to ask> | <what it does instead> |
### 4.2 Hard rules
- <things the skill must never do>

## 5. Output
<Exact shape of what the skill hands back — report, file, comment, return contract.>

## 6. Reference index
| File | Load when |
|---|---|
```

### 4. Add it to the README

Add a row to the [Skill catalog](README.md#skill-catalog) (status ✅ Available) and, if it hands off
to or from other skills, update the flow diagram.

### 5. Add an example

Put a sanitised sample input and output in `examples/<skill-name>/`.

---

## How to write a good skill

Skills are read by an AI agent under pressure — mid-task, with a big context, sometimes
unattended. Write for that reader.

**Do**
- **Decide in code what must be repeatable.** Step counts, verdicts, layouts and file names come
  from scripts or fixed tables, not from "use your judgement".
- **Give a fixed default for every question an unattended agent could hit.** A sub-agent can't
  ask the user; if the skill doesn't say what to do, every run does something different.
- **Write commands, not essays.** "Run X. If it exits 3, report `INCONCLUSIVE — precondition`" beats
  a paragraph about preconditions.
- **Explain *why* for every non-obvious rule**, in one or two sentences.
- **Use tables** for mappings (situation → action, format → build path).
- **Keep `SKILL.md` focused.** Move long background, edge cases and per-tool detail into
  `references/` and say when to load each file. Aim for a `SKILL.md` an agent can follow without
  reading anything else on a normal run.
- **Show one complete, working example** of anything the agent has to write (a capture script, a
  manifest).

**Don't**
- Hard-code a product, URL, account, tracker or selector.
- Leave verdicts or formatting to the model's discretion.
- Use vague wait instructions ("wait a bit"). Give the condition and the budget.
- Write "never" without saying what to do instead.
- Duplicate a rule in two places — when one copy changes, the other rots. Cross-reference instead.
- Restate dependency versions; point to the package manifest.

---

## Testing your skill before you open a PR

Every behaviour change must be exercised at least once, by an agent, against a real application.

### Use a public demo app

Don't test against systems you don't own. These public practice sites are fine:

| App | Good for |
|---|---|
| `https://demo.playwright.dev/todomvc` | Basic web flows, add/complete/delete |
| `https://the-internet.herokuapp.com` | Edge cases: dynamic loading, alerts, iframes, auth |
| `https://www.saucedemo.com` | Login, cart and checkout flows |

### What to check

- [ ] **Run it twice on the same input.** Step count, verdict and report structure must be
      identical. If they differ, something is still left to the model.
- [ ] **Run a failing case on purpose** (e.g. assert text that isn't there). The failure is
      reported as `failed` with bug details — not passed, not crashed.
- [ ] **Run with a missing input** (no account, thin test case). The skill stops with the
      documented `INCONCLUSIVE` / `ABORTED` outcome instead of improvising.
- [ ] **Scripts run.** Any snippet or file under `scripts/` you added or changed has actually been
      executed. Say how in the PR.
- [ ] **Every output format you touched opens correctly** in its real application (Word, Excel,
      Google Docs/Sheets, a browser for HTML/PDF).

Attach a sanitised sample output, or link to one in `examples/`.

---

## Sanitisation checklist

Before every push, check your diff and any attached files:

- [ ] No passwords, API keys, tokens, cookies or session IDs — including in screenshots and logs.
- [ ] No company names, internal hostnames, private URLs, real customer data or personal data.
- [ ] No IDs from private trackers (use `PROJ-123`-style placeholders).
- [ ] Screenshots show only demo apps or redacted UIs.
- [ ] No absolute paths from your machine (`/Users/<you>/…`, `C:\Users\…`).

Run a quick scan before pushing:

```bash
git diff upstream/main --stat
git diff upstream/main | grep -n -i -E 'password|secret|token|api[_-]?key|/Users/|C:\\\\Users'
```

If you accidentally committed a secret, **rotate it first**, then tell a maintainer — deleting the
commit is not enough once it has been pushed.

---

## Pull request checklist

The PR template repeats this. A PR with unticked boxes and no explanation will be sent back.

- [ ] Linked issue (required for new skills and breaking changes)
- [ ] Change type: patch / minor / breaking
- [ ] Tested against a real application — how, and with which agent
- [ ] Ran twice; results were consistent
- [ ] `CHANGELOG.md` updated for every skill touched
- [ ] README catalog / diagram updated if a skill was added or its purpose changed
- [ ] Cross-references (`§N.N`) still point at the right sections
- [ ] Sanitisation checklist done

---

## Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(ai-assisted-manual-testing): add Notion output adapter
fix(ai-assisted-manual-testing): render image captions in italics
docs(readme): add test-data-seeder to catalog
chore: update issue templates
```

The scope is the skill folder name, or `readme` / `contributing` / `repo`.

---

## How review works

1. A maintainer triages new PRs, usually within a week.
2. Reviewers check, in this order: **safety** (no secrets, no unattended writes, no loosened
   rules), **consistency** (would two runs produce the same result?), **portability** (no
   project-specific assumptions), then **clarity**.
3. Behaviour changes need at least one maintainer approval and all review conversations resolved.
4. Maintainers squash-merge and update the skill's version in its changelog.

If your PR has had no response for two weeks, comment on it to bump it.

---

## Licensing of contributions

This project is licensed under the [MIT License](LICENSE). By opening a pull request, you agree
that your contribution is licensed under the same terms, and that you have the right to submit it —
in particular, that it doesn't contain code or content your employer or anyone else owns.
