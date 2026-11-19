# Foundana module-ops workflow (REFERENCE, not installed anywhere)

`module-ops.yml` is the reference implementation of the ops-repo workflow that Central dispatches (R-45, T-20). It was
written and checked offline only: it has never run on GitHub. Install it at `.github/workflows/module-ops.yml` in
`MODULE_OPS_REPO` on branch `main`, after reading "Not implemented or unverified" below.

**J3 re-acceptance needed (FDNA-093b).** `module-ops.yml` changed (new `adopt` verb, new `adopt_map` step, stricter seal and
branch checks, `adopt_code` in `result.json`). A person (Michael) must re-read the changed file against J3-1 to J3-11 and
accept it again before Central dispatches any writing verb (install, upgrade, adopt). Nothing here has run on GitHub.

Flow: `gate` (input validation) -> `ec_fetch` -> `ec_engine` -> `ec_emit` (enable_check) or `engine` (install, upgrade) -> `publish` (job 3) -> `result` (assembles `result.json`).
Central dispatches, finds the run by `operation_id` in the run name, polls it, then downloads artefact `module-result`.

## The adopt verb (FDNA-093b)

`adopt` records a hand-ported module in `modules.lock.json` without copying anything else. Dev only (the gate refuses
production for every writing verb). It runs in the same `engine` job as install and upgrade and opens a PR into `dev`.

- **Map path.** The adopt map is the client file `modules/adopt/<module>.json`, committed by a person through a reviewed PR
  (Central never writes it). Schema and size limits belong to the template engine (`tools/modules/adopt-map.mjs`).
- **How it is read.** The `adopt_map` step runs before any `npm ci` or client code. It reads the file as a git blob at the
  dispatched `ref` (`git ls-tree`, so a symlink or a symlinked directory gives no entry): it must be a regular file (mode
  100644), at most 256 KiB, a JSON object. A copy is written outside the client tree and passed with `--adopt-map`.
  Any refusal is `adopt_map_invalid`, and no client code runs afterwards.
- **Engine arguments.** `--adopt-map`, `--expect-version <input version>` and `--withhold-new-migrations`. `--on-drift` is
  never passed (a guard in the engine step fails the job if it ever appears); reviewed drift goes in the map's
  `acknowledged_drift`.
- **Seal.** For adopt the planned and changed paths must be exactly `modules.lock.json` (regular file), and the committed lock
  must name the module at the dispatched version. The `publish` job repeats the path rule. Seal output is codes and counts only.
- **`adopt_code`** in `result.json` (adopt only, when the gate passed): `adopt_ok`, `adopt_map_invalid`,
  `adopt_version_mismatch`, `adopt_withheld_mismatch`, `adopt_incomplete`, `adopt_drift_unexpected`, `adopt_lock_invalid`,
  `adopt_failed`. For a failure `error_code` carries the same word. engine.json's module and version must equal the inputs,
  otherwise `adopt_lock_invalid`.
- **Branch.** `modules/adopt/<module>-<version>` (install and upgrade keep `modules/<module>-<version>`).
- **Not exercised on GitHub.** The step scripts were run locally against stub inputs only (see "Not implemented or unverified").

## Names to configure (names only, never values)

Repository variables (ops repo):

