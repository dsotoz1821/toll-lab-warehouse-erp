[← Back to the overview](../README.md) · 🇪🇸 [Leer en español](case-study-02-reversal-guardrail.es.md)

# Undoing a document without corrupting the inventory

**Reversing an approved transaction is easy until two documents disagree about where a piece of equipment is. This is what it took to make cancellation safe, and what happened when the data needed to reverse was never captured.**

---

## The operation

A reader fails in a toll lane. A technician swaps it. The document that records the swap — approved, with a final folio — moves real inventory: the incoming unit becomes *installed* in that lane, the outgoing unit becomes *written off*.

Sometimes that document is wrong. Someone picked the wrong serial, or the replacement never happened, or the whole batch was entered twice. It has to be undone.

## The principle: counterpose, never erase

A cancelled document is not deleted. It is **counterposed**, the way a credit note works in accounting:

- The folio is never released, never reused, never set to null. A gap in the series is explained, not covered up.
- The line items and the photographic evidence are kept.
- Each unit's notes are not rewritten — a cancellation mark is prepended to the original text.
- Everything restored is written into a ledger entry with its reason, its author and its evidence.

An inventory that auditors read cannot have history that disappears. The moment you allow a delete, the question "what did this look like in August?" stops having an answer.

## To undo, you must know what to undo *to*

This sounds obvious and is the thing most systems get wrong. The approval mutates five fields per unit — status, inventory state, warehouse, lane, notes. To reverse it you need the values from before.

So the approval **photographs the previous state of every unit before touching anything.** Two details decide whether that photograph is worth having:

**It goes first in the loop, with no exceptions.** Captured after the mutation, it would store the new state, and the reversal would restore the unit to where it already was — a silent no-op. The worst kind of bug: nothing fails, it just quietly lies.

**It is written inside the same transaction and the same row lock as the mutation.** Either the photograph and the change are both saved, or neither is. A unit can never end up installed without its before-picture.

One field is deliberately *not* read from the photograph: the justification state. That one was changed earlier, by the wizard that locked the unit, not by the approval — so the photograph holds an intermediate value, and the true previous value lives elsewhere. Restoring from the photograph would have released units that were legitimately covered by an earlier document, and they would have reappeared as pending. Nothing would have failed; the numbers would simply have stopped reconciling. Knowing *which* field the obvious source is wrong for took reading the wizard, not the approval.

## The guardrail

Having a photograph is not enough. Consider:

```
01 Sep   Document A   installs reader #1 in lane 3
10 Sep   Document B   writes off reader #1, installs reader #2
Today    someone tries to cancel document A
```

Reverting A would put reader #1 back to *installed*, while B — still live — says it was written off. Two valid documents, contradicting each other, and an inventory that matches neither.

So reversal is blocked when a unit has moved on since the approval. Three things can block it: a later replacement document, a later outbound shipment, or a standing exemption on that unit.

## The part that was correct and useless

The block said: *"these units have moved on. Undo in reverse order: cancel the most recent document first."*

True, and no help at all. It did not say **which** document. The person looking at the message had to go and find, by hand, what had touched that unit afterwards, in what order, and where each one is cancelled from.

So the block now builds the route: every blocking document, newest first — which is the order they must be undone in — with its date, its status, the units in dispute, and a link to where each is cancelled. The last step closes the loop: *come back here and cancel this one*.

Two design choices in it:

**It groups by document, not by unit.** If one later folio took three of your units, that is **one** step with three serials listed, not three steps. It gets cancelled once.

**A fourth case is reported separately.** Sometimes a unit's state drifted with no document behind it at all. Those cannot be fixed by cancelling anything, so they are shown apart, in a different colour, saying exactly that. Folding them in with the steps would send someone hunting for a document that does not exist — which is worse than saying nothing, because it looks like an instruction.

The route is shown to superusers only. It names documents from other plazas, other periods, and people who do not report to whoever is looking at the screen.

## When the data to reverse was never captured

The before-picture was added at a certain point in the system's life. Documents approved before that have none — and the block for those says, correctly, that there is nothing to return the unit to.

That was harmless while the cancellation window was 30 days: those documents fell out of reach within a month and the case went extinct on its own. Then the rule changed — a superuser was allowed to cancel regardless of date — and the case became reachable at any age. **A change that was correct in itself invalidated an assumption written in a comment somewhere else.**

An audit measured it before anything was decided:

| | documents | units |
|---|---|---|
| Approved without a before-picture | 11,345 | 17,296 |
| → reconstructible from row history | | 4,064 (23%) |
| → history started after the approval | | 10 |
| → nothing recorded before the approval | | 13,222 (76%) |

Three out of four cannot be recovered. **Mass reconstruction was off the table**, and the honest answer for those is that they cannot be cancelled; a correction is made by raising a new document.

## The repair, for the cases that mattered

A batch of ten units had been entered twice. Five of the documents had to be cancelled, and all five predated the before-picture.

The tempting move is to edit the inventory rows by hand. That is the wrong move, and not for reasons of style: cancellation does much more than move fields. It writes the ledger entry, marks each unit's history with the reason, recomputes the monthly report, notes the period crossing, releases the photographic evidence for reuse and notifies the people involved. Editing by hand leaves inventory that moved with **no document explaining it** — the exact failure the system exists to prevent.

So instead, the *one missing input* was restored and the normal cancellation was allowed to run. A tool generates a plan: for each affected unit it pre-fills the previous state from row history where that exists, and leaves the fields **null** where it does not — null on purpose, because a pre-filled plausible value gets approved at a glance, and a null forces a decision. A person fills those in, a dry run validates the whole plan without writing anything, and only then is it applied — all-or-nothing, never overwriting a real photograph, and stamped with its source, its date and who authorised it. A reconstructed photograph never passes itself off as original, and the confirmation screen says so before executing.

In the end: one field written per unit, and the full reversal ran itself, with its complete paper trail.

## The order was not cosmetic

Reading the plan, one serial appeared twice — as the incoming unit of one document and the outgoing unit of another:

```
as the incoming unit of doc A   →  returns to: in stock, no lane
as the outgoing unit of doc D   →  returns to: installed, lane 3
```

Cancel D first and the unit goes to *installed*, then A takes it to *in stock*. Correct. Cancel A first and it ends *installed* — precisely what you do not want for a duplicate.

The guardrail enforces the right order. It is not a recommendation printed on a screen; A is genuinely blocked until D is gone.

---

## What I would keep from this

**The guardrail should name the way out.** A block that is right but silent costs more than it saves, because the person in front of it will find another way — usually a direct edit to the database. The route turned a wall into an instruction, and that is the difference between a control people respect and one they route around.

**Measure before deciding.** The audit of 11,345 documents took an afternoon and killed an idea I liked. Reconstructing everything from row history was elegant and would have written the post-approval state as if it were the pre-approval state for three quarters of the cases — a cancellation that reverts nothing while reporting success. The number is what stopped it.

**Fix the input, not the output.** Restoring the one missing field and letting the real operation run kept every guarantee the operation carries. Reaching into the inventory directly would have been faster and would have cost the ledger entry, the history, the recalculation and the notifications — all the things that make the number trustworthy later.

[← Back to the overview](../README.md)
