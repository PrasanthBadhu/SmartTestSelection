# Selective Regression Testing — Implementation Plan

## 1. Architecture Overview

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  tr/cobalt_search│  │tr/cobalt_website │  │tr/cobalt_static- │  │tr/cobalt_document│
│  (READ-ONLY)     │  │  (READ-ONLY)     │  │  content         │  │  _netcore        │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                      │                     │
         │           ┌─────────┴──────┐    ┌──────────┴──────┐             │
         │           │ tr/cobalt_alert│    │tr/cobalt_folder-│             │
         │           │  (READ-ONLY)   │    │  ing (READ-ONLY)│             │
         │           └────────┬───────┘    └────────┬────────┘             │
         │                    │                     │                      │
         │       ┌────────────┴──────┐   ┌──────────┴──────┐               │
         │       │tr/cobalt_relatedinfo│  │tr/cobalt_type-  │              │
         │       │  (READ-ONLY)       │  │  ahead          │              │
         │       └────────┬──────────┘   └────────┬────────┘               │
         │                │                        │                        │
         └────────────────┴────────────────────────┴────────────────────────┘
                                         │  GitHub API (read-only polling)
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  tr/CobaltRegressionTesting  (GitHub Actions host)                                   │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  pr-poller.yml   (manual dispatch — "PR Poller (Manual)")                    │   │
│  │                                                                              │   │
│  │  1. Read config/repo-registry.yml                                            │   │
│  │     → 8 source repos (enabled: true)                                         │   │
│  │                                                                              │   │
│  │  2. For each repo: discover PRs (by PR number, scan_date, or date range)    │   │
│  │     → Uses GitHub Search API (merged:) + Pulls API supplement               │   │
│  │     → Writes state/prs_discovered.json                                      │   │
│  │                                                                              │   │
│  │  3. Analyze PR file impact (10 concurrent workers)                          │   │
│  │        gh pr view <N> --repo <full_name> --json files                       │   │
│  │        → Loads per-repo feature map from feature_map_local                  │   │
│  │        → Maps changed file paths → modules / sub_paths                      │   │
│  │        → Writes state/<repo_name>/pr_impacts/pr_<N>.json                   │   │
│  │                                                                              │   │
│  │  4. Map impact to WL_DNet workflows                                         │   │
│  │        python tools/feature_test_mapper.py                                  │   │
│  │        --impact state/<repo_name>/pr_impacts/pr_<N>.json                   │   │
│  │        --map-local config/feature-map-<repo>.yml   ← LOCAL file            │   │
│  │        → maps TestCategory names → WL_DNet_*.yml workflows                 │   │
│  │        → uses CAT_TO_WORKFLOW dict (248 entries) in feature_test_mapper.py  │   │
│  │        → writes state/test_plans/<repo_name>_<N>.json                      │   │
│  │                                                                              │   │
│  │  5. Build per-file impact details                                           │   │
│  │        → per-file classification (matched_by, risk, test_categories)       │   │
│  │        → writes state/per_file_impacts.json                                 │   │
│  │                                                                              │   │
│  │  6. Dispatch WL_DNet_*.yml directly (unless dry_run=true)                  │   │
│  │        gh workflow run <WL_DNet_*.yml> --repo tr/CobaltRegressionTesting   │   │
│  │        -f TEST_SITE_DotNet=<env>                                            │   │
│  │                                                                              │   │
│  │  7. Write Markdown job summary                                              │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                   │                                                                  │
│                   ▼                                                                  │
│      WL_DNet_Edge_*.yml  /  WL_DNet_Next_*.yml  /  WL_DNet_ANZ_*.yml  / …          │
│      (runs on AWS EC2 via CodeBuild runner)                                          │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### Eight feature maps — one per source repo

| Source Repo | Feature Map (local) | Fetched from (fallback) |
|---|---|---|
| tr/cobalt_search | `config/feature-map-search.yml` | tr/Seven-Kingdoms `feature-map-search.yml` |
| tr/cobalt_website | `config/feature-map-website.yml` | tr/Seven-Kingdoms `feature-map-website.yml` |
| tr/cobalt_static-content | `config/feature-map-static-content.yml` | tr/Seven-Kingdoms `feature-map-static-content.yml` |
| tr/cobalt_document_netcore | `config/feature-map-document.yml` | tr/Seven-Kingdoms `feature-map-document.yml` |
| tr/cobalt_alert | `config/feature-map-alert.yml` | tr/Seven-Kingdoms `feature-map-alert.yml` |
| tr/cobalt_foldering | `config/feature-map-foldering.yml` | tr/Seven-Kingdoms `feature-map-foldering.yml` |
| tr/cobalt_relatedinfo | `config/feature-map-relatedinfo.yml` | tr/Seven-Kingdoms `feature-map-relatedinfo.yml` |
| tr/cobalt_typeahead | `config/feature-map-typeahead.yml` | tr/Seven-Kingdoms `feature-map-typeahead.yml` |

