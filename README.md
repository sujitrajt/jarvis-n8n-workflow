# Jarvis — WhatsApp Assistant (n8n workflow)

Version-controlled backup of the **Whatsapp-Automation** n8n workflow (aka "Jarvis"), a personal WhatsApp bot that answers questions against a Google Sheet of saved places ("Google_Maps_Want_To_Go").

- n8n instance: `sujitrajt.app.n8n.cloud`
- Workflow ID: `bNbeiIVM4QMlDvdp`
- Exported: 2026-08-31

## Architecture (current, read-only search)

```
WhatsApp Trigger
  -> If (is this a real text message, not a delivery-status callback?)
    -> Extract Message
      -> Get row(s) in sheet  ─┐
      -> Sync Sheet & Continue ┴─> AI Agent (Gemini + Simple Memory + searchSavedPlaces tool)
        -> Prepare Reply
          -> Send message
```

- **AI Agent**: Google Gemini (`gemini-3.5-flash-lite`), per-sender conversation memory via `Simple Memory`, and a `searchSavedPlaces` code tool that filters the already-fetched sheet rows by city/cuisine/rating/price/visited/keyword.
- **Get row(s) in sheet**: reads the full "Want to go" tab of the Google Sheet on every message.
- Read-only today — no add/update/delete capability yet.

## Planned: CRUD extension

Adding full Create/Read/Update/Delete so Jarvis can manage the sheet directly from WhatsApp:

1. **Places lookup** tool (Google Places API "Text Search", `X-Goog-Api-Key` header auth credential already created in n8n as "Google Places API") — also usable for general discovery of places not yet on the list.
2. **addPlace** — looks up a place by name, and if multiple real-world branches are found, adds *all* of them (not asking which one), deduping against existing rows.
3. **updatePlace** — fuzzy name match against the sheet; asks for clarification only if genuinely ambiguous.
4. **deletePlace** — always confirms the exact row (name + city) with the user before deleting; never deletes silently.
5. System prompt update covering the new tools plus "everything for this trip" region-based filtering (for future trip-planning use, e.g. "show me everything marked for LA").

Future: additional Google Maps saved lists (Beach, Parks, tourist attractions, etc.) will continue to be imported into the sheet via manual browser session, not automated through Jarvis over WhatsApp.

## Notes on this export

- `whatsapp-automation.json` is a straight export of the workflow definition (nodes, connections, settings) via n8n's `/rest/workflows/:id` API, reshaped to match n8n's standard downloadable export format.
- **Pinned test data was stripped.** The live workflow had ~40KB of pinned execution output (sample WhatsApp payloads and all ~199 pinned sheet rows) attached for local testing in the editor; that's cached debug data, not part of the workflow logic, and it included a couple of real phone numbers — so it's excluded here rather than committed.
- **No credential secrets are included.** Nodes reference credentials by id/name only (e.g. `"Google Sheets account"`, `"WhatsApp account 2"`) — n8n never exports the actual stored secret values. Treat this repo as containing configuration, not credentials.
- n8n Cloud's native git "Source Control" sync is gated to Business/Enterprise tiers; this repo is a manual export instead, meant to be re-exported and committed periodically (especially before/after the CRUD build) to track history.

## Re-exporting after changes

In the n8n editor: workflow menu (⋮, top right) → **Download**, or re-fetch `GET /rest/workflows/bNbeiIVM4QMlDvdp` while logged in. Overwrite `whatsapp-automation.json` and commit.
