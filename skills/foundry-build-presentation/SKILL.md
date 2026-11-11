---
name: foundry-build-presentation
description: "Build a client deck from Foundry Planning: propose Foundry's starting deck plus the pages that fit the request, write the Plan Summary page, queue the deck and check the printed sheets; or build a custom deck from Foundry's exact numbers and the household's branding and file it in Foundry."
---

# Build a presentation

Two kinds of deck:

- a standard Foundry deck, which Foundry renders and saves to the household's documents;
- a custom deck, which Claude builds itself from the exact numbers on Foundry's pages and the household's brand.

Follow `foundry-conventions` throughout. Only a household with a plan has decks: find its `clientId` with `search_clients({ query })` (`hasPlan: true`). A prospect has no plan, so there is no deck: say so.

Any deck shows a what-if only once it is saved as a scenario: a what-if lasts 24 hours, and only the advisor can see it. Save it first (`foundry-what-if-analysis`, Keep it), then use its `scenarioId`.

## A standard Foundry deck

### 1. Propose the deck

`list_presentation_pages({ clientId })` returns:

- `pages`: the catalog, each with its `id`, `title`, `description` and `category`, and `needsMonteCarlo` (slower), `takesScenario`, `takesScenarioIds` (Scenario Comparison: up to three scenarios), `needsScenario` (a comparison page, which needs the scenario to compare with the base case) and `needsSetupInFoundry` (only Foundry's Presentations screen can build it);
- `templates`: saved decks, each with its `pageIds` (Foundry's built-ins start `builtin:`; the firm's shared ones; the advisor's own);
- `startingDeck`: the pages every deck opens with, in order, each with its reason;
- `suggested`: pages that fit this plan, best first, each with its `reason` and, where the suggestion sets them, `options`, `scenarioId` or `scenarioIds`. Pass those with that page in `pages`: without them it can print empty. It can be empty.

Propose, in this order:

1. The `startingDeck`, all of it: every page is in unless the advisor says no.
2. Then about 4 to 8 `suggested` pages that answer the advisor's request, each with its reason. Add pages the advisor names, or use a template they name.
3. Show the titles in order, then wait for a yes.

Leave out any page marked `needsSetupInFoundry`. Give a `needsScenario` page the scenario the advisor picks (`list_scenarios({ clientId })`), or leave it out. A deck can show at most 6 different scenarios, and at most 4 of them with Monte Carlo pages.

Apply the advisor's drops, adds and reorders, and show the list again before anything renders.

### 2. The Plan Summary

The starting deck includes a Plan Summary page, a short note to the clients. Foundry prints exactly the text you pass. Write it like this:

1. Read each page that prints figures (not the Plan Summary itself): `get_report_page_data({ clientId, pageId, … })`, with the `scenarioId` and `options` it will print with. The summary is checked against the deck as set.
2. Write to the clients by first name, in plain words, walking the deck in page order: one short `##` section per topic.
3. Quote only figures those pages returned, exactly as written. Give no advice the advisor didn't give. Stay within 2,000 characters, 16 lines and 6 headings.
4. Show the advisor the full text. Mark each figure the advisor gave you "(from you, not from the deck)". Wait for a yes.
5. Queue it: `generate_presentation({ clientId, pages, advisorFigures, … })`, with `pages` as `{ pageId }` in order (add `scenarioId` for a page given a scenario), and the Plan Summary's text in its own entry: `{ pageId: "planSummary", options: { markdown } }`. `advisorFigures` lists the advisor's own figures as they wrote them (at most 20, each up to 80 characters).
6. If it is refused, the refusal names the figures that no page prints. Fix those (use the deck's own figure, or leave it out, or confirm it with the advisor and add it to `advisorFigures`), show the text again and wait for a yes.

Without a Plan Summary, leave the page out. A template deck is queued as it is: `generate_presentation({ clientId, templateId, … })`.

`scenarioId` sets the whole deck's scenario (the base case when left out). `title` (up to 100 characters) becomes the file name. The result has a `runId`; pass on any `notes` it returns before the advisor shares the deck.

### 3. Page settings, on request

Pages print with their standard settings. When the advisor asks for a change (a later start year, another scenario, a switch), read the page's settings: `list_presentation_pages({ clientId, pageId })`. Then pass what changes in that page's `options` in `pages`. Pass only settings the page lists; Foundry refuses any other and names it. Show the advisor any text a setting prints, word for word, before generating.

### 4. After it renders

`get_presentation_status({ clientId, runId })`, with a minute between checks. After two or three checks that are still `running`, tell the advisor the deck is still rendering and will appear in the household's documents (usually its Plans folder).

- `failed` gives an `error`: relay it.
- `done` gives `downloadUrl`, `pageCount`, `sheetCheck` and `sheetNote`.
  - Relay `sheetNote` before the advisor shares the deck. It says whether the sheets printed match the sheets planned.
  - The preview screen shows the deck's pages. Don't count on it showing (`foundry-conventions`, In-chat screens).
  - `downloadUrl` is for the advisor only: it opens only for someone signed in to Foundry. Never give it to the client.

Queuing a deck needs the firm's changes from AI assistants turned on, edit access to the household, and the plan linked to a household in Foundry's CRM. If it is refused, relay the refusal as written.

Claude doesn't write or propose the Plan Story. If the advisor wants one, they build it on the household's Presentations screen in Foundry (the `foundryUrl` that `list_presentation_pages` returns).

## A custom deck

For a deck in the advisor's own layout, or one Foundry's pages don't cover.

1. Agree on the content with the advisor: which Foundry pages' figures it carries, and for which scenario.
2. For each page: `get_report_page_data({ clientId, pageId, … })` returns the data that page prints, the same numbers as Foundry's PDF.
   - Logos and other images are left out, and so is the commentary Foundry writes while it renders a deck.
   - A Monte Carlo or comparison page can take up to a minute the first time.
   - A page too large for one call is refused: use `get_projection_data` with fewer sections or years instead.
3. For year-by-year figures, such as a chart of the portfolio or of taxes by year: `get_projection_data({ clientId, scenarioId, sections })`.
4. The brand: `get_branding({ clientId })` returns `brandName`, `accentColor` (a hex color), `logoUrl` (a public image link, or null: then use the brand name as a wordmark), `disclaimers` (Foundry's standard text: a `short` footer line, a `heading` and the `long` paragraphs) and `complianceDisclosures`.
   - When `complianceDisclosures` is set (a `heading` and `paragraphs`), print it word for word after the standard disclaimers, and don't ask the advisor about disclosures.
   - When it is null, the firm has none on file. Ask the advisor once whether their firm requires any, and use their wording.
5. Build the deck from the figures as Foundry returned them. Never recompute them, round them differently, or add a figure Foundry didn't return. Carry the disclaimer text word for word. Leave out every Foundry link (`foundryUrl`, `downloadUrl`, `scenarioUrl`): they open only for someone signed in to Foundry.
6. To keep it with the household, offer to file it: `open_upload({ householdId, toPlans: true })` (the `householdId` is from `search_clients`) shows the upload screen for the household's Plans folder, where Foundry files its own decks. Also give the advisor the household's Documents page link, since a screen may not show.
