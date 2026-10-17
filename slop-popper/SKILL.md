---
name: slop-popper
description: >-
  Use when the user says deslop, anti-slop, or AI slop, or wants a
  behavior-preserving cleanup of noisy, repetitive, over-abstracted,
  over-defensive, over-commented, or weakly tested AI-generated code. `/deslop`
  runs a quick pre-commit cleanup; `/deslop-full`, `/deslop-since`,
  `/deslop-review`, `/deslop-approve`, `/deslop-reject`, and `--dry-run` cover the rest.
---
# Slop Popper

Remove AI-generated slop from working code without changing behavior. "Slop" is code that runs but carries noise a careful human would not write: duplication, dead code, needless layers, defensive clutter, vague names, noisy comments, and weak tests. This is cleanup, not a rewrite.

## Commands

| Command | Scope | What it does | Report |
|---|---|---|---|
| `/deslop` (or plain `deslop`) | Staged changes; if none, uncommitted changes | Quick pre-commit cleanup, no questions | Short |
| `/deslop-full [path]` | The path, module, or file list given; whole repo if none | Complete Workflow | Full |
| `/deslop-since <commit or tag>` | Files changed since that point, plus uncommitted changes | Complete Workflow, release-sized | Full |
| `/deslop-review` | A cleanup someone else made | Review only, no edits | Verdict |
| `/deslop-approve <IDs \| all>` | Items from the last Needs further review list | Applies only approved items | Short |
| `/deslop-reject <IDs>` | Items from the last Needs further review list | Closes items without changing the code | Short |
| `--dry-run` (on any command) | Same as the command | Plan only, no edits, nothing written | Plan in chat |

`--review` is an alias for `/deslop-review`. A natural-language request with a target ("deslop this module") runs as `/deslop-full` on that target. Details for each command are under Modes below.

**Use for:** explicit deslop, anti-slop, or AI slop requests; cleanup of noisy, repetitive, or over-abstracted code; leftovers from earlier AI work; cleanup before a commit or release.

**Do not use for:** new features, product changes, broad redesigns, formatting-only requests (use the formatter), or code whose behavior is too unclear to protect with tests or a written verification plan.

## Ground Rules

