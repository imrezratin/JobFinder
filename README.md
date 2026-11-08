# CV ⇄ Job Matcher — Bangladesh & Remote

A static web app (plain HTML/CSS/JS, no build step) that you can host for free on **GitHub Pages**.

1. **Upload your CV** (PDF, DOCX, TXT) — parsed entirely in your browser, never uploaded anywhere.
2. The app extracts your **skills, role families, seniority, industry background, degrees & certifications** into an editable profile.
3. It pulls **live job listings** for **Bangladesh** and **remote-friendly** roles worldwide from several public job APIs (plus a cached feed refreshed by a GitHub Action).
4. Every job gets an **explainable match %** — skills you have, skills you're missing, role fit, seniority fit, location fit — plus filters, a skill-gap analysis, saved jobs, CSV/JSON export, pre-filled links into BDJobs/LinkedIn/Indeed, and a "score any pasted job description" tool.

```
cv-job-matcher/
├── index.html                 # the app shell
├── css/styles.css             # styling (light + dark)
├── package.json               # `npm run build` (esbuild bundle), `npm run fetch`, `npm run serve`
├── js/
│   ├── app.bundle.js          # built from app.js — what index.html loads (works from file:// too)
│   ├── app.js                 # UI orchestration
│   ├── cv-parser.js           # PDF/DOCX/TXT → text → profile (pdf.js + mammoth from CDN)
│   ├── analyze.js             # shared text analysis (skills, roles, seniority, location)
│   ├── skills.js              # the skill taxonomy, role families, adjacency, seniority words
│   ├── matcher.js             # the scoring model (explainable match %)
│   ├── sources.js             # job-source adapters (browser + Node)
│   ├── boards.js              # deep links to boards without an API (BDJobs, LinkedIn…)
│   └── storage.js             # localStorage wrapper
├── data/jobs.json             # cached feed written by the GitHub Action
├── scripts/fetch-jobs.mjs     # Node script that builds data/jobs.json
├── assets/                    # favicon, sample CV
└── .github/workflows/
    ├── pages.yml              # deploy to GitHub Pages on push
    └── fetch-jobs.yml         # refresh data/jobs.json every 6 h and redeploy
```

---

## 1 · Deploy to GitHub Pages (5 minutes)

1. Create a new repository on GitHub (public or private — Pages works for both on paid plans; public on free).
   Name it e.g. `job-matcher`, or `<your-username>.github.io` if you want it at the root URL.
2. Push this folder to the `main` branch:
   ```bash
   cd JobFinder
   git init -b main
   git add .
   git commit -m "feat: CV ⇄ Job Matcher"
   git remote add origin https://github.com/<your-username>/<repo>.git
   git push -u origin main
   ```
3. In the repository go to **Settings → Pages → Build and deployment → Source** and choose **GitHub Actions**.
4. The `Deploy to GitHub Pages` workflow runs automatically on the push. Your site will be live at
   `https://<your-username>.github.io/<repo>/` within a minute or two.
5. Go to **Actions → Refresh job feed → Run workflow** once to populate `data/jobs.json`
   (it then runs by itself every 6 hours and redeploys the site).

