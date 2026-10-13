# Jarvis: WhatsApp AI Assistant (n8n)

Jarvis is a personal WhatsApp assistant for a saved-places list. You message it in plain English ("any pubs in San Francisco?", "add Blue Bottle Coffee in Berkeley", "mark Cobi's as visited") and it searches, adds, updates, or deletes entries in a Google Sheet of roughly 200 places imported from Google Maps saves.

It is built as an n8n workflow around a Gemini tool-calling agent with 6 custom tools. Search is deterministic JavaScript, so the model never invents places, ratings, or addresses.

## Architecture

```
WhatsApp Trigger
  -> If (text message, not a delivery-status callback)
    -> Extract Message (text + sender ID)
      -> Get row(s) in sheet ─┐
      -> Sync Sheet & Continue┴─> AI Agent
                                    model:    Gemini 3.5 Flash (primary)
                                              Gemini 3.5 Flash-Lite (fallback)
                                    memory:   Simple Memory, per sender, 8 messages
                                    tools:    searchSavedPlaces, findPlaceRow, placesLookup,
                                              addPlace, updatePlace, deletePlace
                                  -> Prepare Reply (format, trim, friendly error text)
                                    -> Send message (WhatsApp)
```

## Tools

| Tool | Type | What it does |
| --- | --- | --- |
| `searchSavedPlaces` | JavaScript code tool | Filters the saved list by location, cuisine, keyword, rating, price, and visited status. Returns the true match count, up to 25 results sorted by rating, and diagnostics when nothing matches. |
| `findPlaceRow` | JavaScript code tool | Finds the exact sheet row for a place by name (optionally city) before any update or delete. |
| `placesLookup` | HTTP request tool | Google Places API (New) Text Search, for adding a place by name or discovering places not on the list. |
| `addPlace` | Google Sheets tool | Appends one row per real-world location returned by `placesLookup`, after a duplicate check. |
| `updatePlace` | Google Sheets tool | Updates only `Visited?` and `Notes`; every other column is unmapped so it cannot be overwritten. |
| `deletePlace` | Google Sheets tool | Deletes a single row, only after the user confirms in a separate message. |

## Design decisions

**Deterministic search, not model memory.** The agent must call `searchSavedPlaces` before answering any place question, and may only recommend saved places the tool actually returned. Suggestions from `placesLookup` are always labeled as not yet saved.

**Search that understands how people ask.**
- Location matches both the City and Region columns, with aliases such as "Bay Area", "South Bay", "SF", "Tahoe", and "LA".
- Category terms expand through a one-way synonym map ("pub" also finds bars, taprooms, and breweries) and match Category, Type, and Cuisine, but not place names, which caused false positives.
- Multi-word terms stay whole, so "ice cream" no longer matches "Rice" or "Juice".
- Matching runs in three stages (exact word, substring, place name) and stops at the first stage with results.
- When nothing matches, the tool reports how many places are saved in that location and in that category elsewhere, so the agent never wrongly says the list is empty.

**Safe writes.**
- Updates and deletes always look up the exact row with `findPlaceRow` first and ask when the name is ambiguous.
- Deletes require explicit confirmation in the user's next message, never in the same turn, and remove at most one row per call.
- `updatePlace` can only change two columns.

**Resilience to model outages.** Gemini returned frequent 503 "high demand" errors, so the agent has a fallback model, retries up to 5 times with a 5-second wait, and continues on error. `Prepare Reply` turns a failure into a friendly "try again in a minute" message instead of leaving the user with no reply.

## Bugs found and fixed

- **Empty results for common questions.** Searching "pubs in San Francisco" returned nothing because search only checked the City column and matched words literally. The rewrite above took that query from 0 to 7 correct results.
- **Gemini 400 errors after multi-category questions.** Two parallel tool calls broke Gemini's function-call ordering when the conversation was replayed from memory. The agent now makes at most two sequential calls, and a single call handles comma-separated categories.
- **Places lookup returning nothing.** n8n did not evaluate `$fromAI()` inside a raw JSON request body, so Google received the literal template string. Moving the query to a body field fixed it.
- **Fields left blank on new rows.** Several `addPlace` columns were never set to be filled by the model; they are now model-fillable with formatting rules matching existing rows.

## Setup

1. In n8n, import `whatsapp-automation.json` (Workflows, then Import from File).
2. Create credentials for WhatsApp Business Cloud API (trigger and send), Google Sheets OAuth2, Google Gemini, and a Header Auth credential named "Google Places API" using header `X-Goog-Api-Key`.
3. Replace the placeholders `YOUR_GOOGLE_SHEET_ID` and `YOUR_WHATSAPP_PHONE_NUMBER_ID` with your own values.
4. The sheet needs these columns: `#`, `Place Name`, `Category`, `Rating`, `Reviews`, `Price Range`, `Status`, `Visited?`, `Notes`, `City`, `Address`, `Cuisine`, `Type`, `Region`.
5. Activate the workflow. If WhatsApp messages stop arriving after edits, unpublish and publish again so n8n re-registers the webhook.

## Repository contents

- `whatsapp-automation.json`: the n8n workflow export. Test data has been removed, and personal IDs are replaced with placeholders. Credentials are referenced by name only; no secrets are stored.
- `README.md`: this file.