1. **Cleanup, not rewrite.** Behavior stays the same. Leave existing logic, control flow, signatures, file layout, and formatting as written unless a catalog finding needs a minimal change. When in doubt, hold it for review instead of editing.
   - **Applied automatically:** deleting dead or duplicate code; fixing vague names (updating every caller in the same pass); removing wrappers that only pass calls through; moving code to the module it belongs in (updating imports); removing noise comments and debug output; marking deliberate catch-alls; fixing a clearly wrong comment or docstring; adding tests (except in `/deslop`, which lists them for review instead).
   - **Held for review (needs the user's OK):** changing signatures, parameters, or return types; changing logic or behavior; reformatting whole files; swapping libraries or patterns; changing config values, IDs, or constants; renaming anything used outside the codebase (a public API, CLI flag, file format, or saved setting); deleting test-only code.
2. **Lock behavior first** with the narrowest tests that cover what you will touch. If tests cannot come first, write the verification plan before editing.
3. **Follow project conventions.** Always run the style lookup first (Workflow step 1). What it finds overrides this skill's defaults for the lines being changed. It never justifies restyling untouched code.
4. **Delete before you add.** Reuse existing helpers and patterns; add no dependencies unless asked.
5. **Small, reversible diffs.** One catalog category per pass; never mix unrelated changes.
6. **Respect the scope.** Stay inside the files the command or user gave you.
7. **Size limit.** If a run would touch more than about 10 files or 300 changed lines in total, stop after the plan, show it, and ask before editing. The cap is for the whole run's plan, not each pass. `/deslop-full` on a whole repository often hits it; it then stops for approval and, once approved, runs the plan in batches that each stay under the cap.
8. **Never commit or push to the user's repo or branch.** The user does that. Private snapshot commits in a scratch copy or a temporary git repo (see No Git, No Tests) are fine for locking and diffing each pass.

## Slop Catalog

Classify every finding into one of these categories before editing. Categories 1 through 12 are also the pass order and pass numbers; category 13 only adds review items. Reports use these numbers. Each entry gives the definition, what to look for, and the fix. Rule 1 limits every fix: anything on its held-for-review list goes under Needs further review instead of being applied.

### 1. Dead code
Code that never runs or is never used.
- Look for: unused functions, classes, exports, variables, parameters; unreachable branches; stale feature flags; commented-out blocks; debug leftovers.
- Fix: delete it. Version control keeps history.
- **Test-only code is not dead.** If production code never calls something but tests do, or the docs, a style guide, or an API list mention it, don't delete it. Flag it for review:
  - Trace who calls it from the app's entry points (UI handlers, CLI, jobs), not just whether anything references it. Name the code that replaced it, if there is one.
  - Find the helpers that only exist to support it, and flag them together as one group.
  - Add one review comment above each flagged definition, written as a normal comment in that language:
    `REVIEW(deslop): only tests call this; replaced by <new_function>. Helpers: <a>, <b>. Remove after owner OK.`
  - Leave the tests, any public-API listing, and the behavior unchanged. Don't add runtime deprecation warnings unless asked.
  - List the group in the report under "Needs further review", with the callers, what replaced it, the tests and docs that reference it, and roughly how many lines removing it would take out.
  - If removal is approved later, take out the code, its helpers, its doc references, and the tests that only test it, in one pass, and remove the review comments with it. Tests that use it to check other behavior (for example as setup) are repointed to its replacement, not deleted, so coverage stays.

### 2. Duplication
The same logic written more than once.
- Look for: copy-paste branches, near-identical functions, repeated literal values, helpers that redo what the standard library or an existing util already does.
- Fix: consolidate into one place, or replace with the existing utility. Do not merge things that only look alike but change for different reasons.

### 3. Needless abstraction
Structure that adds indirection without adding value.
- Look for: pass-through wrappers, single-use helper layers, interfaces or base classes with one implementation, factories that build one thing, config for values that never change; optional parameters, flags, and `**kwargs` / `...rest` options that no caller ever passes.
- Fix: inline it, and drop options nobody uses (updating callers). Keep an abstraction only when it has more than one real user or isolates a real boundary. Removing a parameter from public or external-facing code is held for review.

### 4. Boundary violations
Code in the wrong place or reaching across layers.
- Look for: UI code doing file or network I/O directly, business logic inside handlers, hidden global state, wrong-direction imports, side effects in functions named like pure getters; hard-coded secrets, tokens, PINs, or credentials in source.
- Fix: move logic to the layer that owns it; keep validation and I/O at system edges. Never move, rotate, or delete a hard-coded secret automatically: flag it under Needs further review, and never copy its value into the report. A secret the keep list names (by name, never by value) is reported as a K item instead.

### 5. Defensive slop
Guards against conditions that cannot happen.
- Look for: `None` checks on values that are never `None`; type checks the type system already guarantees; re-validating data already validated at the boundary; length or empty checks before loops that handle empty input fine; fallback defaults for required values; double-checking a return value the callee guarantees.
- Fix: delete the guard. Keep checks at real boundaries (user input, files, network, external APIs) and where a failure would be silent and costly.

### 6. Error-handling slop
Error handling that hides, duplicates, or fakes recovery.
- Look for:
  - Swallowed errors: `except: pass`, `catch {}`, or a broad catch that returns a default with no log or reason.
  - Blanket try/except wrapping every call instead of the few that can actually fail.
  - Log-and-rethrow at every layer, so one failure is logged many times.
  - Catching an error only to raise a new one that drops the original cause.
  - Generic messages like "An error occurred" or "Something went wrong" with no detail.
  - Retry loops on errors that are not transient.
  - The same try / log / return-default template wrapped around every function, whether it can fail or not.
  - Success flags instead of errors: returning `True`/`False`, `None`, or `{"success": ..., "error": ...}` dicts where the language would normally raise or throw.
- Fix: catch the narrowest error type where you can actually handle it; let the rest propagate. Log once, at the layer that handles it. Chain causes (`raise X from err` in Python, `cause` in JS/Java). Write messages that say what failed and with which input. A deliberate broad catch needs a fallback value and a short comment or lint directive explaining why. Converting success flags to exceptions changes return types, so hold that for review unless the function is private and every caller is updated in the same pass.

### 7. Naming slop
Names that say nothing about meaning.
- Look for: `data`, `result`, `res`, `temp`, `tmp`, `obj`, `item`, `value`, `info`, `stuff`, `val2`; class names ending in `Manager`, `Handler`, `Helper`, `Processor`, `Service` that hide what the class does; `utils` / `helpers` / `common` modules that collect unrelated functions; booleans not phrased as questions (`flag` instead of `is_ready`); names that disagree with what the code does.
- Fix: rename to what the thing is or does (`unpaid_invoices`, `parse_order_row`, `is_expired`). Splitting a catch-all module by topic changes file layout, so hold it for review. Leave short names alone where convention expects them (`i` in a small loop, `e`/`err` in a catch, `x, y` for coordinates).

### 8. Type and import slop
Typing and imports that add noise or hide mistakes.
- Look for: unused imports; wildcard imports; `Any` / `any` / `object` where a real type is known; casts or `# type: ignore` that hide a real mismatch; outdated typing syntax where the language version supports the modern form (for example `List[str]` instead of `list[str]`, `Optional[X]` instead of `X | None` in Python 3.10+); redundant annotations the compiler already infers.
- Fix: remove unused and wildcard imports; replace `Any` with the real type; fix the mismatch instead of silencing it; use the project's language-version idiom.

### 9. Logging and debug slop
Output left behind from development or added for show.
- Look for: `print()`, `console.log`, `System.out.println`, or `dbg!` debugging calls; log lines that only say "entering function" or "done"; logging inside tight loops; logging secrets or full payloads; the same event logged at several layers; every log at the same level; chatty or cheerful messages in logs, errors, or UI text, meaning exclamation marks, emoji, or "Oops"-style phrasing (plain status text like "Saved" or "Order exported" is fine); `if __name__ == "__main__":` blocks or similar entry points holding demo or test code in modules that are never run directly.
- Fix: delete debug narration. Keep output that helps diagnose problems, such as a startup version line or a real error report. Delete a `__main__` or demo entry point only when no docs, scripts, build files, or CI mention running that module; otherwise hold it for review. Rewrite chatty log and error messages to say plainly what happened; hold visible UI text for review. Keep logs that help someone diagnose a real problem, at the right level (debug, info, warning, error), with the useful context (IDs, counts, file names), logged once.

### 10. Placeholder content
Unfinished or sample material left in work that is supposed to be done.
- Look for: stub bodies (`pass`, `...`, `raise NotImplementedError`, `// TODO: implement`) in code that callers reach; fake sample values (`example.com`, `foo`, `bar`, `John Doe`, `test@test.com`, `123456`) outside test fixtures and docs; generic UI text ("Welcome to the App!", "Get Started", "Lorem ipsum", "Your content here") that was never replaced.
- Fix: delete stubs nothing reaches. Hold reachable stubs, real-looking values, and visible UI text for review, since replacing them changes behavior or what the user sees. Sample values in tests are fine when clearly marked as fixtures.

### 11. Comment noise
Comments that do not tell the reader anything the code cannot. See Comment Standards below.

### 12. Missing tests
Behavior that would break silently.
- Look for: changed code with no test, tests that only check that no error was raised, tests that mock the thing under test, missing edge cases (empty, max, invalid input).
- Fix: add focused tests for preserved behavior before or alongside cleanup. In `/deslop`, list proposed tests under Needs further review instead of writing them.

### 13. UI/design defaults
Generic visual patterns that make an AI-built interface feel unreviewed. These change what the user sees, so they are never applied automatically: list them under Needs further review. Use them as review prompts, not bans; keep choices with a clear brand, accessibility, or design-system reason.
- **Readable text:** flag body text that is too small to read comfortably; keep it at the platform's comfortable body size unless a dense-data view has a clear reason.
- **Shadow restraint:** question shadows on every surface, logo, card, or icon; keep them only where they clarify elevation or interaction.
- **Content hierarchy:** remove repetitive eyebrow/title/description stacks when the title already carries the message; avoid decorative emoji badges unless they are part of the product voice. Emoji or icons that carry meaning (file-type icons, status markers) are fine.
- **Palette rationale:** challenge default AI blue/purple palettes when no brand reason exists.
- **Layout rhythm:** avoid perfectly uniform 3- or 4-column grids when emphasis, asymmetry, or varied card weights would serve the content better.
- **Gradient restraint:** tone down extreme gradients unless the brand owns that look.

## Comment Standards

A comment earns its place by telling the reader something the code cannot.

**Doc comments on public API, in the language's convention**
- Python: PEP 257 docstrings (triple double quotes, one-line summary ending in a period, blank line before details). Match the project's style (Google, NumPy, reST) if one is in use.
- C# / .NET: `///` XML doc comments (`<summary>`, `<param>`, `<returns>`).
- Java: Javadoc `/** ... */` with `@param`, `@return`, `@throws`.
- Kotlin: KDoc `/** ... */`.
- JavaScript / TypeScript: JSDoc / TSDoc `/** ... */`; in TypeScript, skip type tags the signature already states.
- Go: a sentence starting with the identifier's name on exported names.
- Rust: `///` for items, `//!` for modules.
- Private or self-explanatory helpers do not need doc comments.

**Keep**
- Why something is done when it is not obvious: business rules, workarounds, ordering or performance constraints.
- References: ticket IDs, spec links, upstream bug links.
- Warnings about side effects, thread safety, units, or required call order.
- License and copyright headers.
- `TODO` / `FIXME` / `HACK` / `NOTE` that say what is left and why, ideally with an owner or ticket.
- Lint or compiler directives (`# noqa`, `// eslint-disable-next-line`, `#pragma`, `# type: ignore`) with a short reason.

**Remove or rewrite**
- Comments that restate the code (`i += 1  # increment i`).
- Step narration (`# Step 1: ...`, `// Now we loop through ...`).
- Change-log or chat residue (`# Added to fix the bug`, `// Updated per your request`, `# New implementation`).
- Commented-out code.
- Section banners and dividers, unless the project already uses them consistently.
- Doc comments on trivial helpers that only repeat the name, and full Args / Returns / Raises blocks on small functions whose name and signature already say it all. Use a one-line summary instead.
- Comments that restate a constant's name (`MAX_RETRIES = 3  # maximum number of retries`). Keep one only if it gives the unit, source, or reason for the value.
- Emoji, marketing tone, or exclamation marks.
- Stale comments that no longer match the code: fix or delete.

**Form:** full sentences, matching the surrounding style; place comments above the line they explain; keep trailing comments short.

## Tooling

Run the project's existing linters and checkers before and after cleanup, and report the difference. If the project has none configured, these are common defaults; run them read-only to find candidates, and do not add them to the project unless asked.

| Language | Lint / style | Types | Dead code | Tests |
|---|---|---|---|---|
| Python | `ruff check` (`F401` unused imports, `BLE` broad except, `E722` bare except, `T20` print calls, `D` docstrings) | `mypy` or `pyright` | `vulture` | `pytest` or `python -m unittest` |
| JS / TS | `eslint` (`no-unused-vars`, `no-console`, `no-empty`) | `tsc --noEmit` | `knip` or `ts-prune` | the project's test runner |
| C# | `dotnet format --verify-no-changes`, analyzers | compiler warnings as errors | IDE0051 / IDE0052 unused members | `dotnet test` |
| Java / Kotlin | Checkstyle / detekt / ktlint | compiler | IDE unused-symbol inspection | Gradle / Maven test |
| Go | `go vet`, `staticcheck` | compiler | `staticcheck` U1000 | `go test ./...` |

Keep checkers from writing caches into the tree, especially in `--dry-run`: for example `ruff check --no-cache`, `mypy --cache-dir=/dev/null` (`--cache-dir=nul` on Windows), and `python -B` or `PYTHONDONTWRITEBYTECODE=1`. Remove any cache that still appears.

Treat tool output as leads, not orders: confirm each finding (for example, code only reached through reflection or dynamic dispatch is not dead).

## Workflow

Every editing command follows these steps. Pre-commit mode runs them quickly and without questions; the other modes note where they differ.

1. **Find style sources.** Before any edit, look for, in this order:
   - A coding-style skill in the available skill list (a personal, team, or language style skill). Read it if one fits the language.
   - Lint and formatter configs in the repo: `pyproject.toml`, `ruff.toml`, `setup.cfg`, `.editorconfig`, `.eslintrc*`, `.prettierrc*`, `tsconfig.json`, `.stylecop*`, `detekt.yml`, `.golangci.yml`, and similar.
   - Style docs: `CONTRIBUTING*`, `STYLE*`, `docs/style*`, or a style section in the README.
   - A project cleanup procedure, such as `DESLOP.md`. Follow it where it is stricter than this skill.
   - A keep list: anything the project marks as off-limits, such as a "do not change" or "keep" section in the coding standards or style docs, or a dedicated file like `KEEP.md` or `.deslopkeep`. Entries can name functions, files, values, or patterns; known secrets are listed by name only, never by value. Never edit, move, rename, or delete anything on it; if a finding touches a kept item, note it in the report and skip it. Only the user can change the keep list. A kept item gets its own ID (`K1`, `K2`) in the report only when a catalog rule would otherwise flag it, and the keep list wins over every other rule, including test-only flagging (it shows as a K item only); they can't be approved with `/deslop-approve` until the user removes them from the keep list.
   - The code itself: the dominant patterns in nearby files (docstring format, naming, error handling).

   If sources conflict, the order is: the keep list (changed only by the user editing it), then explicit user instruction, then repo docs and configs, then the style skill, then this skill's defaults. Note the sources found, including the keep list, or "none found".
2. **Lock behavior.** Identify what must not change; run or add the narrowest tests. Record a baseline of lint, type, and dead-code findings (run checkers cache-free; see Tooling).
3. **Plan.** List findings by catalog category, the files in scope, and the pass order. Check the plan against the size limit.
4. **Run one pass per catalog category, 1 through 12, in order** (safest first). Category 13 adds review items only and is never applied.

   Skip passes with no findings. After any deletion, re-check in the same pass whether the deleted code's callees, helpers, and test fakes now have no callers, and handle them the same way. Re-run targeted tests after each pass.
5. **Quality gates.** No new test failures compared to the baseline (tests that already failed may keep failing, but list them); lint, type, and existing security checks pass or improve against the baseline. If a gate fails, fix it or back out that change; never force it through.
6. **Docs and changelog.** Update stale references (renamed functions, moved files) in docs the cleanup touched. If the project keeps a changelog, add a new bullet for the cleanup (under the current version or an Unreleased heading, as the project does); never edit or remove existing bullets. `REVIEW(deslop)` comments stay in the source until their item is approved or rejected; if the project has a packaging or release step, follow its rule on stripping them, and say in the report that they are present.
7. **Report** (see Report Format).

## No Git, No Tests

- **No git** (a plain folder or an unpacked zip): unpack or copy it into a working copy. Just before the first real edit (never in `--dry-run`), create a temporary repo whose git folder sits outside the tree and commit a baseline snapshot: from the tree, pass the paths on each command rather than exporting them, for example `mkdir -p ../<tree>.deslop` and then `git --git-dir=../<tree>.deslop/.git --work-tree=. init`, then the same prefix for `add -A` and for the commit, giving it a placeholder identity in case git has none set: `git --git-dir=../<tree>.deslop/.git --work-tree=. -c user.name=deslop -c user.email=deslop@localhost commit -m baseline`. If you do export `GIT_DIR` and `GIT_WORK_TREE`, unset them as soon as the run ends. Edit only the working copy; never edit the original or the snapshot commit. Keep the git folder, `last-report.md`, and other deslop files in that sibling `<tree>.deslop/` folder so nothing gets packed. Never pack the sibling folder, and once every review item is closed, delete only its `.git` folder. Keep `<tree>.deslop/last-report.md` (with the Closed list and reject hashes) until the tree is replaced, so rejected items aren't flagged again and numbers aren't reused. `/deslop` and `/deslop-since` have nothing to scope by, so run as `/deslop-full`. The untouched original (for example the source zip) is the undo.
- **No tests:** add none in `/deslop`. In other modes, add focused tests where practical. If no tests cover a change, apply automatically only comment, import, and debug-output cleanup, plus renames and pass-through wrapper removals when every caller is in scope and the code still compiles, imports, or passes the type checker afterward. Hold everything else for review.

## Modes

### Pre-commit (`/deslop`, the default)
- **Scope:** staged changes (`git diff --cached`); if nothing is staged, uncommitted changes (`git diff`). Within those files, prefer the changed lines and the code they call.
- **Save point first.** Before any edit, save the current state inside the repo's `.git` folder so it's never committed: staged changes to `.git/deslop/<timestamp>-staged.patch` (`git diff --cached --binary`) and unstaged changes to `.git/deslop/<timestamp>-unstaged.patch` (`git diff --binary`). To undo the cleanup, reset every file the cleanup touched or the patches list back to the last commit (`git restore --source=HEAD --staged --worktree <files>`), then apply the staged patch with `git apply --index` and the unstaged patch with `git apply`. Give the save-point paths and these undo commands in the report. Run the undo only when the user asks.
- Run the Workflow without asking questions. Don't write new tests; list the tests you'd add under Needs further review. Stop only if the size limit is hit, or tests fail and the fix is not obvious.
- Afterward, re-stage only the files you changed (`git add <those files>`).
- Short report.

### Full (`/deslop-full [path]`)
- **Scope:** the path, module, or file list given; with no target, the whole repository.
- Run every Workflow step in full, including the written plan and before-and-after tool results.
- Full report.

### Since (`/deslop-since <commit or tag>`)
- **Scope:** every file changed between that commit or tag and now, including uncommitted changes (`git diff <ref>`). Within those files, prefer the changed lines and the code they call.
- Use it to tidy a whole release before shipping. Otherwise the same as Full.

### Review (`/deslop-review`, or `--review`)
A reviewer-only pass on a cleanup someone else made. The same pass must not both write and approve high-impact cleanup.
1. Do not edit files.
2. Read the plan, the diff, and the verification evidence.
3. Check against the Slop Catalog and Ground Rules: leftover findings, new slop introduced, changes that alter behavior, and whether the size limit and one-category-per-pass rule were followed.
4. Give a verdict (approve, approve with follow-ups, or request changes) and list follow-ups for a separate cleanup pass.

### Approve (`/deslop-approve <IDs | all>`)
Applies items from the most recent Needs further review list after the user OKs them. Every editing run saves the open list, plus a closed list of approved and rejected keys with their retired numbers, to `.git/deslop/last-report.md` (or `<tree>.deslop/last-report.md` when there's no git). `--dry-run` and `/deslop-review` never overwrite it.
1. Accept one or more IDs or keys (`R2`, `R1 R3`, `billing.py:export_invoice`) or `all`. Match IDs against the saved list; if an ID's key no longer matches the code, re-scan and show the new list instead of guessing. If no recent report is available, re-scan that scope and show the IDs first; never guess.
2. Apply each approved item as its own small change. Approval lifts the held-for-review limit for that item only; every other rule still applies.
3. For test-only code, follow the removal step in category 1.
4. Run tests and checkers after each item. If one fails, back it out and report it.
5. Report what was applied, what was backed out, and which IDs are still open.

Items that aren't approved or rejected stay open and appear again in the next report until they're closed or the code changes.

### Reject (`/deslop-reject <IDs>`)
Closes review items the user doesn't want changed.
1. Accept IDs or keys, matched against the saved list the same way as Approve.
2. Remove that item's `REVIEW(deslop)` comment, if any; change nothing else. This is an edit, so on a fresh copy create the snapshot and `last-report.md` first, as for any edit.
3. Record the key as rejected in the closed list of `last-report.md`, with a short hash of that symbol's source. Don't flag it again unless the hash no longer matches, meaning the code changed; then give it the next new R number and note "previously rejected as Rn".
4. Short report: what was rejected and which IDs are still open.

### Dry run (`--dry-run`)
Add to any editing command. Do the style lookup, scan, and plan, then stop. List each change that would be applied, grouped by pass, plus the Full report's Needs further review, Closed (from the saved report, if any), and Kept items sections. Edit nothing, run no formatters, stage nothing, and write no files: the plan goes in chat only.

## Report Format

**Full report** (`/deslop-full`, `/deslop-since`):
- **Style sources used** (or none found)
- **Changed files**
- **Findings fixed by category** (counts, with one or two examples each)
- **Tool results before and after** (lint, types, dead code)
- **Verification run** (tests and commands, with results)
- **Needs further review:** every finding not applied, numbered `R1`, `R2`, and so on, each with a stable key (`file:symbol`, for example `billing.py:export_invoice`). The key is the primary identity; an R number stays with its key across runs, and closed numbers are retired, never reused. With no earlier saved report (a fresh copy), numbering starts at R1 and items are matched by key. One line each: the file, what it is, and why it was held (held-for-review list, test-only code, over the size limit, or not sure it's safe). For test-only code, add its callers, replacement, references, and the lines removal would cut.
- **Closed:** approved and rejected keys with their retired R numbers.
- **Kept items:** findings skipped because the keep list covers them, numbered `K1`, `K2`, with their keys.
- **Remaining risks**

**Short report** (`/deslop`, `/deslop-approve`, `/deslop-reject`): the save-point paths (for `/deslop`), changed files, a one-line count per category, the test result, the Needs further review list, closed items, and any kept items. Skip empty sections.

## Examples

**Good:** `deslop` (pre-commit cleanup of staged changes)
**Good:** `/deslop-full src/` (full routine on a folder)
**Good:** `/deslop-since v1.16.1 --dry-run` (preview cleanup of a release)
**Good:** `/deslop-approve R2` (apply one held-back item)
**Good:** `/deslop-reject R3` (keep the code as is and close the item)
**Good:** `deslop this module: too many wrappers, duplicate helpers, and dead code`
**Good:** `clean up the error handling in src/sync without changing behavior`
**Bad:** `refactor auth to support SSO` (feature work)
**Bad:** `clean up formatting` (use the formatter)
