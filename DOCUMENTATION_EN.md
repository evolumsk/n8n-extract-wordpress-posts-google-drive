# WordPress to Google Drive Extractor
### n8n Workflow Documentation

Automatically monitors WordPress posts via XML sitemaps, extracts their content, and saves them to Google Drive as Google Docs (readable version + clean HTML). Detects both new and modified posts.

---

## Table of Contents

- [What the workflow does](#what-the-workflow-does)
- [Architecture](#architecture)
- [Main flow](#main-flow)
- [Branch: New posts](#branch-new-posts)
- [Branch: Modified posts](#branch-modified-posts)
- [Google Cloud API](#google-cloud-api)
- [WordPress API](#wordpress-api)
- [Slack](#slack)
- [Google Sheets structure](#google-sheets-structure)
- [Google Drive file structure](#google-drive-file-structure)
- [Extraction statuses](#extraction-statuses)
- [Troubleshooting](#troubleshooting)

---

## What the workflow does

```
WordPress site  →  read sitemaps  →  compare with Sheets  →  extract content  →  save to Drive  →  notify Slack
```

| Scenario | Action |
|---|---|
| New post on the site | Add to Sheets as `pending`, create 2 Google Docs |
| Post was modified in WP | Delete old Google Docs, reset to `pending`, re-extract in the same run |
| Post already processed | Mark as `already_extracted`, skip |
| WP API returns an error | Mark as `skip_error`, continue |
| Nothing new | Send an info message to Slack |

---

## Architecture

```mermaid
graph TB
    subgraph TRIGGER["TRIGGER"]
        S[Schedule Trigger<br/>every 6h]
        M[Manual Trigger]
    end

    subgraph CONFIG["CONFIGURATION"]
        C[Set Config<br/>site_url · spreadsheet_id<br/>sheet_name · gdrive_folder_id]
    end

    subgraph SOURCES["DATA SOURCES"]
        direction LR
        WP[WordPress<br/>sitemap_index.xml]
        GS[Google Sheets<br/>memory tab]
    end

    subgraph COMPARE["COMPARISON"]
        MRG[Merge Sources]
        NEW[Find New Posts]
        MOD[Find Modified Posts]
    end

    subgraph BRANCH1["BRANCH 1 — NEW POSTS"]
        APP[Sheets Append Pending]
        GET[GET Full WP Post]
        PREP[Prepare Content]
        DOC1[Docs Create<br/>Readable GDoc]
        DOC2[Docs Create<br/>HTML GDoc]
        UPD[Sheets Update Extracted]
    end

    subgraph BRANCH2["BRANCH 2 — MODIFIED POSTS"]
        DEL[Delete Old GDocs]
        RST[Sheets Reset to Pending]
        REEXT[Re-extraction<br/>in the same run]
    end

    subgraph NOTIFY["NOTIFICATIONS"]
        SLK1[Slack — Extraction summary]
        SLK2[Slack — Modified posts]
        SLK3[Slack — Nothing new]
    end

    S --> C
    M --> C
    C --> WP
    C --> GS
    WP --> MRG
    GS --> MRG
    MRG --> NEW
    MRG --> MOD
    NEW --> APP --> GET
    GET --> PREP --> DOC1 --> DOC2 --> UPD --> SLK1
    MOD --> DEL --> RST --> SLK2
    RST --> REEXT --> DOC1
```

---

## Main flow

```mermaid
flowchart LR
    T([Trigger]) --> SC[Set Config]
    SC --> FI[Fetch Sitemap Index]
    SC --> SRA[Sheets Read All]

    FI --> PFS[Parse & Filter Sitemaps]
    PFS --> FSS[Fetch Sub-Sitemap]
    FSS --> RAP[Re-attach Post Type]
    RAP --> PSU[Parse Sub-Sitemap URLs]

    SRA --> TAG[Tag Sheet Rows]

    PSU --> MRG{Merge Sources}
    TAG --> MRG

    MRG --> FNP[Find New Posts]
    MRG --> FMP[Find Modified Posts]
```

---

## Branch: New posts

```mermaid
flowchart TD
    FNP[Find New Posts] --> IHN{IF Has\nNew Posts?}

    IHN -->|YES| APP[Sheets Append Pending]
    IHN -->|NO| SGP2[Sheets Get Pending]

    APP --> CTR[Collapse to Trigger]
    CTR --> SGP[Sheets Get Pending]
    SGP --> FPO[Filter Pending Only]
    SGP2 --> FPO

    FPO --> IHP{IF Has\nPending?}

    IHP -->|NO| SAM[Send message\n'Nothing new' to Slack]
    IHP -->|YES| GWP[GET Full WP Post]

    GWP --> PRC[Prepare Content\nHTML parser + clean]

    PRC --> ISP{IF Skip Post?\nHTTP error?}

    ISP -->|SKIP| MSE[Sheets Mark\nSkip Error]
    ISP -->|OK| DSD[Drive Search Doc\nDoes GDoc exist?]

    DSD --> IDE{IF Doc\nExists?}

    IDE -->|YES| MAE[Sheets Mark\nAlready Extracted]
    IDE -->|NO| DC1[Docs Create\nReadable GDoc]

    DC1 --> DC2[Docs Create\nHTML GDoc]
    DC2 --> DIH[Docs Insert HTML]
    DIH --> SUE[Sheets Update\nExtracted]
    SUE --> CPD[Collect Post Data]
    CPD --> AGG[Aggregate Posts]
    AGG --> BSS[Build Slack Summary]
    BSS --> IHE{IF Has\nExtractions?}
    IHE -->|YES| SLS[Slack Summary]
```

---

## Branch: Modified posts

Runs **in parallel** with Branch 1 and does not interfere with it.

```mermaid
flowchart TD
    FMP[Find Modified Posts\ncompares sitemap lastmod vs Sheets] --> IHM{IF Has\nModified?}

    IHM -->|NO| END([end of branch])
    IHM -->|YES| PDI[Prepare Doc IDs\ngdoc_id + gdoc_html_id]

    PDI --> DDD[Delete Drive Doc\ncontinueOnFail: true]

    DDD --> DS[Deduplicate Slugs\n2 doc-items → 1 item per slug]

    DS --> SRP[Sheets Reset to Pending\nextraction_status = pending\nClears gdoc_id, gdoc_url,\ngdoc_html_id, gdoc_html_url, extracted_at]

    SRP -->|branch A| BMM[Build Modified Message]
    SRP -->|branch B| GWP[GET Full WP Post\nIMMEDIATE RE-EXTRACTION]

    BMM --> SMN[Slack Modified Notify]
    GWP --> PREP[Prepare Content]
    PREP --> DC1[Docs Create Readable]
    DC1 --> DC2[Docs Create HTML]
    DC2 --> SUE[Sheets Update Extracted]
```

**Key:** change detection = `sitemap lastmod` > `sheet date_modified`

---

## Google Cloud API

All Google integrations (Sheets, Drive, Docs) require a Google Cloud project with OAuth2.

### Creating a project and OAuth2 credentials

```mermaid
flowchart LR
    A[console.cloud.google.com] --> B[New project]
    B --> C[APIs & Services\n→ Enable APIs]
    C --> D[Google Sheets API\nGoogle Drive API\nGoogle Docs API]
    D --> E[Credentials\n→ Create OAuth 2.0\nClient ID]
    E --> F[Web application\nRedirect URI:\nn8n callback URL]
    F --> G[Copy\nClient ID\n+ Client Secret]
    G --> H[n8n Credentials\n→ New → Google *]
```

**Redirect URI for n8n:**
```
https://[your-n8n-host]/rest/oauth2-credential/callback
```

### Credentials in n8n

Create a separate credential for each Google service:

| Credential type | Used in node |
|---|---|
| Google Sheets OAuth2 API | Sheets Read All, Append, Update, Get, Mark... |
| Google Drive OAuth2 API | Drive Search Doc, Delete Drive Doc, Docs Create* |
| Google Docs OAuth2 API | Docs Insert HTML, Docs Insert Content |

> `Docs Create` and `Docs Create HTML` use the **Drive API** (not Docs API) — they upload via multipart upload.

---

## WordPress API

The workflow reads post content via the WordPress REST API.

### Setting up an Application Password

```mermaid
flowchart LR
    A[WP Admin] --> B[Users → Your Profile]
    B --> C[Application Passwords\n→ Add New]
    C --> D[Name: n8n-api\n→ Add Application Password]
    D --> E[Copy the password\nshown only once!]
    E --> F[n8n → Credentials\n→ New → WordPress API]
    F --> G[URL: https://your-site.com\nUsername: wp-login\nPassword: copied password]
```

### WordPress requirements

| Requirement | Detail |
|---|---|
| WordPress REST API | Must be publicly accessible at `/wp-json/wp/v2/` |
| sitemap_index.xml | Generated by a plugin (e.g. Yoast, Rank Math) |
| Post types in sitemaps | `post-sitemap.xml`, custom CPT sitemaps |
| Application Passwords | WordPress 5.6+ |

### Sitemaps — which post types are read

The workflow looks for these patterns in `sitemap_index.xml` (configurable in the `Parse & Filter Sitemaps` node):

```
post-sitemap.xml           →  post_type: posts
terminologia-sitemap.xml   →  post_type: terminologia
hub-destinacii-sitemap.xml →  post_type: hub-destinacii
```

For other post types, edit the `Parse & Filter Sitemaps` node and add a pattern to the `relevantPatterns` array.

---

## Slack

The workflow sends 3 types of messages:

| Message | When | Content |
|---|---|---|
| Extraction summary | After a successful run | List of extracted posts with links to Google Docs |
| Modified posts | When a change is detected | List of slugs + new modification date |
| Nothing new | When there are no pending posts | Simple info message |

### Setting up a Slack App

```mermaid
flowchart LR
    A[api.slack.com/apps] --> B[Create New App\nFrom scratch]
    B --> C[OAuth & Permissions\n→ Bot Token Scopes]
    C --> D[chat:write\nchannels:read]
    D --> E[Install to Workspace]
    E --> F[Copy\nBot User OAuth Token]
    F --> G[n8n → Credentials\n→ New → Slack OAuth2 API]
    G --> H[Add bot to channel\n/invite @bot-name]
```

---

## Google Sheets structure

### Creating the spreadsheet

1. Create a new spreadsheet at [sheets.google.com](https://sheets.google.com).
2. Rename the tab to `memory`.
3. Paste the following header into row 1:

```
wp_id	post_type	slug	title	status	date_published	date_modified	link	extraction_status	gdoc_id	gdoc_url	gdoc_html_id	gdoc_html_url	extracted_at	site_url
```

### Columns and their purpose

| Column | When filled | Description |
|---|---|---|
| `wp_id` | On extraction | WordPress post ID |
| `post_type` | On sitemap write | `posts`, `terminologia`, custom CPT |
| `slug` | On sitemap write | URL slug — **key column for matching** |
| `title` | On extraction | Post title |
| `status` | On write | Always `publish` |
| `date_published` | From sitemap | Date of first publication |
| `date_modified` | From sitemap / on change | Date of last modification |
| `link` | From sitemap | Full post URL |
| `extraction_status` | Automatically | See status table below |
| `gdoc_id` | After extraction | Readable GDoc ID |
| `gdoc_url` | After extraction | Link to Readable GDoc |
| `gdoc_html_id` | After extraction | HTML GDoc ID |
| `gdoc_html_url` | After extraction | Link to HTML GDoc |
| `extracted_at` | After extraction | Timestamp of last extraction |
| `site_url` | On write | Site label from Set Config |

---

## Google Drive file structure

For each WordPress post, **2 Google Docs** are created in one folder:

```
your-folder/
├── [123] Post Title               ← Readable GDoc
│       Content: readable text + metadata
│       Use: review, AI processing
│
└── [123] Post Title - HTML        ← HTML GDoc
        Content: clean HTML for WordPress
        Use: copy-paste back into WordPress
```

### What the Readable GDoc contains

```
[Title]
─────────────────────
URL:         https://...
WP ID:       123
Slug:        post-slug
Type:        posts
Published:   2024-01-15
Modified:    2024-03-22
Site:        your-site.com
─────────────────────
[clean content without images, scripts, or shortcodes]
```

### What the HTML GDoc contains

```html
<!-- WP ID: 123 | SLUG: post-slug | TYPE: posts -->
<!-- TITLE: Post Title -->
<!-- URL: https://... -->
<!-- PUBLISHED: 2024-01-15 | MODIFIED: 2024-03-22 -->

<h2>Section</h2>
<p>Content...</p>
<table>...</table>
```

---

## Extraction statuses

```mermaid
stateDiagram-v2
    [*] --> pending : New post detected\nfrom sitemap

    pending --> extracted : Google Docs successfully created
    pending --> already_extracted : GDoc already existed in Drive
    pending --> skip_error : WP API returned an error

    extracted --> pending : Post modified in WP\n(branch 2 — reset)
    already_extracted --> pending : Post modified in WP\n(branch 2 — reset)

    extracted --> [*]
    already_extracted --> [*]
    skip_error --> [*]
```

| Status | Description |
|---|---|
| `pending` | Waiting for extraction |
| `extracted` | Google Docs successfully created |
| `already_extracted` | GDoc already existed in Drive (skipped, Sheets updated) |
| `skip_error` | WP API returned 404/5xx or post does not exist |

---

## Troubleshooting

### Sitemap not loading

```
[Parse Sitemaps] No relevant sitemaps found
```
- Check `site_url` in Set Config (no trailing slash).
- Open `https://your-site.com/sitemap_index.xml` in a browser.
- Check sitemap names — they must match the patterns in `Parse & Filter Sitemaps`.

### Drive Search Doc returns empty results

- Check `gdrive_folder_id` in Set Config.
- Confirm the OAuth2 account has access to the folder.
- The folder must be owned by or shared with the Google account used in Drive credentials.

### Docs Create fails with 403

- Check that **Google Drive API** is enabled in Google Cloud Console.
- Check that the OAuth2 consent screen has the correct scopes: `drive.file` or `drive`.

### WP API returns 401

- Regenerate the Application Password in WP Admin → Users → Profile.
- Check `site_url` — `/wp-json/wp/v2/` must be accessible.
- Check whether the WordPress REST API is disabled by a plugin.

### Slack not receiving messages

- Check that the bot has the `chat:write` scope.
- Use the Channel ID in Slack nodes (not the channel name, but an ID like `C0AT3KG6H2B`).
- Check that the bot is invited to the channel: `/invite @bot-name`.

### Post keeps being re-extracted

- Check `date_modified` in Sheets — if it is older than the sitemap `lastmod`, the post will be reset.
- Check whether a WordPress plugin is updating the modification date on every save.
