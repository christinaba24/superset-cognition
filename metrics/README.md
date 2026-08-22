<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Engineering metrics

Tracked, committed baseline metrics for this fork, so automations **read a file instead of
re-running scans**. A measure agent that parses `engineering-metrics.json` finishes in seconds;
reproducing the same numbers live costs a full dev-environment build (Docker, `npm run
plugins:build`, `load-examples`) plus `pip-audit`, `npm audit`, `bandit` and `oxlint` runs.

| File | Purpose |
| --- | --- |
| `engineering-metrics.json` | Canonical, machine-readable metrics. Stable schema (`schema_version`). |
| `engineering-metrics.csv` | Flattened export of the same values for spreadsheet/BI use. |
| `measure-agent-prompt.md` | Prompt text for the automation's "measure" child agent. |

## Provenance

Every baseline number comes from [`../ISSUES.md`](../ISSUES.md), the security & code-quality audit
of commit `f2610e9dca`. Nothing here is estimated. `generated_from` records the source file, the
audited commit and the repo; `last_updated` is the date the file was last refreshed.

## Schema

### `metrics.dependency_vulnerabilities` (pre-computed)

Source: `ISSUES.md` lines 13-16 (summary table) and 58-86 (per-ecosystem breakdown).

- `totals.advisories` — sum across ecosystems (4 + 19 + 13 + 2).
  `totals.python_packages_affected` is the distinct Python package count (`python_dev`'s 8 already
  include the 3 prod packages); `ISSUES.md` reports no package count for the npm ecosystems.
- `by_ecosystem.<python_prod|python_dev|frontend|websocket>` — `tool`, `target`,
  `advisories`, `packages`, `package_names`, `by_severity` (`high`/`moderate`/`low`, plus
  `unclassified` where `ISSUES.md` gives a total without a per-advisory severity).
  `python_prod` is `pip-audit` on `requirements/base.txt`; `python_dev` is the dev+prod env;
  `frontend` and `websocket` are `npm audit` in their respective workspaces.
- `stale_overrides[]` — `package`, `current`, `fix`, `note`. These are the cases where the repo
  already pins an override but the pinned version is itself inside the advisory range.

### `metrics.sast_code_quality` (pre-computed)

Source: `ISSUES.md` lines 17-20 (summary table), 43 and 50-54 (rule breakdowns).

- `bandit.total`, `bandit.by_severity` (`high`/`medium`/`low`), `bandit.by_rule.<Bxxx>` with
  `count` + `description`. `by_rule` covers only the rules enumerated in `ISSUES.md`; the rest of
  the 266 are unenumerated low-severity findings.
- `oxlint.total_warnings`, `oxlint.errors`, `oxlint.by_rule.<rule>`, and
  `oxlint.correctness_relevant_rules` (`rules-of-hooks`, `jsx-key` — the only classes worth
  hand-review; everything else is mechanical).

`mypy` is deliberately **not** tracked here: a repo-wide run is slow and its 3458 errors are
existing debt, not a moving quality signal.

### `metrics.agent_pr_throughput` (live)

Source: `ISSUES.md` lines 93-103 for the automation framing; values from the GitHub API.

- `prs_opened`, `prs_merged`, `review_comments_raised`, `review_comments_resolved` — Devin-authored
  PRs in this repo. Seeded on `as_of`; the automation refreshes these four fields from the GitHub
  API on every run and reports `null` for anything it cannot determine (never a stale value).

## Updating

- Dependency and SAST buckets change only when a new audit is run. Re-run the scans from
  `ISSUES.md` § Reproducing, update the values, and bump `last_updated` (and `source_commit`).
- PR throughput is refreshed by the automation and does not need a commit.
- Keep the shape stable — the measure agent's output contract mirrors it. Additive changes only;
  bump `schema_version` for anything breaking.