### Workflow

| Workflow | Trigger | Purpose |
|---|---|---|
| `pr-poller.yml` | Manual only (workflow_dispatch) | On-demand PR discovery, impact analysis, and direct WL_DNet dispatch |

---

## 2. Mapping Chain

```
source-repo changed files   (any of the 8 monitored repos)
    │
    ├─ sub_path match   (longest prefix wins — highest priority)
    ├─ module match     (top-level folder prefix)
    └─ keyword match    (PR title + body, case-insensitive)
         │
         ▼
    TestCategory names   (from the repo's feature-map-*.yml)
         │
         ▼
    WL_DNet_*.yml workflows   (CAT_TO_WORKFLOW dict in feature_test_mapper.py, 248 entries)
         │
         ▼
    gh workflow run <WL_DNet_*.yml> dispatched directly from pr-poller.yml
         │
         ▼
    WL_DNet tests run on AWS EC2 (CodeBuild runner)
```

---

## 3. Feature Map Coverage (as of 2026-06-18)

### feature-map-search.yml  (cobalt_search)

| Section | Count | Notes |
|---|---|---|
| Modules | 12 | SearchCommon, SearchMetadataObjects, SearchSerialized, CarswellEcosystem, CobaltPlatformSearch, CarswellSearchWeb, TNPCarswellSearchWeb, WFASearchWeb, WLNSearch, CorrectionalSearch, DraftingSearch, WeblinksSearch |
| Sub-paths | 8 | All within WLNSearch: aunz, wlglobal, casesuk, docketuk, researchuk, wluk, publicrecords, WLUK result service |
| Keywords | 30+ | brazil, wluk, aunz, anz, canada, enhancements, blc, business law, docket, keycite, portal manager, court express, california, alerts, document, delivery, precision, advantage, analytics, related info, foldering, annotation, quick check, tax migration, website, redlining, research report, smart folder, 10k, search 4k, trillium |
| high_risk_always_add | 3 | CoreSearch, GlobalSearch, EdgeSmokeFeatures |

### feature-map-website.yml  (cobalt_website)

| Section | Count | Notes |
|---|---|---|
| Modules | 19 | WLNWebsite, WLNWebsiteCommon, CobaltPlatformWeb (high-risk), CarswellWebsite, ANZWebsite, WFAWebsite, WebsiteDelivery, WebsiteMobile, WebsiteAlerts, WebsiteFoldering, WebsiteAnalytics, WebsitePrecision, WebsiteKeyCite, WebsiteAnnotations, WebsiteResearchReports, WebsiteRedlining, WebsiteQuickCheck, WebsiteWeblinks, WebsiteEnhancements |
| Sub-paths | 22+ | Top-level module shortcuts + deep Java package paths under WLNWebsite/src/main/java/com/tr/cobalt/web/\<feature\>/ |
| Keywords | 35+ | website, homepage, navigation, delivery, mobile, search, aunz, anz, canada, alert, folder, analytics, precision, advantage, keycite, key cite, related info, annotation, highlight, redlining, quick check, research report, smart folder, trillium, enhancements, blc, business law, tax, tax migration, docket, california, weblinks, portal manager, court express, document, wluk, 10k, search 4k |
| high_risk_always_add | 5 | CoreSearch, GlobalSearch, EdgeSmokeFeatures, WebsiteCore1, WebsiteCore2 |

### feature-map-static-content.yml  (cobalt_static-content)

| Section | Count | Notes |
|---|---|---|
| Modules | 17 | CommonStaticContent, WLNStaticContent, CobaltPlatformStaticContent (high-risk), CarswellStaticContent, ANZStaticContent, WFAStaticContent, StaticContentDelivery, StaticContentMobile, StaticContentAlerts, StaticContentAnalytics, StaticContentPrecision, StaticContentFoldering, StaticContentKeyCite, StaticContentAnnotations, StaticContentResearchReports, StaticContentRedlining, StaticContentWeblinks |
| Sub-paths | 40+ | Typed by content type (css/, js/, templates/) AND product area: CommonStaticContent/css\|js\|templates, WLNStaticContent/css/common, plus product-specific paths for aunz, canada, mobile, delivery, alerts, analytics, folder, precision, keycite, annotation, homepage, redlining, researchreports, weblinks, tax, blc |
| Keywords | 35+ | Same as website map plus: css, template, static, responsive |
| high_risk_always_add | 5 | CoreSearch, GlobalSearch, EdgeSmokeFeatures, WebsiteCore1, WebsiteCore2 |

