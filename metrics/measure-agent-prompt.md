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

# MEASURE child-agent prompt

Paste the block below as the prompt for the automation's **measure** child agent. It replaces the
previous metric set entirely.

---

Work in `christinaba24/superset-cognition` (ref: `master`).

You are the MEASURE agent. Report exactly three metrics, replacing whatever metrics were
previously tracked:

1. `dependency_vulnerabilities`
2. `sast_code_quality`
3. `agent_pr_throughput`

## How to get the numbers (this run must be fast)

**Read `metrics/engineering-metrics.json` from the repo. Do not re-derive the pre-computed
numbers.** That file is the committed, versioned baseline for metrics 1 and 2, generated from
`ISSUES.md` at commit `f2610e9dca`. Reading it is the whole point: it replaces a multi-minute scan
with a single file read. See `metrics/README.md` for the field-by-field schema.

Concretely:

- Metrics 1 and 2 (`dependency_vulnerabilities`, `sast_code_quality`): copy the values straight out
  of `metrics/engineering-metrics.json`. Do **not** run `pip-audit`, `npm audit`, `bandit`, or
  `oxlint`.
- Metric 3 (`agent_pr_throughput`): this is the **only** live-derived metric. Refresh
  `prs_opened`, `prs_merged`, `review_comments_raised` and `review_comments_resolved` from the
  GitHub API (e.g. `gh pr list --state all --json number,state,mergedAt,author` and
  `gh api repos/christinaba24/superset-cognition/pulls/<n>/comments`, or the GraphQL
  `reviewThreads { isResolved }` field for resolved counts), scoped to Devin-authored PRs. If a
  value cannot be determined, emit `null` — never a stale number.

## Do not do

- Do **not** build a dev environment: no Docker, no webpack/dev server, no
  `npm run plugins:build`, no `superset load-examples`, no `npm ci` of the frontend monorepo.
- Do **not** run repo-wide `mypy` (or any other full-repo type check). It is slow and the numbers
  are known existing debt, not a signal being tracked.
- Do **not** run the dependency or SAST scanners. If the JSON file is missing or unparseable, say
  so explicitly in your output and emit `null` values for the affected metrics rather than falling
  back to a live scan.

## Output

Emit a single consolidated JSON object mirroring the shape of `metrics/engineering-metrics.json`,
with metric 3 refreshed:

```json
{
  "schema_version": "1.0",
  "generated_from": {
    "source_file": "ISSUES.md",
    "source_commit": "f2610e9dca",
    "repo": "christinaba24/superset-cognition"
  },
  "last_updated": "<date of this run, YYYY-MM-DD>",
  "metrics": {
    "dependency_vulnerabilities": {
      "refresh": "pre-computed",
      "totals": { "advisories": 38, "packages_affected": 11 },
      "by_ecosystem": {
        "python_prod":  { "advisories": 4,  "packages": 3, "package_names": ["flask", "setuptools", "paramiko"] },
        "python_dev":   { "advisories": 19, "packages": 8 },
        "frontend":     { "advisories": 13, "by_severity": { "high": 12, "moderate": 1 } },
        "websocket":    { "advisories": 2,  "by_severity": { "high": 2 } }
      },
      "stale_overrides": [
        { "package": "brace-expansion", "current": "5.0.8", "fix": ">=5.0.9", "note": "..." },
        { "package": "js-yaml", "current": "4.3.0", "fix": ">4.3.0", "note": "..." },
        { "package": "dompurify", "current": "3.4.12", "fix": ">3.4.12", "note": "..." },
        { "package": "nanoid", "current": "<3.3.18", "fix": "3.3.18", "note": "..." }
      ]
    },
    "sast_code_quality": {
      "refresh": "pre-computed",
      "bandit": {
        "total": 266,
        "by_severity": { "high": 6, "medium": 41, "low": 219 },
        "by_rule": { "B324": 6, "B704": 7, "B608": 25, "B301": 3, "B310": 3, "B102": 2, "B104": 1 }
      },
      "oxlint": {
        "total_warnings": 1480,
        "errors": 0,
        "by_rule": {
          "prefer-destructuring": 573,
          "react-hooks/exhaustive-deps": 382,
          "no-unstable-nested-components": 151,
          "no-console": 139,
          "jsx-key": 80,
          "rules-of-hooks": 47
        },
        "correctness_relevant_rules": ["rules-of-hooks", "jsx-key"]
      }
    },
    "agent_pr_throughput": {
      "refresh": "live",
      "as_of": "<date of this run>",
      "prs_opened": 0,
      "prs_merged": 0,
      "review_comments_raised": 0,
      "review_comments_resolved": 0
    }
  }
}
```

Follow the JSON with at most five lines of prose: any metric that moved since the committed
baseline, and anything you could not determine.
