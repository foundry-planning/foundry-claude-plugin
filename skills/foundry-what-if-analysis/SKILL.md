---
name: foundry-what-if-analysis
description: "Model and compare what-ifs on a Foundry Planning household's plan: retirement age, spending, Roth conversions, Social Security claiming, savings, stress tests. Runs Foundry's engine without saving, presents results with their caveats, and saves a what-if as a scenario only when the advisor asks."
---

# What-if analysis

Model changes to a household's plan through Foundry's engine, and compare them with the plan as it stands. Nothing is saved unless the advisor asks to keep a what-if, and a what-if never changes the plan itself. Follow `foundry-conventions` throughout:

- Changes and solve targets take fractions: 0.03 is 3%, and an 85% target is 0.85.
- Every figure is Foundry's. Narrate it; never recompute it.

## Set it up

1. Find the household: `search_clients({ query })`, and keep its `clientId`. Only a household with a plan (`hasPlan: true`) has one. A prospect has nothing to model yet: say so.
2. Get the ids the changes need. Never guess an id.
   - Accounts: `list_plan_details({ clientId, kind: "account" })`.
   - The others: the same call with `kind` set to `"income"`, `"expense"`, `"liability"`, `"family_member"`, `"external_beneficiary"`, `"roth_conversion"` or `"asset_transaction"`.
   - Start from the base plan unless the advisor names a saved scenario. `list_scenarios({ clientId })` lists them, which also answers "what have we already modeled?"; add a `scenarioId` there for that scenario's changes. Pass `scenarioId` to a what-if tool to start from one.
3. Pick the tool. references/what-if-recipes.md has the worked calls (retirement-age ladders, Roth strategies, claiming ages, spending ceilings), which change answers which question, the solve kinds and ranking.
   - One what-if: `compare_what_ifs({ clientId, variants: [{ label, changes }] })` with one variation. Add `solveFor` to find a number, or `includeMonteCarlo: true` for the probability of success. Every solve except the claim age runs Monte Carlo anyway.
   - Several variations side by side: `compare_what_ifs({ clientId, variants, … })`. Each variation has a `label`, its `changes` and an optional `solveFor`; `sharedChanges` apply to every variation, and `rankBy` ranks them. It takes at most 10 variations when Monte Carlo or a success-based solve is involved, 25 otherwise. Split a bigger set into calls of that size.
   - The most the base plan or a saved scenario can spend at a target: `solve_max_spending({ clientId, targetPoS: 0.85 })`. It's quick when the plan hasn't changed since the last solve.
   - The best Social Security claiming age on the base plan or a saved scenario: `analyze_social_security({ clientId, person: "client" })`. Use `person: "spouse"` for the co-client.
   - Two scenarios already saved in Foundry: `compare_scenarios({ clientId, left: "base", right })`, with `"base"` or a scenario id on each side. It answers "what does this change cost them?" on net to heirs after estate tax, estate tax at each death, each heir's change, lifetime tax and the ending liquid portfolio. A side can also be `"do-nothing"` or `"snap:<id>"`. For estate ideas such as trusts, follow `foundry-estate-review`.

## Run it

1. A run with Monte Carlo or a solve can take minutes. If it isn't done in about 45 seconds, it returns `status: "running"` with a `jobId`.
   - Keep the `whatIfId` (or `whatIfIds`) from this first answer: the later result doesn't repeat them.
   - Call `get_what_if_results({ jobId })`. Each call waits up to about 50 seconds.
   - After two or three checks that are still running, tell the advisor it's still running (its `progress` gives how many variations are done), and that you can check again when they ask. A run is kept for 24 hours, only for the advisor who started it.
2. A refusal before the run names what to fix: "Change 2: …", or a variation by its label. Fix that change (an id from `list_plan_details`, a year inside the plan, the right kind of account) and run again.
3. A `failed` run gives its `error`: relay it.

## Present it

