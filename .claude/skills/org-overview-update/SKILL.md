---
name: org-overview-update
description: Weekly update for the Org Overview tab (Tab 5) of the dashboard at /Users/craghav/Downloads/KPI_Dashboard.html. Needs only the KPI CSV. Recomputes all 10 Org Overview data constants — L6 weekly, Manager weekly, sub-cloud split, and Signature P&T breakdowns — then patches the HTML and pushes to GitHub Pages.
tools: Bash, Read, Edit, Write
---

# Org Overview Update Skill

Updates the **Org Overview tab** data. Use weekly after getting the KPI CSV export.

## Dashboard file
`/Users/craghav/Downloads/KPI_Dashboard.html`

## What to provide

```
[ ] KPI CSV export — UTF-8, comma-delimited
    Required columns: Metric, FISCAL_DATE, Level 6 Manager, Manager Nm,
                      Product Topic Name, Success Plan, SUM_METRIC

[ ] New week start date (Sunday) in YYYY-MM-DD format
    e.g. 2026-09-07
```

The same CSV used for the KPI Trends tab update works here — it's the same file.

---

## Step 1 — Save CSV and compute all constants

Ask the user to paste/share the CSV path, save to temp dir, then run:

```python
import csv, json, re
from collections import defaultdict
from datetime import datetime, timedelta

CSV_PATH  = "/Users/craghav/.claude/jobs/<job-id>/tmp/kpi_new.csv"
NEW_WEEK  = "YYYY-MM-DD"   # replace with user-provided date

# Read current 4 weeks from the dashboard before running
# grep for ORG_L6_WEEKLY to find the existing weeks:
#   grep -o '"20[0-9][0-9]-[0-9][0-9]-[0-9][0-9]"' /Users/craghav/Downloads/KPI_Dashboard.html | \
#     grep -A4 -B4 'ORG_L6' | head -8
EXISTING_WEEKS = ["2026-08-02","2026-08-09","2026-08-16","2026-08-23"]  # update from dashboard

# 4-week rolling window: drop oldest, add new
ALL_WEEKS = EXISTING_WEEKS[1:] + [NEW_WEEK]   # drop first (oldest), keep 3 + new = 4

SIG_PLANS = {"Signature","Signature Success","Gov Signature Success","US Signature Success"}

PT_CLOUD_MAP = {
  "Industry-Financial Services":"Industry","Revenue Cloud (Core)-Price Management":"Revenue",
  "Industry-Life Sciences":"Industry","Revenue-Salesforce CPQ":"Revenue",
  "Revenue Cloud (Core)-Advanced Approvals":"Revenue","Industry-Omnistudio":"Industry",
  "Revenue Cloud (Core)-Product Configurator":"Revenue","Industry-Education Cloud":"Industry",
  "Industry-Public Sector Solutions":"Industry","Industry-Health Cloud":"Industry",
  "Revenue Cloud (Core)-Transaction Management":"Revenue","Revenue-CPQ Developer Support":"Revenue",
  "Industry-Nonprofit Success Pack (NPSP)":"Industry","Revenue Cloud (Core)-Contract Lifecycle Management with DocGen":"Revenue",
  "Industry-Retail and Consumer Goods":"Industry","Revenue Cloud (Core)-Billing & Invoicing":"Revenue",
  "Industry-CPQ / Order Management / Digital Commerce":"Industry","Revenue Cloud (Core)-Product Catalog Management":"Revenue",
  "Industry-Mandatory Security Controls":"Industry","Industry-Nonprofit Cloud":"Industry",
  "Revenue-Document Generation":"Revenue","Revenue-Salesforce Billing":"Revenue",
  "Industry-Loyalty Management":"Industry","Industry-Energy & Utilities Cloud":"Industry",
  "Revenue Cloud (Core)-Usage & Ratings":"Revenue","Industry-Document Generation":"Industry",
  "Industry-Security & Activations":"Industry","Revenue Cloud (Core)-Developer Support - Product to Order":"Revenue",
  "Industry-Developer Support":"Industry","Revenue Cloud (Core)-Dynamic Revenue Orchestration":"Revenue",
  "Revenue-Billing Developer Support":"Revenue","Revenue-Salesforce Subscription Management":"Revenue",
  "Industry-Communication Cloud":"Industry","Industry-Media Cloud":"Industry",
  "Industry-Health & Insurance":"Industry","Industry-Email Delivery & Desktop Integrations":"Industry",
  "Industry-Manufacturing Cloud":"Industry","Industry-Automotive Cloud":"Industry",
  "Industry-CRM Analytics":"Industry","Industry-Business Rules Engine (BRE)":"Industry",
  "Industry-Net Zero Cloud":"Industry","Revenue-Salesforce Contracts":"Revenue",
  "Industry-Nonprofit Packages (Other SFDO)":"Industry","Revenue Cloud (Core)-Revenue Cloud Analytics":"Revenue",
  "Revenue Cloud (Core)-OmniStudio":"Revenue","Revenue Cloud (Core)-Developer Support - Invoice to Cash":"Revenue",
  "Industry-How-to, Setup, Configuration, Reports & Dashboards":"Industry",
}

with open(CSV_PATH, encoding='utf-8') as f:
    rows = list(csv.DictReader(f, delimiter=','))

def parse_week(s):
    try:
        dt = datetime.strptime(s.strip()[:10], '%m/%d/%Y')
        return (dt - timedelta(days=(dt.weekday()+1)%7)).strftime('%Y-%m-%d')
    except: return None

# ── Accumulators ──────────────────────────────────────────────────────────────
def ddd(): return defaultdict(lambda: defaultdict(float))
def ddi(): return defaultdict(lambda: defaultdict(int))

l6_inflow   = ddi(); l6_closures  = ddi(); l6_esc      = ddi()
l6_ttr_sum  = ddd(); l6_ttr_cnt   = ddi()
l6_csat_sum = ddd(); l6_csat_cnt  = ddi()

mgr_inflow   = ddi(); mgr_closures  = ddi(); mgr_esc      = ddi()
mgr_ttr_sum  = ddd(); mgr_ttr_cnt   = ddi()
mgr_csat_sum = ddd(); mgr_csat_cnt  = ddi()

l6_cloud   = ddi()   # [wk][l6][cloud] = count
mgr_cloud  = ddi()   # [wk][mgr][cloud] = count
l6_pt_sig  = ddi()   # [wk][l6][pt] = count
mgr_pt_sig = ddi()   # [wk][mgr][pt] = count

for r in rows:
    wk  = parse_week(r.get('FISCAL_DATE',''))
    if not wk or wk not in ALL_WEEKS: continue
    l6  = r.get('Level 6 Manager','').strip()
    mgr = r.get('Manager Nm','').strip()
    metric = r.get('Metric','').strip()
    pt  = r.get('Product Topic Name','').strip()
    plan = r.get('Success Plan','').strip()
    try:
        val = float(r.get('SUM_METRIC','') or 0)
    except:
        val = 0.0

    if metric == 'Inflow':
        cloud = PT_CLOUD_MAP.get(pt, 'Industry')
        if l6:
            l6_inflow[wk][l6] += 1
            l6_cloud[wk][l6][cloud] += 1
            if plan in SIG_PLANS and pt:
                l6_pt_sig[wk][l6][pt] += 1
        if mgr and mgr != l6:
            mgr_inflow[wk][mgr] += 1
            mgr_cloud[wk][mgr][cloud] += 1
            if plan in SIG_PLANS and pt:
                mgr_pt_sig[wk][mgr][pt] += 1

    elif metric == 'Closures/TTR':
        if l6:
            l6_closures[wk][l6] += 1
            if val > 0:
                l6_ttr_sum[wk][l6] += val
                l6_ttr_cnt[wk][l6] += 1
        if mgr and mgr != l6:
            mgr_closures[wk][mgr] += 1
            if val > 0:
                mgr_ttr_sum[wk][mgr] += val
                mgr_ttr_cnt[wk][mgr] += 1

    elif metric == 'CSAT':
        if l6 and val > 0:
            l6_csat_sum[wk][l6] += val
            l6_csat_cnt[wk][l6] += 1
        if mgr and mgr != l6 and val > 0:
            mgr_csat_sum[wk][mgr] += val
            mgr_csat_cnt[wk][mgr] += 1

    elif metric == 'Escalations':
        if l6:
            l6_esc[wk][l6] += 1
        if mgr and mgr != l6:
            mgr_esc[wk][mgr] += 1

def avg(s, c): return round(s/c, 2) if c else 0.0

# ── Build ORG_L6_WEEKLY ───────────────────────────────────────────────────────
ORG_L6_WEEKLY = {}
for wk in ALL_WEEKS:
    ORG_L6_WEEKLY[wk] = {}
    for l6 in set(list(l6_inflow.get(wk,{}).keys())+list(l6_closures.get(wk,{}).keys())):
        ORG_L6_WEEKLY[wk][l6] = {
            "inflow":      l6_inflow[wk].get(l6,0),
            "closures":    l6_closures[wk].get(l6,0),
            "ttr":         avg(l6_ttr_sum[wk].get(l6,0), l6_ttr_cnt[wk].get(l6,0)),
            "csat":        avg(l6_csat_sum[wk].get(l6,0), l6_csat_cnt[wk].get(l6,0)),
            "escalations": l6_esc[wk].get(l6,0),
        }

# ── Build ORG_MGR_WEEKLY ─────────────────────────────────────────────────────
ORG_MGR_WEEKLY = {}
for wk in ALL_WEEKS:
    ORG_MGR_WEEKLY[wk] = {}
    for mgr in set(list(mgr_inflow.get(wk,{}).keys())+list(mgr_closures.get(wk,{}).keys())):
        ORG_MGR_WEEKLY[wk][mgr] = {
            "inflow":      mgr_inflow[wk].get(mgr,0),
            "closures":    mgr_closures[wk].get(mgr,0),
            "ttr":         avg(mgr_ttr_sum[wk].get(mgr,0), mgr_ttr_cnt[wk].get(mgr,0)),
            "csat":        avg(mgr_csat_sum[wk].get(mgr,0), mgr_csat_cnt[wk].get(mgr,0)),
            "escalations": mgr_esc[wk].get(mgr,0),
        }

# ── Build cloud / sig PT weekly dicts ────────────────────────────────────────
def to_plain(d):
    if isinstance(d, defaultdict):
        return {k: to_plain(v) for k,v in d.items()}
    return d

ORG_L6_CLOUD_WEEKLY   = {wk: to_plain(l6_cloud[wk])  for wk in ALL_WEEKS}
ORG_MGR_CLOUD_WEEKLY  = {wk: to_plain(mgr_cloud[wk]) for wk in ALL_WEEKS}
ORG_L6_PT_SIG_WEEKLY  = {wk: to_plain(l6_pt_sig[wk]) for wk in ALL_WEEKS}
ORG_MGR_PT_SIG_WEEKLY = {wk: to_plain(mgr_pt_sig[wk]) for wk in ALL_WEEKS}

# ── Build rolling totals (all 4 weeks) ───────────────────────────────────────
combined_pt  = defaultdict(int)
l6_pt_totals = defaultdict(lambda: defaultdict(int))
mgr_pt_totals = defaultdict(lambda: defaultdict(int))

for wk in ALL_WEEKS:
    for l6, pts in l6_pt_sig[wk].items():
        for pt, cnt in pts.items():
            l6_pt_totals[l6][pt] += cnt
            combined_pt[pt] += cnt
    for mgr, pts in mgr_pt_sig[wk].items():
        for pt, cnt in pts.items():
            mgr_pt_totals[mgr][pt] += cnt

ORG_COMBINED_PT_TOTALS = dict(combined_pt)
ORG_L6_PT_SIG_TOTALS   = to_plain(l6_pt_totals)
ORG_MGR_PT_TOTALS      = to_plain(mgr_pt_totals)

# ── Output summary ────────────────────────────────────────────────────────────
print(f"=== New week: {NEW_WEEK} ===")
print(f"L6 data: {json.dumps(ORG_L6_WEEKLY.get(NEW_WEEK,{}), indent=2)}")
print(f"\nTop managers (new week inflow):")
for mgr, d in sorted(ORG_MGR_WEEKLY.get(NEW_WEEK,{}).items(), key=lambda x:-x[1]['inflow'])[:5]:
    print(f"  {mgr}: inflow={d['inflow']} closures={d['closures']} ttr={d['ttr']} csat={d['csat']}")
print(f"\nTop 5 combined P&T Sig totals:")
for pt, cnt in sorted(ORG_COMBINED_PT_TOTALS.items(), key=lambda x:-x[1])[:5]:
    print(f"  {pt}: {cnt}")
print(f"\nRolling weeks: {ALL_WEEKS}")

# ── Save all constants for Step 2 ────────────────────────────────────────────
out = {
    "ALL_WEEKS": ALL_WEEKS,
    "ORG_L6_WEEKLY":       ORG_L6_WEEKLY,
    "ORG_MGR_WEEKLY":      ORG_MGR_WEEKLY,
    "ORG_L6_CLOUD_WEEKLY": ORG_L6_CLOUD_WEEKLY,
    "ORG_MGR_CLOUD_WEEKLY":ORG_MGR_CLOUD_WEEKLY,
    "ORG_L6_PT_SIG_WEEKLY": ORG_L6_PT_SIG_WEEKLY,
    "ORG_MGR_PT_SIG_WEEKLY":ORG_MGR_PT_SIG_WEEKLY,
    "ORG_COMBINED_PT_TOTALS":ORG_COMBINED_PT_TOTALS,
    "ORG_L6_PT_SIG_TOTALS": ORG_L6_PT_SIG_TOTALS,
    "ORG_MGR_PT_TOTALS":    ORG_MGR_PT_TOTALS,
}
with open('/Users/craghav/.claude/jobs/<job-id>/tmp/org_update_data.json','w') as f:
    json.dump(out, f, separators=(',',':'))
print("\nSaved org_update_data.json")
```

