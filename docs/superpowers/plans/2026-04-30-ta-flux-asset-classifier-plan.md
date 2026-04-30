# TA-flux Asset Role Classifier Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Splunk Cloud-vetted add-on that uses MLTK's RandomForestClassifier to predict asset roles from firewall-observed port patterns, with a daily classification job and a human-confirmation dashboard.

**Architecture:** Pure-SPL Splunk add-on (no scripts, no custom commands). Two KV Store collections hold confirmed labels and pending predictions. SPL macros encapsulate the feature pivot. Five saved searches handle bootstrap, weekly training, daily inference, and dashboard-driven confirm/reject. A Simple XML dashboard provides row-level review.

**Tech Stack:** Splunk Cloud / Enterprise 9.x, Splunk Machine Learning Toolkit 5.4+, Splunk KV Store, Simple XML dashboards. Validation via `splunk-appinspect`.

**Spec:** [docs/superpowers/specs/2026-04-29-ta-flux-asset-classifier-design.md](../specs/2026-04-29-ta-flux-asset-classifier-design.md)

---

## File structure

```
TA-flux/
├── package/
│   ├── app.manifest                       # Cloud manifest, MLTK dependency
│   ├── README.md                          # User-facing install + usage
│   ├── default/
│   │   ├── app.conf                       # App metadata
│   │   ├── collections.conf               # 2 KV Store collections
│   │   ├── transforms.conf                # 4 lookup definitions
│   │   ├── macros.conf                    # 3 SPL macros
│   │   ├── savedsearches.conf             # 5 saved searches
│   │   └── data/ui/
│   │       ├── nav/default.xml
│   │       └── views/flux_asset_review.xml
│   ├── lookups/
│   │   ├── odin_classify_ports.csv        # Moved from /lookups
│   │   └── classified_hosts.csv           # Moved from /lookups, schema fixed
│   └── metadata/
│       └── default.meta                   # Permissions
├── docs/
│   └── superpowers/
│       ├── specs/2026-04-29-ta-flux-asset-classifier-design.md
│       └── plans/2026-04-30-ta-flux-asset-classifier-plan.md
├── .gitignore
└── README.md                              # Repo-level dev doc
```

## Prerequisites for the implementer

Install once:

```bash
pip install splunk-appinspect
```

For end-to-end SPL/MLTK testing, you'll also want a Splunk instance with MLTK installed (Cloud trial, Enterprise local install, or `docker run --rm -p 8000:8000 splunk/splunk:latest`). Without it, you can still write and AppInspect-validate every task; only manual SPL execution requires a Splunk instance.

## Spec → plan deviation

**`top3_roles` field semantics renamed.** The spec describes `top3_roles` as the top three roles by probability. Computing a true top-3 sort in pure SPL is awkward; the plan stores **all** class probabilities pipe-separated in this field. The dashboard parses and displays the top 3. Field name retained for KV Store schema compatibility with the spec; semantics is "all class probabilities, dashboard re-ranks at display time."

---

### Task 1: Initialize repo skeleton and git

**Files:**
- Create: `.gitignore`
- Create: `README.md` (repo-level)
- Create: `package/` directory tree

- [ ] **Step 1: Initialize git**

```bash
cd /Users/mbjerkel/Documents/Git/TA-flux
git init -b main
```

- [ ] **Step 2: Create the package directory tree**

```bash
mkdir -p package/default/data/ui/nav package/default/data/ui/views package/lookups package/metadata
```