In hosts that show screens, `compare_what_ifs` also shows the comparison on the Projection chart, and keeps checking a run that is still going. The screen is a view only: read the numbers with `get_what_if_results` (or from the tool's own result) before you narrate them, and never narrate a figure only because a screen showed it. `run_projection` and `get_monte_carlo` show charts the same way.

The results are under `result`: `baseline` and one row per variation in `variants`. A single what-if is the one row.

- Give each figure as Foundry returns it, beside the baseline (current plan) figure and the difference (`deltas`). Name the plan year where one applies.
- The metrics:
  - `successRate`: the probability of success, only when Monte Carlo ran;
  - `endingPortfolio`: the liquid portfolio in the plan's last year;
  - `lifetimeTax`;
  - `netToHeirs`: already net of the heirs' income tax on inherited retirement accounts, and including lifetime gifts. It matches the Solver's "Total to Heirs";
  - `heirIncomeTax`, `yearsFullyFunded` and `firstShortfallYear`.
- A solve gives its `value`, `status` and `achievedSuccess` (the probability of success at that value):
  - `converged`: it found the value that meets the target;
  - `max-iterations`: it stopped early, so the value is approximate. Say so;
  - `unreachable`: moving that one lever can't reach the target. The value is the closest the search got, and it misses: never present it as the answer.

  A max-spending value is in today's dollars, rounded to the nearest $5,000.
- Pass on every caveat in `caveats`, in your words. In a comparison they cover the whole table.
- A variation that `failed` gives its `error`. A variation listed in `unfinished` ran out of time: say so, and offer to run it in a follow-up call.
- With `rankBy`, give each row's `rank` and the `best` label. A failed row, or one whose solve is `unreachable`, has no rank.

## Keep it

Only when the advisor asks ("save that as a scenario"):

1. Say which what-if you will save and the name you'll give it (up to 60 characters, such as "Retire at 62"). Wait for a yes.
2. Save it: `save_what_if_as_scenario({ clientId, whatIfId, name })`, with that what-if's `whatIfId`. For a comparison, use the variation's id from `whatIfIds`.
   - It makes a new scenario and leaves the base plan and every other scenario alone. Give the advisor its `scenarioUrl`, which opens the Solver on it.
   - It needs the firm's changes from AI assistants turned on. If it is refused, relay the refusal as written.
3. A solve's answer isn't part of the what-if. To keep a solved value, run `compare_what_ifs` with one variation holding that value as a change, then save that `whatIfId`. For a solved age, that's a `retirement_age` change; for a solved spend, a `type: "spending_amount"` change.
4. A what-if id lasts 24 hours. After that, run the what-if again.

## Work on a saved scenario

Only when the advisor asks, and each change is shown and approved first, as under Keep it:

- Add a what-if to a scenario that already exists: `save_what_if_as_scenario({ clientId, whatIfId, targetScenarioId })`. It appends that what-if's changes to that scenario instead of making a new one. `list_scenarios({ clientId })` gives the ids.
- Turn a scenario's changes on or off without losing them: `toggle_scenario_changes({ clientId, scenarioId, on: false, toggleGroupId })` for a whole group, or `changeIds` for single changes. A change that is off stays saved and is left out of the plan. A sale to a trust is one group, its account change and its note: switch it by `toggleGroupId`. Switching off or removing the change alone is refused, since the note would stay in the plan.
- Take a change out for good: `remove_scenario_change({ clientId, scenarioId, changeId })`. That thing goes back to the base plan's value. Name the change and wait for a yes.
- A new scenario to start from: `create_scenario({ clientId, name, copyFrom })`, with `copyFrom` as `"empty"`, `"base"` or a scenario id.

## Make a scenario the base plan

Promoting is rare and cannot be undone: it replaces base-plan records with the scenario's and permanently deletes every other scenario. A snapshot of the old base plan is kept.

1. Make sure the idea is saved as a scenario (see Keep it). The base plan itself is refused.
2. Preview it: `promote_scenario_to_base({ clientId, scenarioId })`. Nothing is written. You get `lines` (what changes, in plain words), `deletedScenarios`, `snapshotName` and a `previewToken`.
3. Show the advisor every line and every scenario that will be deleted, and say a snapshot of the old base plan is kept. Wait for a plain yes.
4. Confirm with that token: `promote_scenario_to_base({ clientId, scenarioId, confirm: true, previewToken })`. It then gives `promoted`, `snapshotName`, `deletedScenarios` and `counts`.
5. If the answer has `stale: true`, the plan changed since the preview and nothing was written. Show the fresh `lines` and `deletedScenarios`, ask again, and use the fresh `previewToken` only after a new yes.

Never confirm for the advisor: not on a hint, and not because they asked for the preview. Confirm only with the `previewToken` of the list the advisor said yes to. After a `stale` answer, show the fresh list and get a new yes before you confirm again.

## Chart it and take it further

- Year-by-year figures for a chart in the chat: `get_projection_data({ clientId, whatIfId, sections })`.
  - `sections` come from `"cashflow"`, `"balances"`, `"tax"` and `"estate"`. Narrow with `startYear` and `endYear`.
  - For a saved scenario, pass its `scenarioId` in place of `whatIfId`. Only a saved scenario has `"monte_carlo"`.
  - Too much data for one call is refused: ask for fewer sections or years.
- A what-if goes in any deck, standard or custom, only once it's saved as a scenario. Then follow `foundry-build-presentation` with its `scenarioId`.
- Changing the plan itself isn't a what-if. It's a plan change, shown to the advisor and approved first, as `foundry-after-the-meeting` says under Changing the plan.
