# TA-flux

Splunk Cloud-vetted add-on that classifies asset roles from firewall traffic
port patterns using MLTK LogisticRegression.

User-facing documentation: [package/README.md](package/README.md)

Design spec: [docs/superpowers/specs/2026-04-29-ta-flux-asset-classifier-design.md](docs/superpowers/specs/2026-04-29-ta-flux-asset-classifier-design.md)

Implementation plan: [docs/superpowers/plans/2026-04-30-ta-flux-asset-classifier-plan.md](docs/superpowers/plans/2026-04-30-ta-flux-asset-classifier-plan.md)

## Building the .spl

    tar -czvf TA-flux.spl --exclude='.DS_Store' package/

## Validating before submission

    splunk-appinspect inspect package/ --mode test --included-tags cloud