- [ ] **Step 3: Move existing lookups under package/**

```bash
git mv lookups/odin_classify_ports.csv package/lookups/odin_classify_ports.csv 2>/dev/null \
  || mv lookups/odin_classify_ports.csv package/lookups/odin_classify_ports.csv
git mv lookups/classified_hosts.csv package/lookups/classified_hosts.csv 2>/dev/null \
  || mv lookups/classified_hosts.csv package/lookups/classified_hosts.csv
rmdir lookups
```

- [ ] **Step 4: Write `.gitignore`**

```
.DS_Store
*.spl
*.tar.gz
build/
dist/
__pycache__/
.venv/
.idea/
.vscode/
```

- [ ] **Step 5: Write repo-level `README.md`**

```markdown
# TA-flux

Splunk Cloud-vetted add-on that classifies asset roles from firewall traffic
port patterns using MLTK RandomForestClassifier.

User-facing documentation: [package/README.md](package/README.md)

Design spec: [docs/superpowers/specs/2026-04-29-ta-flux-asset-classifier-design.md](docs/superpowers/specs/2026-04-29-ta-flux-asset-classifier-design.md)

Implementation plan: [docs/superpowers/plans/2026-04-30-ta-flux-asset-classifier-plan.md](docs/superpowers/plans/2026-04-30-ta-flux-asset-classifier-plan.md)

## Building the .spl

    tar -czvf TA-flux.spl --exclude='.DS_Store' package/

## Validating before submission

    splunk-appinspect inspect package/ --mode test --included-tags cloud
```

- [ ] **Step 6: Verify the layout**

```bash
find package -type d | sort
```

Expected:

```
package
package/default
package/default/data
package/default/data/ui
package/default/data/ui/nav
package/default/data/ui/views
package/lookups
package/metadata
```

- [ ] **Step 7: Commit**

```bash
git add .gitignore README.md package/lookups/ docs/
git commit -m "chore: initialize repo skeleton and move lookups under package/"
```

---

### Task 2: Write `app.manifest` and `app.conf`

**Files:**
- Create: `package/app.manifest`
- Create: `package/default/app.conf`

- [ ] **Step 1: Write `package/app.manifest`**

```json
{
  "schemaVersion": "2.0.0",
  "info": {
    "id": {
      "group": null,
      "name": "TA-flux",
      "version": "0.1.0"
    },
    "author": [
      { "name": "Mikael Bjerkeland", "email": "mikael@bjerkeland.com", "company": null }
    ],
    "releaseDate": null,
    "description": "Asset role classifier using firewall traffic ports and Splunk MLTK.",
    "classification": {
      "intendedAudience": "Anyone",
      "categories": ["Security, Fraud & Compliance"],
      "developmentStatus": "Beta"
    },
    "commonInformationModels": null,
    "license": null,
    "privacyPolicy": null,
    "releaseNotes": null
  },
  "dependencies": {
    "Splunk_ML_Toolkit": {
      "version": "5.4.0",
      "package": "Splunk_ML_Toolkit"
    }
  },
  "tasks": [],
  "inputGroups": null,
  "incompatibleApps": null,
  "platformRequirements": {
    "splunk": {
      "Enterprise": "9.0.0"
    }
  },
  "supportedDeployments": ["_standalone", "_search_head_clustering"],
  "targetWorkloads": ["_search_heads"]
}
```

- [ ] **Step 2: Write `package/default/app.conf`**

```ini
[install]
is_configured = false
state = enabled

[ui]
is_visible = true
label = TA-flux Asset Classifier

[launcher]
author = Mikael Bjerkeland
description = Asset role classifier using firewall traffic ports and Splunk MLTK.
version = 0.1.0

[package]
id = TA-flux
check_for_updates = false
```

- [ ] **Step 3: Verify manifest parses as JSON**

```bash
python3 -m json.tool package/app.manifest > /dev/null && echo OK
```

Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add package/app.manifest package/default/app.conf
git commit -m "feat: add app.manifest and app.conf with MLTK dependency"
```

---

### Task 3: Permissions (`metadata/default.meta`)

**Files:**
- Create: `package/metadata/default.meta`

- [ ] **Step 1: Write `package/metadata/default.meta`**

```ini
[]
access = read : [ * ], write : [ admin, power ]
export = system

[macros]
export = system

[transforms]
export = system

[collections]
export = system

[savedsearches]
export = none

[views]
export = system

[nav]
export = system
```

`savedsearches` is `export = none` because the searches are internal machinery; only the dashboard surfaces them. Nav and views export to `system` so the dashboard appears wherever the add-on is installed.

- [ ] **Step 2: Commit**

```bash
git add package/metadata/default.meta
git commit -m "feat: add default.meta with permissions and exports"
```

---

### Task 4: KV Store schemas (`collections.conf`)

**Files:**
- Create: `package/default/collections.conf`

- [ ] **Step 1: Write `package/default/collections.conf`**

```ini
[flux_classifications]
enforceTypes = true
field.dest = string
field.asset_role = string
field.confirmed_by = string
field.confirmed_at = number
field.source = string
field.notes = string
accelerated_fields.idx_dest = {"dest": 1}
accelerated_fields.idx_asset_role = {"asset_role": 1}
accelerated_fields.idx_source = {"source": 1}

[flux_pending_review]
enforceTypes = true
field.dest = string
field.predicted_role = string
field.predicted_probability = number
field.top3_roles = string
field.observed_ports = string
field.predicted_at = number
accelerated_fields.idx_dest = {"dest": 1}
accelerated_fields.idx_predicted_role = {"predicted_role": 1}
accelerated_fields.idx_predicted_probability = {"predicted_probability": -1}
```

`_key` is implicit on every KV Store collection — no need to declare it. We'll write rows with `_key = dest` from SPL so updates upsert idempotently.

- [ ] **Step 2: Commit**

```bash
git add package/default/collections.conf
git commit -m "feat: declare KV Store collections flux_classifications and flux_pending_review"
```

---

### Task 5: Lookup definitions (`transforms.conf`)

**Files:**
- Create: `package/default/transforms.conf`

- [ ] **Step 1: Write `package/default/transforms.conf`**

```ini
[odin_classify_ports]
filename = odin_classify_ports.csv

[classified_hosts_seed]
filename = classified_hosts.csv

[flux_classifications_lookup]
external_type = kvstore
collection = flux_classifications
fields_list = _key, dest, asset_role, confirmed_by, confirmed_at, source, notes

[flux_pending_review_lookup]
external_type = kvstore
collection = flux_pending_review
fields_list = _key, dest, predicted_role, predicted_probability, top3_roles, observed_ports, predicted_at
```

- [ ] **Step 2: Commit**

```bash
git add package/default/transforms.conf
git commit -m "feat: add lookup definitions for CSV files and KV Store collections"
```

---

### Task 6: Internal-dest filter macro

**Files:**
- Create: `package/default/macros.conf`

- [ ] **Step 1: Write the first stanza of `package/default/macros.conf`**

```ini
# -----------------------------------------------------------------------------
# flux_internal_dest_filter
#
# Site-specific scoping macro. EDIT THIS POST-INSTALL to restrict classification
# to your internal hosts only. Default ships permissive (matches everything).
#
# Example replacements:
#
#   IP-based:
#     where cidrmatch("10.0.0.0/8", dest)
#        OR cidrmatch("172.16.0.0/12", dest)
#        OR cidrmatch("192.168.0.0/16", dest)
#
#   Hostname-based:
#     where match(dest, "\.crimsonia\.net$")
#
[flux_internal_dest_filter]
definition = where 1=1
iseval = 0
```

- [ ] **Step 2: Commit**

```bash
git add package/default/macros.conf
git commit -m "feat: add flux_internal_dest_filter macro (permissive default)"
```

---

### Task 7: Model name macro

**Files:**
- Modify: `package/default/macros.conf` (append)

- [ ] **Step 1: Append to `package/default/macros.conf`**

```ini

# -----------------------------------------------------------------------------
# flux_model_name
#
# Single source of truth for the MLTK model name. Used as
#   | fit ... into `flux_model_name`
#   | apply `flux_model_name`
#
[flux_model_name]
definition = flux_asset_classifier
iseval = 0
```

- [ ] **Step 2: Commit**

```bash
git add package/default/macros.conf
git commit -m "feat: add flux_model_name macro"
```

---

### Task 8: Feature-pivot macro

**Files:**
- Modify: `package/default/macros.conf` (append)

- [ ] **Step 1: Append to `package/default/macros.conf`**

```ini

# -----------------------------------------------------------------------------
# flux_feature_pivot(earliest_arg, latest_arg)
#
# Produces one row per dest with port_* binary features (1 if the port was
# observed accepting traffic on this dest in the time window, else 0). Drops
# unrecognized ports via the odin_classify_ports lookup. Adds a multivalue
# `observed_ports` field as a passenger for dashboard display.
#
# Args:
#   earliest_arg - relative or absolute time, e.g. -7d@d
#   latest_arg   - relative or absolute time, e.g. now
#
[flux_feature_pivot(2)]
args = earliest_arg, latest_arg
definition = tstats summariesonly=true count as conn_count \
    from datamodel=Network_Traffic.All_Traffic \
    where earliest=$earliest_arg$ latest=$latest_arg$ \
    by All_Traffic.dest, All_Traffic.dest_port \
| rename All_Traffic.dest as dest All_Traffic.dest_port as dest_port \
| `flux_internal_dest_filter` \
| lookup odin_classify_ports port as dest_port OUTPUT category \
| where isnotnull(category) \
| eval dest_port_clean = replace(dest_port, "[^a-zA-Z0-9]", "_") \
| eval port_{dest_port_clean} = 1 \
| stats max(port_*) AS port_*, values(dest_port) AS observed_ports BY dest \
| fillnull value=0
iseval = 0
```

The continuation backslashes are required — Splunk reads multi-line `definition` values only when the prior line ends with `\`.

- [ ] **Step 2: Commit**

```bash
git add package/default/macros.conf
git commit -m "feat: add flux_feature_pivot(2) macro for binary port matrix"
```

---

### Task 9: Bootstrap saved search

**Files:**
- Create: `package/default/savedsearches.conf`

- [ ] **Step 1: Write `package/default/savedsearches.conf`**

```ini
# =============================================================================
# flux_bootstrap_classifications
#
# One-time import of classified_hosts.csv into flux_classifications KV Store.
# Skips destinations already present (so re-running can't clobber human
# confirmations). Disabled by default; enable via UI, run once, disable again.
#
[flux_bootstrap_classifications]
disabled = 1
enableSched = 0
is_visible = 0
dispatch.earliest_time = 0
dispatch.latest_time = now
search = | inputlookup classified_hosts_seed \
| lookup flux_classifications_lookup dest OUTPUT asset_role as existing_role \
| where isnull(existing_role) \
| eval _key = dest, \
       source = "seed", \
       confirmed_by = coalesce(confirmed_by, "seed"), \
       confirmed_at = coalesce(strptime(confirmed_at, "%Y-%m-%d"), now()) \
| table _key dest asset_role confirmed_by confirmed_at source notes \
| outputlookup flux_classifications_lookup append=true
```

- [ ] **Step 2: Commit**

```bash
git add package/default/savedsearches.conf
git commit -m "feat: add flux_bootstrap_classifications saved search"
```

---

### Task 10: Weekly training saved search

**Files:**
- Modify: `package/default/savedsearches.conf` (append)

- [ ] **Step 1: Append to `package/default/savedsearches.conf`**

```ini

# =============================================================================
# flux_train_model
#
# Builds the feature matrix over the last 30 days, attaches asset_role from
# flux_classifications, drops unlabeled rows, fits a RandomForestClassifier.
# Scheduled weekly Sundays at 02:00 UTC.
#
[flux_train_model]
disabled = 0
enableSched = 1
cron_schedule = 0 2 * * 0
is_visible = 0
dispatch.earliest_time = 0
dispatch.latest_time = now
search = | `flux_feature_pivot(-30d@d, now)` \
| fields - observed_ports \
| lookup flux_classifications_lookup dest OUTPUT asset_role \
| where isnotnull(asset_role) \
| fit RandomForestClassifier asset_role from port_* into `flux_model_name`
```

- [ ] **Step 2: Commit**

```bash
git add package/default/savedsearches.conf
git commit -m "feat: add flux_train_model weekly saved search"
```

---

### Task 11: Daily classification saved search

**Files:**
- Modify: `package/default/savedsearches.conf` (append)

This is the most complex search. It applies the model, extracts the predicted role's probability, captures the full class-probability distribution as a pipe-separated string (the dashboard re-ranks for top-3 display), and upserts non-confirmed hosts into `flux_pending_review`.

- [ ] **Step 1: Append to `package/default/savedsearches.conf`**

```ini

# =============================================================================
# flux_classify_assets
#
# Builds the feature matrix over the last 7 days, applies the trained model,
# computes per-host predicted_probability (probability of the predicted class)
# and top3_roles (pipe-separated "role:prob" pairs across ALL classes; the
# dashboard re-ranks at display time). Excludes hosts already confirmed.
# Upserts into flux_pending_review keyed by dest. Scheduled daily at 03:00 UTC.
#
[flux_classify_assets]
disabled = 0
enableSched = 1
cron_schedule = 0 3 * * *
is_visible = 0
dispatch.earliest_time = 0
dispatch.latest_time = now
search = | `flux_feature_pivot(-7d@d, now)` \
| apply `flux_model_name` probabilities=True \
| rename "predicted(asset_role)" as predicted_role \
| rename "probability(asset_role=*)" as "p_*" \
| lookup flux_classifications_lookup dest OUTPUT asset_role as confirmed_role \
| where isnull(confirmed_role) \
| eval predicted_probability = 0 \
| foreach p_* [eval predicted_probability = if("<<MATCHSTR>>"==predicted_role, round('<<FIELD>>', 4), predicted_probability)] \
| eval top3_roles = "" \
| foreach p_* [eval top3_roles = top3_roles . if(len(top3_roles)==0, "", "|") . "<<MATCHSTR>>" . ":" . tostring(round('<<FIELD>>', 4))] \
| eval observed_ports = mvjoin(observed_ports, ","), \
       predicted_at   = now(), \
       _key           = dest \
| table _key dest predicted_role predicted_probability top3_roles observed_ports predicted_at \
| outputlookup flux_pending_review_lookup append=true
```

Key SPL idioms:

- `foreach p_*` iterates over every column matching `p_*`. Inside the bracket:
  - `<<FIELD>>` expands to the literal field name (e.g. `p_web_server`).
  - `<<MATCHSTR>>` expands to just the wildcard portion (e.g. `web_server`).
- `'<<FIELD>>'` (single-quoted) dereferences the value.
- `if(len(top3_roles)==0, "", "|")` avoids a leading `|` separator on the first iteration.
- `outputlookup … append=true` upserts on `_key` matches; rows for hosts that disappear from the daily search remain untouched (you may want a separate prune search later, out of scope for v1).

- [ ] **Step 2: Commit**

```bash
git add package/default/savedsearches.conf
git commit -m "feat: add flux_classify_assets daily saved search"
```

---

### Task 12: Confirm-prediction saved search

**Files:**
- Modify: `package/default/savedsearches.conf` (append)

- [ ] **Step 1: Append to `package/default/savedsearches.conf`**

```ini

# =============================================================================
# flux_confirm_prediction
#
# Dashboard-driven, on-demand. Takes args dest, role (optional override),
# user. Promotes a row from flux_pending_review to flux_classifications with
# source="confirmed", then rewrites flux_pending_review without the row.
#
[flux_confirm_prediction]
disabled = 0
enableSched = 0
is_visible = 0
dispatchAs = user
dispatch.earliest_time = 0
dispatch.latest_time = now
search = | inputlookup flux_pending_review_lookup \
| where dest = "$dest$" \
| eval asset_role     = if("$role$" != "", "$role$", predicted_role), \
       source         = "confirmed", \
       confirmed_by   = "$user$", \
       confirmed_at   = now(), \
       _key           = dest, \
       notes          = "$notes$" \
| table _key dest asset_role confirmed_by confirmed_at source notes \
| outputlookup flux_classifications_lookup append=true \
| append [ \
    | inputlookup flux_pending_review_lookup \
    | where dest != "$dest$" \
    | outputlookup flux_pending_review_lookup \
  ] \
| stats count
```

The `append` block is the standard "delete one KV Store row via SPL" pattern: rewrite the entire collection minus the row. Acceptable for the size of the pending queue (thousands at most).

- [ ] **Step 2: Commit**

```bash
git add package/default/savedsearches.conf
git commit -m "feat: add flux_confirm_prediction saved search"
```

---

### Task 13: Reject-prediction saved search

**Files:**
- Modify: `package/default/savedsearches.conf` (append)

- [ ] **Step 1: Append to `package/default/savedsearches.conf`**

```ini

# =============================================================================
# flux_reject_prediction
#
# Dashboard-driven, on-demand. Takes arg dest. Drops the row from
# flux_pending_review without writing to flux_classifications.
#
[flux_reject_prediction]
disabled = 0
enableSched = 0
is_visible = 0
dispatchAs = user
dispatch.earliest_time = 0
dispatch.latest_time = now
search = | inputlookup flux_pending_review_lookup \
| where dest != "$dest$" \
| outputlookup flux_pending_review_lookup \
| stats count
```

- [ ] **Step 2: Commit**

```bash
git add package/default/savedsearches.conf
git commit -m "feat: add flux_reject_prediction saved search"
```

---

### Task 14: Navigation

**Files:**
- Create: `package/default/data/ui/nav/default.xml`

- [ ] **Step 1: Write `package/default/data/ui/nav/default.xml`**

```xml
<nav search_view="search" color="#65A637">
  <view name="flux_asset_review" default="true" />
  <view name="search" />
</nav>
```

`flux_asset_review` is the dashboard added in the next task; setting it as the default view makes it the landing page when users open the add-on.

- [ ] **Step 2: Commit**

```bash
git add package/default/data/ui/nav/default.xml
git commit -m "feat: add nav with flux_asset_review as default view"
```

---

### Task 15: Review dashboard

**Files:**
- Create: `package/default/data/ui/views/flux_asset_review.xml`

This is the largest single file in the add-on. It implements the entire row-level review UX from the spec.

- [ ] **Step 1: Write `package/default/data/ui/views/flux_asset_review.xml`**

```xml
<form version="1.1" theme="light">
  <label>TA-flux — Asset Role Review</label>
  <description>
    Review and confirm predicted asset roles. Click a row to load it into the
    review panel below.
  </description>

  <fieldset submitButton="false" autoRun="true">
    <input type="dropdown" token="tok_role" searchWhenChanged="true">
      <label>Filter by predicted role</label>
      <choice value="*">All</choice>
      <fieldForLabel>predicted_role</fieldForLabel>
      <fieldForValue>predicted_role</fieldForValue>
      <search>
        <query>| inputlookup flux_pending_review_lookup | stats count by predicted_role</query>
      </search>
      <default>*</default>
      <initialValue>*</initialValue>
    </input>
    <input type="text" token="tok_min_conf" searchWhenChanged="true">
      <label>Minimum confidence</label>
      <default>0.0</default>
      <initialValue>0.0</initialValue>
    </input>
    <input type="text" token="tok_search" searchWhenChanged="true">
      <label>Filter by dest substring</label>
      <default>*</default>
      <initialValue>*</initialValue>
    </input>
  </fieldset>

  <row>
    <panel>
      <single>
        <title>Pending</title>
        <search>
          <query>| inputlookup flux_pending_review_lookup | stats count</query>
          <refresh>30s</refresh>
        </search>
        <option name="colorBy">value</option>
        <option name="rangeColors">["0x55AA55","0xDD6644"]</option>
        <option name="rangeValues">[0]</option>
      </single>
    </panel>
    <panel>
      <single>
        <title>Confirmed</title>
        <search>
          <query>| inputlookup flux_classifications_lookup | stats count</query>
          <refresh>30s</refresh>
        </search>
      </single>
    </panel>
    <panel>
      <single>
        <title>Last prediction run</title>
        <search>
          <query>| inputlookup flux_pending_review_lookup | stats max(predicted_at) as last | eval last=strftime(last, "%Y-%m-%d %H:%M")</query>
          <refresh>30s</refresh>
        </search>
      </single>
    </panel>
  </row>

  <row>
    <panel>
      <title>Pending predictions</title>
      <table>
        <search>
          <query>
            | inputlookup flux_pending_review_lookup
            | where predicted_probability >= tonumber("$tok_min_conf$")
            | search predicted_role="$tok_role$" dest="$tok_search$"
            | sort - predicted_probability
            | table dest predicted_role predicted_probability top3_roles observed_ports predicted_at
            | eval predicted_at = strftime(predicted_at, "%Y-%m-%d %H:%M")
          </query>
          <refresh>30s</refresh>
        </search>
        <option name="drilldown">row</option>
        <option name="count">25</option>
        <drilldown>
          <set token="tok_selected_dest">$row.dest$</set>
          <set token="tok_selected_role">$row.predicted_role$</set>
          <set token="tok_selected_prob">$row.predicted_probability$</set>
          <set token="tok_selected_top3">$row.top3_roles$</set>
          <set token="tok_selected_ports">$row.observed_ports$</set>
        </drilldown>
      </table>
    </panel>
  </row>

  <row depends="$tok_selected_dest$">
    <panel>
      <title>Review: $tok_selected_dest$</title>
      <html>
        <h3>$tok_selected_dest$</h3>
        <p>
          <strong>Predicted:</strong> $tok_selected_role$
          (confidence $tok_selected_prob$)
        </p>
        <p>
          <strong>Class probabilities (all):</strong>
          <code>$tok_selected_top3$</code>
        </p>
        <p>
          <strong>Observed ports:</strong>
          <code>$tok_selected_ports$</code>
        </p>
      </html>
    </panel>
  </row>

  <row depends="$tok_selected_dest$">
    <panel>
      <title>Confirm or reject</title>
      <input type="dropdown" token="tok_role_override">
        <label>Confirm as role</label>
        <fieldForLabel>asset_role</fieldForLabel>
        <fieldForValue>asset_role</fieldForValue>
        <search>
          <query>| inputlookup flux_classifications_lookup | stats count by asset_role | sort asset_role</query>
        </search>
        <default>$tok_selected_role$</default>
      </input>
      <input type="text" token="tok_notes">
        <label>Notes (optional)</label>
        <default></default>
      </input>
      <input type="link" token="tok_confirm_action">
        <label>Action</label>
        <choice value="confirm">Confirm</choice>
        <choice value="reject">Reject</choice>
        <change>
          <condition value="confirm">
            <set token="tok_run_confirm">1</set>
            <unset token="tok_run_reject"/>
          </condition>
          <condition value="reject">
            <set token="tok_run_reject">1</set>
            <unset token="tok_run_confirm"/>
          </condition>
        </change>
      </input>
    </panel>
  </row>

  <row depends="$tok_run_confirm$">
    <panel>
      <title>Confirming…</title>
      <table>
        <search>
          <query>
            | savedsearch flux_confirm_prediction
                dest="$tok_selected_dest$"
                role="$tok_role_override$"
                user="$env:user$"
                notes="$tok_notes$"
          </query>
          <done>
            <unset token="tok_selected_dest"/>
            <unset token="tok_run_confirm"/>
          </done>
        </search>
      </table>
    </panel>
  </row>

  <row depends="$tok_run_reject$">
    <panel>
      <title>Rejecting…</title>
      <table>
        <search>
          <query>
            | savedsearch flux_reject_prediction dest="$tok_selected_dest$"
          </query>
          <done>
            <unset token="tok_selected_dest"/>
            <unset token="tok_run_reject"/>
          </done>
        </search>
      </table>
    </panel>
  </row>
</form>
```

Notes for the implementer:

- The `<row depends="$tok_selected_dest$">` blocks render only after a row is clicked.
- The action runs by setting `tok_run_confirm` or `tok_run_reject`. The hidden execution panels chain to the saved searches and clear tokens on completion (causing the review panel and pending table to refresh).
- `$env:user$` resolves to the logged-in user — passed to `flux_confirm_prediction` for `confirmed_by`.

- [ ] **Step 2: Commit**

```bash
git add package/default/data/ui/views/flux_asset_review.xml
git commit -m "feat: add flux_asset_review dashboard with row-level confirm/reject"
```

---

### Task 16: User-facing README

**Files:**
- Create: `package/README.md`

- [ ] **Step 1: Write `package/README.md`**

````markdown
# TA-flux — Asset Role Classifier

Splunk Cloud-vetted add-on that classifies destinations from firewall logs by
the combination of ports they accept traffic on. Trains a RandomForest model
nightly, queues predictions for human review, and grows the labeled corpus
over time as you confirm or override predictions.

## What it does

1. Builds a binary `port_*` feature matrix per destination from the
   `Network_Traffic` data model.
2. Trains a RandomForestClassifier (MLTK) once per week from confirmed labels.
3. Predicts asset roles for unconfirmed hosts every night, writes them to a
   review queue.
4. Provides a dashboard for human reviewers to confirm, override, or reject
   predictions. Confirmations feed back into the next training run.

## Requirements

- Splunk Enterprise 9.0+ or Splunk Cloud.
- Splunk Machine Learning Toolkit (MLTK) 5.4+ installed.
- The `Network_Traffic` CIM data model populated and accelerated.
- `admin` or `power` role to install and to modify confirmations.

## Install

1. Install **Splunk Machine Learning Toolkit** from Splunkbase.
2. Install this add-on (`TA-flux.spl`).
3. Edit `flux_internal_dest_filter` in `default/macros.conf` (or via Settings →
   Advanced search → Search macros) to scope to your internal hosts. Examples:

   ```
   # IP-based
   where cidrmatch("10.0.0.0/8", dest)
      OR cidrmatch("172.16.0.0/12", dest)
      OR cidrmatch("192.168.0.0/16", dest)

   # Hostname-based
   where match(dest, "\.crimsonia\.net$")
   ```

4. Verify the seed lookup loaded:

   ```
   | inputlookup classified_hosts_seed | stats count
   ```

5. Run the bootstrap once. From Settings → Searches, reports, and alerts,
   enable `flux_bootstrap_classifications`, run it, then disable it again.
   Verify:

   ```
   | inputlookup flux_classifications_lookup | stats count by source
   ```

6. Run the training search once manually:

   ```
   | savedsearch flux_train_model
   ```

   Verify:

   ```
   | summary flux_asset_classifier
   ```

7. Run the daily classification once manually:

   ```
   | savedsearch flux_classify_assets
   ```

   Open the **TA-flux** app — the review dashboard should be populated.

8. The schedules on `flux_train_model` (weekly, Sundays 02:00 UTC) and
   `flux_classify_assets` (daily, 03:00 UTC) are enabled by default. Adjust
   timing to your environment's traffic patterns.

## Using the review dashboard

Open the **TA-flux Asset Classifier** app — the review dashboard is the
landing page.

- The pending table lists predictions awaiting confirmation.
- Filter by predicted role, minimum confidence, or `dest` substring.
- Click a row to populate the **Review** panel below.
- Optionally override the role from the dropdown, add a note, then click
  **Confirm** or **Reject**.
- The pending table auto-refreshes; confirmed rows disappear from it.

## When to retrain manually

The weekly schedule is the baseline. Retrain extra in three cases:

1. After modifying `odin_classify_ports.csv` (added or recategorized ports).
2. After confirming hosts under a new `asset_role` value.
3. After confirming a noticeably-wrong batch of predictions.

To retrain:

```
| savedsearch flux_train_model
```

## Field reference

`flux_classifications` (KV Store) — confirmed labels:

| field | type | notes |
|---|---|---|
| `dest` | string | host identifier; matches `tstats` output |
| `asset_role` | string | the label |
| `confirmed_by` | string | username, or `seed` for bootstrap rows |
| `confirmed_at` | epoch number | when confirmed |
| `source` | string | `seed` \| `confirmed` \| `manual_override` |
| `notes` | string | reviewer comment |

`flux_pending_review` (KV Store) — predictions awaiting review:

| field | type | notes |
|---|---|---|
| `dest` | string | |
| `predicted_role` | string | |
| `predicted_probability` | number | 0–1 |
| `top3_roles` | string | pipe-separated `role:prob` for all classes |
| `observed_ports` | string | comma-separated list |
| `predicted_at` | epoch number | |

## Useful introspection searches

- Confirmation activity by source:

  ```
  | inputlookup flux_classifications_lookup | timechart count by source
  ```

- Pending count over time (run as a saved search and chart):

  ```
  | inputlookup flux_pending_review_lookup | stats count
  ```

- Model freshness:

  ```
  | summary flux_asset_classifier
  ```

- Saved-search run history:

  ```
  | rest /services/saved/searches/flux_train_model/history
  ```

## Troubleshooting

**The model doesn't exist (`apply` fails).** Run `flux_train_model` manually
once, verify with `| summary flux_asset_classifier`. If training fails, check
that you have at least 5–10 labeled rows per role in `flux_classifications`.

**Pending queue is empty after running classify.** Either every host is
already confirmed (good), or the feature pivot returned no rows (check that
`flux_internal_dest_filter` isn't too restrictive and that
`Network_Traffic.All_Traffic` is accelerated).

**Schema-mismatch errors at apply time.** The port lookup grew between train
and apply, so the model has fewer feature columns than the current matrix.
Retrain: `| savedsearch flux_train_model`.

## Limitations (v1)

- No bulk confirm — review one row at a time.
- No UI for editing already-confirmed labels (edit the KV Store directly via
  SPL or the Lookup Editor).
- New `asset_role` values must come via the seed CSV or by direct write to
  `flux_classifications`.
- No "Retrain now" dashboard button (run the saved search manually).
````

- [ ] **Step 2: Commit**

```bash
git add package/README.md
git commit -m "docs: add user-facing README with install and usage"
```

---

### Task 17: AppInspect validation

**Files:** none modified — validation only.

- [ ] **Step 1: Run AppInspect against the package**

```bash
splunk-appinspect inspect package/ --mode test --included-tags cloud --output-file appinspect-report.txt
```

- [ ] **Step 2: Read the report**

```bash
cat appinspect-report.txt | head -80
```

Expected: zero failures, zero errors. Warnings are acceptable for v1 (review them but don't block).

If failures are reported:

- **`check_for_app_dependencies_in_default_inputs_conf`** — irrelevant; we don't ship inputs.conf.
- **`check_validate_no_default_local_directories`** — make sure no `local/` directory got committed.
- **`check_collections_conf_definition`** — re-validate the field types in `collections.conf`.
- **`check_macros_conf_use_args_correctly`** — `flux_feature_pivot(2)` must declare `args = earliest_arg, latest_arg`.

Fix in place, re-run, until clean.

- [ ] **Step 3: Commit any fixes**

```bash
git add package/
git commit -m "fix: address AppInspect findings"
```

(Skip this commit if no fixes needed.)

---

### Task 18: Build the .spl package

**Files:**
- Create: `TA-flux.spl` (build artifact, not committed)

- [ ] **Step 1: Build the package**

```bash
tar -czvf TA-flux.spl --exclude='.DS_Store' package/
```

- [ ] **Step 2: Verify the archive**

```bash
tar -tzvf TA-flux.spl | head -20
```

Expected: file listing rooted at `package/` containing `app.manifest`, `default/`, `lookups/`, `metadata/`, `README.md`.

- [ ] **Step 3: Final AppInspect against the .spl**

```bash
splunk-appinspect inspect TA-flux.spl --mode test --included-tags cloud
```

Expected: same clean result as Task 17.

- [ ] **Step 4: Tag the release**

```bash
git tag -a v0.1.0 -m "Initial release"
```

(Don't push the tag without explicit user confirmation.)

---

## Spec coverage map

| Spec section | Tasks |
|---|---|
| Add-on directory layout | 1 |
| `app.conf`, `app.manifest` | 2 |
| Permissions (`default.meta`) | 3 |
| KV Store schemas | 4 |
| Lookup definitions | 5 |
| `flux_internal_dest_filter` macro | 6 |
| `flux_model_name` macro | 7 |
| `flux_feature_pivot` macro | 8 |
| `flux_bootstrap_classifications` | 9 |
| `flux_train_model` | 10 |
| `flux_classify_assets` | 11 |
| `flux_confirm_prediction` | 12 |
| `flux_reject_prediction` | 13 |
| Navigation | 14 |
| Review dashboard | 15 |
| User README (operations, install, troubleshooting) | 16 |
| AppInspect / Cloud-vetting | 17 |
| Package build | 18 |

All spec sections covered. The `top3_roles` semantic divergence is documented at the top of this plan.
