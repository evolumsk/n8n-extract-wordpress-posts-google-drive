# WordPress → Google Drive Extractor
### Dokumentácia n8n workflow

Workflow automaticky sleduje WordPress príspevky cez XML sitemapy, extrahuje ich obsah a ukladá do Google Drive ako Google Docs (čitateľná verzia + čisté HTML). Detekuje nové aj upravené príspevky.

---

## Obsah

- [Čo workflow robí](#čo-workflow-robí)
- [Architektúra](#architektúra)
- [Hlavný flow](#hlavný-flow)
- [Vetva: Nové príspevky](#vetva-nové-príspevky)
- [Vetva: Upravené príspevky](#vetva-upravené-príspevky)
- [Google Cloud API](#google-cloud-api)
- [WordPress API](#wordpress-api)
- [Slack](#slack)
- [Štruktúra Google Sheets](#štruktúra-google-sheets)
- [Štruktúra súborov v Google Drive](#štruktúra-súborov-v-google-drive)
- [Stavy extrakcie](#stavy-extrakcie)
- [Riešenie problémov](#riešenie-problémov)

---

## Čo workflow robí

```
WordPress web  →  čítanie sitemáp  →  porovnanie so Sheets  →  extrakcia obsahu  →  uloženie do Drive  →  notifikácia Slack
```

| Scenár | Akcia |
|---|---|
| Nový príspevok na webe | Pridanie do Sheets ako `pending`, vytvorenie 2 Google Docs |
| Príspevok bol upravený vo WP | Zmazanie starých Google Docs, reset na `pending`, re-extrakcia v tom istom behu |
| Príspevok už bol spracovaný | Označenie ako `already_extracted`, preskočenie |
| WP API vráti chybu | Označenie ako `skip_error`, pokračovanie ďalej |
| Nič nové | Odoslanie info správy do Slacku |

---

## Architektúra

```mermaid
graph TB
    subgraph TRIGGER["SPUSTENIE"]
        S[Schedule Trigger<br/>každých 6h]
        M[Manual Trigger]
    end

    subgraph CONFIG["KONFIGURÁCIA"]
        C[Set Config<br/>site_url · spreadsheet_id<br/>sheet_name · gdrive_folder_id]
    end

    subgraph SOURCES["ZDROJE DÁT"]
        direction LR
        WP[WordPress<br/>sitemap_index.xml]
        GS[Google Sheets<br/>záložka memory]
    end

    subgraph COMPARE["POROVNANIE"]
        MRG[Merge Sources]
        NEW[Find New Posts]
        MOD[Find Modified Posts]
    end

    subgraph BRANCH1["VETVA 1 — NOVÉ PRÍSPEVKY"]
        APP[Sheets Append Pending]
        GET[GET Full WP Post]
        PREP[Prepare Content]
        DOC1[Docs Create<br/>Readable GDoc]
        DOC2[Docs Create<br/>HTML GDoc]
        UPD[Sheets Update Extracted]
    end

    subgraph BRANCH2["VETVA 2 — UPRAVENÉ PRÍSPEVKY"]
        DEL[Delete Old GDocs]
        RST[Sheets Reset to Pending]
        REEXT[Re-extrakcia<br/>v tom istom behu]
    end

    subgraph NOTIFY["NOTIFIKÁCIE"]
        SLK1[Slack — Súhrn extrakcie]
        SLK2[Slack — Upravené príspevky]
        SLK3[Slack — Nič nové]
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

## Hlavný flow

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

## Vetva: Nové príspevky

```mermaid
flowchart TD
    FNP[Find New Posts] --> IHN{IF Has\nNew Posts?}

    IHN -->|ÁNO| APP[Sheets Append Pending]
    IHN -->|NIE| SGP2[Sheets Get Pending]

    APP --> CTR[Collapse to Trigger]
    CTR --> SGP[Sheets Get Pending]
    SGP --> FPO[Filter Pending Only]
    SGP2 --> FPO

    FPO --> IHP{IF Has\nPending?}

    IHP -->|NIE| SAM[Odoslanie správy\nNič nové do Slacku]
    IHP -->|ÁNO| GWP[GET Full WP Post]

    GWP --> PRC[Prepare Content\nHTML parser + clean]

    PRC --> ISP{IF Skip Post?\nHTTP chyba?}

    ISP -->|SKIP| MSE[Sheets Mark\nSkip Error]
    ISP -->|OK| DSD[Drive Search Doc\nExistuje GDoc?]

    DSD --> IDE{IF Doc\nExists?}

    IDE -->|ÁNO| MAE[Sheets Mark\nAlready Extracted]
    IDE -->|NIE| DC1[Docs Create\nReadable GDoc]

    DC1 --> DC2[Docs Create\nHTML GDoc]
    DC2 --> DIH[Docs Insert HTML]
    DIH --> SUE[Sheets Update\nExtracted]
    SUE --> CPD[Collect Post Data]
    CPD --> AGG[Aggregate Posts]
    AGG --> BSS[Build Slack Summary]
    BSS --> IHE{IF Has\nExtractions?}
    IHE -->|ÁNO| SLS[Slack Summary]
```

---

## Vetva: Upravené príspevky

Beží **paralelne** s Vetvou 1 a neruší ju.

```mermaid
flowchart TD
    FMP[Find Modified Posts\nporovnáva lastmod sitemapy vs Sheets] --> IHM{IF Has\nModified?}

    IHM -->|NIE| END([koniec vetvy])
    IHM -->|ÁNO| PDI[Prepare Doc IDs\ngdoc_id + gdoc_html_id]

    PDI --> DDD[Delete Drive Doc\ncontinueOnFail: true]

    DDD --> DS[Deduplicate Slugs\n2 doc-items → 1 item per slug]

    DS --> SRP[Sheets Reset to Pending\nextraction_status = pending\nVymaže gdoc_id, gdoc_url,\ngdoc_html_id, gdoc_html_url, extracted_at]

    SRP -->|vetva A| BMM[Build Modified Message]
    SRP -->|vetva B| GWP[GET Full WP Post\nOKAMŽITÁ RE-EXTRAKCIA]

    BMM --> SMN[Slack Modified Notify]
    GWP --> PREP[Prepare Content]
    PREP --> DC1[Docs Create Readable]
    DC1 --> DC2[Docs Create HTML]
    DC2 --> SUE[Sheets Update Extracted]
```

**Kľúčové:** detekcia zmeny = `sitemap lastmod` > `sheet date_modified`

---

## Google Cloud API

Všetky Google integrácie (Sheets, Drive, Docs) vyžadujú Google Cloud projekt s OAuth2.

### Vytvorenie projektu a OAuth2

```mermaid
flowchart LR
    A[console.cloud.google.com] --> B[Nový projekt]
    B --> C[APIs & Services\n→ Enable APIs]
    C --> D[Google Sheets API\nGoogle Drive API\nGoogle Docs API]
    D --> E[Credentials\n→ Create OAuth 2.0\nClient ID]
    E --> F[Web application\nRedirect URI:\nn8n callback URL]
    F --> G[Skopírovanie\nClient ID\n+ Client Secret]
    G --> H[n8n Credentials\n→ New → Google *]
```

**Redirect URI pre n8n:**
```
https://[vas-n8n-host]/rest/oauth2-credential/callback
```

### Credentials v n8n

Pre každú Google službu je potrebné vytvoriť samostatný credential:

| Typ credentialu | Použitie v node |
|---|---|
| Google Sheets OAuth2 API | Sheets Read All, Append, Update, Get, Mark... |
| Google Drive OAuth2 API | Drive Search Doc, Delete Drive Doc, Docs Create* |
| Google Docs OAuth2 API | Docs Insert HTML, Docs Insert Content |

> `Docs Create` a `Docs Create HTML` používajú **Drive API** (nie Docs API) — nahrávajú cez multipart upload.

---

## WordPress API

Workflow číta obsah príspevkov cez WordPress REST API.

### Nastavenie Application Password

```mermaid
flowchart LR
    A[WP Admin] --> B[Users → Your Profile]
    B --> C[Application Passwords\n→ Add New]
    C --> D[Názov: n8n-api\n→ Add Application Password]
    D --> E[Skopírovanie hesla\nzobrazí sa len raz]
    E --> F[n8n → Credentials\n→ New → WordPress API]
    F --> G[URL: https://vasa-stranka.sk\nUsername: wp-login\nPassword: skopírované heslo]
```

### Požiadavky na WordPress

| Požiadavka | Detail |
|---|---|
| WordPress REST API | Musí byť verejne dostupné na `/wp-json/wp/v2/` |
| sitemap_index.xml | Generovaný pluginom (napr. Yoast, Rank Math) |
| Typy príspevkov v sitemape | `post-sitemap.xml`, vlastné CPT sitemaps |
| Application Passwords | WordPress 5.6+ |

### Sitemaps — aké typy príspevkov sa čítajú

Workflow hľadá v `sitemap_index.xml` tieto vzorce (upraviteľné v node `Parse & Filter Sitemaps`):

```
post-sitemap.xml           →  post_type: posts
terminologia-sitemap.xml   →  post_type: terminologia
hub-destinacii-sitemap.xml →  post_type: hub-destinacii
```

Pre iné typy príspevkov je potrebné upraviť node `Parse & Filter Sitemaps` — pridať vzorec do poľa `relevantPatterns`.

---

## Slack

Workflow posiela 3 typy správ:

| Správa | Kedy | Obsah |
|---|---|---|
| Súhrn extrakcie | Po úspešnom behu | Zoznam extrahovaných príspevkov s odkazmi na Google Docs |
| Upravené príspevky | Pri detekcii zmeny | Zoznam slugov + nový dátum zmeny |
| Nič nové | Keď nie sú pending príspevky | Jednoduchá info správa |

### Nastavenie Slack App

```mermaid
flowchart LR
    A[api.slack.com/apps] --> B[Create New App\nFrom scratch]
    B --> C[OAuth & Permissions\n→ Bot Token Scopes]
    C --> D[chat:write\nchannels:read]
    D --> E[Install to Workspace]
    E --> F[Skopírovanie\nBot User OAuth Token]
    F --> G[n8n → Credentials\n→ New → Slack OAuth2 API]
    G --> H[Pridanie bota do kanála\n/invite @nazov-bota]
```

---

## Štruktúra Google Sheets

### Vytvorenie tabuľky

1. Vytvorte novú tabuľku na [sheets.google.com](https://sheets.google.com).
2. Premenujte záložku na `memory`.
3. Vložte nasledujúcu hlavičku do riadku 1:

```
wp_id	post_type	slug	title	status	date_published	date_modified	link	extraction_status	gdoc_id	gdoc_url	gdoc_html_id	gdoc_html_url	extracted_at	site_url
```

### Stĺpce a ich účel

| Stĺpec | Kedy sa vyplní | Popis |
|---|---|---|
| `wp_id` | Pri extrakcii | WordPress ID príspevku |
| `post_type` | Pri zápise zo sitemapy | `posts`, `terminologia`, vlastný CPT |
| `slug` | Pri zápise zo sitemapy | URL slug — **kľúčový stĺpec pre matching** |
| `title` | Pri extrakcii | Titulok príspevku |
| `status` | Pri zápise | Vždy `publish` |
| `date_published` | Zo sitemapy | Dátum prvého publikovania |
| `date_modified` | Zo sitemapy / pri zmene | Dátum poslednej úpravy |
| `link` | Zo sitemapy | Plná URL príspevku |
| `extraction_status` | Automaticky | Pozri tabuľku stavov nižšie |
| `gdoc_id` | Po extrakcii | ID Readable GDoc |
| `gdoc_url` | Po extrakcii | Odkaz na Readable GDoc |
| `gdoc_html_id` | Po extrakcii | ID HTML GDoc |
| `gdoc_html_url` | Po extrakcii | Odkaz na HTML GDoc |
| `extracted_at` | Po extrakcii | Timestamp poslednej extrakcie |
| `site_url` | Pri zápise | Label webu z Set Config |

---

## Štruktúra súborov v Google Drive

Pre každý WordPress príspevok vzniknú **2 Google Docs** v jednom priečinku:

```
vas-priecinok/
├── [123] Titulok článku               ← Readable GDoc
│       Obsah: čitateľný text + metadáta
│       Použitie: review, AI spracovanie
│
└── [123] Titulok článku – HTML        ← HTML GDoc
        Obsah: čisté HTML pre WordPress
        Použitie: copy-paste späť do WordPress
```

### Čo obsahuje Readable GDoc

```
[Titulok]
─────────────────────
URL:         https://...
WP ID:       123
Slug:        nazov-clanku
Typ:         posts
Publikované: 2024-01-15
Upravené:    2024-03-22
Web:         vasa-stranka.sk
─────────────────────
[čistý obsah bez obrázkov, skriptov, shortcodes]
```

### Čo obsahuje HTML GDoc

```html
<!-- WP ID: 123 | SLUG: nazov-clanku | TYP: posts -->
<!-- TITULOK: Titulok článku -->
<!-- URL: https://... -->
<!-- PUBLIKOVANÉ: 2024-01-15 | UPRAVENÉ: 2024-03-22 -->

<h2>Sekcia</h2>
<p>Obsah...</p>
<table>...</table>
```

---

## Stavy extrakcie

```mermaid
stateDiagram-v2
    [*] --> pending : Nový príspevok detekovaný\nzo sitemapy

    pending --> extracted : Google Docs úspešne vytvorené
    pending --> already_extracted : GDoc už existoval v Drive
    pending --> skip_error : WP API vrátilo chybu

    extracted --> pending : Príspevok upravený vo WP\n(vetva 2 — reset)
    already_extracted --> pending : Príspevok upravený vo WP\n(vetva 2 — reset)

    extracted --> [*]
    already_extracted --> [*]
    skip_error --> [*]
```

| Stav | Popis |
|---|---|
| `pending` | Čaká na extrakciu |
| `extracted` | Google Docs úspešne vytvorené |
| `already_extracted` | GDoc existoval v Drive (preskočený, Sheets aktualizovaný) |
| `skip_error` | WP API vrátilo 404/5xx alebo príspevok neexistuje |

---

## Riešenie problémov

**Sitemap sa nenačíta**
- Overte `site_url` v Set Config — nesmie obsahovať lomku na konci.
- Skontrolujte `https://vasa-stranka.sk/sitemap_index.xml` v prehliadači.
- Overte názvy sitemáp — musia zodpovedať vzorcom v node `Parse & Filter Sitemaps`.

**Drive Search Doc vracia prázdne výsledky**
- Overte `gdrive_folder_id` v Set Config.
- Skontrolujte, či má OAuth2 účet prístup k priečinku.
- Priečinok musí byť vlastnený alebo zdieľaný na použitý Google účet.

**Docs Create zlyhá s chybou 403**
- Overte, že `Google Drive API` je zapnuté v Google Cloud Console.
- Overte, že OAuth2 consent screen má správne scopes: `drive.file` alebo `drive`.

**WP API vracia 401**
- Vygenerujte nové Application Password v WP Admin → Users → Profile.
- Overte `site_url` — `/wp-json/wp/v2/` musí byť dostupné.
- Skontrolujte, či REST API nie je zakázané pluginom.

**Slack nedostáva správy**
- Overte, že bot má scope `chat:write`.
- Použite Channel ID v Slack nodoch (napr. `C0AT3KG6H2B`), nie názov kanála.
- Skontrolujte, že bot je pozvaný do kanála: `/invite @nazov-bota`.

**Príspevok sa stále znova extrahuje**
- Skontrolujte `date_modified` v Sheets — ak je starší ako `lastmod` zo sitemapy, bude resetovaný.
- Skontrolujte, či WordPress plugin nepribúda dátum úpravy pri každom uložení.
