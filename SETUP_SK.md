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
- [Riešenie problémov](#riešenie-problémov)

---

## Google Cloud — nastavenie OAuth2

Všetky Google integrácie (Sheets, Drive, Docs) vyžadujú Google Cloud projekt s OAuth2. Vytvorí sa jeden projekt a jeden OAuth2 klient, ktorý sa použije pre všetky tri credentials v n8n.

### 1. Vytvorenie Google Cloud projektu

1. Otvorte [console.cloud.google.com](https://console.cloud.google.com).
2. Kliknite na výber projektu hore → **New Project**.
3. Zadajte názov (napr. `n8n-wordpress-extractor`) a kliknite **Create**.

### 2. Zapnutie požadovaných API

1. V ľavom menu prejdite na **APIs & Services → Library**.
2. Vyhľadajte a zapnite každé z nasledujúcich API:
   - **Google Sheets API**
   - **Google Drive API**
   - **Google Docs API**

### 3. Nastavenie OAuth consent screen

1. Prejdite na **APIs & Services → OAuth consent screen**.
2. Vyberte **External** → **Create**.
3. Vyplňte povinné polia (App name, support email, developer email).
4. V sekcii **Scopes** kliknite **Add or remove scopes** a pridajte:
   - `https://www.googleapis.com/auth/spreadsheets`
   - `https://www.googleapis.com/auth/drive`
   - `https://www.googleapis.com/auth/documents`
5. Pridajte váš Google účet ako **Test user**.
6. Uložte a pokračujte.

### 4. Vytvorenie OAuth2 credentials

1. Prejdite na **APIs & Services → Credentials**.
2. Kliknite **+ Create Credentials → OAuth 2.0 Client ID**.
3. Vyberte **Web application**.
4. V sekcii **Authorized redirect URIs** pridajte:
   ```
   https://[vas-n8n-host]/rest/oauth2-credential/callback
   ```
   Nahraďte `[vas-n8n-host]` URL adresou vašej n8n inštancie (napr. `https://n8n.vasa-firma.sk`).
5. Kliknite **Create**.
6. Skopírujte **Client ID** a **Client Secret** — budete ich potrebovať v ďalších krokoch.

---

## Google Sheets Credential

1. V n8n prejdite na **Credentials → New**.
2. Vyhľadajte a vyberte **Google Sheets OAuth2 API**.
3. Vložte **Client ID** a **Client Secret** z Google Cloud.
4. Kliknite **Sign in with Google** a autorizujte účet, ktorý má prístup k vašej tabuľke.
5. Uložte credential napr. pod názvom `Google Sheets — WP Extractor`.

### Priradenie k nodom

Tento credential priraďte ku všetkým nodom s názvom obsahujúcim „Sheets":

- Sheets Read All
- Sheets Append Pending
- Sheets Get Pending
- Sheets Update Extracted
- Sheets Mark Skip Error
- Sheets Mark Already Extracted
- Sheets Reset to Pending

---

## Google Drive Credential

1. V n8n prejdite na **Credentials → New**.
2. Vyhľadajte a vyberte **Google Drive OAuth2 API**.
3. Vložte rovnaký **Client ID** a **Client Secret**.
4. Autorizujte rovnaký Google účet.
5. Uložte credential napr. pod názvom `Google Drive — WP Extractor`.

### Priradenie k nodom

- Drive Search Doc
- Delete Drive Doc
- Docs Create (Readable GDoc)
- Docs Create HTML (HTML GDoc)

> Poznámka: nody „Docs Create" používajú Drive API (multipart upload), nie Docs API.

---

## Google Docs Credential

1. V n8n prejdite na **Credentials → New**.
2. Vyhľadajte a vyberte **Google Docs OAuth2 API**.
3. Vložte rovnaký **Client ID** a **Client Secret**.
4. Autorizujte rovnaký Google účet.
5. Uložte credential napr. pod názvom `Google Docs — WP Extractor`.

### Priradenie k nodom

- Docs Insert HTML
- Docs Insert Content

---

## WordPress Application Password

Application Passwords umožňujú n8n autentifikovať sa voči WordPress REST API bez použitia hlavného prihlasovacieho hesla.

### Vygenerovanie hesla vo WordPress

1. Prihláste sa do WordPress Adminu.
2. Prejdite na **Users → Your Profile** (alebo profil používateľa, ktorý sa má používať pre API prístup).
3. Zrolujte nadol do sekcie **Application Passwords**.
4. Zadajte názov (napr. `n8n-extractor`) a kliknite **Add New Application Password**.
5. Skopírujte vygenerované heslo okamžite — zobrazí sa iba raz.

### Požiadavky

| Požiadavka | Detail |
|---|---|
| Verzia WordPress | 5.6 alebo novší |
| REST API | Musí byť verejne dostupné na `/wp-json/wp/v2/` |
| XML Sitemaps | Musí ich generovať plugin (napr. Yoast SEO, Rank Math) |

### Nastavenie credentialu v n8n

1. V n8n prejdite na **Credentials → New**.
2. Vyhľadajte a vyberte **WordPress API**.
3. Vyplňte polia:
   - **WordPress URL:** `https://vasa-stranka.sk`
   - **Username:** vaše WordPress používateľské meno
   - **Password:** skopírované application password
4. Uložte credential.

### Priradenie k nodom

- GET Full WP Post

---

## Slack Bot Token

### Vytvorenie Slack App

1. Otvorte [api.slack.com/apps](https://api.slack.com/apps).
2. Kliknite **Create New App → From scratch**.
3. Zadajte názov (napr. `n8n-notifier`) a vyberte váš workspace.

### Nastavenie oprávnení

1. V ľavom menu prejdite na **OAuth & Permissions**.
2. V sekcii **Bot Token Scopes** kliknite **Add an OAuth Scope** a pridajte:
   - `chat:write`
   - `channels:read`

### Inštalácia aplikácie

1. Na tej istej stránke zrolujte hore a kliknite **Install to Workspace**.
2. Autorizujte aplikáciu.
3. Skopírujte **Bot User OAuth Token** (začína `xoxb-`).

### Pridanie bota do Slack kanála

V Slacku otvorte kanál, do ktorého chcete dostávať notifikácie, a zadajte:

```
/invite @nazov-vasho-bota
```

### Zistenie Channel ID

Slack nody vyžadujú Channel ID, nie názov kanála.

1. Otvorte Slack v prehliadači.
2. Prejdite do daného kanála.
3. URL bude vyzerať takto: `https://app.slack.com/client/TXXXXXXXX/CXXXXXXXXX`
4. Posledná časť (začínajúca `C`) je Channel ID.

Alternatívne: pravý klik na názov kanála → **View channel details** → ID je zobrazené na konci dialógu.

### Nastavenie credentialu v n8n

1. V n8n prejdite na **Credentials → New**.
2. Vyhľadajte a vyberte **Slack API**.
3. Vložte **Bot User OAuth Token**.
4. Uložte credential.

### Priradenie k nodom

- Slack Summary
- Slack Modified Notify
- Send a message (nič nové)

---

## Priradenie credentials k nodom

Po vytvorení všetkých credentials otvorte každý node vo workflow a vyberte správny credential z rozbaľovacieho zoznamu.

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
- Overte, či `site_url` v Set Config nemá lomku na konci.
- Otvorte `https://vasa-stranka.sk/sitemap_index.xml` v prehliadači a skontrolujte dostupnosť.
- Overte, že názvy sitemáp zodpovedajú vzorcom v node `Parse & Filter Sitemaps`.

**Drive Search Doc vracia prázdne výsledky**
- Overte `gdrive_folder_id` v Set Config.
- Skontrolujte, či autorizovaný Google účet má prístup k priečinku.

**Docs Create zlyhá s chybou 403**
- Overte, že **Google Drive API** je zapnuté v Google Cloud Console.
- Overte, že OAuth2 scope obsahuje `drive` alebo `drive.file`.

**WordPress API vracia 401**
- Vygenerujte nové Application Password v WP Admin → Users → Profile.
- Overte, že `/wp-json/wp/v2/` je verejne dostupné.
- Skontrolujte, či nejaký bezpečnostný plugin neblokuje REST API.

**Slack nedostáva správy**
- Overte, že bot má scope `chat:write`.
- Použite Channel ID (napr. `C0AT3KG6H2B`), nie názov kanála.
- Overte, že bot je pozvaný do kanála.

**Post sa stále znova extrahuje**
- Skontrolujte `date_modified` v Sheets — ak je starší ako `lastmod` zo sitemapy, post bude pri každom behu resetovaný.
- Skontrolujte, či nejaký WordPress plugin neaktualizuje `date_modified` pri každom uložení.
