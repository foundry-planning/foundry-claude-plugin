# Plan sections

Which details of a plan Claude can change, and how. Before you write a section whose columns you don't know, call `describe_plan_section({ sectionId })`: it lists the columns, their allowed values and their units. Without a `sectionId` it lists every section's id.

## Two ways in

- Fact finder sections (people, accounts, income, expenses, savings, policies, trusts, wills, gifts…) go through the fact finder: `fill_fact_finder_step` while building a plan, or `update_plan_detail` for one change. Their cells take percents (6.5 means 6.5%).
- Direct sections go through `update_plan_detail` and save straight to the base plan at once, with no draft and no review. Show the advisor the change and wait for a yes first. Their rates and growth columns take fractions (0.03 means 3%).

## The direct sections

Each is a `sectionId` of `update_plan_detail`. `recordId` updates that record; leaving it out adds one, for the sections that can add. A record's id comes from `list_plan_details` (the kind is in brackets), unless the line says otherwise.

- `deductions` [deduction]: charitable, above-the-line, itemized and property-tax deductions, each with a year range.
- `tax_adjustments` [tax_adjustment]: income the tax return counts that cash flow doesn't, or the other way round, each with a year range.
- `medicare`: a person's Medicare coverage. `recordId` is the person: `client` or `spouse`. It only changes coverage.
- `income_schedule`, `expense_schedule`, `savings_schedule`: a year-by-year amount for an income, an expense or a savings rule. `recordId` is that income's or expense's id (kind `income` or `expense`), or the savings rule's id (kind `savings_rule`). They only change what the plan has.
- `roth_conversions` [roth_conversion], `transfers` [transfer], `reinvestments` [reinvestment], `withdrawal_order` [withdrawal_strategy], `relocations` [relocation], `asset_transactions` [asset_transaction]: the plan's techniques. Their account columns take account ids from `list_plan_details` kind `account`.
- `account_allocations`, `account_beneficiaries`, `account_growth`: an account's asset mix, beneficiaries and growth. `recordId` is the account's id. They only change accounts the plan has.
- `life_cash_values`: a life insurance policy's cash value by year. `recordId` is the policy's account id.
- `trust_assets`: puts a business, an account or a debt into a trust, or takes it back. An account's or debt's add is an owner's share; a business's add to an irrevocable trust is a gift. `recordId` is the trust's id (kind `entity`). Each edit is an action (`op` add or remove) applied at once. A business's add sent twice moves the share twice; an account's or debt's add sets the trust's share, so sent twice it changes nothing more. The result lists the trust's assets after the change. See Funding a trust.
- `trust_beneficiaries`: a trust's beneficiaries. `recordId` is the trust's id.
- `gift_series` [gift_series]: recurring gifts, including a gift paid straight from an IRA to charity (a QCD).
- `asset_gifts` [asset_gift]: a gift of a share of an account or a debt in a plan year; `share` is a fraction (0.25 means 25%). A debt linked to the property goes with it. Ids are `list_plan_details` kind `asset_gift`; remove one with the removal kind `gift`.
- `revocable_trust_accounts` [revocable_trust]: a revocable living trust's accounts. `accountIds` (cash, taxable and real estate accounts) replaces the list; it is a label that moves no ownership. Each row carries its `accountIds`.
- `account_groups` [account_group]: a custom account group. `accountIds` (taxable, cash and retirement accounts) replaces its members. A reinvestment names a custom group by that group's id in `groupKeys`: take the id from `list_plan_details` kind `account_group`.
- `open_items` [open_item]: the plan's own to-do list on Details > Profile, not a CRM task (`create_tasks` makes those). `done` true marks an item done now, false reopens it.
- `business_flows`, `entity_flows`: a business account's or an entity's year-by-year income, expense and distribution %. `recordId` is the business account's or entity's id; they only change what the plan has. Send the schedule with `flowMode: "schedule"`; one sent in `annual` mode is refused. A business account ignores its schedule in annual mode, but an entity's years still replace its yearly amounts there, so switching an entity to `annual` is refused while it has years unless you send `schedule: []` with it (that clears them). In schedule mode a blank distribution % is 0 for an entity and the business's policy % for a business account. A trust has no distribution %. Read the rows on the `flowOverrides` of the kind `account` (business) or `entity` row.
- `liability_extra_payments` [extra_payment]: one extra payment on a debt. Adding takes `liabilityId` (the debt's id), `year`, `type` (per payment, added to every payment that year, or lump sum, paid once) and `amount`; `recordId` is the payment's id, from `extraPayments` on the debt's `liability` row. `note_extra_payments`: a note's whole list of extra payments, sent as `extraPayments` and replacing every row; the rows get new ids on every save, so read the note (kind `note_receivable`) again before the next change. `recordId` is the note's id.
- `annuity_contract`: an annuity account's contract (income rider or annuitized income, payout, cost basis, fees). `recordId` is the annuity account's id.
- `charitable_trusts`: a charitable lead or remainder trust (CLT or CRT) with its split-interest terms; adding one takes its terms too. `recordId` is the trust's id (kind `entity`).
- `business_holdings`: holds an existing account or debt inside a business account, or releases one the business holds to a new owner. `recordId` is the business account's id (kind `account`). Each edit is one action (`op` hold or release) applied at once; the result lists what the business holds after the change.
- `account_settings`: an account's realization mix and turnover (taxable accounts on a portfolio or asset mix), its RMD switch (retirement accounts) and its margin-call level (taxable accounts). `recordId` is the account's id (kind `account`). A value the projection would ignore is refused.

## Rules

- Lists replace. A schedule, an asset mix, a set of beneficiaries, a list of accounts or a list of cash values is sent whole and replaces what is there. Read the current list first, change it, send all of it back. An empty list clears it where the description says so. Where each list is read, on its `list_plan_details` row:
  - an income's or expense's schedule: `scheduleOverrides` (year to amount) on its kind `income` or `expense` row. Send it as `{ year, amount }` rows.
  - a savings rule's schedule: `scheduleOverrides` on its kind `savings_rule` row, sent as `{ year, amount }` rows like an income's. Never send part of a schedule.
  - an account's asset mix: `allocations` on its kind `account` row, in the shape `account_allocations` takes.
  - an account's or life policy's beneficiaries: `beneficiaries` on its kind `account` row (a trust there is `entityIdRef`; send it as `entityId`).
  - a life policy's cash values: `lifeInsurance.cashValueSchedule` on its kind `account` row.
  - a trust's beneficiaries: `beneficiaries` (primary and contingent), `incomeBeneficiaries` and `remainderBeneficiaries` on its kind `entity` row. `trust_beneficiaries` takes all four tiers in one list.
  - a Roth conversion's `sourceAccountIds` and a transfer's `schedules`: on their kind `roth_conversion` and `transfer` rows.
  - a reinvestment's accounts and groups: `pickedAccountIds` and `groupKeys` on its kind `reinvestment` row (a custom group's id is on its kind `account_group` row). Its `accountIds` there also counts the groups' accounts, so send `pickedAccountIds` as `accountIds`.
