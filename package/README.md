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

**Scheduled `flux_train_model` failed with "no events".** No labeled hosts in
`flux_classifications` yet. Run the bootstrap (`flux_bootstrap_classifications`)
to load the seed labels, then re-run `flux_train_model` manually.

## Limitations (v1)

- No bulk confirm — review one row at a time.
- No UI for editing already-confirmed labels (edit the KV Store directly via
  SPL or the Lookup Editor).
- New `asset_role` values must come via the seed CSV or by direct write to
  `flux_classifications`.
- No "Retrain now" dashboard button (run the saved search manually).