### feature-map-document.yml  (cobalt_document_netcore)

| Section | Count | Notes |
|---|---|---|
| Modules | 9 | Document (high-risk), WestlawNext (high-risk), Carswell, Correctional, Drafting, SLWB, TNPCarswell, Weblinks, DocumentSuperbuild |
| Sub-paths | 17 | Document/Document/Controllers\|Pipeline\|Domain\|Xslt\|Views, Document/Document/Domain/DocumentOutline, Document/Document/Provider/CrossVertical/DocCompare, WestlawNext/WestlawNextDocument, WestlawNext/WLNDocument (+ /Domain/DocumentOutline), WestlawNext/WestlawNextIntegrationTests, Carswell/CarswellDocument, Correctional/CorrectionalDocument, Drafting/DraftingDocument, SLWB/SLWBDocument, TNPCarswell/TNPCarswellDocument, Weblinks/WeblinksDocument |
| Keywords | 45+ | document, doc display, docx, delivery, related info, rendering, xslt, pipeline, canada, carswell, correctional, drafting, weblinks, government weblinks, keycite, research report, synopsis, print, pdf, litigation analytics, localization, globalization, focus highlight, table of contents, toc, tco, versions compare, compare documents, snippet, save snippet, snippet compare, snippet navigation, citing reference, public domain, procedural posture, find and print, find & print, arbitration, practical law, public utility, new york digest, portal manager, related documents, copy with reference, scrolling, content type, inline keycite |
| high_risk_always_add | 4 | Document, DocDisplay, DOCX, EdgeSmokeFeatures |

### feature-map-alert.yml  (cobalt_alert)

| Section | Count | Notes |
|---|---|---|
| Modules | 19 | Alert (high-risk), AlertCommon (high-risk), AlertProcessor (high-risk), AlertEventListener (high-risk), AlertProductService (high-risk), AlertSerialized, AlertStatsListener, AlertVeracode, AnalyticsAlert, CarswellAlert, ClassifierProcessor, JudicialPracticeAlert, PubProducerMaintainer, SLWBAlert, TNPCarswellAlert, WFBAlert, WLNAlert, Config, LoggingConfig |
| Sub-paths | 19 | Alert/src\|WebContent\|IntegrationTests, AlertCommon/src, AlertProcessor/src\|integration, AlertEventListener/src, AlertProductService/src, AlertSerialized/src, AnalyticsAlert/src, CarswellAlert/src, TNPCarswellAlert/src, ClassifierProcessor/src, JudicialPracticeAlert/src, SLWBAlert/src, WFBAlert/src, WLNAlert/src\|EndpointTests, PubProducerMaintainer/src |
| Keywords | 34+ | alert, alerts, alert access, capitol watch, mobile alert, subscription, notification, analytics alert, alert analytics, analytics, aunz, anz, canada, carswell, judicial practice, classifier, processor, serialization, event listener, westlaw today, wlt, custom digest, digest, alert admin, alert center, court wire, docket alert, westclip, email delivery, newsletter, company investigator, alert engine, keycite alert, uk alert |
| high_risk_always_add | 3 | AlertAccess, Alerts, EdgeSmokeFeatures |

### feature-map-foldering.yml  (cobalt_foldering)

| Section | Count | Notes |
|---|---|---|
| Modules | 13 | Foldering (high-risk), FolderingCommon (high-risk), FolderingSerialized (high-risk), FormsFolderingCore, SharedFoldering, SharedFolderingCache, History, CarswellFoldering, CorrectionalFoldering, DraftingFoldering, TNPCarswellFoldering, WFBFoldering, eReaderFoldering |
| Sub-paths | 18 | Foldering/src\|WebContent\|IntegrationTests\|FunctionalTests, FolderingCommon/src, FolderingSerialized/src, FormsFolderingCore/src\|EndpointTests, SharedFolderingCache/src, History/HistoryPlatform\|HistoryListener, CarswellFoldering/src, TNPCarswellFoldering/src, CorrectionalFoldering/src, DraftingFoldering/src, WFBFoldering/src\|EndpointTests, eReaderFoldering/src |
| Keywords | 25+ | folder, foldering, smart folder, research organizer, folder recommendation, folder redesign, folder analysis, keycite, history, cache, serialization, aunz, anz, nz forms, canada, carswell, correctional, drafting, ereader, e-reader, snippet, save snippet, non root, private files, private |
| high_risk_always_add | 3 | Foldering, ResearchOrganizer, EdgeSmokeFeatures |

