---
name: foundry-estate-review
description: "Run an estate review for a Foundry Planning household: where the estate goes at each death and the taxes on it, whether beneficiaries match the client's wishes, then trying ideas such as SLATs, a family bank, CRTs and CLTs, or an IDGT sale against today's plan, and making the chosen idea the real plan. Use when the advisor asks for an estate review, an estate-tax look, gifting to a trust, or whether beneficiaries are aligned."
---

# Estate review

Look at the estate as it stands, check it against what the client wants, try ideas without saving anything, compare them with today's plan, and keep the one the advisor picks. Follow `foundry-conventions` throughout, and `foundry-what-if-analysis` for running, saving and promoting. Every figure is Foundry's: narrate it, never recompute it. Name the plan year each figure belongs to.

Foundry's output is the advisor's analysis, for the advisor to review. Never present it as legal or tax advice. The advisor and the client's attorney decide what to set up.

## 1. Read today's estate

Find the household with `search_clients` (it needs a `clientId`; a prospect has no plan to review). Then:

`get_estate_summary({ clientId, sections: ["transfers", "will_conflicts"] })`

- Lead with who gets what at each death and the taxes on it: today (if both died now), at the first death and at the second, as the projected deaths fall in the plan.
- Give the estate tax at each death (federal, state, admin) and what is left to the heirs.
- Add `"liquidity"` when the question is whether the estate can pay its taxes and costs, and `"gift_tax"` when gifts are already being made. Pass `scenarioId` to review a saved scenario instead of the base plan.
- Pass on `taxGrounded: false` next to the tax figures if you see it.
- For the estate at another year (say, if both died in 2040), add `year`; it applies to `transfers` and `will_conflicts`.

## 2. Check beneficiaries against the client's wishes

- `will_conflicts` lists assets that pass differently than the will directs, checked at each death. Show each one in plain words.
- Read the designations in `list_plan_details({ clientId, kind: "account" })` and the family and charities in the same call with `kind: "family_member"` or `"external_beneficiary"`.
- Ask the advisor what the client wants, then name every mismatch: an ex-spouse or a child who has died still named, a minor named outright, a retirement account going to the estate. Do not decide which side is right; ask.
- To fix a designation on approval, the plain way is the direct `account_beneficiaries` section of `update_plan_detail` (`describe_plan_section` lists it; it saves to the base plan at once, after a yes). A what-if saved and promoted would also fix it, but promoting deletes every other scenario, so don't use it for a small fix. To see the effect first, try `change_beneficiaries` in step 3.

## 3. Try ideas

Each idea is a `structure` change. Nothing is saved, and the base plan doesn't change (except the empty trust added below, when an idea needs one). Call `describe_what_if_changes({ action })` for an action's fields before you use it; with no `action` it lists them all. Never guess a field. Then run the ideas side by side with `compare_what_ifs`:

```
compare_what_ifs({ clientId, variants: [
  { label: "SLAT", changes: [
    { type: "structure", action: "create_trust", params: { ref: "slat", name: "Spousal Access Trust", subType: "irrevocable", grantor: "client", lifetimeBeneficiary: "spouse" } },
    { type: "structure", action: "gift_to_trust", params: { trustRef: "slat", grantor: "client", startYear: 2027, amount: 5000000 } }
  ] }
] })
```

