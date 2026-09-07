# Contact Quick Editor — Zoho CRM Widget

Widget for the Contact record detail page: shows and edits First name, Last
name, Phone and Email, plus a mailing address section with postal-code lookup.
All values are read from and saved to the CRM record via the Zoho JS SDK —
nothing is hardcoded.

One file (`app/widget.html`), vanilla HTML/CSS/JS, no build step.

## Install and run

1. Pack it: `zet validate && zet pack` (zip lands in `dist/`).
2. In CRM: **Setup → Developer Space → Widgets → Create New Widget**
   - Type: **Related List**
   - Hosting: **Zoho**, upload the zip
   - Index Page: `/widget.html` (relative to the `app/` folder — the Base URL
     already ends in `/app`, so `/app/widget.html` gives a 404)
3. Open a Contact → **Add Related List → Widgets** → pick the widget.

For local dev: `zet run`, accept the self-signed cert at
`https://127.0.0.1:5000/app/widget.html` once, and create the widget with
Hosting: **External** pointing at that URL. On macOS, I had a problem with the port, so free port 5000 first by
disabling AirPlay Receiver (System Settings → General → AirDrop & Handoff).

## Address API: Zippopotam.us + ViaCEP (hybrid)

Postal codes aren't globally unique ("10001" exists in several countries), so I had to create
the Country field as a selector that routes the lookup:

- **Brazil → ViaCEP**: free, keyless, CORS-enabled, street-level accuracy from
  the Correios base — far more precise for CEPs than any international API.
- **Everywhere else → Zippopotam.us**: free, **no API key**, CORS-enabled,
  ~60 countries. Returns city and state for a postal code, which is what the
  task asks for.

Keyless matters more than convenience here: a widget runs entirely in the
browser, so any embedded key would be public — key-based geocoders (Google,
HERE) would need a proxy backend just to hide the credential. Both chosen APIs
avoid that entirely. Setup is just whitelisting both domains in the manifest's
`cspDomains`.

Limitations: Zippopotam is city-level (not street), covers ~60 countries (the
selector only lists supported ones; a country loaded from CRM outside the list
stays selectable but without lookup), and some countries match on prefix codes
(UK outward code, Canadian FSA — normalized in code). ViaCEP and Zippopotam
are community services without formal SLAs — fine for interactive lookups,
wrong for bulk enrichment. Unknown codes return `{"erro": true}` (ViaCEP) or
HTTP 404 (Zippopotam); both are handled with clear messages, and lookup
failure never blocks manual entry or saving.

## What I'd do differently with more time

- Per-country postal code format validation and input masks.
- Send only changed fields and warn about unsaved edits.
- Guard against concurrent edits (current behavior is last-write-wins).
- Auto-lookup once a valid code is typed (debounced) instead of a button.
- Improvement in the error return for the widget
- pt-BR translations and other languages, and unit tests for the pure logic (code normalization,
  field map, response parsing).

## Assumptions

- Standard Contact field names (`First_Name`, `Mailing_City`, etc.); custom
  layouts only require editing the `FIELD_MAP` constant.
- "Address" means the Mailing address block.
- The Country selector defaults to Brazil for new/empty records, but any
  country already on the record is preserved as-is on load and save.
- Saves should trigger workflows (`Trigger: ["workflow"]`), behaving like a
  native edit.
