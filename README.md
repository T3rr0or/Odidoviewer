# Odido Account Dump Viewer

A single-file HTML5 viewer for the Odido/T-Mobile NL Salesforce CRM data breach dump. Drop it in a browser, load your files, and you have a searchable, filterable interface over millions of records — no server, no installation, no dependencies.

> ⚠️ **This tool is intended for security researchers, CTI analysts, and incident responders working with this dataset in a professional capacity. Handle the data in accordance with applicable privacy laws (AVG/GDPR). Do not share, publish, or distribute the underlying data.**

---

## What this is

The Odido breach exposed a Salesforce Account export containing personal data of Dutch telecom customers. Each record holds up to 257 fields including names, addresses, phone numbers, email addresses, account status, billing details, and a full communication history log. The raw dump is an NDJSON file (one JSON object per line) and is completely unreadable without tooling.

This viewer solves that. It is one `.html` file. You open it in Chrome, load your dump files, and work.

---

## Features

### Import
- Drag and drop or click to select files
- Queue up to 21 files at once and walk away — they load sequentially without any interaction
- Only `.txt` files with `odido` in the filename are accepted; everything else is silently skipped
- Files are streamed in 4 MB chunks via a Web Worker so the browser stays fully responsive during loading
- A per-file and overall progress bar with ETA updates as files parse
- Duplicate records (by Salesforce ID) are automatically skipped when loading multiple files
- The table becomes usable as soon as the first file finishes, while the rest continue loading in the background

### Filters and search
- Free-text search across name, city, postal code, Salesforce ID, and source filename
- Dropdown slicers for Status, Sales Channel, City, Year Created, and Source File
- All dropdowns are rebuilt dynamically after each import — they always reflect exactly what is loaded
- Sorting on every column, click again to reverse
- Paginated table (250 records per page) so the DOM stays fast even with 600k+ records loaded

### Record detail panel
Click **View** on any row to open the detail panel. The full record is read on demand by seeking directly to the relevant bytes in the source file — no need to keep 257 fields per record in memory.

The detail panel has two tabs.

#### Fields tab
Displays all non-empty fields organized into sections:

| Section | What it contains |
|---|---|
| Core | ID, name, type, status, brand, segment, portal flags, sensitivity permissions |
| Address | Street, house number, extension, city, postal code, country code |
| Contact | Email address (`vlocity_cmt__BillingEmailAddress__c`), phone, fax |
| Account Info | Sales channel, subscriptions, open opportunities, hierarchy level |
| Dates | Created, modified, last activity, inactivation date |
| Payment | Payment method, terms, bank details, paper invoice flag |
| System IDs | Salesforce and legacy system identifiers |
| All Other Fields | Any remaining field that has a value |

Boolean fields (`true`/`false`) are handled properly: `true` displays as **Yes**, `false` is hidden to reduce noise.

#### Activity Log tab

This is where you find the complete communication history between Odido and the customer.

The raw data is stored in a single field called `SObjectLog__c`. It is a concatenated string of log entries with no delimiter between them — entries are only separated by the timestamp that starts each one. Each entry follows this structure:

```
DD-MM-YYYY HH:MM:SS - actor - direction - message
```

When the direction field is empty, Salesforce writes a double dash (`system - - Email verstuurd...`). The viewer parses this correctly and reconstructs clean entries.

Each entry is classified by type and rendered accordingly:

| Type | Icon | Border | What it represents |
|---|---|---|---|
| Email | 📧 | Blue | Automated or manual email sent to the customer |
| SMS | 📱 | Purple | SMS notification sent to the customer |
| Inbound | 📞 | Amber | Inbound contact — agent summaries, case notes |
| Other | 📋 | Grey | Any other system event |

**The Activity Log tab is the most forensically relevant part of the detail view.** It shows:

- Every email sent to the customer, including the recipient address, the template code, and the template description (e.g. `2223 - Factuurnotificatie B2C`, `3024 - Jouw pukcode`, `1999 - Start renewal`)
- Every SMS notification, including the destination number
- All inbound agent interactions — who handled the contact, when, and what was summarised
- Non-system actors (agents like `anilkumar.kota`, `id055894`, Salesforce user IDs) are shown with a **by** label so human actions are distinguishable from automated ones
- A direction badge (**Inbound**) is shown on entries that have it

At the top of the log tab there is a filter bar:

```
All (76)   Email (71)   SMS (3)   Inbound (2)
```

Click any button to narrow the timeline to that entry type. The count next to each type updates based on the actual entries for that account.

---

## Requirements

- **Chrome** (recommended) — tested on Chrome 120+
- Any modern Chromium-based browser should also work
- No internet connection required after opening the file
- No installation, no Node, no Python, nothing

---

## Usage

1. Download `odido_viewer.html`
2. Open it in Chrome (File > Open, or drag the file onto the browser)
3. Click the drop zone or drag your dump files onto it
4. Files must be `.txt` and must have `odido` in the filename — the viewer skips anything else automatically
5. Click **Load All Files**
6. Use the search box and filter dropdowns to find records
7. Click **View** on any row to open the detail panel
8. Switch to the **Activity Log** tab to see the full communication history for that account

---

## Performance

Each record in the dump is approximately 8 KB of raw JSON (257 Salesforce fields). The viewer handles this by keeping only a lean index in memory (9 fields, ~200 bytes per record) and reading full records on demand from the file using byte offsets recorded during parsing.

| Metric | Value |
|---|---|
| Records per 250 MB file | ~30,000 |
| Total across 21 files | ~630,000 |
| Memory for full dataset (lean index) | ~125 MB |
| Memory per full record (on View click) | negligible — single file seek |
| Parse time per 250 MB file | ~2–3 seconds |

The parser runs in a Web Worker so the main thread — and therefore the UI — is never blocked.

---

## Data notes

- The `vlocity_cmt__BillingEmailAddress__c` field is present on all records but may be empty on accounts migrated from legacy systems. Email addresses are more reliably found in the Activity Log via the `SObjectLog__c` field.
- All records in this dump are B2C consumer accounts under the TMNL (T-Mobile NL / Odido) brand.
- Account source for all records is `Migration` — these are legacy accounts moved into Salesforce.
- The `SObjectLog__c` field can contain years of communication history per account, going back to 2019 in some cases.

---

## License

This tool is released for research and incident response purposes. The viewer code itself is MIT licensed. The underlying data is not part of this repository and must not be included in any commit.