### feature-map-relatedinfo.yml  (cobalt_relatedinfo)

| Section | Count | Notes |
|---|---|---|
| Modules | 11 | RelatedInfo (high-risk), RelatedInfoSerialized (high-risk), WLNRelatedInfo (high-risk), AccelusRelatedInfo, CarswellRelatedInfo, WestKMRelatedInfo, CorrectionalRelatedInfo, DraftingRelatedInfo, SlwbRelatedInfo, TNPCarswellRelatedInfo, RelatedInfoConfig |
| Sub-paths | 15 | RelatedInfo/src\|GraphicalKeyciteTests\|IntegrationTests\|EndpointTests, RelatedInfoSerialized/src, WLNRelatedInfo/src\|EndpointTests\|IntegrationTests, WestKMRelatedInfo/src, AccelusRelatedInfo/src, CarswellRelatedInfo/src, TNPCarswellRelatedInfo/src, CorrectionalRelatedInfo/src, DraftingRelatedInfo/src, SlwbRelatedInfo/src |
| Keywords | 26+ | related info, relatedinfo, keycite, key cite, graphical history, graphical keycite, ip tools, citing references, delivery, facets, flags, negative treatment, negative history, citing reference, serialization, canada, carswell, correctional, drafting, accelus, inline keycite, search facets, related documents, copy with reference, related information, keycite alerts |
| high_risk_always_add | 4 | RelatedInfoContent, RelatedInfoDelivery, RelatedInfoFlags, EdgeSmokeFeatures |

### feature-map-typeahead.yml  (cobalt_typeahead)

| Section | Count | Notes |
|---|---|---|
| Modules | 5 | CobaltTypeahead (high-risk), WLNTypeahead, CarswellTypeahead, CorrectionalTypeahead, TNPCarswellTypeahead |
| Sub-paths | 7 | CobaltTypeahead/src, WLNTypeahead/src\|WebContent, CarswellTypeahead/src\|WebContent, CorrectionalTypeahead/src, TNPCarswellTypeahead/src |
| Keywords | 17 | typeahead, autocomplete, suggestion, search suggest, smart search, browse page, advanced search, canada, carswell, correctional, trd typeahead, tco, tco section, indigo typeahead, search platform, wln core, wln enhancements |
| high_risk_always_add | 3 | CoreSearch, GlobalSearch, EdgeSmokeFeatures |

### CAT_TO_WORKFLOW coverage summary (248 total entries)

| Category | Count | Examples |
|---|---|---|
| Covered in all / most maps | ~145 | ANZ suite, Canada suite, Analytics, Precision, Foldering, KeyCite, Annotations, Alerts, Delivery, Redlining, QuickCheck, ResearchReports, BLC, Tax, Weblinks, Website Core |
| cobalt_search map only (intentional) | 13 | AdvancedSearchTemplate, BrowsePageSearch, MultipleSearchWithin_Edge, SearchCore, SearchMetadata, SearchblePdfs, SmartSearch, WlnEdgeSearch, WLNCorrectional, Foldering (Edge variant), AnzUnderDevelopment, AnzUnderDevelopment1, KeyCiteTestFlag |
| In website but **missing from static-content** | 8 | QuickCheckUiCheckWork_1–4, QuickCheckUiJudicial_1–3, QuickCheckUiOpponent ← **known gap** |
| Not in any map | ~82 | All 18 Axe integration tests + search/UI-specific tests (see §10) |

---

## 4. File Inventory

### What goes in `tr/CobaltRegressionTesting`

| File | Status | Change Summary |
|---|---|---|
| `.github/workflows/pr-poller.yml` | **CURRENT** | Manual dispatch; discovers PRs, analyzes impact, dispatches WL_DNet_*.yml directly |
| `config/repo-registry.yml` | **CURRENT** | Registers all 8 source repos |
| `config/feature-map-search.yml` | **CURRENT** | 12 modules, 8 sub-paths, 30+ keywords; duplicate analytics fix applied; TrdSmoke added |
| `config/feature-map-website.yml` | **CURRENT** | 19 modules, 22+ sub-paths, 35+ keywords |
| `config/feature-map-static-content.yml` | **CURRENT** | 17 modules, 40+ sub-paths, 35+ keywords |
| `config/feature-map-document.yml` | **CURRENT** | 9 modules, 17 sub-paths, 45+ keywords (cobalt_document_netcore) |
| `config/feature-map-alert.yml` | **CURRENT** | 19 modules, 19 sub-paths, 34+ keywords (cobalt_alert) |
| `config/feature-map-foldering.yml` | **CURRENT** | 13 modules, 18 sub-paths, 25+ keywords (cobalt_foldering) |
| `config/feature-map-relatedinfo.yml` | **CURRENT** | 11 modules, 15 sub-paths, 26+ keywords (cobalt_relatedinfo) |
| `config/feature-map-typeahead.yml` | **CURRENT** | 5 modules, 7 sub-paths, 17 keywords (cobalt_typeahead) |
| `tools/pr_impact_analyzer.py` | **CURRENT** | `--repo`, `--state-dir`, `--feature-map`, `--force-prs` flags; null-safe YAML loading |
| `tools/feature_test_mapper.py` | **CURRENT** | Null-safe YAML loading; 248-entry CAT_TO_WORKFLOW dict |
| `tools/requirements.txt` | **UNCHANGED** | PyGithub>=2.1.1, pyyaml>=6.0.1 |

