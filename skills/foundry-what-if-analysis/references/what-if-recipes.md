# What-if recipes

Worked calls for the questions advisors ask most, then the solve kinds, ranking and limits. Ids come from `list_plan_details`. Years must fall inside the plan, amounts are dollars, and rates are fractions. `person` is `"client"` or `"spouse"` (the co-client). The tools list each change's fields and what they mean.

## Which change answers which question

- Retire at another age: `type: "retirement_age"`, with `person` and `age` (40–85; `month` optional).
- Spend more or less in retirement: `type: "spending_scale"` with a `multiplier` (0.9 spends 10% less), or `type: "spending_amount"` with an `annualAmount` in today's dollars.
- One expense or income: `type: "expense_amount"`, `"income_amount"`, `"income_years"` (when it starts or stops) or `"income_growth"`, with the row's id.
  - `"income_years"` can't move a year tied to a milestone such as retirement (change the retirement age instead), or a Social Security row (change its claim age instead).
- Social Security, each with `person`: `type: "ss_claim_age"` (`age` 62–70, `months` optional), `"ss_pia"` (the monthly benefit at full retirement age), `"ss_annual_benefit"`, `"ss_no_benefit"` or `"ss_cola"`. The person needs a Social Security row in the plan.
- Savings: `type: "savings_amount"`, `"savings_percent"`, `"savings_roth_percent"`, `"savings_max"` or `"savings_years"`, with an `accountId` that already has a savings rule. Foundry refuses an account without one; `type: "add_account"` adds an account with savings.
- Life expectancy: `type: "life_expectancy"`, with `person` and `age`.
- Stress tests:
  - `type: "stress_inflation"` (`rate`);
  - `"stress_ss_haircut"` (`pct`, `startYear`);
  - `"stress_market_crash"` (`year`, `drawdownPct`);
  - `"stress_tax_rates"` (`points`, `startYear`);
  - `"stress_exemption_cap"` (`cap`);
  - `"stress_disability"` (`person`, `startYear`, and `endYear` if it ends).

  Put one in `sharedChanges` to stress every variation of a comparison.
- Techniques: `type: "roth_conversion"` (below), `"asset_sale"`, `"asset_purchase"`, `"gift"`, `"relocation"`, `"pay_off_debt"`, `"add_income"`, `"add_expense"` and `"add_account"`. `type: "remove"` leaves an existing item out of the what-if.
- Trusts, wills and beneficiaries: `type: "structure"` with an `action` and its `params` (the estate recipes below). `describe_what_if_changes({ action })` lists an action's fields.
- One what-if or several: `compare_what_ifs` (one variation for one what-if). Two scenarios already saved in Foundry: `compare_scenarios` instead.

## Retirement-age ladder

"What could they spend if they retired at 60, 61 … 67?" One comparison. Each age solves for the most they can spend at the advisor's target (85% here), and an "As planned" variation gives the same solve on today's plan:

```
compare_what_ifs({ clientId, variants: [
  { label: "As planned", changes: [], solveFor: { kind: "max_spending", targetSuccess: 0.85 } },
  { label: "Retire 60", changes: [{ type: "retirement_age", person: "client", age: 60 }], solveFor: { kind: "max_spending", targetSuccess: 0.85 } },
  { label: "Retire 61", changes: [{ type: "retirement_age", person: "client", age: 61 }], solveFor: { kind: "max_spending", targetSuccess: 0.85 } },
  …
  { label: "Retire 67", changes: [{ type: "retirement_age", person: "client", age: 67 }], solveFor: { kind: "max_spending", targetSuccess: 0.85 } }
], rankBy: { metric: "solvedValue" } })
```

- Nine variations with a solve fit the 10-variation cap.
- For a couple retiring together, put both people's `retirement_age` changes in each variation.
- Give each row's `solve.value` (today's dollars, rounded to $5,000) with its `status`.

"When can they retire and still be at 85%?": `compare_what_ifs({ clientId, variants: [{ label: "Earliest age", changes: [], solveFor: { kind: "retirement_age", person: "client", targetSuccess: 0.85 } }] })` finds the earliest age that reaches the target.

## Roth strategies

"What's the best way to convert to Roth?"

- Get the account ids: `list_plan_details({ clientId, kind: "account" })`.
  - The sources are pre-tax retirement accounts: a traditional IRA, 401(k), 403(b), SEP or SIMPLE IRA.
  - The destination is a Roth IRA with the same owner.
  - An inherited account can't convert.
- If the plan already has a conversion (`list_plan_details({ clientId, kind: "roth_conversion" })`), put `{ type: "remove", kind: "roth_conversion", id }` in `sharedChanges`, so each strategy replaces it.

