# Merck Pipeline Observability (prototype)

Static dashboard built from Acceldata ADM snapshot data (26 Aug 2026, 04:30 UTC). It does **not** query ADM while you use it.

## Public links

After GitHub Pages is on:

- Dashboard: https://ad-krish.github.io/merck-pipeline-observability/pipeline-observability.html
- Data lineage: https://ad-krish.github.io/merck-pipeline-observability/data-lineage.html

The repo root URL redirects to the dashboard.

## How GitHub is used

GitHub stores this folder as a **public repository** (a shared copy of the files). Anyone with the link can view the files. **GitHub Pages** then serves those HTML files as a website.

| Step | What it does |
|---|---|
| `git add` / `git commit` | Save a snapshot of your local files |
| `git push` | Upload that snapshot to GitHub |
| GitHub Pages | Turns the HTML on the `main` branch into a public URL |

## Update the live site

1. Edit `pipeline-observability.html` (or the lineage docs) locally.
2. From this folder:

```bash
git add -A
git commit -m "Describe what changed"
git push
```

3. Wait about a minute, then hard-refresh the Pages URL.

## Local files

| File | Role |
|---|---|
| `pipeline-observability.html` | Dashboard |
| `data-lineage.html` | Lineage (where numbers come from) |
| `DATA-LINEAGE.md` | Same lineage in Markdown |
| `ACR-2934.txt` / `ACR-2935.txt` | Merck requirements |