> **Running locally:** just double-click `index.html` — it works from a local file (the cached feed `data/jobs.json`
> can't be read from `file://`, but live sources still load). For the full experience run `npx serve .` or
> `python -m http.server 8080` and open `http://localhost:8080`.
>
> **Editing the code:** the browser loads `js/app.bundle.js`, which is built from `js/app.js` and its imports.
> After changing anything in `js/`, run `npm run build` (needs Node 18+; it uses esbuild via npx). The GitHub
> workflows run the build automatically, so the deployed site is always current even if you forget.

---

## 2 · Job sources

| Source | Where it runs | Key needed | Coverage |
|---|---|---|---|
| **Cached feed** (`data/jobs.json`) | GitHub Action, every 6 h | — | Everything below that the Action could reach |
| Remotive | Action only (≤4 calls/day per their terms) | — | Remote, tech-leaning |
| Jobicy | Browser + Action | — | Remote, with candidate-region info |
| Himalayas | Action only (no CORS) | — | Remote, with location restrictions |
| RemoteOK | Browser + Action | — | Remote, tech |
| Arbeitnow | Browser + Action | — | Europe ATS feeds (remote ones only) |
| The Muse | Action only (no CORS) | — | Flexible/remote, business & IT |
| **JSearch** (RapidAPI, Google Jobs) | Browser + Action | ✅ free tier ~200 req/month | **Bangladesh** — aggregates LinkedIn, Indeed, BDJobs, Glassdoor, company sites |
| **Jooble** | Action only (no CORS) | ✅ free key | **Bangladesh** aggregator |
| BDJobs, LinkedIn, Indeed BD, Glassdoor, Skill.jobs… | "Job boards" tab (deep links) | — | No public API → one-click pre-filled searches + "Score a posting" |

### Adding Bangladesh coverage (recommended)

Bangladeshi boards don't publish APIs. The two aggregators that do cover Bangladesh need free keys:

* **JSearch** — sign up at <https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch>, copy your `x-rapidapi-key`.
  * In the app: tick **JSearch** in step 3 and paste the key (stored only in your browser), **or**
  * In GitHub: **Settings → Secrets and variables → Actions → New repository secret** `JSEARCH_API_KEY` so the Action
    includes Bangladesh jobs in the cached feed for all visitors.
* **Jooble** — request a key at <https://jooble.org/api/about>, add it as the secret `JOOBLE_API_KEY`.

Optionally set a repository **variable** `SEARCH_QUERIES` (comma-separated, e.g.
`business analyst,erp implementation,it manager,power bi`) to control what the Action searches for.

If a live source fails in the browser (CORS policy changes, rate limits), the app shows it as *failed* and keeps going —
the cached feed still contains that source's jobs from the last Action run.

---

## 3 · How the match % is calculated

| Component | Weight | What it measures |
|---|---|---|
| **Skills** | 40 % | Skills detected in the posting (title/tag mentions count 2×) that also appear in your CV, weighted by skill importance. Postings with < 4 detectable skills are damped so one keyword can't give 100 %. |
| **Role fit** | 25 % | Does the job title belong to one of your role families? Adjacent families (e.g. *Business Analyst* ↔ *ERP Consultant*) earn 60 %. |
| **Seniority** | 10 % | Level inferred from the title / "X+ years" vs. your years of experience. One step up = 70 %, one step down = 85 %. |
| **Location** | 15 % | Bangladesh = 100 %. Remote worldwide / Asia-friendly = 100 %. Remote locked to USA/EU = 15 % (60 % if you opt in). On-site abroad = 5 % (50 % if open to relocation). |
| **Industry** | 5 % | Overlap between your industry background (garments, textile, manufacturing…) and the posting. |
| **Title similarity** | 5 % | Token overlap between your target titles and the job title. |

Bands: **Strong ≥ 75**, **Good 55–74**, **Fair 35–54**, **Weak < 35**. Each card has a *"Why N %?"* panel with the per-component bars and notes, and a **confidence** label (based on description length and number of detectable skills).

Tune the weights in `js/matcher.js` (`WEIGHTS`) and the vocabulary in `js/skills.js` — adding a skill alias there improves both CV parsing and job parsing at once.

---

## 4 · Privacy

* The CV is parsed with [pdf.js](https://mozilla.github.io/pdf.js/) / [mammoth](https://github.com/mwilliamson/mammoth.js) **in the browser**.
* The profile, saved jobs, API key and preferences live in `localStorage` on your device only. "Clear all data" in the footer wipes them.
* Job APIs are called directly from your browser (or from the GitHub Action for the cached feed); your CV text is never sent to them.

## 5 · Customising

* **Brand / colours:** CSS variables at the top of `css/styles.css`.
* **Repository link:** edit the `#gh-link` href in `index.html`.
* **Default search roles for the Action:** `DEFAULT_QUERIES` in `scripts/fetch-jobs.mjs` or the `SEARCH_QUERIES` variable.
* **More sources:** add an adapter in `js/sources.js` returning `makeJob({...})` objects and register it in `LIVE_ADAPTERS` (browser) and/or `scripts/fetch-jobs.mjs` (Action).

## Attribution

Listings are provided by Remotive, Jobicy, Himalayas, RemoteOK, Arbeitnow, The Muse and — when configured — JSearch and Jooble. Each card links back to the original posting as those services require. MIT licensed.