| Name | Used for | Relation to Central |
| --- | --- | --- |
| `MODULE_ENGINE_COMMIT` | the pinned template commit; the `engine_commit` input must equal it | same name as Central `MODULE_ENGINE_COMMIT` (src/env.ts); set both to the same 40-hex value |
| `MODULE_TEMPLATE_REPO` | `owner/name` of foundana-build-template | ops-only name, assumed by this reference |
| `MODULE_CLIENT_OWNER` | the only owner a `target_repo` may belong to; also the App installation owner | ops-only, assumed |
| `MODULE_READ_APP_ID` | read-only App (contents: read) for the template and client checkouts | ops-only, assumed (see "Identities") |
| `MODULE_RUNNER_APP_ID` | the runner App (contents + pull requests on client repos), job 3 only | the contract's "Runner App" |
| `MODULE_TEMPLATE_REF` | FDNA-088b: the template branch the pinned commit must be on while provisional mode is on (for the AEBS dev window: `integration/FDNA-023-build-modules`). Ignored unless provisional is allowed | ops-only |
| `MODULE_PROVISIONAL_ALLOWED` | FDNA-088b: exactly `true` only for the AEBS dev window; any other value (or unset) means off. Refused for production dispatches | ops-only |
| `MODULE_RUNNER_BOT_NAME`, `MODULE_RUNNER_BOT_EMAIL` | identity for the engine's local work-branch commit (never published) | ops-only, assumed |

Secrets, by environment (so a job cannot read a key that is not its own):

| Environment | Secret | Read by |
| --- | --- | --- |
| `module-ops-read` | `MODULE_READ_APP_PRIVATE_KEY` | `enable_check` and `engine` jobs, only the token-minting step |
| `module-ops-publish` | `MODULE_RUNNER_APP_PRIVATE_KEY` | `publish` job, only the token-minting step |

Central-side names that matter to this workflow but are NOT set in the ops repo: `MODULE_OPS_REPO` (where this file
lives), `ARTEFACT_DOWNLOAD_HOSTS` (the host GitHub redirects the artefact download to; still unverified),
`MODULE_DISPATCH_APP_ID`, `MODULE_DISPATCH_APP_PRIVATE_KEY`, `MODULE_DISPATCH_INSTALLATION_ID` (the dispatch App: actions
write on the ops repo, contents/checks read on client repos). The dispatch App key never enters the ops repo.

## Identities

The contract names two Apps: the Central dispatch App and the runner App (job 3 only). It does not say what token the
engine job uses to read client repos. This reference adds a third, read-only App (`MODULE_READ_APP_*`, contents: read)
so that no job but job 3 can ever see the write-capable key. If you prefer two Apps, the alternative is to mint a
down-scoped (`permission-contents: read`) token from the runner App in the engine job; that exposes the runner key to
the job that runs client code (in a minting step only), which is why it was not chosen.

## Clause map: J3-1 to J3-11, R-44 and T-14

Line numbers are for the `module-ops.yml` in this folder at the time of writing; the `# J3-n` comments in the file
mark the same places.

| Clause | Implemented by |
| --- | --- |
| J3-1 job 3 `needs:` the engine job | line 472 `needs: [gate, engine]` |
| J3-2 job 3 only when `npm ci --ignore-scripts`, `tsc --noEmit`, `vitest run` succeeded, by GitHub outcome in `if:` | job outputs lines 297-300 take `steps.npm_ci/tsc/vitest.outcome`; job 3 `if:` lines 473-482 test `needs.engine.result` and those outputs. No file or `result.json` is read. The three steps run in the engine's Docker sandbox (lines 362-369, 451-465) |
| J3-3 `result.json` assembled by a job with no client checkout | job `result`, lines 605-684: no `actions/checkout`, only `needs.*.outputs` through env |
| J3-4 never force-push | no `git push` exists; the branch is created with `POST git/refs` only (line 594), so an existing branch is a 422 failure. No PATCH, no `--force` |
| J3-5 never merge a pull request | only `gh pr create` (line 596); no merge call, no auto-merge in the file |
| J3-6 never set git identity | job 3 has no `git config user.*` or `-c user.*`; commits are made by the API and attributed to the App bot (line 590). The engine job passes `GIT_AUTHOR_*`/`GIT_COMMITTER_*` env for its local, never-pushed commit (engine step, lines 371-396): that is not job 3 and is not `user.*` config, but a strict reading of "no identity anywhere" would object; see below |
| J3-7 PR base is `dev` | `BASE_BRANCH="dev"` constant, line 577; not an input |
| J3-8 PR body from a file | `--body-file "$cs/pr-body.md"`, line 597 |
| J3-9 hooks off | `core.hooksPath` pointed at an empty directory in the scrub env (lines 566-570) and in the `git archive` step (line 182). No git push exists to run hooks |
| J3-10 scrubbed env, token only for `gh` | `export -n APP_TOKEN` (565), every child through `scrub` (`env -i`, line 568), `GH_TOKEN` passed only inside `gh_t` (line 571) |
| J3-11 `--allow-unmerged-template` only when `MODULE_PROVISIONAL_ALLOWED` is exactly `true` AND the environment is dev (CHANGED by FDNA-088b D9, from "never"; recorded deviation) | the gate sets `provisional_allowed` true only for that pair and refuses `provisional_refused_production`; the flag is appended only in the `engine` job's engine step and the `ec_engine` engine step, each guarded by `provisional_allowed == 'true'`. Grep the flag: exactly those two places |
| R-44 / T-14 no secret-bearing job on a pull request; module files never touch `.github/` | the workflow has only `workflow_dispatch`, no `pull_request` trigger; the verify step refuses any path with a `.github`, `.git`, `.husky` or `node_modules` segment (path rules, lines 518-524). The client-side secret-free CI job of R-44 is a human PR in the client repo, not part of this file |
| T-20 push step has no client checkout and gets only a diff artefact | job 3 uses `download-artifact` only; the manifest is checked against the planned file list (lines 508-540) before a token is minted (line 543) |

