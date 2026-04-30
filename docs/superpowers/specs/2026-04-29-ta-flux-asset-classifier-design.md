# TA-flux — Asset Role Classifier Design

**Date:** 2026-04-29
**Status:** Design approved, pending spec review.
**Owner:** mikael@bjerkeland.com

## Problem

Classify the most likely asset role of a destination host based on the combination of TCP/UDP ports observed in firewall logs. Train daily-updated predictions, and let an operator validate and confirm them so the corpus of labeled hosts grows over time.

## Decisions log

| # | Decision | Rationale |
|---|---|---|
| 1 | Target Splunk Cloud (App Inspect-vetted) | Most restrictive deployment target; design that passes here also runs on Splunk Enterprise. |
| 2 | Binary port matrix as features (`port_22`, `port_80`, …) | User's existing SPL produces this directly; preserves precision for distinctive port-combination signatures. |
| 3 | Labels seeded from `classified_hosts.csv`; source of truth migrated to a KV Store collection on first run | Labels evolve via human confirmations — needs upserts, audit fields, and concurrent writes that a CSV can't safely provide. |
| 4 | Daily inference, weekly retrain | Inference is cheap; training is more expensive and benefits from a larger labeled corpus accumulating between runs. |
| 5 | Row-level confirm dashboard (Simple XML) | Best reviewer UX while staying inside Cloud-vetted SPL primitives. |
| 6 | Algorithm: `RandomForestClassifier` (not Regressor) | Label is categorical. |
| 7 | Labels file holds `dest` (not port-pattern) as the join key | A host's observed ports drift over time; the role does not. Labels are attached to the host, not to a port snapshot. |

## Architecture

### Components

1. **Static port lookup** `odin_classify_ports.csv` — port → category mapping. Input-only; maintained manually.
2. **Seed lookup** `classified_hosts.csv` — one-time bootstrap of `dest, asset_role` from hand-labeling.
3. **KV Store collections:**
   - `flux_classifications` — confirmed labels (seed + human-confirmed predictions). Source of truth for training.
   - `flux_pending_review` — predictions awaiting human sign-off.
4. **SPL macros** — encapsulate the feature-pivot SPL so training, inference, and the dashboard share one source of truth.
5. **Saved searches:**
   - `flux_bootstrap_classifications` — manual / on-install. Imports the seed CSV into the KV Store.
   - `flux_train_model` — weekly. Builds features, joins labels, calls `fit RandomForestClassifier`, saves model.
   - `flux_classify_assets` — daily. Builds features for all hosts, calls `apply`, writes new/changed predictions to `flux_pending_review`.
   - `flux_confirm_prediction` — dashboard-driven. Promotes a row from pending to classifications.
   - `flux_reject_prediction` — dashboard-driven. Removes from pending without confirming.
6. **Review dashboard** `flux_asset_review` — pending table with row-level confirm action.

### Data flow

```
odin_classify_ports.csv ──┐
                          ▼
   tstats Network_Traffic ─► [pivot SPL] ──► port_* feature matrix per dest
                          │                              │
                          │   (training, weekly)         │   (inference, daily)
                          │                              │
   flux_classifications ─► join asset_role               apply <model>
                              │                              │
                              ▼                              ▼
                          fit RandomForestClassifier   predicted_role + prob
                              │                              │
                              ▼                              ▼
                          saved model               flux_pending_review (KV)
                                                          │
                                                          ▼
                                                  Review dashboard
                                                          │
                                                          ▼
                                              "Confirm"  → flux_classifications
```

### Bootstrap

`classified_hosts.csv` is loaded into `flux_classifications` once on first install via `flux_bootstrap_classifications`. After that, the KV Store is the source of truth. The seed CSV stays in the package as a deterministic re-install fallback.

## Add-on directory layout

```
TA-flux/
├── package/
│   ├── app.manifest
│   ├── README.md
│   ├── default/
│   │   ├── app.conf
│   │   ├── collections.conf
│   │   ├── transforms.conf
│   │   ├── macros.conf
│   │   ├── savedsearches.conf
│   │   ├── data/ui/nav/default.xml
│   │   ├── data/ui/views/flux_asset_review.xml
│   │   └── ui-prefs.conf
│   ├── lookups/
│   │   ├── odin_classify_ports.csv
│   │   └── classified_hosts.csv
│   ├── metadata/
│   │   └── default.meta
│   └── static/
│       └── appIcon.png
└── docs/
    └── superpowers/specs/2026-04-29-ta-flux-asset-classifier-design.md
```