The result already carries a baseline row (today's plan), so no "Today" variation is needed; add one only to rank today's plan beside the ideas.

Ids come from `list_plan_details` (trusts are `kind: "entity"`). A what-if can't change the same record twice: one change per account, trust or will (a new trust and then its funding gift are fine). Put the same idea in more than one variation to compare amounts or years.

- **SLAT.** `create_trust` with `lifetimeBeneficiary: "spouse"` (a grantor trust by default), then `gift_to_trust` with its `ref` as `trustRef`.
  - A trust made inside the idea holds the cash given to it at 0% growth in the projection, as Foundry's Solver does, so its out-of-estate value and net to heirs are understated. Tell the advisor so; if they want the trust's growth shown, add the trust to the base plan first (see Adding the empty trust below) and gift into it by `trustId`.
  - If two SLATs would mirror each other (each spouse the grantor of a trust for the other, similar terms and funding), warn the advisor that the reciprocal-trust doctrine can undo the benefit. Whether the two are different enough is a question for the client's attorney. Say so; don't judge it.
- **Family bank.** A trust has to be in the base plan before it can lend, so add the empty trust first (see Adding the empty trust below). Then fund it with `gift_to_trust` using its `trustId`, and lend with a `trust_loan` (`borrower`, `amount`, `interestRate`, `paymentType`, `termMonths`, `startYear`). Without funding, the loan comes from $0 cash. Only the trust's side is modeled. If the trust lends more than its cash, the run's caveats say so, naming the trust, the loan and the year: tell the advisor, and fund it first or lend less.
- **CRT and CLT.** `create_split_interest_trust` (`subType: "crt"` or `"clt"`) funded by whole accounts, needing `name`, `grantor`, `charityId` (from `external_beneficiary`), `fundingAccountIds`, `inceptionYear`, `payoutType`, `irc7520Rate` and `termType`. Read the notes from `describe_what_if_changes` for the payout and the term and measuring lives.
- **IDGT.** Add the trust to the base plan first (see Adding the empty trust below). Then a `gift_to_trust` (a gift uses the exemption), then `sell_to_trust` of a whole account for a note. The sale needs a grantor trust, so give it `trustType` "IDGT" or `grantorTrust` "Yes" when you add it; otherwise the sale is refused.
- **Beneficiaries and wills.** `change_beneficiaries` replaces one account's designations; `change_will` replaces one person's whole will (read it first, and send all of it). A trust made in the same what-if takes `trustRef` as a beneficiary or recipient.
- **Remove a trust.** `remove_trust` takes a trust out only inside a what-if. To take a trust out of the base plan, see below.

Refusals to expect, and to explain in plain words:

- A life insurance policy can't be moved into a trust as a what-if yet (its premiums would count differently in the preview and the saved scenario). Say so; the advisor can do it in the Solver.
- A life insurance policy can't be gifted in kind or sold to a trust either, or fund a CRT or CLT. A trust that holds a policy can't be removed in a what-if, for the same reason.
- A retirement account can't be retitled, gifted in kind, sold or used to fund a CRT or CLT. Name the trust as its beneficiary instead, with `change_beneficiaries`.
- `sell_to_trust` and `trust_loan` refuse a trust made in the same what-if (it needs to be in the plan). `sell_to_trust` also refuses a trust that isn't a grantor trust, a retirement account, a policy, checking, and an account the trust already holds.
- A change that fails names the field to fix, with a pointer back to `describe_what_if_changes`. Fix it and run again.

### Adding the empty trust

For a family bank or an IDGT, ask the advisor, then add the trust to the base plan: `update_plan_detail({ clientId, sectionId: "trusts", cells: { … } })`, with no `recordId` so it adds one. Read the columns first with `describe_plan_section({ sectionId: "trusts" })`.

- Give it a `trustName`, and use the fact finder's labels: `grantor` "Client" or "Co-client", `grantorTrust` "Yes", `trustType` "IDGT" or "Irrevocable". Not "Revocable living trust", and not "CRT" or "CLT" (those are `create_split_interest_trust`).
- An empty trust changes no numbers. It is in the plan until removed. If the advisor drops the idea, offer to remove it only once no saved scenario uses the trust: `remove_plan_detail({ clientId, kind: "entity", recordId })`, only after a plain yes, naming the trust. If a saved scenario still uses it (a loan, a sale, a gift or a retitle), the removal is refused and names the scenarios: say to delete those first (`delete_scenario` also removes the notes they hold, after its own yes), or keep the trust.

Run, wait for and narrate a run as `foundry-what-if-analysis` says (a run past about 45 seconds comes back as a job: `get_what_if_results`).

## 4. Compare with today's plan

In the run, rank by `netToHeirsAfterEstateTax`. Lead the advisor's read with the order the Estate Planning comparison uses:

1. `netToHeirsAfterEstateTax`: net to heirs after estate tax, today's plan against the idea.
2. `estateTaxFirstDeath` and `estateTaxFinalDeath`: the estate tax at each death.
3. Each heir's change (`heirDeltas`).
4. Then `netToHeirs`, the Solver's "Total to Heirs". It is lower because it also takes off the income tax the heirs owe on inherited pre-tax accounts. Say which one you are reading.

Once an idea is saved as a scenario (see `foundry-what-if-analysis`, Keep it), show Foundry's own comparison:

`compare_scenarios({ clientId, left: "base", right: scenarioId })`

- Net to heirs after estate tax leads. Then estate tax at each death, the difference (right minus left), `heirs`, lifetime tax and the ending portfolio. Add `year` to see the estate at another year.
- `heirs` counts only what the estates pass at the deaths. Money already in an irrevocable trust (the out-of-estate figure) isn't credited to the trust's beneficiaries, so say so whenever out-of-estate is above zero, naming the trust's remainder beneficiaries from the plan if you know them.
- Each side has an `ownFinalDeathYear`. A `note` appears when the year isn't that side's own last death: say what year the figure is for. `preDeath: true` means the year is before the last death, so estate tax reads $0: say that it is a year too early to compare tax.
- A side can also be `"do-nothing"` or a frozen snapshot (`"snap:<id>"`). The result then has no "Open in Foundry" link; say why, in the words of `linkNote`. Otherwise offer the link (`foundryCompareUrl`): it opens Estate Transfer's comparison as of today, not at the compared year, so say to pick that year there (`linkNote` says so).

## 5. Life insurance for the estate tax

When liquidity at death comes up, or the estate can't pay its tax without selling:

`solve_life_insurance_need({ clientId, coverEstateTaxes: true })`

It can come back as a job (`get_what_if_results`). Add `scenarioId` to solve for the idea that was saved. Fields left out take the plan's saved settings; ask the advisor before you change the death year or the amount to leave heirs.

## 6. Make the chosen idea the real plan

The idea the advisor picks usually becomes the plan. Follow `foundry-what-if-analysis`, "Make a scenario the base plan": save the idea (only after the advisor asks, and a yes to the name), preview it with `promote_scenario_to_base`, show every line and every scenario that will be deleted, get a plain yes, then confirm with the `previewToken`. Never confirm for the advisor. If the answer is `stale`, show the fresh list and ask again.

## 7. Investments

When the review turns to how the money is invested, `get_asset_allocation({ clientId })` gives the current mix against the recommended portfolio. It is read-only: the recommended portfolio is picked on the client's Investments page in Foundry, so give the advisor that page if none is picked. Its percents are of classified value; the Unallocated total stands apart. Don't recommend a mix.

## Close

Summarize in this order: where the estate goes and the tax today; any beneficiary that doesn't match the client's wishes; each idea tried, with net to heirs after estate tax beside today's; what the advisor chose. Name what the attorney has to review (any trust, a SLAT pair, a sale to a trust, a CRT or CLT). Draft for the advisor; send nothing to the client.
