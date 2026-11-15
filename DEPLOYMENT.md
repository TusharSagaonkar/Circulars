# Circulars — Static Website Deployment

This folder is a **ready-to-deploy static website** generated from
`https://github.com/TusharSagaonkar/Circulars`.

## What this is

The repo is a collection of Microsoft-Word-saved HTML pages ("Web Circulars"
— an internal circulars / office-orders portal). The pages link to each other
and to per-document asset folders (`*_files/`).

## Why it "worked on Windows" but broke on a real static host

The original links use **mixed case** that does not match the actual filenames
on disk — for example a link to `INDEX-OC.htm` while the file is `index-oc.htm`,
or `../oc/2016-17/...` while the directory is `Oc/2016-17/`. Windows'
filesystem is **case-insensitive**, so every such link resolved. Linux static
hosts (GitHub Pages, Netlify, S3, nginx, `python -m http.server`) are
**case-sensitive**, so the same links would 404.

A second, pre-existing problem: the `oo/` and `policy/` content is
**doubly nested** (`oo/oo/<year>/`, `policy/policy/`), but their "back to
index" links were written for a shallower depth. Those links were *already
broken on Windows* (case-insensitivity does not fix a wrong number of `../`).

## What was fixed

Two automated passes (scripts in `/home/z/my-project/scripts/`) rewrote the
HTML in place — **no files were renamed, no content was removed**:

1. **`deploy.py`** — for every `href`/`src` that resolves case-insensitively to
   an existing file but is not exact-case, rewrote it to the real on-disk path.
   **1,580 case-mismatched links fixed** (e.g. `INDEX-OC.htm` → `index-oc.htm`,
   `../oc/2016-17/...` → `../Oc/2016-17/...`).
2. **`depth_fix.py`** — for back-links whose `../` count is wrong, searched
   upward to the site root for the real target and rewrote with the correct
   depth **and** case. **1,709 depth-broken links fixed** (e.g. a policy page's
   `../INDEX/INDEX-POLICIES.htm` → `../../index/index-policies.htm`).

A root **`index.html`** (and `index.htm`) redirects to `index/index.htm`, the
real home page. FrontPage `_vti_cnf/` metadata folders were removed.

### Link-health after fixing

| metric | before | after |
| --- | --- | --- |
| exact-match (working) links | 8,399 | **11,674** |
| case-mismatched links | 1,568 | **0** |
| depth-broken back-links | ~1,700 | **0** |

### Remaining broken links (not fixable — content is absent from the repo)

These 404 on **any** host, including Windows. They are not deployment defects;
the target files simply do not exist in the repository:

- **Older circular years not committed** — the indexes reference years such as
  `1977-95`, `2001-02`, `2009-10`, `2010-11`, `2011-12`, `2014-15`, but the
  repo only contains `2016-17` onward for most series.
- **The author's local Windows paths** — e.g.
  `../../../../../../My Webs BackUp/acbl_20180904/...` (left over from Word's
  editing history on the original machine).
- **Office metadata** — `editdata.mso`, `filelist.xml`, `themedata.thmx`,
  `colorschememapping.xml`, `item00XX.xml`. Browsers ignore these; they do not
  affect rendering.
- A couple of items that point outside the repo: `Computer_Basics.pps`,
  `../../../Tel_Code/0-Home Page.htm`, `../../../Shortkey/00-OMNI.htm`,
  `INDEX-DURING 6 MONTHS.htm`.

## Site layout

```
circulars-site/            <- site root (deploy this folder)
├── index.html / index.htm  <- redirect to index/index.htm
├── index/                  <- home page + all index pages + their *_files
│   ├── index.htm           <- the real home page (entry point)
│   ├── index-oc.htm        <- Circulars index
│   ├── index-oo.htm        <- Office Orders index
│   ├── index-fm.htm        <- Fraud Master index
│   ├── index-fx.htm        <- Forex index
│   ├── index-cr.htm        <- Credit & Recovery index
│   ├── index-policies.htm  <- Policies index
│   └── index-dept.htm      <- Department-wise index
├── Oc/<year>/              <- Circular documents + *_files + PDFs
├── oo/oo/<year>/           <- Office Order documents
├── fm/                     <- Fraud Master documents
├── fx/<year>/              <- Forex documents
├── Cr/<year>/              <- Credit & Recovery documents
└── policy/policy/          <- Policy documents
```

The entry point is **`index/index.htm`**. Visiting the site root redirects
there automatically.

## How to deploy

This is a plain static site — no build step, no server-side code. Pick any
static host and upload the **contents of this folder** as the site root.

### Option A — GitHub Pages
1. Create a new public repository (e.g. `circulars`).
2. Copy the **contents** of `circulars-site/` into the repo root
   (`git init`, add all, commit, push).
3. Settings → Pages → Source = `Deploy from a branch`, branch = `main`,
   folder = `/ (root)`.
4. Site goes live at `https://<user>.github.io/circulars/`.

### Option B — Netlify / Vercel / Cloudflare Pages
- Drag-and-drop the `circulars-site/` folder into Netlify Drop
  (https://app.netlify.com/drop), or connect the repo. No build command,
  publish directory = `.` (the folder root).

### Option C — any static file host (S3, nginx, Apache, Caddy)
- Upload the folder contents to the document root. Default directory index
  `index.html` is already present (redirects to `index/index.htm`).

### Option D — preview locally
```bash
cd circulars-site
python3 -m http.server 8000
# open http://localhost:8000/
```

## Verified

Served locally and confirmed HTTP 200 for: the root redirect, the home page,
every category index (`index-oc`, `index-oo`, `index-fm`, `index-fx`,
`index-cr`, `index-policies`, `index-dept`), sample documents from every
category, embedded image assets, and the depth-fixed back-to-index links.
