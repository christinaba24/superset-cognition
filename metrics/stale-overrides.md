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

# Stale dependency overrides

This is the workflow problem the daily repo-health sweep exists to catch, written down with the
evidence, so it can be read without running anything.

## The pattern

An `overrides` entry in `superset-frontend/package.json` is a deliberate remediation: a human read an
advisory, worked out which transitive dependency was vulnerable, and pinned it. That fix is correct
on the day it merges.

It does not stay correct. Advisory ranges get widened, new advisories land on the pinned version, and
the pin itself is never revisited — nothing in CI fails when a remediation stops remediating. The
override still looks intentional in review. It just no longer fixes anything.

That is the failure mode: remediation does not break, it decays, silently.

## The evidence in this repo

Overrides as written, versus what the lockfile actually resolves (`superset-frontend/package.json`
and `package-lock.json`):

| Override | As written | Resolved in the lockfile | State |
| --- | --- | --- | --- |
| `lerna` → `js-yaml` | `^4.3.0` | `4.3.0` (under `lerna`, `stylelint`, `cosmiconfig`, `react-diff-viewer-continued`) | decayed — `4.3.0` is the top of the vulnerable range `4.0.0 - 4.3.0`, so the pin sits on the last bad version |
| `minimatch@>=10` → `brace-expansion` | `>=5.0.8` | `5.0.8` under `nx`; `5.0.9` for 13 other copies in the tree | decayed — the range floor is the vulnerable version, and one copy resolved to exactly the floor |
| `dompurify` | `^3.4.11` | `3.4.12` at the root, `3.4.13` in `packages/superset-ui-core` | drifted — one tree patched, the other did not, in the same install |
| `nanoid@>=3 <4` | `3.3.18` | `3.3.18` | holding — an exact pin, and it is absent from `npm audit` output |

`nanoid` matters as much as the other three: it shows the override mechanism works fine. What decays
is a range pin left unattended, not the technique.

Unrelated to any override, `js-yaml` also resolves to `3.15.0` at the top level of
`node_modules` — the top of the other vulnerable range, `3.0.0 - 3.15.0`.

Live `npm audit` on the frontend at the time of writing: 14 advisories — 1 critical, 12 high,
1 moderate.

## Why the obvious automated fix does not work

`npm audit fix --force` (run with `--dry-run`, nothing modified) proposes moving *backwards*:

- `lerna` → `6.6.2` (from 10.x), which is how it "fixes" both `js-yaml` and `brace-expansion`
- `@deck.gl/geo-layers` → `8.9.36`
- `@deck.gl/mesh-layers` → `8.6.5`

All three are SemVer-major moves in the wrong direction, on packages that back the geospatial charts.
The two invocations do not even agree with each other: the JSON run reported `8.9.36` for
`mesh-layers`, the text run reported `8.6.5`.

And then it does not complete. It exits:

```
npm error code EOVERRIDE
npm error Override for @deck.gl/geo-layers@8.9.36 conflicts with direct dependency
```

The proposed fix conflicts with the overrides the team already wrote. So the tool that is supposed to
maintain the remediation cannot reason about the remediation that is already there.

None of this proves those downgrades would break a specific chart — it proves the automated
remediation is not a path anyone should accept unread, which is why teams stop reading it, and why
advisories accumulate on a repo that is otherwise well maintained.

## What the sweep does with it

Every stale override in this file is tracked as a `stale_overrides` entry in
[`engineering-metrics.json`](engineering-metrics.json), so the count is a number that moves rather
than a thing someone remembers to check.

- **MAINTAIN** turns each actionable one into a single spawnable fix prompt — one package, the
  advisory ID, the fixed version, the command, the verification gates. One fix per session, never
  batched, so a bad bump isolates to one upgrade.
- **MEASURE** reports whether the count went down.

The judgement is the product. Choosing a forward path through the dependency graph instead of a
downgrade, regenerating locks, running the gates, and escalating rather than guessing when no fixed
release exists (as with `paramiko`) — none of that is a lookup, which is why the daily run is an
agent and not a cron job wrapping `npm audit`.
