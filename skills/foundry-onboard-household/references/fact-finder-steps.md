# Foundry's onboarding steps

The steps a household's plan is built in, in the order to follow. Each heading gives the step's number, its id (what `start_fact_finder_step` and `get_plan_checklist` take as `step`) and its name. Below each is what the step holds, so you can tell which documents feed it and skip a step with nothing to add. When you open a step, its response gives the exact sections, columns, options and fixed rows, and its `guidance`: follow it.

## 0. household: Household

The client and the co-client, one fixed row each ("Client" and "Co-client"): names, date of birth, retirement age and month, plan-to age (the life expectancy), email and phone, and whether each is covered by a workplace retirement plan. The household's filing status, state of residence and address (one fixed row, "Household"). The family: children, grandchildren and other heirs, one row each, with their relationship and date of birth. Charities and other beneficiaries outside the family, one row each.

It comes first because later steps link to these people: account owners, education account beneficiaries, will provisions. `create_household` opens this step's draft itself.

## 1. balance_sheet: Balance sheet

One row per account, annuity, education account, equity grant, note receivable and debt, each with its current value: cash accounts, taxable investment accounts, retirement accounts (each with one owner: retirement accounts can't be joint), annuities (an income rider's or annuitization's terms aren't asked here: `annuity_contract` sets them after the step saves), 529 and other education savings, stock options and equity awards, notes receivable (money owed to the household, such as a loan to family), and liabilities (mortgages, HELOCs, auto, student and personal loans, credit cards). Account numbers go in as the last 4 digits only.

A retirement account's ongoing contributions belong in the savings step. Account beneficiaries are columns on each account's row here, not in the estate step. A new plan starts with a $0 "Household Cash" checking account (`seeded: true`): link the household's real checking account to it rather than adding a second one.

## 2. real_estate: Real estate

Each property, with its value, cost basis, owner, address and annual property tax, and each business interest with its value, owners and their share, tax treatment, the share of earnings distributed, and its annual income and expense. The accounts and debts a business owns are held in it afterwards with `business_holdings`.

Each property's annual property tax goes in its row. A mortgage saved in the balance sheet step links to its property: name it in the property's "Mortgage on it" column.

## 3. income: Income

Each person's salary, one fixed row each ("Client" and "Co-client"): employer, annual salary, annual raise, start and end year (a blank end year stops at retirement). Social Security, one fixed row each, from a statement or an estimate the advisor gives (an ssa.gov statement is the best source). Other income, one row each: pensions, deferred compensation, business, rental and trust distributions.

The fixed rows link to that person's salary and Social Security records on their own. A new plan starts each person's Social Security at $0 (`seeded: true`), so fill in the real benefit here.

## 4. savings: Savings

Each row is one ongoing contribution, and its account column names an account from the balance sheet step: an annual amount or a percent of salary, the Roth share of a 401(k) or 403(b), the employer's match or flat contribution, and the years it starts and ends.

## 5. goals: Goals

Each row is an expense outside living expenses, such as education, travel, a large purchase or care for a parent, with an annual amount and the years it starts and ends. "A goal?" marks a goal. A goal needs an amount and the years it happens.

## 6. living_expenses: Living expenses

Two fixed rows, "Current" and "In retirement", each with an annual amount, its yearly increase and the years it covers, plus an optional itemized list of current living expenses.

Living expenses are what the household spends after taxes, property taxes, insurance premiums, debt payments and the goals listed separately. When a figure includes any of those (a budget, a stated monthly spend, bank outflows), take them out and tell the advisor which and why: Foundry models each one on its own, so leaving them in counts them twice. A new plan starts with two default living expense rows (`seeded: true`); the fixed rows link to them.

## 7. insurance: Insurance

One row per life, disability and long-term care policy. Group or employer coverage without a policy of its own goes in the notes.

## 8. estate: Estate

The documents each person has in place, one fixed row each ("Client" and "Co-client"), ticking each one that exists today. Trusts (a charitable lead or remainder trust's split-interest terms aren't asked here: `charitable_trusts` adds it), will provisions (specific gifts first, then who receives the rest), prior lifetime gifts (gifts above the annual exclusion, or any gift to a trust), and anything else worth knowing, in the notes. Account beneficiaries are on each account's row in the balance sheet step.
