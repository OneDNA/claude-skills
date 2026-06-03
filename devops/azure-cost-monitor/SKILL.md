---
name: azure-cost-monitor
description: >
  Azure cost monitor: analyze subscription costs, detect idle Container Apps,
  apply scale-to-zero via Bicep params, and generate a markdown cost report.
  Use monthly or after infrastructure changes. Never installs plugins or MCP tools.
tools: Bash, Read, Edit, Write
---

# Azure Cost Monitor

Analyze Azure costs, detect idle resources, apply scale-to-zero, and generate
a cost report. Use only `az` CLI — never install plugins or MCP tools.

---

## Constraints (read before starting)

- **NEVER set `max-replicas` to 0** — Azure Container Apps does not allow it. Always use `min-replicas: 0` / `max-replicas: 1` for scale-to-zero.
- **NEVER use imperative `az` CLI for persistent infrastructure changes** — update Bicep parameter files instead.
- **NEVER install plugins, MCP tools, or any tooling** — use only what is already on PATH.
- Resource group naming: `dna-${environment}-portfolio-${location}-rg`
- Get subscription ID: `az account show --query id -o tsv`

---

## Step 1 — Authenticate and set context

```bash
az account show --query "{name:name, id:id, tenantId:tenantId}" -o table
```

If not logged in: `az login`

---

## Step 2 — Get last 7-day costs by resource

```bash
SUBSCRIPTION=$(az account show --query id -o tsv)
START=$(date -d "-7 days" +%Y-%m-%d 2>/dev/null || date -v-7d +%Y-%m-%d)
END=$(date +%Y-%m-%d)

az consumption usage list \
  --subscription "$SUBSCRIPTION" \
  --start-date "$START" \
  --end-date "$END" \
  --query "[].{resource:instanceName, cost:pretaxCost, currency:currency}" \
  -o table 2>/dev/null || echo "consumption usage not available — check billing permissions"
```

If `az consumption` is unavailable, try Cost Management:

```bash
az costmanagement query \
  --type Usage \
  --scope "subscriptions/$SUBSCRIPTION" \
  --timeframe Custom \
  --time-period from="$START" to="$END" \
  --dataset-aggregation '{"totalCost":{"name":"PreTaxCost","function":"Sum"}}' \
  --dataset-grouping '[{"type":"Dimension","name":"ResourceGroupName"}]' \
  -o table
```

---

## Step 3 — Check Container App activity (last 48 hours)

For each Container App in both resource groups:

```bash
for RG in "dna-dev-portfolio-we-rg" "dna-prd-portfolio-we-rg"; do
  echo "=== $RG ==="
  az containerapp list --resource-group "$RG" --query "[].{name:name, minReplicas:properties.template.scale.minReplicas, maxReplicas:properties.template.scale.maxReplicas}" -o table 2>/dev/null || echo "RG not found"
done
```

Check request metrics for each app (replace `<app-name>` and `<rg>`):

```bash
az monitor metrics list \
  --resource "$(az containerapp show -n <app-name> -g <rg> --query id -o tsv)" \
  --metric "Requests" \
  --start-time "$(date -u -d '-48 hours' +%Y-%m-%dT%H:%MZ 2>/dev/null || date -u -v-48H +%Y-%m-%dT%H:%MZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%MZ)" \
  --interval PT1H \
  --aggregation Total \
  --query "value[0].timeseries[0].data[*].total" \
  -o tsv | awk '{s+=$1} END {print "Total requests:", s}'
```

If total requests = 0 → container is idle.

---

## Step 4 — Apply scale-to-zero for idle apps (via Bicep)

Do **NOT** run `az containerapp update` — update the Bicep parameter file instead.

For idle dev apps, edit `infra/parameters.dev.bicepparam`:

```
# Ensure scale-to-zero is configured:
# minReplicas: 0
# maxReplicas: 1   ← NEVER set this to 0
```

For idle prod apps (proceed cautiously — confirm with user first):

Edit `infra/parameters.prod.bicepparam` with the same values.

After editing params, note: changes take effect on next infrastructure deployment — do not deploy immediately without user approval.

---

## Step 5 — Generate cost report

Create the report at `docs/reports/azure-cost-YYYY-MM-DD.md`:

```bash
REPORT_DATE=$(date +%Y-%m-%d)
REPORT_PATH="docs/reports/azure-cost-$REPORT_DATE.md"
mkdir -p docs/reports
```

Write the report using this template:

```markdown
# Azure Cost Report — YYYY-MM-DD

## Summary

- **Period**: last 7 days
- **Subscription**: <name>

## Resource Costs

| Resource | 7-day cost | Currency |
| -------- | ---------- | -------- |
| ...      | ...        | EUR      |

## Container App Activity (last 48h)

| App                        | Resource Group          | Requests | min/max Replicas | Action taken                |
| -------------------------- | ----------------------- | -------- | ---------------- | --------------------------- |
| dna-dev-portfolio-we-ca-fe | dna-dev-portfolio-we-rg | 0        | 0/1              | scale-to-zero (already set) |
| dna-prd-portfolio-we-ca-fe | dna-prd-portfolio-we-rg | 142      | 0/1              | no change                   |

## Recommendations

- <list any cost optimization recommendations>

## Notes

- Scale-to-zero changes require a Bicep redeploy to take effect.
- Never set max-replicas below 1.
```

---

## Step 6 — Commit and open PR

```bash
git checkout -b "chore/azure-cost-report-$REPORT_DATE"
git add docs/reports/azure-cost-$REPORT_DATE.md
git add infra/parameters.*.bicepparam   # only if params were changed
git commit -m "chore(infra): azure cost report $REPORT_DATE

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git push -u origin HEAD
gh pr create --title "chore(infra): azure cost report $REPORT_DATE" \
  --body "Monthly Azure cost analysis and scale-to-zero review. See \`docs/reports/azure-cost-$REPORT_DATE.md\`."
```