### What goes in `tr/Seven-Kingdoms` (fallback copies)

| File | Action |
|---|---|
| `feature-map-search.yml` | Keep in sync with `config/feature-map-search.yml` |
| `feature-map-website.yml` | Keep in sync with `config/feature-map-website.yml` |
| `feature-map-static-content.yml` | Keep in sync with `config/feature-map-static-content.yml` |
| `feature-map-document.yml` | Keep in sync with `config/feature-map-document.yml` |
| `feature-map-alert.yml` | Keep in sync with `config/feature-map-alert.yml` |
| `feature-map-foldering.yml` | Keep in sync with `config/feature-map-foldering.yml` |
| `feature-map-relatedinfo.yml` | Keep in sync with `config/feature-map-relatedinfo.yml` |
| `feature-map-typeahead.yml` | Keep in sync with `config/feature-map-typeahead.yml` |

> Note: `feature_map_local` in repo-registry.yml points to local `config/` copies, so the Seven-Kingdoms API fetch is only a fallback. Both locations must be kept in sync.

### What stays in the source repos

| Repo | Action |
|---|---|
| tr/cobalt_search | **NO CHANGES** — read-only |
| tr/cobalt_website | **NO CHANGES** — read-only |
| tr/cobalt_static-content | **NO CHANGES** — read-only |
| tr/cobalt_document_netcore | **NO CHANGES** — read-only |
| tr/cobalt_alert | **NO CHANGES** — read-only |
| tr/cobalt_foldering | **NO CHANGES** — read-only |
| tr/cobalt_relatedinfo | **NO CHANGES** — read-only |
| tr/cobalt_typeahead | **NO CHANGES** — read-only |

---

## 5. Code Fixes Applied

### Null-safety fix: YAML `modules:` / `sub_paths:` with no value

**Root cause:** YAML parses a key with no value (e.g. `modules:`) as Python `None`. `dict.get("key", {})` returns `None` when the key exists with a `None` value; `None.keys()` raises `AttributeError` and crashes the analyze step silently, producing no impact file and thus no changed files recorded.

**Fix:** All five feature-map section lookups changed from `fm.get("key", {})` to `(fm.get("key") or {})`:

```python
# tools/feature_test_mapper.py  (lines ~320-324)
fm_modules   = (feature_map.get("modules")             or {})
fm_sub_paths = (feature_map.get("sub_paths")           or {})
fm_keywords  = (feature_map.get("keywords")            or {})
fm_ct_rules  = (feature_map.get("change_type_rules")   or {})
fm_high_risk = (feature_map.get("high_risk_always_add") or {})
```

Same pattern applied in `tools/pr_impact_analyzer.py` and the `classify_file` function inside `pr-poller.yml`.

### Duplicate analytics keyword fix (feature-map-search.yml)

**Root cause:** Two `analytics:` entries in the keywords section. YAML silently drops the first when a key appears twice; the shorter second entry survived, losing `LegalAnalyticsApi`, `LitigationAnalytics`, `TrdLegalAnalyticsUi`, `TrdSmoke`, `TrdApi`, `TrdFacets`.

**Fix:** Removed the duplicate shorter entry. The surviving entry now contains all 8 analytics categories including `TrdSmoke`.

### Direct WL_DNet dispatch (selective-regression.yml removed)

**Change:** The original design routed pr-poller.yml → selective-regression.yml → WL_DNet_*.yml. `selective-regression.yml` has been removed. pr-poller.yml now dispatches each WL_DNet_*.yml directly via `gh workflow run`.

**Why:** Eliminates an unnecessary indirection layer, simplifying debugging and reducing the number of GitHub Actions workflows to maintain.

---

## 6. Step-by-Step Deployment