- Units differ by column, and each column's description names its unit. Rates and growth columns are fractions (0.03 means 3%); the tool refuses a rate above 1. The exceptions: a beneficiary's `percentage` runs 0 to 100 (50 means 50%, and each tier you list totals 100), and `trust_assets` `percent` is 0 to 100 (30 means 30%). Years are four digits. Money is dollars.
- Asset classes and model portfolios are given by the firm's name, not an id. The match ignores capitalization. An unknown name is refused with the firm's names.
- An update that changes nothing is refused ("Nothing to change") and writes nothing. Say so.
- A result's `saved` lists each field from its value on record to the new one (`trust_assets` lists the trust's assets instead).
- A result's `warnings` are advisories: the save went through. Say each one to the advisor in your own words; never drop one.
- A record id from another household, or from another section, is refused as not found.
- Direct sections read and write the base plan only. A change that exists only in a what-if scenario isn't found.

## Annuities, charitable trusts, businesses and account settings

- **Annuity income.** When a fact finder review warns "Income from a rider or annuitization isn't saved… Set it in Details", save the step (the account is added), then set the contract with `annuity_contract` (`recordId` is the annuity account's id; its `annuity` object is on its kind `account` row). It edits and never adds; an update keeps every column it doesn't send. Income is never guessed: switching to `rider` or `annuitized` is refused unless you send `incomeStartYear` or the contract already has a start. A year you send replaces a stored milestone. A joint-and-survivor payout needs `survivorPct` and a period-certain one needs `periodCertainYears`. Fields the projection would ignore (rider fields on an annuitized contract) are refused.
- **A CLT or CRT.** When a fact finder review blocks a line (`cannotSave`) with "A charitable lead or remainder trust needs its split-interest terms — add it in Details", add the trust with `charitable_trusts` (no `recordId`; `describe_plan_section` lists the terms it needs). The terms move no money. Fund it with `trust_assets` (an account's owner's share: no gift record, no exemption used; the share is the trust's from the plan's start). Foundry already books a new CLT's remainder-interest gift itself, so an `asset_gifts` gift to a CLT or CRT, or a business added through `trust_assets` (that add is a gift), counts its whole value against lifetime exemption on top: ask the advisor before using either. A CLT's heirs are `trust_beneficiaries` rows in the primary tier (the remainder tier shows in reports only); a CLT with no primary rows hands nothing on. The charity must be a charity of this household, and a measuring life the client or co-client with a date of birth. A trust's type can't change.
  - A result of an add lists only the columns you sent. Foundry computes `originalIncomeInterest` and `originalRemainderInterest` (a CLT's deduction and taxable gift; a CRT's retained income and charitable deduction), so read the trust back (kind `entity`, its `splitInterest`) before you quote them.
  - Sending a term or the grantor on an update must say `origin`. Foundry doesn't store how the trust was funded, and saving the terms redoes what depends on it. `new`: funded in the plan, so Foundry recomputes the deduction and remainder interest from the terms. `existing`: funded before the plan, with amounts already filed, so Foundry keeps the stored ones (or the ones you send). On a CLT, `existing` also deletes the remainder gift Foundry computed for it, and with it the lifetime exemption that gift used: say so to the advisor before you send it. A name, trustee or grantor-trust edit alone needs no `origin`.
- **A business's own accounts and debts.** "The S-corp owns the building and its mortgage": after the fact finder adds each record, hold it with `business_holdings` (`recordId` is the business account's id; `op` `hold`, `assetType`, `assetId`), one record per call. A held record keeps no owners of its own: it follows the business. Foundry refuses to hold a retirement, 529, stock-option, life insurance or annuity account, a business, a record a trust or entity owns part of, one an asset gift names, and one already held. A release needs its new `owner`: `client`, `spouse` or `joint` (half each). A held 529 is the exception: it releases with no `owner`, and one sent is refused. The result's `after.held` lists what the business holds.
- **Account settings.** `account_settings` sets an account's realization mix and turnover (a taxable account that grows by a model portfolio, a fund portfolio or its own asset mix), `rmdEnabled` (a retirement account other than an inherited IRA; `true` is refused on a Roth IRA or an HSA, which have no RMDs, since the projection would still take and tax one) and `marginCall` (a taxable account). The four realization percentages may total at most 1; send all four together, since one left null takes the portfolio's share and nothing is rescaled. A value the projection would ignore or misuse is refused with the reason.
- **`warnings`.** An `annuity_contract` or `charitable_trusts` result may carry them: a QLAC over its premium cap; a CRT payout under 5% or over 50%, a remainder under 10%, or an annuity likely to run the trust dry. The save went through. Tell the advisor each one.

