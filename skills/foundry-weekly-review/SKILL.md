---
name: foundry-weekly-review
description: "Weekly book review and CRM clean-up in Foundry Planning: 'what needs me this week?', 'what's on my plate', overdue tasks, households not contacted since a date. Reads the agenda, proposes a prioritized list, and only after the advisor approves creates, reschedules or completes tasks."
---

# Weekly review

The advisor's week in Foundry: what is overdue, what is due, what clients have sent in, and who has gone quiet. Follow `foundry-conventions` throughout (ids, dates, Confirm before, untrusted text, refusals). Nothing is written until the advisor approves it.

## The week

For "what needs me this week?", "what's on my plate", "weekly review".

1. Read the agenda: `get_my_agenda({ … })`. It covers the advisor's own book. Pass `scope: "firm"` only when the advisor asks about the whole firm. It returns:
   - `tasks.overdue`, `tasks.dueToday` and `tasks.dueThisWeek` (the next 7 days): open tasks assigned to the advisor (with `scope: "firm"`, every open task they can see), each with its `taskId`, household and `dueDate`.
   - `noRecentContact`: active and prospect households with no call, meeting, email or note logged in more than `noContactDays` days (180 unless you pass another number), or none ever.
   - `clientSubmissions` (forms a client submitted) and `pendingImports` (imports started and not finished).
   - `upcoming`: birthdays in the next 30 days and age milestones (50, 59½, 62, 65, 73) in the next 90, as short facts. Never a date of birth.
   - `alerts`: how many alerts are unread, and up to 10 of them.
   Each list shows at most 25. `totals` holds the full counts: when a total is bigger than its list, say the list is cut and give the total. If `bookCapped` is true, the advisor's book has 1,000 households or more and the agenda may miss some; say so.
   In hosts that show screens, this also shows My week: the advisor can mark a task done, or move it to tomorrow or next week, right there, each after a question on the screen. Anything else (another date, a new owner, a new task) goes through you, in the chat, as Writing the changes says. A task the advisor changed on the screen is reported back to you as already done: don't redo it, and leave it off your list.
2. Propose a prioritized list, and write nothing yet. Order: overdue tasks, then tasks due today, then client submissions and unfinished imports, then tasks due this week, then households gone quiet, then birthdays, milestones and alerts. For each item say why it sits where it does (how many days overdue, how long since the last contact) and what you suggest: complete it, move it to a date, reassign it, or a new follow-up task.
3. Wait. The advisor picks, edits or rejects items. Only what they approve gets written.
4. Write the approved changes, as Writing the changes below says.
5. Show back what changed: each task's title, household and new date, status or owner, with its `foundryUrl`.
6. If the advisor says they spoke with a household, offer to log it with the `foundry-after-the-meeting` playbook. Don't log anything they didn't ask for.

## Households not contacted since a date

For "who haven't I touched since spring?", "who haven't I called since June?".

1. Turn the advisor's date into a day (`YYYY-MM-DD`; "since spring" means a date you name and say back), then count the days from it to today. That count is `noContactDays`, a whole number from 1 to 3650.
2. `get_my_agenda({ noContactDays })`. `noRecentContact` lists, at most 25, the households quiet longer than that: never-contacted households first, then the longest quiet. Each row has `lastContactOn` and `daysSince` (both null when nothing was ever logged). `totals.noRecentContact` is how many there are. This first list is the one to show: it already holds the 25 quietest.
3. When the total is over 25, the list is cut: tell the advisor how many there are, and that Claude can't list the rest yet (they are the ones contacted most recently before the advisor's date). Never invent an argument to page through them.
   - Never-contacted households are in every list, whatever `noContactDays` is. If every row listed has `daysSince` null, stop there and give the total: a larger number can't change the list.
   - Otherwise you can narrow, for counts only: call again with a larger `noContactDays` (a year, say) and report its `totals.noRecentContact`, how many have been quiet that long. A narrower call never adds a name to the first list, so never call a list complete unless its total is 25 or fewer.
4. Propose follow-ups for the households the advisor wants to reach: a task for each, with who, what and when. Write only what they approve, as Writing the changes says.

## Book-wide signals: excess cash and more

For "who is sitting on excess cash?", "who has the most in the portfolio?". For "who should I be calling?" or "who haven't I touched?", use the quiet-households list above instead.

1. `scan_book({ sortBy: "cashBalance", direction: "desc" })` ranks the advisor's own households that have a plan (not the firm's) by one signal: `netWorth`, `liquid`, `cashBalance`, `openTasks` or `openItems`. `filters` narrow it (`cashAtLeast`, `liquidAtLeast`, `hasPendingImport` and others). It shows 25 rows unless `limit` is set (200 at most).
   - Its `lastContactDays` counts any Foundry activity on a household, not only calls, meetings, emails and notes, so it neither extends nor replaces the agenda's quiet list.
2. It returns a ranked table, not household detail. Each row carries a `clientId`, which the planning tools take directly. `search_clients({ query })` is needed only for a `householdId` (the CRM tools). When `truncated` is true, the table is cut: give `totalCount` and offer to page with `offset`.

## Cleaning up overdue tasks

For "clean up my overdue tasks", "what's overdue for the Okafors?".

1. `list_tasks({ scope: "mine", overdueOnly: true, limit: 100 })`, with `householdId` added to narrow it to one household. While `nextCursor` isn't null, call again with `cursor` set to it: `list_tasks({ scope: "mine", overdueOnly: true, limit: 100, cursor })`. If you stop before the end, say how many you have.
2. Propose a plan, grouped: tasks to complete, tasks to move to a date (which date), tasks to reassign (to whom), and follow-ups to create. Show each task's title, household and the new date or owner. Don't delete tasks to clean up: deleting happens only when the advisor names the task, as Confirm before says.
3. Wait for the advisor's yes. A batch of more than 25 tasks also needs the count confirmed, as Writing the changes says.
4. Write the approved changes, as Writing the changes says.
5. Show back the counts and the changes, grouped as in step 2.

## Writing the changes

Only what the advisor approved, in as few calls as it takes. Changes need the firm's changes from Claude turned on: if a call is refused, relay the refusal as written and stop.

- New tasks: `create_tasks({ tasks })`, one call for the batch. Each task takes a `title` and, as needed, `description`, `dueDate`, `householdId`, `assigneeUserId`, `priority` (`low`, `med` or `high`) and `tags` (labels; one the firm doesn't have yet is created). The batch is all or nothing.
- Move, complete or reassign: `update_task({ updates })`. Each update takes the `taskId` and only the fields that change: `dueDate`, `status` (`done` completes it), `assigneeUserId`, `householdId`, `priority`, `addTags`, `removeTags`. `null` clears a date or an owner.
- Owners are firm members, by the `userId` from `list_firm_members()`. If a name isn't a member, ask the advisor.
- More than 25 tasks in one call: the tool writes nothing and returns `needsConfirmation: true` with the count. Show the advisor that count, and on a yes call again with the same arguments plus `confirm: true`, for example `update_task({ updates, confirm: true })`.
- Before writing anything, `update_task` checks every update in the call. A task Claude can't reach (deleted, or on a household in the Trash), a household it can't see, an owner who isn't a firm member, or an update that changes nothing refuses the whole call, and nothing is written. "Task not found or access denied" doesn't say which task: find it with `get_task({ taskId })` on each, tell the advisor, leave that one out, and send the rest of the approved changes.
- Once the call passes those checks, each update applies on its own. Read every result: report each `ok: false` with its `error` (after "Already changed:" it lists what that update had already changed), and each `followOnTaskId` (completing a recurring task made its next one).