Verify the output looks right — especially `ORG_L6_WEEKLY[NEW_WEEK]` values for Taher Jamboo and Abdellah Sabere.

---

## Step 2 — Patch the HTML file

Write and run a Python patch script:

```python
import json, re

HTML_PATH = "/Users/craghav/Downloads/KPI_Dashboard.html"
DATA_PATH = "/Users/craghav/.claude/jobs/<job-id>/tmp/org_update_data.json"

with open(HTML_PATH, encoding='utf-8') as f:
    src = f.read()
with open(DATA_PATH) as f:
    d = json.load(f)

def replace_const(src, const_name, new_value):
    """Replace the entire JS constant value using regex."""
    pattern = r'(const ' + re.escape(const_name) + r'\s*=\s*)(\{[^;]+\});'
    new_js = json.dumps(new_value, separators=(',',':'))
    replacement = r'\g<1>' + new_js + ';'
    result, count = re.subn(pattern, replacement, src, count=1, flags=re.DOTALL)
    assert count == 1, f"{const_name}: found {count} matches"
    return result

# Replace all 10 Org Overview constants
for const in [
    'ORG_L6_WEEKLY', 'ORG_MGR_WEEKLY',
    'ORG_L6_CLOUD_WEEKLY', 'ORG_MGR_CLOUD_WEEKLY',
    'ORG_L6_PT_SIG_WEEKLY', 'ORG_MGR_PT_SIG_WEEKLY',
    'ORG_COMBINED_PT_TOTALS', 'ORG_L6_PT_SIG_TOTALS', 'ORG_MGR_PT_TOTALS',
]:
    src = replace_const(src, const, d[const])
    print(f"  Updated {const}")

with open(HTML_PATH, 'w', encoding='utf-8') as f:
    f.write(src)
print(f"Done. File size: {len(src):,} bytes")
```