## Removing

`remove_plan_detail({ clientId, kind, recordId })` removes one record permanently, the way Foundry's own delete does. Ask first, naming the record, as `foundry-conventions` says. Its kinds:

- Plan details with a direct section: `deduction`, `tax_adjustment`, `roth_conversion`, `transfer`, `reinvestment`, `withdrawal_strategy`, `relocation`, `asset_transaction`, `gift_series`. The three schedules clear a schedule rather than remove a record: `income_schedule`, `expense_schedule` and `savings_schedule`, each by the id of its income, expense or savings rule.
- Client-level records: `account_group` (refused while a reinvestment still names it) and `open_item`.
- `extra_payment`: one extra payment on a debt (a note's list is replaced through `note_extra_payments`).
- Cash flow and balance sheet: `account`, `liability`, `income`, `expense`, `savings_rule`, `holding` (one position on an account), `note_receivable`, `stock_option_grant` (one grant; its account stays).
- People, policies and estate: `family_member`, `external_beneficiary` (a charity or other outside beneficiary), `life_policy`, `disability_policy`, `ltc_policy`, `entity`, `revocable_trust`, `will`, `will_bequest` (one will provision; the will and its other provisions stay), `gift`.

Ids come from `list_plan_details` for most kinds, including a savings rule (`savings_rule`), a note receivable (`note_receivable`), a revocable trust (`revocable_trust`), an account group (`account_group`) and an open item (`open_item`). A disability or long-term care policy's id is in `get_insurance` and the insurance step's `existingRecords`. A will and a will provision are only in the estate step's `existingRecords`, a stock-option grant and a holding only in the balance sheet step's (a holding's id is its `holdingId`).

A `tax_return` is removed by its year as the recordId ("2024"); `get_tax_return` lists the years. Whoever read it, the year goes; the source PDF stays in the vault.

Foundry refuses, with a sentence naming what to change first:

- the plan's client and co-client, and a household (nothing removes one);
- a family member or entity that still owns part of an account or debt, or an outside beneficiary that still owns part of an account or note;
- an entity named as a beneficiary;
- an account, life policy or debt that a gift gives;
- a family member who is the measuring life of a split-interest trust, or an outside beneficiary that is its charity.

When a removal also takes other records, the result's `alsoRemoved` lists them. Tell the advisor what else went. An entity's removal takes its default checking accounts. A family member's, an entity's or an outside beneficiary's takes the gifts and recurring gifts that name them and the will designations that name them. A family member's or an entity's removal also takes their shares of a business and of a note (an outside beneficiary who owns part of a note is refused first). A family member's or outside beneficiary's also takes the beneficiary designations that name them and, for a family member, the CRM contact linked to them. A QCD (a recurring gift from an IRA to a charity) is counted in `alsoRemoved` as a recurring gift, though `list_plan_details` lists it as kind `qcd`.

## Funding a trust

An account reaches a trust three ways, and each does something different to the estate; a debt reaches one two ways, because a debt can't join a revocable living trust (and `asset_gifts` refuses a revocable trust as the recipient). Ask which one the advisor means before you write.

- An owner's share from the plan's start: `trust_assets` with `assetType` `account` or `liability` and `percent` (0 to 100). The trust is a recorded owner in every year (a retirement account takes a trust only at 100), the other owners' shares shrink in proportion, and nothing is a gift: no year, no gift record, no lifetime exemption used. An irrevocable trust's share is out of the household's estate. A business is the default `assetType` (`entity`): to an irrevocable trust that add is a gift (it uses exemption and takes a `year` and `valuationDiscount`), to a revocable trust it is an owner's share.
- A gift in a plan year: `asset_gifts`. The share moves from the gift's year and the gift uses lifetime exemption: a trust's whole value, a family member's or an individual's value after one annual exclusion, a charity's none. `valuationDiscount` (a fraction) works for any recipient: it lowers the gift's value and so the exemption used, while the full share still moves. A gift of a debt counts $0 against exemption, so a discount on one is refused.
- A revocable living trust: `revocable_trust_accounts` (accounts only). It moves no ownership: the accounts stay the household's, taxed to it and in its estate, and who receives them doesn't change. What changes is probate: no probate cost is charged on them at a death.

## Plan assumptions

`update_plan_assumptions({ clientId, changes })` takes fractions for rates (0.03 means 3%) and dollars for amounts. Besides growth and inflation rates it covers each asset class's growth source (inflation, a custom rate, the account's own asset mix for taxable and retirement, or a firm model portfolio by name for taxable, cash and retirement), the share of surplus cash flow spent and where the rest is saved, the tax rate on trust income paid outside the household, whether a person is covered by a workplace plan, the plan's start year (this year or next), Medicare premium growth, probate and administration costs, the flat state estate rate, the present-value discount rate, the lifetime exemption cap, capital loss carryforwards, prior taxable gifts and the tax engine mode. `inflationSource` is `"asset_class"` (inflation follows the firm's Inflation asset class; refused when the firm has none, and refused with `inflationRate`) or `"custom"` (the plan's own `inflationRate`; sending `"custom"` alone switches back to the stored rate). A plan end year is refused when the plan ends at the later life expectancy: change life expectancy instead.