### Conf-file responsibilities

- **`app.conf`** — `id = TA-flux`, `version`, `is_visible = true`.
- **`collections.conf`** — declares `flux_classifications` and `flux_pending_review`.
- **`transforms.conf`** — four lookup stanzas:
  - `odin_classify_ports` (file-backed CSV) — used by the pivot macro.
  - `classified_hosts_seed` (file-backed CSV) — used only by the bootstrap search.
  - `flux_classifications_lookup` (KV Store, `external_type = kvstore`) — used by training and inference.
  - `flux_pending_review_lookup` (KV Store) — used by the dashboard and by `flux_confirm_prediction` / `flux_reject_prediction`.
- **`macros.conf`** — three macros: `flux_feature_pivot(2)`, `flux_model_name`, `flux_internal_dest_filter`.
- **`savedsearches.conf`** — five searches above.
- **`default.meta`** — `export = system` for macros, lookups, KV Store collections, and the dashboard. `read = *`, `write = admin,power`.

### App Inspect / Cloud-vetting constraints honored

- No `bin/`, no scripts, no custom search commands, no modular inputs.
- No `inputs.conf`.
- No writes outside KV Store / MLTK model store.
- No external network calls.
- No `local/` shipped.
- MLTK declared as a dependency in `app.manifest`.

## KV Store schemas

### `flux_classifications`

Source of truth. Holds seed labels plus every confirmed prediction.

| field | type | required | notes |
|---|---|---|---|
| `_key` | string | auto | Set to `dest` for idempotent upsert. |
| `dest` | string | yes | Host identifier; matches what `tstats` returns. |
| `asset_role` | string | yes | Single-valued role label. |
| `confirmed_by` | string | yes | Username; `seed` for bootstrap rows. |
| `confirmed_at` | number | yes | Epoch seconds. |
| `source` | string | yes | `seed` \| `confirmed` \| `manual_override`. |
| `notes` | string | no | Free-text reviewer comment. |

Accelerated fields: `dest`, `asset_role`, `source`.

### `flux_pending_review`

Predictions awaiting human sign-off. Daily classify writes here; dashboard reads here; confirm/reject removes by key.

| field | type | required | notes |
|---|---|---|---|
| `_key` | string | auto | Set to `dest` (one pending row per host). |
| `dest` | string | yes | |
| `predicted_role` | string | yes | Top-class prediction. |
| `predicted_probability` | number | yes | Probability of the top class (0–1). |
| `top3_roles` | string | yes | Pipe-separated `role:prob` triples for UI display. |
| `observed_ports` | string | yes | Comma-separated list of host's known ports (context for reviewer). |
| `predicted_at` | number | yes | Epoch seconds. Doubles as the run-recency indicator on the dashboard. |

Accelerated fields: `dest`, `predicted_role`, `predicted_probability`.

`model_version` is intentionally omitted for v1: deriving an accurate training timestamp from inside the classify search (vs. the run timestamp) requires either a third KV Store collection or `| summary <model>` plumbing that's overkill for the value. `predicted_at` is what reviewers actually care about.

### Why `_key = dest`

Both collections key on `dest` so daily updates upsert idempotently. Confirming a host = `outputlookup` upsert into `flux_classifications` plus a rewrite of `flux_pending_review` excluding the row.

## SPL macros

### `flux_feature_pivot(2)` — args: `earliest_arg`, `latest_arg`

```spl
tstats summariesonly=true count as conn_count
    from datamodel=Network_Traffic.All_Traffic
    where earliest=$earliest_arg$ latest=$latest_arg$
    by All_Traffic.dest, All_Traffic.dest_port
| rename All_Traffic.dest as dest All_Traffic.dest_port as dest_port
| `flux_internal_dest_filter`
| lookup odin_classify_ports port as dest_port OUTPUT category
| where isnotnull(category)
| eval dest_port_clean = replace(dest_port, "[^a-zA-Z0-9]", "_")
| eval port_{dest_port_clean} = 1
| stats max(port_*) AS port_*, values(dest_port) AS observed_ports BY dest
| fillnull value=0
```

Differences from the user's original SPL:

- `category` filtering retained (only ports we recognize), but the field is dropped before output (no longer used as a label).
- `dc_category` and `values(category)` removed.
- `observed_ports` preserved as a multivalue passenger for dashboard display; not used as a feature.

