# Návod na nastavenie

Postup získania všetkých potrebných API kľúčov a konfigurácie prihlasovacích údajov v n8n.

---

## Obsah

- [Google Cloud — nastavenie OAuth2](#google-cloud--nastavenie-oauth2)
- [Google Sheets credential](#google-sheets-credential)
- [Google Drive credential](#google-drive-credential)
- [Google Docs credential](#google-docs-credential)
- [WordPress Application Password](#wordpress-application-password)
- [Slack Bot Token](#slack-bot-token)
- [Priradenie credentials k nodom](#priradenie-credentials-k-nodom)

---

## Google Cloud — nastavenie OAuth2

Všetky Google integrácie (Sheets, Drive, Docs) vyžadujú Google Cloud projekt s OAuth2. Vytvoríš jeden projekt a jeden OAuth2 klient, ktorý použiješ pre všetky tri credentials v n8n.

### 1. Vytvor Google Cloud projekt

1. Choď na [console.cloud.google.com](https://console.cloud.google.com).
2. Klikni na výber projektu hore → **New Project**.
3. Zadaj názov (napr. `n8n-wordpress-extractor`) a klikni **Create**.

### 2. Zapni požadované API

1. V ľavom menu choď na **APIs & Services → Library**.
2. Vyhľadaj a zapni každé z nasledujúcich API:
   - **Google Sheets API**
   - **Google Drive API**
   - **Google Docs API**

### 3. Nastav OAuth consent screen

1. Choď na **APIs & Services → OAuth consent screen**.
2. Vyber **External** → **Create**.
3. Vyplň povinné polia (App name, support email, developer email).
4. V sekcii **Scopes** klikni **Add or remove scopes** a pridaj:
   - `https://www.googleapis.com/auth/spreadsheets`
   - `https://www.googleapis.com/auth/drive`
   - `https://www.googleapis.com/auth/documents`
5. Pridaj svoj Google účet ako **Test user**.
6. Ulož a pokračuj.

### 4. Vytvor OAuth2 credentials

1. Choď na **APIs & Services → Credentials**.
2. Klikni **+ Create Credentials → OAuth 2.0 Client ID**.
3. Vyber **Web application**.
4. V sekcii **Authorized redirect URIs** pridaj:

```
https://[tvoj-n8n-host]/rest/oauth2-credential/callback
```

Nahraď `[tvoj-n8n-host]` URL adresou tvojej n8n inštancie (napr. `https://n8n.tvoja-firma.sk`).

5. Klikni **Create**.
6. Skopíruj **Client ID** a **Client Secret** — budeš ich potrebovať v ďalších krokoch.

---

## Google Sheets Credential

1. V n8n choď na **Credentials → New**.
2. Vyhľadaj a vyber **Google Sheets OAuth2 API**.
3. Vlož **Client ID** a **Client Secret** z Google Cloud.
4. Klikni **Sign in with Google** a autorizuj účet, ktorý má prístup k tvojej tabuľke.
5. Ulož credential napr. pod názvom `Google Sheets — WP Extractor`.

### Priraď k nodom

Tento credential priraď ku všetkým nodom s názvom obsahujúcim „Sheets":

- Sheets Read All
- Sheets Append Pending
- Sheets Get Pending
- Sheets Update Extracted
- Sheets Mark Skip Error
- Sheets Mark Already Extracted
- Sheets Reset to Pending

---

## Google Drive Credential

1. V n8n choď na **Credentials → New**.
2. Vyhľadaj a vyber **Google Drive OAuth2 API**.
3. Vlož rovnaký **Client ID** a **Client Secret**.
4. Autorizuj rovnaký Google účet.
5. Ulož credential napr. pod názvom `Google Drive — WP Extractor`.

### Priraď k nodom

- Drive Search Doc
- Delete Drive Doc
- Docs Create (Readable GDoc)
- Docs Create HTML (HTML GDoc)

> Poznámka: nody „Docs Create" používajú Drive API (multipart upload), nie Docs API.

---

## Google Docs Credential

1. V n8n choď na **Credentials → New**.
2. Vyhľadaj a vyber **Google Docs OAuth2 API**.
3. Vlož rovnaký **Client ID** a **Client Secret**.
4. Autorizuj rovnaký Google účet.
5. Ulož credential napr. pod názvom `Google Docs — WP Extractor`.

### Priraď k nodom

- Docs Insert HTML
- Docs Insert Content

---

## WordPress Application Password

Application Passwords umožňujú n8n autentifikovať sa voči WordPress REST API bez použitia hlavného prihlasovacieho hesla.

### Vygeneruj heslo vo WordPress

1. Prihlás sa do WordPress Adminu.
2. Choď na **Users → Your Profile** (alebo profil používateľa, ktorého chceš použiť pre API prístup).
3. Zrolovaj nadol do sekcie **Application Passwords**.
4. Zadaj názov (napr. `n8n-extractor`) a klikni **Add New Application Password**.
5. Skopíruj vygenerované heslo okamžite — zobrazí sa iba raz.

### Požiadavky

| Požiadavka | Detail |
|---|---|
| Verzia WordPress | 5.6 alebo novší |
| REST API | Musí byť verejne dostupné na `/wp-json/wp/v2/` |
| XML Sitemaps | Musí ich generovať plugin (napr. Yoast SEO, Rank Math) |

### Nastav credential v n8n

1. V n8n choď na **Credentials → New**.
2. Vyhľadaj a vyber **WordPress API**.
3. Vyplň polia:
   - **WordPress URL:** `https://tvoja-stranka.sk`
   - **Username:** tvoje WordPress používateľské meno
   - **Password:** skopírované application password
4. Ulož credential.

### Priraď k nodom

- GET Full WP Post

---

## Slack Bot Token

### Vytvor Slack App

1. Choď na [api.slack.com/apps](https://api.slack.com/apps).
2. Klikni **Create New App → From scratch**.
3. Zadaj názov (napr. `n8n-notifier`) a vyber svoj workspace.

### Nastav oprávnenia

1. V ľavom menu choď na **OAuth & Permissions**.
2. V sekcii **Bot Token Scopes** klikni **Add an OAuth Scope** a pridaj:
   - `chat:write`
   - `channels:read`

### Nainštaluj aplikáciu

1. Na tej istej stránke zrolovaj hore a klikni **Install to Workspace**.
2. Autorizuj aplikáciu.
3. Skopíruj **Bot User OAuth Token** (začína `xoxb-`).

### Pridaj bota do Slack kanála

V Slacku otvor kanál, do ktorého chceš dostávať notifikácie, a zadaj:

```
/invite @nazov-tvojho-bota
```

### Zisti Channel ID

Slack nody vyžadujú Channel ID, nie názov kanála.

1. Otvor Slack v prehliadači.
2. Prejdi do daného kanála.
3. URL bude vyzerať takto: `https://app.slack.com/client/TXXXXXXXX/CXXXXXXXXX`
4. Posledná časť (začínajúca `C`) je Channel ID.

Alternatívne: pravý klik na názov kanála → **View channel details** → ID je zobrazené na konci dialógu.

### Nastav credential v n8n

1. V n8n choď na **Credentials → New**.
2. Vyhľadaj a vyber **Slack API**.
3. Vlož **Bot User OAuth Token**.
4. Ulož credential.

### Priraď k nodom

- Slack Summary
- Slack Modified Notify
- Send a message (nič nové)

---

## Priradenie credentials k nodom

Po vytvorení všetkých credentials otvor každý node vo workflow a vyber správny credential z rozbaľovacieho zoznamu.

| Názov nodu | Typ credentialu |
|---|---|
| Sheets Read All | Google Sheets OAuth2 API |
| Sheets Append Pending | Google Sheets OAuth2 API |
| Sheets Get Pending | Google Sheets OAuth2 API |
| Sheets Update Extracted | Google Sheets OAuth2 API |
| Sheets Mark Skip Error | Google Sheets OAuth2 API |
| Sheets Mark Already Extracted | Google Sheets OAuth2 API |
| Sheets Reset to Pending | Google Sheets OAuth2 API |
| Drive Search Doc | Google Drive OAuth2 API |
| Delete Drive Doc | Google Drive OAuth2 API |
| Docs Create (Readable) | Google Drive OAuth2 API |
| Docs Create HTML | Google Drive OAuth2 API |
| Docs Insert HTML | Google Docs OAuth2 API |
| Docs Insert Content | Google Docs OAuth2 API |
| GET Full WP Post | WordPress API |
| Slack Summary | Slack API |
| Slack Modified Notify | Slack API |
| Send a message (nič nové) | Slack API |

---

## Riešenie problémov

**Sitemap sa nenačíta**
- Over, či `site_url` v Set Config nemá lomku na konci.
- Otvor `https://tvoja-stranka.sk/sitemap_index.xml` v prehliadači a skontroluj, či je dostupná.
- Over, že názvy sitemáp zodpovedajú vzorcom v node `Parse & Filter Sitemaps`.

**Drive Search Doc vracia prázdne výsledky**
- Over `gdrive_folder_id` v Set Config.
- Skontroluj, či autorizovaný Google účet má prístup k priečinku.

**Docs Create zlyhá s chybou 403**
- Over, že **Google Drive API** je zapnuté v Google Cloud Console.
- Over, že OAuth2 scope obsahuje `drive` alebo `drive.file`.

**WordPress API vracia 401**
- Vygeneruj nové Application Password v WP Admin → Users → Profile.
- Over, že `/wp-json/wp/v2/` je verejne dostupné.
- Skontroluj, či nejaký bezpečnostný plugin neblokuje REST API.

**Slack nedostáva správy**
- Over, že bot má scope `chat:write`.
- Použi Channel ID (napr. `C0AT3KG6H2B`), nie názov kanála.
- Over, že bot je pozvaný do kanála.

**Post sa stále znova extrahuje**
- Skontroluj `date_modified` v Sheets — ak je starší ako `lastmod` zo sitemapy, post bude pri každom behu resetovaný.
- Skontroluj, či nejaký WordPress plugin neaktualizuje `date_modified` pri každom uložení.
