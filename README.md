# Foundry Planning for Claude

This plugin gives Claude the playbooks for running an advisory practice with Foundry Planning: the weekly review, meeting prep, logging a meeting afterward, onboarding a household, what-if analysis, estate reviews and client decks. The playbooks use the tools of Foundry's connector. The plugin names that connector but doesn't connect it: add the connector by its address, as Foundry's Settings → AI assistants page shows. Connect Foundry once; if the plugin's Connectors tab also lists Foundry Planning, you don't need to connect it there as well.

## Set up

1. In Foundry, open Settings → AI assistants. It shows the connector address and the steps for each Claude app.
2. Add the connector: in Claude, open Customize → Connectors, choose Add, then Add custom connector, paste the address, sign in to Foundry and pick your firm. (Claude Code loads the connector directly.)
3. Open Customize → Plugins, choose Add, then Upload plugin, and pick the file you downloaded from that page. In Claude Code, install it instead with `/plugin install foundry-planning --marketplace foundry-planning/foundry-claude-plugin`.
4. Turn on "Code execution and file creation" in Claude's Settings → Capabilities; the playbooks need it.

Claude can read your households right away. Changes from AI assistants stay off until a firm admin turns them on under Settings → AI assistants.

## What is inside

Skills, which Claude loads when a task needs one:

- `foundry-conventions`: ground rules every Foundry task follows, such as which id a tool takes, how to present Foundry's numbers, what to confirm before deleting, and that anything sent to a client goes only through a preview you approved.
- `foundry-weekly-review`: the week's tasks, client submissions and households gone quiet, and cleaning up overdue tasks.
- `foundry-meeting-prep`: a meeting brief filed in the household's documents, and Foundry's standard deck.
- `foundry-after-the-meeting`: logging a meeting or call with its follow-up tasks and any plan change, checking the plan checklist afterwards, adding a prospect when the family is new, and saving the plan's observations and next steps.
- `foundry-onboard-household`: taking a new prospect to a sent data-collection form, reading and applying the form the client submits, reopening one the client left incomplete, and building a household's plan from statements, one fact finder step at a time. It starts every chat from the plan checklist, estimates what nobody knows (at most three questions, on your yes), completes a list only on your yes, and ends with what's still open.
- `foundry-tax-return-review`: reading a federal tax return (Form 1040) into Foundry, then applying only the plan updates you accept.
- `foundry-what-if-analysis`: what-ifs and comparisons through Foundry's engine, kept as scenarios only when you ask.
- `foundry-estate-review`: today's estate and its taxes, beneficiaries against the client's wishes, and trust ideas (SLATs, a family bank, CRTs and CLTs, IDGTs) compared with today's plan.
- `foundry-build-presentation`: proposes Foundry's starting deck and the pages that fit your request, writes the Plan Summary page for you to approve, checks the printed deck, and builds custom decks from Foundry's numbers and the household's brand, filed in the household's Plans folder.

Commands, each of which starts one of those skills:

- `review`: a weekly review of your Foundry book.
- `prep`: prepare a client meeting brief.
- `log`: log a client meeting or call.
- `onboard`: onboard a household from documents.

Foundry's numbers come from its planning engine. Claude explains them and never recomputes them.
