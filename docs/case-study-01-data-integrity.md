[← Back to the overview](../README.md) · 🇪🇸 [Leer en español](case-study-01-data-integrity.es.md)

# Silent corruption in a frozen monthly report

**87% of a regulatory report had drifted out of sync and nobody knew. The fix could not live in the application, because the application was the thing bypassing it.**

---

## What the report is

Each warehouse keeps a monthly min/max stock form: for every concept it tracks, what the minimum and maximum should be, what was consumed, and what was left over. It is an official form. Once a month closes, that month is a **frozen photograph** — a figure that has been reported cannot change afterwards, or the months stop reconciling with what was filed.

The leftover column is the one that matters here. It is computed from **current stock** — it has no date filter, because stock is a thing that exists now, not a thing that happened in August.

That is the whole vulnerability, stated in one sentence: a column that always means "today", stored in a row that means "August".

The only thing turning it into a monthly photograph was a condition that refused to write outside the month's own window.

## The symptom

The report came back showing the same leftover figure in every month of the year. Then, after a change, October came back as zeros.

Two different wrong answers from the same cause, which is typical: the condition that was supposed to pin each month in place had holes in it, and depending on which hole you went through you either smeared one month across all of them or wrote nothing at all.

## Measuring before fixing

The first move was not to fix anything. It was to find out how far the damage went.

I wrote a reconciliation command: for every concept in every warehouse, recompute the figure from source data and compare it to what was stored. Crucially, it **repeats the formulas rather than calling the method that writes them** — the method writes, and a verification that writes always reports agreement.

The answer came back: **1,303 of 1,505 concepts were out of sync.** Eighty-seven percent.

That number changed the problem. This was not a rare edge case to patch; it was the normal state of the data. Anything built on top of that report had been built on sand, and the first deliverable was no longer a fix — it was repairing 1,303 rows and being able to prove they were right afterwards.

## The two root causes

**A closed period could still be written to.** The guard existed but depended on callers passing the right flag, and there were six of them. I moved it to the first line of the method: a closed period returns immediately unless the caller is the close itself. A rule that six callers have to remember is not a rule.

**The leftover could be written into a past month.** The window check had two escape hatches — one for rows being created for the first time, one for a force-close path — and both wrote today's stock into whatever month the row belonged to. The leftover now only ever writes into the month in progress; a row created outside its own month gets an explicit note saying it has no leftover figure, instead of a number that looks real.

The consumption column needed no such lock: it filters by date, so recomputing August genuinely gives August. It is reconstructible. The leftover is not. That asymmetry is the reason the two columns are treated differently, and it is worth noticing before writing the lock, not after.

## The deeper problem

With both locks in place the figures were correct — as long as every change to the inventory announced itself. The application used Django signals for that.

**Django signals are bypassed by design.** They do not fire on a queryset `.update()`, on `bulk_create`, or on raw SQL. And the codebase had roughly ten places doing a bulk update on exactly the field that feeds the leftover formula, for good reasons: that is what you use when you need to change a thousand rows without loading a thousand objects.

So no amount of work on the signals would close the gap. The gap was not a missing signal; it was a category of write that signals cannot see.

## The options

**Remove the bulk updates.** Honest, and wrong. They exist because they are the right tool for those operations. Replacing ten of them with row-by-row loops would trade a correctness problem for a performance problem, and the next engineer would reintroduce one within a year.

**Recompute everything on a schedule.** Simple, and too slow to be safe. The leftover can only be computed while the month is open; if the nightly job runs after the close, the wrong figure is frozen forever.

**Put the rule where it cannot be bypassed.** A database trigger fires on every write, from any code path, including the ones nobody has written yet. This is what was built.

## What was built

A trigger watches the six columns that feed the leftover formula. When any of them changes, it writes a `(warehouse, concept)` pair into a small queue table. A worker drains the queue every five minutes and recomputes only those pairs.

The trigger is **deliberately stupid**: it marks and returns. It contains no business logic at all. All the logic stays in Python, where it can be read, tested and changed by someone who is not a database specialist. The trigger's only job is to be impossible to skip.

Four details decided whether this worked:

**`IS DISTINCT FROM`, not `<>`.** Comparing a value against `NULL` with `<>` yields `NULL`, not `true` or `false`, and the condition silently misfires. Half those columns are nullable. This is the kind of bug that produces a system which works correctly on the data you tested with.

**Both the old and the new pair are recorded.** If a unit moves from one plaza to another, two reports are affected, not one. A trigger that only records the new state leaves the old plaza quietly wrong.

**A uniqueness constraint plus "do nothing on conflict" keeps the queue bounded.** A thousand movements of the same pair leave one row. Without it, a bulk operation on ten thousand rows would enqueue ten thousand jobs for the same recomputation.

**The Python side reads the instance dictionary, not the attributes.** The signal that snapshots a row's previous values has to read those values without touching the model's attribute descriptors. On a queryset fetched with `.only()` or `.defer()`, touching a deferred field fires one query *per object* — a list of five thousand entries would have become five thousand and one queries, with the cause hidden inside a signal belonging to a different module. The cost as built is zero queries when nothing relevant changed.

## Proving it worked

A trigger that does not fire looks exactly like a system with nothing to do. So it was tested on the path the signals cannot see: a bulk update, inside a transaction that was then rolled back, checking that the queue row appeared. It did.

The reconciliation command then ran across production and reported **zero differences** — after repairing the 1,303 it had found.

## Two scheduling decisions worth explaining

**The reconciliation runs at 2:45 AM, fifteen minutes before the 3:00 close.** Not weekly, and not after. The leftover figure can only be computed while the month is still open; the close freezes it permanently. If the figure is wrong on the last day of the month, it stays wrong forever. A weekly audit might land after the close, and would then exist only to document the error.

**The alert goes to superusers, by email, and never to the in-app notifications.** Those notifications go to site supervisors and administrators, and "40 concepts repaired" is not something they can act on. Putting it there would train the people who receive real alerts to ignore their inbox. The alerting code also never raises: if the mail fails, the data has already been repaired, and failing the repair because an email bounced would be absurd.

---

## What I would keep from this

The part I would do the same way again is **measuring before fixing**. The instinct on seeing a wrong figure is to find the bug and patch it. Running the reconciliation first turned a bug report into a number — 87% — and that number is what justified building a trigger rather than adding another guard clause. Without it I would have patched the symptom and shipped, and the bulk-update paths would still be open today.

The part I would watch is the **two-layer design**: the Python signals now overlap with the trigger. The signals are instant, the trigger is guaranteed, and they both end up calling the same recomputation. That redundancy is deliberate — but it is the kind of thing that confuses whoever reads the code next, so it is documented at both ends.

[← Back to the overview](../README.md)