Other contract items implemented: workflow-level and job-level `permissions: {}` (59 and each job); concurrency group per
client repo (61-64); run name embeds the operation id (16); explicit `timeout-minutes` (3+10 or 25+8+3 minutes, under
Central's 40-minute cancel); artefact `module-result` holding exactly one entry `result.json` (line 681), well under
256 KB compressed and 512 KB unpacked (an extra 400 KB guard at line 671); the closed input set (lines 19-56)
validated with Central's regexes in `gate`; engine commit must equal `vars.MODULE_ENGINE_COMMIT`; production refused for
install and upgrade; no input reaches a `run:` script except through `env:`.

## Not implemented or unverified (read this before relying on it)

1. Nothing ran. YAML parse passed, every `run:` block passed `bash -n`, the embedded Node scripts passed
   `node --check`, and the gate, seal, verify and result scripts were exercised locally with fake inputs (below).
   The checkout, token minting, Docker sandbox, artefact upload/download and the GitHub API calls were not.
2. `actionlint` is not installed on the authoring machine: it was NOT run. Run it before installing.
3. Every action is `<pin sha>`; real commit SHAs must be resolved and pinned. None were invented.
4. FDNA-088b W1-W6 (replaces the old "`seam_checks_ok` is always `null`" note and the assumed `pack_hash`):
   - W1 input `project_ref` (ninth input). W2 `gate`: refuses `project_ref_invalid`, `provisional_refused_production`,
     `template_ref_invalid`; outputs `environment`, `project_ref`, `provisional_allowed`, `template_ref`.
   - W3 `ec_fetch` (read token, no client code): template at `engine_commit` (`fetch-depth: 0`), refuses
     `provisional_not_allowed` unless the pin is an ancestor of `origin/main` (when provisional: of `origin/$MODULE_TEMPLATE_REF`,
     `template_commit_not_on_ref`); fetches the client ref into a bare repo; `pack_hash` = the template's `computePackHash` over
     `git cat-file blob <ref>:<path>`; `readable_tables` parsed as before; uploads artefacts `ec-template` (tar) and `ec-client` (bundle).
   - W4 `ec_engine` (`permissions: {}`, no `environment:`, no token, no secret, no job outputs): restores the template, clones the bundle,
     `npm ci --ignore-scripts` (template and client), runs `cli.mjs enable-check --acceptance <template constant> --env --project-ref
     --template-root --template-commit` (plus the provisional flag under the J3-11 rule), stdout to `engine.json`, uploaded as `ec-engine`.
   - W5 `ec_emit` (no token, no client code): `ops-emit.mjs engine.json "$GITHUB_OUTPUT"` writes `seam_checks_ok`, `error_stage`,
     `error_code`, `acceptance_decision_ref`, `acceptance_expires_at`, `provisional`.
   - W6 `result.json` gains `environment`, `project_ref`, `provisional` (never null: unknown after a passed gate is `true`, fail closed)
     and `acceptance` (object or null); none of the echo fields when the gate refused. Install/upgrade also send `checks`
     (`npm_ci`, `tsc`, `vitest` = `passed`), which Central requires and the earlier version never sent. The `engine` job passes the
     provisional flag under the same rule and reports the engine result's `provisional`.
5. Judgement calls and gaps against the plan (FDNA-088b):
   a. `pack_hash`, `source_sha`, `readable_tables` leave `ec_fetch` through a workflow-authored strict single-line writer, not `ops-emit.mjs`
      (that CLI only maps the engine file to six fields). It checks hex lengths, code grammars and re-serialised JSON.
   b. The client is fetched at full depth (not `--depth=1`) because `git bundle` refuses a shallow repository.
   c. `ec_fetch` imports the template's `seams.mjs` without `npm ci`; if it needs dependencies, `compute` fails with
      `schema_inventory_unavailable`. Not checked.
   d. A client `npm ci` failure does not fail `ec_engine`; the engine reports what it cannot check.
   e. In provisional mode the pin must be on `origin/$MODULE_TEMPLATE_REF` (the plan only says the variable names that branch).
   f. Unverified: whether the engine's `enable-check` needs client refs other than the detached `ref`; the `ec-*` and `ec-{template,engine}`
      download patterns; a tar of the template with its `.git` as the trusted copy.
   g. **U2 (open):** network egress of `ec_engine` is not restricted to the npm registry; a GitHub-hosted runner cannot enforce that here. The
      default is a job with no token, no secret and no environment, `env -i` children and `--ignore-scripts`. Decide U2 (self-hosted runner with an
      egress policy, or the engine's Docker sandbox) before real client data.
   h. `template_ref_invalid` (branch grammar) and the engine-output grammar checks in the `engine` job are additions, not in the plan.
6. `readable_tables` is parsed statically from literal `grant select (...) on <table> to governed_reader` statements in
   `supabase/migrations/*.sql`. Packs that grant through `format()`/dynamic SQL (the Shopify pack does) cannot be read
   this way: the job reports `error_code: readable_tables_unparsed` and sends `null` rather than guess.
   Free text per column comes from the engine's own `schemaInventory`; an unknown column counts as free text.
7. `table_list_hash` is not sent (optional; Central recomputes it from `readable_tables`, so sending it can only cause
   `table_hash_mismatch`).
8. Install, upgrade and adopt run `npm ci --ignore-scripts` and the tests in the engine's Docker sandbox (the pinned
   `node` image), as the engine's own `sandbox.mjs` does; the runner must have Docker. `--update-lock` and
   `--applied-migrations` are never passed, and `--adopt-map` only for adopt (FDNA-093b), so a module with dependency changes will be blocked (`E_DEPS_BLOCKED`) and no
   applied-migrations check is possible (no database credential on the runner, by design).
9. The engine needs the checkout HEAD to equal the dev tip; the workflow passes `--dev-sha <ref>` and
   `--main-sha origin/main`. If `ref` is not the current dev tip the branch is still cut from `ref` and the PR targets `dev`.
10. Job 3 publishes through the Git data API (blobs, tree, commit, ref, then `gh pr create`), not `git push`. The
    published commit therefore differs from the engine's local commit (different SHA, authored by the App bot, trailers
    from the engine's message preserved). This was chosen so J3-6 and J3-9 hold literally. A failed PR creation leaves
    the new branch behind; nothing deletes it.
11. A 40-minute worst case is not modelled across jobs queued behind a busy runner; Central's cancel handles it.
12. `permissions: {}` for GITHUB_TOKEN assumes artefact upload/download needs only the runtime token. Believed true,
    untested.
13. Run conclusion: the workflow concludes `success` whenever `result.json` could be written, even if the operation
    failed (the failure is in `error_stage`/`error_code`). Central's worker today maps a non-success conclusion to
    `runner_run_failed` and does not yet call `downloadArtefact`; confirm Central reads the artefact on success runs.
14. `source_sha` in `result.json` is the verified checkout SHA when a job got that far, otherwise the dispatch `ref`,
    always with `error_stage` set.
15. Secret masking: tokens are masked by the minting action and again by `::add-mask::`. A client-code step could in
    principle write to the runner's file-command directory of its own step; the design limits the blast radius (outcomes
    are read from GitHub's step outcomes, the changeset is uploaded before any client-code step after install, and tsc
    and vitest run in a no-network, no-env container) but this is unverified.