### `flux_model_name`

```
flux_asset_classifier
```

Used as `| fit … into \`flux_model_name\`` and `| apply \`flux_model_name\``.

### `flux_internal_dest_filter`

Default ships permissive:

```spl
where 1=1
```

User customizes post-install. Examples documented in README:

```spl
# IP-based
where cidrmatch("10.0.0.0/8", dest) OR cidrmatch("172.16.0.0/12", dest) OR cidrmatch("192.168.0.0/16", dest)

# Hostname-based
where match(dest, "\.crimsonia\.net$")
```

## Saved searches

### `flux_bootstrap_classifications` — manual, disabled by default

```spl
| inputlookup classified_hosts_seed
| lookup flux_classifications_lookup dest OUTPUT asset_role as existing_role
| where isnull(existing_role)
| eval _key = dest,
       source = "seed",
       confirmed_by = coalesce(confirmed_by, "seed"),
       confirmed_at = coalesce(strptime(confirmed_at, "%Y-%m-%d"), now())
| table _key dest asset_role confirmed_by confirmed_at source notes
| outputlookup flux_classifications_lookup append=true
```

`disabled = 1`. Skips destinations already present so re-running can't clobber human-confirmed labels.

### `flux_train_model` — weekly, Sunday 02:00

```spl
| `flux_feature_pivot(-30d@d, now)`
| fields - observed_ports
| lookup flux_classifications_lookup dest OUTPUT asset_role
| where isnotnull(asset_role)
| fit RandomForestClassifier asset_role from port_* into `flux_model_name`
```

`-30d@d` is the default training window: wide enough to capture periodic services, narrow enough to be current.

### `flux_classify_assets` — daily, 03:00

```spl
| `flux_feature_pivot(-7d@d, now)`
| apply `flux_model_name` probabilities=True
| rename "predicted(asset_role)" as predicted_role
| rename "probability(asset_role=*)" as "p_*"
| lookup flux_classifications_lookup dest OUTPUT asset_role as confirmed_role
| where isnull(confirmed_role)
| eval predicted_probability = <foreach p_*: pick the prob whose suffix == predicted_role>
| eval top3_roles = <foreach p_* → mv "role:prob" pairs → sort desc → mvslice(0,3) → mvjoin "|">
| eval observed_ports = mvjoin(observed_ports, ","),
       predicted_at   = now(),
       _key           = dest
| table _key dest predicted_role predicted_probability top3_roles observed_ports predicted_at
| outputlookup flux_pending_review_lookup append=true
```

The `<…>` blocks are standard `foreach`/`mvappend` patterns; exact SPL nailed in the implementation plan.

The 7-day inference window is shorter than training so silent hosts aren't given stale predictions.

### `flux_confirm_prediction` — dashboard-dispatched, on-demand

Args: `dest`, optional `role`, `user`.

```spl
| inputlookup flux_pending_review_lookup
| where dest = "$dest$"
| eval asset_role     = if("$role$" != "", "$role$", predicted_role),
       source         = "confirmed",
       confirmed_by   = "$user$",
       confirmed_at   = now(),
       _key           = dest
| table _key dest asset_role confirmed_by confirmed_at source notes
| outputlookup flux_classifications_lookup append=true
| append [
    | inputlookup flux_pending_review_lookup
    | where dest != "$dest$"
    | outputlookup flux_pending_review_lookup
  ]
| stats count
```

The `append` block rewrites pending without the confirmed row — Splunk's standard "delete one row from a KV Store collection" pattern.

### `flux_reject_prediction` — dashboard-dispatched, on-demand

Args: `dest`.

```spl
| inputlookup flux_pending_review_lookup
| where dest != "$dest$"
| outputlookup flux_pending_review_lookup
| stats count
```

### Search-time properties

- `dispatch.earliest_time = 0`, `dispatch.latest_time = now` on all five (time picking happens inside the SPL).
- `is_visible = false` on the four non-bootstrap searches.
- `enableSched = 1` only on `flux_train_model` and `flux_classify_assets`.
- `dispatchAs = user` on `flux_confirm_prediction` and `flux_reject_prediction` (so confirmations attribute correctly).

## Review dashboard `flux_asset_review`

Single Simple XML view, fully Cloud-compatible (no custom JS, no extensions).

### Layout