### Prerequisites — GitHub Secrets

Ensure these secrets exist in `tr/CobaltRegressionTesting` (Settings → Secrets → Actions):

| Secret | Permission | Used by |
|---|---|---|
| `COBALT_READ_TOKEN` | `read:repo` on all 8 source repos | pr-poller.yml (PR discovery + file fetching) |
| `SEVEN_KINGDOMS_TOKEN` | `read:repo` on tr/Seven-Kingdoms | feature_test_mapper.py (API fallback) |
| `REGRESSION_TRIGGER_PAT` | `workflow:write` on tr/CobaltRegressionTesting | pr-poller.yml (WL_DNet dispatch) |

> All 8 source repos share the same `COBALT_READ_TOKEN` — they are in the same org.

### Deployment Steps

**Step 1 — Deploy config files**
```bash
git add config/repo-registry.yml
git add config/feature-map-search.yml
git add config/feature-map-website.yml
git add config/feature-map-static-content.yml
git add config/feature-map-document.yml
git add config/feature-map-alert.yml
git add config/feature-map-foldering.yml
git add config/feature-map-relatedinfo.yml
git add config/feature-map-typeahead.yml
git commit -m "feat: add feature maps for all 8 monitored repos"
git push origin main
```

**Step 2 — Deploy Python tools**
```bash
git add tools/pr_impact_analyzer.py   # --repo, --state-dir, --feature-map, --force-prs; null-safe
git add tools/feature_test_mapper.py  # null-safe YAML loading
git commit -m "feat: multi-repo support and null-safe feature map loading"
git push origin main
```

**Step 3 — Deploy pr-poller.yml**
```bash
git add .github/workflows/pr-poller.yml
git commit -m "feat: manual PR poller — direct WL_DNet dispatch"
git push origin main
```

**Step 4 — Deploy feature maps to Seven-Kingdoms (fallback copies)**
```bash
# In tr/Seven-Kingdoms
cp <solution>/config/feature-map-search.yml .
cp <solution>/config/feature-map-website.yml .
cp <solution>/config/feature-map-static-content.yml .
cp <solution>/config/feature-map-document.yml .
cp <solution>/config/feature-map-alert.yml .
cp <solution>/config/feature-map-foldering.yml .
cp <solution>/config/feature-map-relatedinfo.yml .
cp <solution>/config/feature-map-typeahead.yml .
git add *.yml
git commit -m "feat: add/update feature maps for 8 monitored repos"
git push origin main
```

**Step 5 — Smoke test (dry run)**
```
GitHub UI → tr/CobaltRegressionTesting → Actions → "PR Poller (Manual)"
  → Run workflow → dry_run: true → Run
```
Verify the job summary shows detected PRs from all 8 repos and their mapped workflows, without triggering actual test runs.

**Step 6 — Go live**
```
GitHub UI → Actions → "PR Poller (Manual)"
  → Run workflow → dry_run: false → Run
```

---

## 7. Expected Workflow — End to End (example: cobalt_website PR)

```
T+0:00   Developer merges a PR in tr/cobalt_website
         Changed files include: WLNWebsite/src/main/java/com/tr/cobalt/web/analytics/...

T+0–5m   pr-poller.yml triggered manually from GitHub UI

T+0:05s  Discover PRs step
           GitHub Search API: gh pr list --search "is:merged merged:<date>"
           Supplement: gh pr list --state merged --limit 1000
           Writes state/prs_discovered.json

T+0:10s  Analyze PR impact (10 parallel workers)
           gh pr view <N> --repo tr/cobalt_website --json files
           → Loads config/feature-map-website.yml
           → sub_path match: WLNWebsite/.../analytics/ → product:Analytics
           → Writes:
             state/cobalt_website/pr_impacts/pr_<N>.json
               {pr_number, modules_affected:[WebsiteAnalytics],
                sub_products:[analytics], risk_level:medium, ...}

T+0:25s  Map impact to workflows
           python tools/feature_test_mapper.py
             --impact state/cobalt_website/pr_impacts/pr_<N>.json
             --map-local config/feature-map-website.yml
           → analytics sub_path → TestCategories:
             [WLAnalytics, WLAnalyticsIndigo, AnalyticsAlertsEnhancements,
              LegalAnalyticsApi, LitigationAnalytics, TrdLegalAnalyticsUi,
              TrdSmoke, TrdApi, TrdFacets]
           → Maps via CAT_TO_WORKFLOW to WL_DNet workflows:
             [WL_DNet_Analytics_Regression_Next, WL_DNet_Analytics_Edge,
              WL_DNet_Analytics_Alerts_Enhancements_Next,
              WL_DNet_Edge_LegalAnalyticsApi_Edge,
              WL_DNet_Precision_LitigationAnalytics,
              WL_DNet_Edge_TrdLegalAnalyticsUi,
              WL_DNet_Edge_TrdSmoke_Edge,
              WL_DNet_Edge_TrdApi_Edge, WL_DNet_Edge_TrdFacets_Edge]
             + high_risk_always_add: [WebsiteCore1, WebsiteCore2, CoreSearch,
                                      GlobalSearch, EdgeSmokeFeatures]
           → Writes state/test_plans/cobalt_website_<N>.json

T+0:35s  Dispatch WL_DNet workflows directly
           gh workflow run WL_DNet_Analytics_Regression_Next.yml \
             --repo tr/CobaltRegressionTesting \
             -f TEST_SITE_DotNet=DEMO
           (… repeated for each workflow in the plan)

T+1–30m  WL_DNet workflows run on AWS EC2 (CodeBuild runner)

T+end    Job summary written to GitHub Actions step summary
           state/per_file_impacts.json and state/all_impacted_workflows.json
           retained for this run (not committed — transient)
```