```
compare_what_ifs({ clientId, sharedChanges, variants: [
  { label: "No conversions", changes: [] },
  { label: "Fill 22% to 2032", changes: [{ type: "roth_conversion", fromAccountIds: [iraId], toAccountId: rothId, mode: "fill_bracket", bracket: 0.22, startYear: 2027, endYear: 2032 }] },
  { label: "Fill 24% to 2032", changes: [{ type: "roth_conversion", fromAccountIds: [iraId], toAccountId: rothId, mode: "fill_bracket", bracket: 0.24, startYear: 2027, endYear: 2032 }] },
  { label: "Fill 24%, IRMAA-free", changes: [{ type: "roth_conversion", fromAccountIds: [iraId], toAccountId: rothId, mode: "fill_bracket", bracket: 0.24, irmaaCapTier: 0, startYear: 2027, endYear: 2032 }] },
  { label: "$100k a year to 2032", changes: [{ type: "roth_conversion", fromAccountIds: [iraId], toAccountId: rothId, mode: "fixed_amount", amount: 100000, startYear: 2027, endYear: 2032 }] }
], rankBy: { metric: "netToHeirs" } })
```

- The modes:
  - `"fill_bracket"` needs a `bracket`: 0.1, 0.12, 0.22, 0.24, 0.32 or 0.35;
  - `"fixed_amount"` needs an `amount`, and `indexingRate` can grow it each year;
  - `"deplete_over_period"` needs an `endYear`;
  - `"full_account"` converts the whole account.