- **KPI row:** pending count, confirmed count, last prediction run (max `predicted_at` from `flux_pending_review`).
- **Filter inputs:** predicted-role dropdown, min-confidence slider, free-text search.
- **Pending table:** rows from `flux_pending_review_lookup`. Drilldown sets selection tokens.
- **Review panel:** `<panel depends="$tok_selected_dest$">` — only renders after a row is selected. Shows row context (top alternatives, observed ports), an editable role dropdown defaulting to the predicted role, a notes input, and Confirm / Reject buttons.

### Tokens and actions

- Drilldown on the pending table sets `tok_selected_dest`, `tok_selected_role`, `tok_selected_top3`, `tok_selected_ports`.
- Role override dropdown is populated by `| inputlookup flux_classifications_lookup | stats values(asset_role)` so users can only pick existing roles.
- **Confirm button** runs a hidden search panel: `| savedsearch flux_confirm_prediction dest="$tok_selected_dest$" role="$tok_role_override$" user="$env:user$"`. On completion, clears selection tokens and refreshes the pending table.
- **Reject button** runs `| savedsearch flux_reject_prediction dest="$tok_selected_dest$"` with the same lifecycle.

### Refresh

Pending table and KPI panels: `<refresh>30s</refresh>`.

### Auditability

`flux_classifications` itself is the audit trail — `confirmed_by` is stamped from `$env:user$` and `confirmed_at` from `now()` by `flux_confirm_prediction`. Inspect with `| inputlookup flux_classifications_lookup | table dest asset_role confirmed_by confirmed_at source`.

## Out of scope for v1

- Bulk confirm of multiple pending predictions in one action.
- Editing already-confirmed labels via the UI (relabeling means editing KV Store directly).
- Adding new `asset_role` values from the UI (must come via seed CSV or direct SPL).
- "Retrain now" button on the dashboard (manual `| savedsearch flux_train_model` works).

These are deferred to keep v1 small and AppInspect-clean. Each is a small extension layered on top of the existing saved searches.

## Operations

### Pre-install: convert `classified_hosts.csv` to the new schema

The current file in `lookups/` is the wide pre-pivoted matrix (`dest;category;port_*`). The add-on expects only `dest, asset_role` plus optional metadata. Replace it with a comma-delimited CSV of the form:

```csv
dest,asset_role,confirmed_by,confirmed_at,notes
adcs.air.01.crimsonia.net,directory_services,seed,2026-04-29,
…
```

External CDN-style destinations (akamai, msedge, etc.) should be removed from the seed unless `flux_internal_dest_filter` is configured to keep them in scope.

### Install order

1. Install MLTK on the search head. Verify with `| fit`.
2. Install the TA-flux package.
3. Edit `flux_internal_dest_filter` to match the environment.
4. Verify lookups: `| inputlookup classified_hosts_seed | stats count`.
5. Run `flux_bootstrap_classifications` once manually. Verify `| inputlookup flux_classifications_lookup | stats count by source`.
6. Run `flux_train_model` once manually. Verify with `| summary flux_asset_classifier`.
7. Run `flux_classify_assets` once manually. Open the dashboard.
8. Enable schedules on `flux_train_model` (weekly) and `flux_classify_assets` (daily).

### When to retrain manually (beyond the weekly schedule)

- After modifying `odin_classify_ports.csv`.
- After confirming hosts under a new `asset_role` value.
- After confirming a noticeably-wrong batch.

### Observability searches (documented in README)

- Pending count trend: `| inputlookup flux_pending_review_lookup | stats count` (run on a schedule and chart).
- Confirmation rate: `| inputlookup flux_classifications_lookup | timechart count by source`.
- Model freshness: `| summary flux_asset_classifier`.
- Search health: `| rest /services/saved/searches/<name>/history`.

### Capacity

- Pending review collection: bounded by # of unique non-confirmed `dest`s.
- Confirmations: linear in distinct classified hosts.
- Daily classify: one wide pivot per run; seconds to a couple of minutes on an accelerated DM.
- Weekly train: scales with confirmed-corpus size, not traffic volume.

## README structure (`package/README.md`)

1. What it does (one paragraph).
2. Requirements (MLTK, accelerated Network_Traffic data model).
3. Install & first-run (the 8-step list).
4. Customizing `flux_internal_dest_filter` (with both example forms).
5. Using the review dashboard (screenshots, click-flow).
6. When to retrain manually.
7. Field reference (KV Store schemas, useful introspection searches).
8. Troubleshooting (model missing, empty pending queue, schema-mismatch errors).
