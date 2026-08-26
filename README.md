# Merck Pipeline Observability (prototype)

Static dashboard built from Acceldata ADM snapshot data (26 Aug 2026, 04:30 UTC). It does **not** query ADM while you use it.

## Public URL

Share this with Merck (root path, same style as the Mantis demo):

**https://merck-pipeline-observability.vercel.app/**

Data lineage: https://merck-pipeline-observability.vercel.app/data-lineage.html

GitHub Pages (backup): https://ad-krish.github.io/merck-pipeline-observability/

Repo: https://github.com/ad-krish/merck-pipeline-observability

## How GitHub is used

GitHub stores this folder as a **public repository**. **GitHub Pages** serves `index.html` at the root URL.

## Update the live site

Edit **`index.html`** (that is the dashboard). Then:

```bash
cd /Users/krish/Downloads/Customers/Merck/Pipeline_Dashboard
git add -A
git commit -m "Describe what changed"
git push
```

Wait about a minute, then hard-refresh the Pages URL.

## Local files

| File | Role |
|---|---|
| `index.html` | Dashboard (this is what the public URL loads) |
| `data-lineage.html` | Lineage (where numbers come from) |
| `DATA-LINEAGE.md` | Same lineage in Markdown |
| `ACR-2934.txt` / `ACR-2935.txt` | Merck requirements |