- `irmaaCapTier` (0–4) is the highest Medicare IRMAA tier the conversion may push income into. 0 stays surcharge-free.
- Rank by `netToHeirs` (higher first) or `lifetimeTax` (lower first).
- An IRMAA-capped conversion alongside another conversion raises a caveat that the cap may not hold: pass it on.
- The annual amount that reaches a target: add `solveFor: { kind: "roth_amount", changeIndex: 0, targetSuccess: 0.85 }` to a variation whose change at that index (counted from 0, within that variation's own changes) is a `"fixed_amount"` conversion.

## Claiming ages

- The age, 62 to 70, that leaves the largest liquid portfolio at the end of the plan, on the base plan or a saved scenario: `analyze_social_security({ clientId, person: "client" })`.
  - It runs no Monte Carlo, and ties go to the earliest age.
  - It returns `solvedClaimAge`, and in `candidates` each age with its `endingPortfolio`.
  - It sets the claim by age for every candidate, whatever the household's claiming setting is today.
- With other changes, or with the probability of success:

  ```
  compare_what_ifs({ clientId, variants: [
    { label: "Claim at 62", changes: [{ type: "ss_claim_age", person: "client", age: 62 }] },
    { label: "Claim at 67", changes: [{ type: "ss_claim_age", person: "client", age: 67 }] },
    { label: "Claim at 70", changes: [{ type: "ss_claim_age", person: "client", age: 70 }] }
  ], includeMonteCarlo: true })
  ```

  Or add `solveFor: { kind: "ss_claim_age", person: "client" }` to a what-if, to find the best age alongside its other changes.
- Someone whose Social Security has no benefit entered (a new plan starts at $0) ties at every age, so the answer means nothing. Get the estimate first: `get_plan_checklist({ householdId })` lists it.

## Spending ceilings

- The most the base plan or a saved scenario can spend each year in retirement at a target: `solve_max_spending({ clientId, targetPoS: 0.85 })`.
  - `realAnnualSpend` is in today's dollars, rounded to the nearest $5,000.
  - `achievedPoS` is the probability of success at that spend.
  - `scaleFactor` is that spend as a multiple of the plan's retirement living expense.
  - Foundry serves it from its cache when the plan hasn't changed; otherwise it takes longer.
- The same with changes: `compare_what_ifs({ clientId, variants: [{ label: "What-if", changes, solveFor: { kind: "max_spending", targetSuccess: 0.85 } }] })`.
- To test one spending level: `compare_what_ifs({ clientId, variants: [{ label: "Spend 120,000", changes: [{ type: "spending_amount", annualAmount: 120000 }] }], includeMonteCarlo: true })`. Or scale the spending: `type: "spending_scale"` with `multiplier: 0.9`.
- To keep a solved spend as a scenario, run it as a `type: "spending_amount"` change and save that what-if.

## Estate ideas

A what-if can't change the same record twice: one `structure` change per account, trust or will (a new trust and its funding gift are fine). `describe_what_if_changes({ action })` gives each action's fields; `ref` names a trust made in the same what-if, and the later change uses it as `trustRef`. `foundry-estate-review` has the whole review. Rank by `netToHeirsAfterEstateTax`.

- A SLAT: `create_trust` with `lifetimeBeneficiary: "spouse"` (a grantor trust by default), then `gift_to_trust`:

  ```
  compare_what_ifs({ clientId, variants: [{ label: "SLAT", changes: [
    { type: "structure", action: "create_trust", params: { ref: "slat", name: "Spousal Access Trust", subType: "irrevocable", grantor: "client", lifetimeBeneficiary: "spouse" } },
    { type: "structure", action: "gift_to_trust", params: { trustRef: "slat", grantor: "client", startYear: 2027, amount: 5000000 } }
  ] }] })
  ```

  Two SLATs with opposite grantors are two `create_trust` changes. Warn that mirror-image terms raise the reciprocal-trust question for the attorney.
- A family bank: add the empty trust to the base plan first (`update_plan_detail` on the `trusts` section; `foundry-estate-review` has the labels), fund it with `{ type: "structure", action: "gift_to_trust", params: { trustId, grantor, startYear, amount } }` (without funding the loan comes from $0 cash), then `{ type: "structure", action: "trust_loan", params: { trustId, borrower, amount, interestRate, paymentType, termMonths, startYear } }`. A trust made in the same what-if can't lend.
- A CRT or CLT: `create_split_interest_trust` with `name`, `subType`, `grantor`, `charityId`, `fundingAccountIds` (whole accounts), `inceptionYear`, `payoutType`, `irc7520Rate` and `termType` (the payout amount or percent and the term years or measuring lives come from `describe_what_if_changes`).
- An IDGT sale: the trust is in the base plan first; `gift_to_trust` for the seed gift, then `sell_to_trust` with `trustId`, `accountId`, `interestRate`, `paymentType`, `termMonths` and `noteStartYear`.
- An account moved whole into a trust: `retitle_to_trust` with the `accountId` and the trust. `remove_trust` takes a trust out.
- Beneficiaries and wills: `change_beneficiaries` (`accountId`, `beneficiaries`) replaces an account's designations; `change_will` (`grantor`, `bequests`, `residuaryRecipients`) replaces a will.
- Refused: a life insurance policy into a trust (retitled, gifted in kind or sold); a retirement account retitled, gifted in kind, sold or used to fund a CRT or CLT (name the trust as beneficiary instead).

## Solve kinds

| `solveFor` | Finds | Better |
|---|---|---|
| `{ kind: "max_spending", targetSuccess }` | the most they can spend each year in retirement (today's dollars) and still reach the target | higher |
| `{ kind: "retirement_age", person, targetSuccess }` | the earliest retirement age that reaches the target | lower |
| `{ kind: "savings", accountId, targetSuccess }` | the least annual saving into one account that reaches the target | lower |
| `{ kind: "roth_amount", changeIndex, targetSuccess }` | the annual amount of the `"fixed_amount"` Roth conversion at `changeIndex` that reaches the target | the advisor says |
| `{ kind: "ss_claim_age", person }` | the claim age, 62 to 70, that leaves the largest portfolio at the end of the plan | the advisor says |

- `targetSuccess` is a fraction from 0.5 to 0.99.
- Every solve but the claim age runs Monte Carlo, so it counts toward the 10-variation cap.
- A savings solve needs an account with a savings rule in dollars. If it saves a percent of salary, add a `type: "savings_amount"` change for that account first; if it contributes the IRS maximum, add a `type: "savings_max"` change with `contributeMax: false` first.

## Ranking

`compare_what_ifs` takes `rankBy: { metric, direction }`. Rows keep the order you gave them; each gets a `rank`, and `best` is the top row's label. A failed row, or one whose solve is `unreachable`, gets no rank. The metrics, with the direction each ranks by default:

- `successRate`, higher first. It needs Monte Carlo: `includeMonteCarlo: true` or a success-based solve.
- `endingPortfolio`, higher first.
- `lifetimeTax`, lower first.
- `netToHeirs`, higher first: the Solver's figure, after the heirs' income tax.
- `netToHeirsAfterEstateTax`, higher first: the Estate Planning grid's figure, after estate tax and before the heirs' income tax.
- `estateTaxFirstDeath` and `estateTaxFinalDeath`, lower first.
- `heirIncomeTax`, lower first.
- `yearsFullyFunded`, higher first.
- `solvedValue`, the solve's value. Every variation that solves must solve for the same kind. Max spending ranks higher first; retirement age and savings lower first. A Roth amount or claim age needs `direction: "asc"` or `"desc"`.

## Limits

- Up to 40 changes in one list.
- Up to 10 variations in a comparison when Monte Carlo or a success-based solve is involved, 25 otherwise.
- A run waits about 45 seconds before it becomes a job, and each `get_what_if_results` call waits about 50 seconds. A comparison stops starting new variations once its time budget is spent, and lists the rest in `unfinished`.
- A run still going after 15 minutes is reported as failed ("The run was interrupted. Start it again.").
- What-ifs draw on a compute limit. A rate-limit message means wait and try again later.
- What-ifs and runs are kept for 24 hours, only for the advisor who ran them.