> **Note**: `ORG_MGR_L6` (the manager→L6 mapping) is static — no need to update it unless the org structure changes. Update it manually if managers join/leave.

---

## Step 3 — Sync and push

```bash
cp /Users/craghav/Downloads/KPI_Dashboard.html /Users/craghav/orgcs-dashboard-data/kpi_dashboard.html
cd /Users/craghav/orgcs-dashboard-data
git add kpi_dashboard.html
git commit -m "Org Overview update Wk MM/DD"
git push origin main
```

---

## Verification checklist

After pushing, open the dashboard → **Org Overview** tab (allow ~30s for GitHub Pages):

- [ ] KPI tiles show the new week's Inflow / Closures for both L6 managers
- [ ] WoW arrows (▲/▼) on Inflow and Closures point the right direction
- [ ] Inflow, Closure & SLO Miss chart has a new data point on the right
- [ ] CSAT, Escalations & TTR chart updated
- [ ] Sub Cloud chart shows new week for Industry vs Revenue
- [ ] Sig Industry P&T chart has new rightmost point for each series
- [ ] Sig Revenue P&T chart has new rightmost point for each series
- [ ] Switch to "Taher Jamboo" filter — tiles and charts show Taher's data only
- [ ] Switch to "Abdellah Sabere" filter — tiles and charts show Abdellah's data only
- [ ] Manager dropdown works under each L6 filter

