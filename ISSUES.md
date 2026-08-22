# Superset Security & Code-Quality Audit

Scan of `christinaba24/superset-cognition` @ `f2610e9dca` (fork of Apache Superset), run on a
freshly built dev environment (see [Reproducing](#reproducing)).

Everything below was produced by running the tools the repo already configures — no new
lint/scan configuration was introduced.

## Summary

| Area | Tool | Result |
| --- | --- | --- |
| Python deps (prod, `requirements/base.txt`) | `pip-audit` | **4 advisories / 3 packages** |
| Python deps (dev+prod env) | `pip-audit` | **19 advisories / 8 packages** |
| Frontend deps (`superset-frontend`) | `npm audit` | **13 advisories** (12 high, 1 moderate) |
| Websocket deps (`superset-websocket`) | `npm audit` | **2 advisories** (2 high) |
| Python SAST | `bandit -r superset/` | **266 findings** (6 high, 41 medium, 219 low) |
| Python lint | `ruff check .` | clean |
| Python types | `mypy superset/ --check-untyped-defs` | **3458 errors in 400 files** (repo-wide; CI only checks changed files) |
| Frontend lint | `npm run lint` (oxlint) | **1480 warnings**, 0 errors |
| Frontend custom rules | `npm run check:custom-rules` | 21 warnings, 0 errors |
| Frontend styles | `npm run stylelint` | clean |
| Frontend types | `npm run type` (tsc) | clean **only after** `npm run plugins:build` (591 `TS6305` otherwise) |
| CI config | manual review | CodeQL runs default queries only; `security-extended`/`security-and-quality` commented out |

Tests are runnable and green in this environment: `pytest tests/unit_tests/utils` → 941 passed;
`npx jest src/utils` → 653 passed / 59 suites.

## Ranked issues

Severity is the advisory severity where one exists (OSV/GitHub), otherwise an assessment of
blast radius. "Auto" = how amenable the fix is to a Devin automation.

| # | Severity | Category | Description | Affected | Remediation | Auto |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | High | dependency (Python, prod-adjacent) | 3 MCP Python SDK advisories: HTTP transports serve session requests without verifying the authenticating user (GHSA-jpw9-pfvf-9f58, CVSS 3.1 H), WebSocket transport lacks Host/Origin validation (GHSA-vj7q-gjh5-988w), experimental task handlers let any client access/cancel others' tasks (GHSA-hvrp-rf83-w775) | `mcp==1.24.0` (`requirements/development.txt:552`, pulled in via the `fastmcp` extra used by `superset/mcp_service/`) | Bump to `mcp>=1.28.1` | High — lockfile recompile + test run |
| 2 | High | dependency (Python) | `jaraco.context` Zip-Slip path traversal (GHSA-58pv-8j8x-9vj2, CVSS 8.6) | `jaraco-context==6.0.1` (`requirements/development.txt:455`) | Bump to `6.1.0` | High |
| 3 | High | dependency (Python) | `python-multipart` quadratic-time querystring parsing → DoS (GHSA-5rvq-cxj2-64vf, CVSS 7.5); plus two low advisories (`;` separator param pollution, negative `Content-Length` unbounded buffering) | `python-multipart==0.0.29` (`requirements/development.txt:840`) | Bump to `0.0.31` | High |
| 4 | High | dependency (JS) | `brace-expansion` DoS via unbounded intermediate arrays, bypassing the CVE-2026-14257 mitigation (GHSA-rgw5-rvv9-x895, CVSS 7.5). The repo already has an override (`"minimatch@>=10": {"brace-expansion": ">=5.0.8"}`) but `5.0.8` is itself in the vulnerable range | `superset-frontend` (via `nx`/`minimatch`), `superset-websocket` | Raise the override to `>=5.0.9` and re-lock; `npm audit fix` resolves it | High — pure lockfile change |
| 5 | High | dependency (JS) | `js-yaml` quadratic CPU consumption in `!!omap` resolution (GHSA-5p4m-2wfm-xmqj, CVSS 7.5). Same stale-override pattern: `overrides.lerna.js-yaml = "^4.3.0"` but `4.3.0` is vulnerable; a 3.15.0 copy is also present | `lerna@10.0.0` → `js-yaml@4.3.0` / `3.15.0` (`superset-frontend/package.json:412`) | Raise override to `>=4.3.1`; dev-only (build tooling) | High |
| 6 | High | dependency (JS) | `image-size` infinite loops in ICNS/JXL/HEIF parsers → DoS, no fixed release; reaches the bundle through the deck.gl texture stack. `npm audit` proposes a `@deck.gl/mesh-layers@8.6.5` *downgrade*, which is not a real fix | `image-size@0.7.5` ← `texture-compressor@1.0.2` ← `@loaders.gl/textures@4.3.4` ← `@luma.gl/gltf@9.2.6` ← `@deck.gl/{geo,mesh}-layers` | Track upstream `@loaders.gl` dropping `texture-compressor`; interim: `overrides` to a maintained `image-size`, or verify the code path is unreachable (Superset does not load compressed textures from user data) | Medium — needs a judgment call, not a version bump |
| 7 | High | dependency (JS) | `nanoid` predictable/`<3.3.18` advisory in the websocket service lockfile | `superset-websocket` | `npm audit fix` (fix available, non-breaking) | High |
| 8 | High | SAST | 6 `B324` weak-MD5 findings. All are cache/interface-fingerprint hashes, not credential hashing, but they are unannotated so they will keep re-surfacing in every scan | `superset/utils/hashing.py:34`, `superset/utils/public_interfaces.py:43,49`, `superset/key_value/utils.py:98`, `superset/config.py:3286`, one migration | Pass `usedforsecurity=False` to `hashlib.md5(...)` at each site (semantically accurate, FIPS-safe) | High — mechanical, 6 sites |
| 9 | Moderate | dependency (JS) | DOMPurify: `IN_PLACE` hook removal leaves a detached subtree executable → XSS (GHSA-55q2-fjhq-7xh7). Override pins `^3.4.11`, resolved `3.4.12`, advisory covers `<=3.4.12` | `dompurify@3.4.12` (override at `superset-frontend/package.json:395`) | Bump the override past `3.4.12` when released; XSS sanitization is directly security-relevant for Superset's markdown/label rendering | High once a fixed version exists |
| 10 | Moderate | dependency (Python) | `setuptools` MANIFEST.in exclusion bypass via Unicode normalization (GHSA-h35f-9h28-mq5c) — sdist may ship files intended to be excluded | `setuptools==80.9.0` (`requirements/base.txt:370`) | Bump to `83.0.0` | High |
| 11 | Moderate | dependency (Python) | 7 `pip` advisories (path traversal in entry-point names, tar/ZIP confusion, doubly-encoded URL handling, symlink extraction) | `pip==25.1.1` in the built venv | Only affects the install-time toolchain, not the shipped app. Pin `pip>=26.2` in the dev/CI image | High |
| 12 | Moderate | dependency (Python) | `pytest` insecure `/tmp/pytest-of-{user}` handling (GHSA-6w46-j5rx-g56g) — local privilege issue on shared CI runners | `pytest==7.4.4` (`requirements/development.txt:795`) | Bump toward `9.0.3`; major-version jump, needs a test-suite pass | Medium — bump is trivial, fallout is not |
| 13 | Moderate | CI config | `codeql-analysis.yml` runs CodeQL with the default query suite; `queries: security-extended,security-and-quality` is present but commented out, so higher-recall security queries and all quality queries never run. Everything else in the workflow is well hardened (SHA-pinned actions, `permissions: {}` defaults, `zizmor` annotations on the `pull_request_target` workflows) | `.github/workflows/codeql-analysis.yml:78` | Enable `security-extended` on the scheduled run only (keeps PR latency), triage the delta, then decide about `security-and-quality` | High — one-line change + triage of results |
| 14 | Moderate | process / dependency hygiene | The lockfiles drift silently: `uv pip compile` preserves existing pins unless run with `--upgrade`, so e.g. `flask==2.3.3` persists although `pyproject.toml` allows `<4.0.0` and `3.1.3` is current. This is the root cause behind several rows above | `requirements/*.txt`, `superset-frontend/package-lock.json` | Scheduled recompile with `--upgrade` (or per-package `-P`) plus test verification | **Highest-value automation** — see below |
| 15 | Medium | SAST | 7 `B704` `markupsafe.Markup` uses on model-derived strings (potential stored XSS if any component is attacker-controlled) | `superset/models/{dashboard,helpers,slice,sql_lab}.py`, `superset/connectors/sqla/models.py:1642`, `superset/utils/core.py:609` | Audit each site; prefer `Markup(...).format()`/escaping over interpolation | Low — needs per-site reasoning |
| 16 | Medium | SAST | 3 `B301` pickle deserialization (`superset/key_value/types.py:92` plus a migration) and 2 `B102` `exec` (`superset/config.py:3302`, `superset/extensions/utils.py:69`) | as listed | Both are by-design extension points (config loading, key-value codecs). Add scoped `# nosec` with rationale so real regressions stand out | Medium |
| 17 | Medium | SAST | 25 `B608` hardcoded-SQL-expression findings, 3 `B310` `urlopen` with unvalidated schemes (`superset/tasks/utils.py:161`, `superset/reports/notifications/slackv2.py:90`, `superset/db_engine_specs/lint_metadata.py:111`), 1 `B104` bind-all-interfaces | `superset/` | `B608` is largely inherent to a SQL tool; the `B310` sites should assert `http(s)` schemes | Medium |
| 18 | Low | types | Repo-wide `mypy` (the repo's own strict `[tool.mypy]` config) reports 3458 errors in 400 files, concentrated in `superset/connectors/sqla/models.py` (227), `superset/models/helpers.py` (158), `superset/security/manager.py` (124). CI never sees these: the pre-commit hook only type-checks changed files | `superset/` | Not a regression to "fix" — adopt file-by-file via a strictness allowlist so new code cannot add to the debt | Medium — per-module, parallelizable |
| 19 | Low | lint | 1480 oxlint warnings, none failing CI: `prefer-destructuring` 573, `react-hooks/exhaustive-deps` 382, `no-unstable-nested-components` 151, `no-console` 139, `jsx-key` 80, `rules-of-hooks` 47. `rules-of-hooks` and `jsx-key` are the only correctness-relevant classes | `superset-frontend/src`, `plugins/`, `packages/` | Auto-fix the mechanical rules (`npm run lint-fix`); hand-review the 47 `rules-of-hooks` and 80 `jsx-key` hits | High for the mechanical set |
| 20 | Low | DX / CI | `npm run type` fails with 591 `TS6305` on a clean checkout because project references are unbuilt; only `npm run plugins:build && npm run type` works (as CI does). Locally this looks like 591 type errors | `superset-frontend/package.json` | Make `type` depend on `plugins:build`, or document the prerequisite in the script | High |
| 21 | Low | dependency (Python) | Flask missing `Vary: Cookie` (GHSA-68rp-wp8r-4726, CVSS low) and Paramiko allowing SHA-1 in `rsakey.py` (GHSA-r374-rxx8-8654, **no fixed release**; `pyproject.toml` also caps `paramiko<4.0`) | `flask==2.3.3`, `paramiko==3.5.1` (`requirements/base.txt:108,272`) | Flask → `3.1.3` (allowed by `pyproject.toml`, needs a full test run). Paramiko: no action available; track upstream | Medium / none |

### Dependency findings grouped by ecosystem

**Python — `requirements/base.txt` (production)**

| Package | Current | Target | Advisories |
| --- | --- | --- | --- |
| `flask` | 2.3.3 | 3.1.3 | GHSA-68rp-wp8r-4726 (low) |
| `setuptools` | 80.9.0 | 83.0.0 | GHSA-h35f-9h28-mq5c (moderate) |
| `paramiko` | 3.5.1 | — (no fix) | GHSA-r374-rxx8-8654 (low) |

**Python — `requirements/development.txt` (adds)**

| Package | Current | Target | Advisories |
| --- | --- | --- | --- |
| `mcp` | 1.24.0 | 1.28.1 | 3 × high |
| `jaraco-context` | 6.0.1 | 6.1.0 | 1 × high |
| `python-multipart` | 0.0.29 | 0.0.31 | 1 high + 2 low |
| `pytest` | 7.4.4 | 9.0.3 | 1 × moderate |
| `pip` (venv toolchain) | 25.1.1 | 26.2 | 7 × low–moderate |

**JavaScript — `superset-frontend` (13) and `superset-websocket` (2)**

| Package | Current | Fix | Notes |
| --- | --- | --- | --- |
| `brace-expansion` | 5.0.8 (override) | `>=5.0.9` | stale override |
| `js-yaml` | 4.3.0 / 3.15.0 | `>4.3.0` | stale override, dev tooling |
| `dompurify` | 3.4.12 | `>3.4.12` | stale override, security-relevant |
| `nanoid` (websocket) | `<3.3.18` | `3.3.18` | `npm audit fix` |
| `image-size` chain (`texture-compressor`, `@loaders.gl/*`, `@luma.gl/gltf`, `@deck.gl/{geo,mesh}-layers`) | see #6 | no clean fix | audit's suggested deck.gl 8.x downgrade is not a fix |

## Automation notes

Per-issue automatability is in the `Auto` column. The pattern across findings #1–#5, #7, #9–#12 and
#21 is the same: the version bump itself is trivial, the cost is *verifying* it.

**Highest-impact automation — dependency vulnerability remediation with build/test verification.**
A scheduled Devin automation that, per ecosystem:

1. runs `pip-audit` / `npm audit` and diffs against the last run;
2. for each fixable advisory, recompiles the lock (`./scripts/uv-pip-compile.sh -P <pkg>`) or raises
   the `overrides` entry in `superset-frontend/package.json` — one PR per package, so a failure
   isolates to one upgrade;
3. verifies: `pytest tests/unit_tests`, `npm run plugins:build && npm run type`, `npm test`,
   `pre-commit run --all-files`;
4. opens the PR with the advisory IDs, CVSS, and the verification log; drops to a "needs human"
   comment when the upgrade is semver-major or has no fixed release (#6, #21).

This alone clears 12 of the 21 rows, and — importantly — fixes the *class* of problem in #14:
stale overrides and preserved pins are invisible until something scans for them. Secondary
candidates, in order: enabling `security-extended` CodeQL plus auto-fix PRs for its findings (#13);
the mechanical `usedforsecurity=False` and `lint-fix` sweeps (#8, #19); and a per-module mypy
strictness ratchet (#18).

## Reproducing

Environment used for this report: Ubuntu 22.04, Python 3.11.12 (via `uv python`), Node 24.16.0
(from `superset-frontend/.nvmrc`), ruff 0.9.7, mypy 1.15.0.

```bash
# system libs required by the Python deps (mysqlclient, python-ldap, psycopg2)
sudo apt-get install -y libsasl2-dev libldap2-dev libpq-dev default-libmysqlclient-dev pkg-config

# backend
uv venv --python 3.11 .venv && source .venv/bin/activate
uv pip install -r requirements/development.txt   # installs -e . as well
uv pip install pip-audit bandit mypy==1.15.0 pre-commit
uv pip install types-cachetools types-simplejson types-python-dateutil types-requests \
  types-pytz types-croniter types-PyYAML types-setuptools types-paramiko types-Markdown
pre-commit install

# frontend
cd superset-frontend && nvm install && npm ci && npm run plugins:build

# scans
pip-audit -f json -o pip-audit-env.json
pip-audit -r requirements/base.txt --no-deps
bandit -r superset/ -f json -o bandit.json -q
ruff check . && mypy superset/ --check-untyped-defs
cd superset-frontend && npm audit --json && npm run lint && npm run type && npm run stylelint

# tests
pytest tests/unit_tests/utils -q
cd superset-frontend && npx jest src/utils --silent
```