---

## 8. State Management

### State directory layout (written into the GitHub Actions runner workspace)

```
state/
├── prs_discovered.json          # transient: PR list for this run
├── per_file_impacts.json        # transient: per-file classification for this run
├── all_impacted_workflows.json  # transient: merged workflow list for this run
├── <repo_name>/
│   └── pr_impacts/
│       └── pr_<N>.json          # per-PR impact analysis output
└── test_plans/
    └── <repo_name>_<N>.json     # per-PR test plan (TestCategories + workflows)
```

Examples with 8 repos:
```
state/cobalt_search/pr_impacts/pr_2460.json
state/cobalt_website/pr_impacts/pr_1234.json
state/cobalt_alert/pr_impacts/pr_567.json
state/test_plans/cobalt_search_2460.json
state/test_plans/cobalt_website_1234.json
```

> **Note:** State files are written to the GitHub Actions runner's workspace on each run. They are not committed to any branch — they exist only within a single workflow run's filesystem.

### Change detection

The workflow re-processes every PR it discovers within the requested date range or PR number list. There is no cross-run deduplication by default. To re-analyze a specific PR from any earlier period, pass its number via the `pr_numbers` input.

---

## 9. Risk Escalation Rules

| Condition | Effect | Applies to |
|---|---|---|
| Module has `risk_multiplier: high` | Medium risk floor applied | All 8 repos |
| Risk level = `high` | Always adds all `high_risk_always_add` categories | All 8 repos |
| cobalt_search high-risk always-add | CoreSearch, GlobalSearch, EdgeSmokeFeatures | cobalt_search |
| cobalt_website / static-content high-risk always-add | CoreSearch, GlobalSearch, EdgeSmokeFeatures, WebsiteCore1, WebsiteCore2 | cobalt_website, cobalt_static-content |
| cobalt_document_netcore high-risk always-add | Document, DocDisplay, DOCX, EdgeSmokeFeatures | cobalt_document_netcore |
| cobalt_alert high-risk always-add | AlertAccess, Alerts, EdgeSmokeFeatures | cobalt_alert |
| cobalt_foldering high-risk always-add | Foldering, ResearchOrganizer, EdgeSmokeFeatures | cobalt_foldering |
| cobalt_relatedinfo high-risk always-add | RelatedInfoContent, RelatedInfoDelivery, RelatedInfoFlags, EdgeSmokeFeatures | cobalt_relatedinfo |
| cobalt_typeahead high-risk always-add | CoreSearch, GlobalSearch, EdgeSmokeFeatures | cobalt_typeahead |
| Change type = `docs` | `skip_all: true` — no tests triggered | All 8 repos |
| Change type = `test` | Integration tests skipped (smoke only) | All 8 repos |
| Change type = `chore` | Only smoke tests run (`EdgeSmokeFeatures` or repo-specific override) | All 8 repos |
| No categories matched | Fallback: CoreSearch + EdgeSmokeFeatures | All 8 repos |

---

## 10. Known Gaps in Feature Map Coverage

| Gap | Repos affected | Priority |
|---|---|---|
| **QuickCheck 8 categories missing from static-content** | cobalt_static-content | Medium — QuickCheck has CSS/JS assets |
| Axe accessibility tests (18) not mapped anywhere | All repos | Low — these are run on a fixed schedule, not PR-driven |
| Search-UI features not in any map (~64) | cobalt_search | Low — most need a keyword or sub_path addition |