16. Environments `module-ops-read` and `module-ops-publish` must exist with no required reviewers (the run is unattended).

17. FDNA-093b: actionlint could not be run here (no binary; the npm `node-actionlint` build is too old to know `vars`,
    `run-name` or typed `inputs`, so its output is not usable). Validation was: YAML parse, `bash -n` on every run block, and
    local runs of the `adopt_map`, engine, seal, `publish.verify`, gate and result scripts against throwaway git repos and
    a stub engine (map refusals including symlink, executable and oversize, version mismatch, extra path, tampered
    manifest, every result ordering). Run actionlint and a real adopt on a scratch client before accepting.
18. FDNA-093b open points: Central's artefact reader must accept the new `adopt_code` key; the map blob SHA is not bound
    to a Central-supplied value (there is no input for it); the engine step only guards the arguments it builds itself.

## Local dry run (no GitHub)

Needs bash, git, jq, node. From this folder:

```bash
mkdir .v && python - <<'PY'
import yaml
d = yaml.safe_load(open('module-ops.yml', encoding='utf8'))
for job, steps in ((j, d['jobs'][j]['steps']) for j in d['jobs']):
    for i, s in enumerate(steps):
        if 'run' in s:
            open(f".v/{job}.{s.get('id', i)}.sh", 'w', newline='\n').write(s['run'])
PY
bash -n .v/*.sh                                   # syntax of every run block
export RUNNER_TEMP="$PWD/.v/tmp" GITHUB_OUTPUT="$PWD/.v/out"; mkdir -p "$RUNNER_TEMP"; : > "$GITHUB_OUTPUT"
SHA=$(printf 'a%.0s' {1..40})
env OP_ID=11111111-2222-3333-4444-555555555555 TARGET_REPO=foundana/client-x ENGINE_COMMIT=$SHA MODULE=governed-mcp \
  VERSION=1.2.3 ENVIRONMENT=dev VERB=install REF_SHA=$SHA PINNED_ENGINE_COMMIT=$SHA CLIENT_OWNER=foundana \
  TEMPLATE_REPO=foundana/foundana-build-template RUN_REF=refs/heads/main bash .v/gate.check.sh && cat "$GITHUB_OUTPUT"
```

The same pattern runs `engine.seal.sh` against a throwaway git repo with a fake `$RUNNER_TEMP/engine.json`, then
`publish.verify.sh` against the changeset it wrote (a tampered blob must fail with `blob_hash`), then
`result.<id>.sh` with `EN_*`/`PUB_*` env to see each `result.json` variant. The publish step needs a real token and
the GitHub API: replace `gh_t` with a function that echoes its arguments to dry-run it. For a full local run of the
jobs, `act` can execute the shell steps but not `actions/create-github-app-token` or artefact exchange without
stubs; prefer the bash emulation above. Remove `.v` afterwards.