---

## Key notes

- **Week boundaries**: Sun–Sat. `FISCAL_DATE` is `m/d/yyyy` → roll back to Sunday with `(weekday+1)%7`.
- **CSAT**: average of `SUM_METRIC` where `Metric == 'CSAT'` and val > 0. Rows with no CSAT score are excluded.
- **TTR**: average of `SUM_METRIC` where `Metric == 'Closures/TTR'` and val > 0.
- **Closures**: row count where `Metric == 'Closures/TTR'` (not the SUM_METRIC value).
- **Sig plans**: `{"Signature", "Signature Success", "Gov Signature Success", "US Signature Success"}` — matches the `Success Plan` column.
- **Cloud mapping**: use `PT_CLOUD_MAP` above, not any CSV column — the mapping is curated.
- **Rolling 4-week window**: always exactly 4 weeks. `ALL_WEEKS = EXISTING_WEEKS[1:] + [NEW_WEEK]`.
- **Totals constants** (`ORG_COMBINED_PT_TOTALS`, `ORG_L6_PT_SIG_TOTALS`, `ORG_MGR_PT_TOTALS`): recomputed across all 4 rolling weeks, not just the new week.
- **Manager list** (`ORG_MGR_L6`): static mapping, not recomputed from CSV. Edit manually if org changes:
  - Taher Jamboo: Angad Rajpal Singh, Chaten Raghav, Farbeena Hussain, Pushkar Dwivedi, Rashi Saraswat, Wahaj Ali Shaik
  - Abdellah Sabere: Adam Dillman, Eugene Pugach, Sam Faulkner, Sreenivas Voore, Victor Soto
- **CSV encoding**: UTF-8, comma-delimited (NOT UTF-16 TSV like the KPI Trends CSV — they differ).
- If the CSV is UTF-16 TSV, use `encoding='utf-16'` and `delimiter='\t'` instead.