Categories missing from `feature-map-static-content.yml` that are in `feature-map-website.yml`:
```
QuickCheckUiCheckWork_1    QuickCheckUiCheckWork_2
QuickCheckUiCheckWork_3    QuickCheckUiCheckWork_4
QuickCheckUiJudicial_1     QuickCheckUiJudicial_2
QuickCheckUiJudicial_3     QuickCheckUiOpponent
```

Axe categories not in any map (18):
```
AxeIntegrationWLAustralia  AxeIntegrationWLCANext  AxeIntegrationWLNewZealand
AxeIntegrationWestlawEdgeSearch  AxeIntegrationWestlawEdgeWebsite
WLAnalyticsAxeIntegartion  WLEdgeAxeIntegration    WLMobileAxeIntegartion
WLNCorrectionalAxeTest     WLNLinksAxeIntegartion  WLPrecisionAxeIntegration
WestlawNextFeatureAxeIntegartion  WestlawPatronAxeIntegartion
CaseNoteBookAxeIntegartion  OpenWebAxe              RIandDocumnetAxeIntegration
TaxnetProAxe               WLCanadaEdgeAxe
```

---

## 11. Scaling to Additional Repositories

To add a new developer repository (e.g. `tr/cobalt_documents`):

1. **Add entry to `config/repo-registry.yml`**
   ```yaml
   - name:               cobalt_documents
     full_name:          tr/cobalt_documents
     enabled:            true
     description:        "Cobalt Documents delivery platform"
     read_token_secret:  COBALT_READ_TOKEN
     feature_map_repo:   tr/Seven-Kingdoms
     feature_map_path:   feature-map-documents.yml
     feature_map_local:  "config/feature-map-documents.yml"
     test_environment:   DEMO
     lookback_days:      7
   ```

2. **Create `config/feature-map-documents.yml`**
   Follow the same YAML schema:
   ```yaml
   version: "3.0"
   modules:
     MyModule:
       test_categories: [...]
   sub_paths:
     "MyModule/src/path/to/feature/":
       product: MyProduct
       test_categories: [...]
   keywords:
     myfeature:
       product: MyProduct
       test_categories: [...]
   high_risk_always_add:
     test_categories: [CoreSearch, GlobalSearch, EdgeSmokeFeatures]
   ```
   Run `git ls-tree HEAD --name-only` in the target repo to confirm exact top-level folder names before writing module names.

3. **Deploy both files to `tr/CobaltRegressionTesting`** (config/) and **`tr/Seven-Kingdoms`** (repo root).

4. **Merge to main** — the next manual run of pr-poller.yml picks it up automatically. No workflow code changes needed.

---

## 12. Secrets Configuration Reference

```
tr/CobaltRegressionTesting → Settings → Secrets and variables → Actions
```

| Secret name | Scope | Notes |
|---|---|---|
| `COBALT_READ_TOKEN` | read:repo on all 8 source repos | One token for all repos (same org) |
| `SEVEN_KINGDOMS_TOKEN` | read:repo on tr/Seven-Kingdoms | Used by feature_test_mapper.py API fallback |
| `REGRESSION_TRIGGER_PAT` | workflow:write on tr/CobaltRegressionTesting | Dispatches WL_DNet_*.yml workflows |

---

## 13. Quick Reference — File Locations

```
C:\SmartTestSelection\
│
├── .github\workflows\
│   └── pr-poller.yml                ← Manual dispatch; direct WL_DNet dispatch
│
├── config\
│   ├── repo-registry.yml            ← 8 monitored repos
│   ├── feature-map-search.yml       ← cobalt_search
│   ├── feature-map-website.yml      ← cobalt_website
│   ├── feature-map-static-content.yml ← cobalt_static-content
│   ├── feature-map-document.yml     ← cobalt_document_netcore
│   ├── feature-map-alert.yml        ← cobalt_alert
│   ├── feature-map-foldering.yml    ← cobalt_foldering
│   ├── feature-map-relatedinfo.yml  ← cobalt_relatedinfo
│   └── feature-map-typeahead.yml    ← cobalt_typeahead
│
└── tools\
    ├── pr_impact_analyzer.py        ← multi-repo flags; null-safe YAML
    ├── feature_test_mapper.py       ← null-safe YAML; 248 CAT_TO_WORKFLOW entries
    └── requirements.txt             ← PyGithub>=2.1.1, pyyaml>=6.0.1

Source repos — NO CHANGES ever:
  tr/cobalt_search          tr/cobalt_alert
  tr/cobalt_website         tr/cobalt_foldering
  tr/cobalt_static-content  tr/cobalt_relatedinfo
  tr/cobalt_document_netcore tr/cobalt_typeahead
```
