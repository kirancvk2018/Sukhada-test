# Hackathon — Implementation Notes

`index.html` only. No new files. `reference-runbooks.html` untouched.

110,074 → 165,080 bytes. Existing code was extended, not replaced.

---

## 1. Summary of changes

**Modified (10 existing functions)**

| Function | Change |
|---|---|
| `Store` | +3 methods for the `h:` namespace; `removeParticipant` now also deletes `h:<key>` |
| `showScreen()` | accepts `'hackathon'` |
| `participantLogin()` | resolves mode once at login and routes accordingly |
| `loadProblem()` | `blockedByHackathon()` guard as first statement |
| `submitSolution()` | same guard |
| `enterParticipant()` | same guard |
| `doLogout()` | clears hackathon state, resets the active tray |
| `makeSlot()` | writes `data-correct` **only** when a `correctId` is passed |
| `refreshRoster()` | Hackathon column, Assign/Edit button, split detail panel, `colSpan` 7→8, two new stat chips |
| `returnCardToTray` · `refreshTrayEmptyState` · `wireTrayTarget` · `buildTray` | `getElementById('tray')` → `getTray()` |

**Added** — six labelled sections (`HACKATHON DATA`, `ACCESS CONTROL`, `SCORING`, `PARTICIPANT EXPERIENCE`, `EMPLOYEE ADMIN`, plus persistence inside `Store`), one participant screen, one config modal, ~60 CSS rules.

**Untouched** — `RAW_PROBLEMS`, `PROBLEMS`, `CARD_INDEX`, `renderAnswerKey()`, `recordAttempt()`, `refreshParticipantStats()`, `submitSolution()` grading logic, all 19 practice scenarios, all Excel alignment.

### The tray decision

Four functions hardcoded `getElementById('tray')`. Rather than fork the drag/drop layer, a single `activeTrayId` variable and `getTray()` accessor were introduced. Practice sets it to `'tray'`, Hackathon to `'hackTray'`. Everything else — `buildCardEl`, `onDragStart`, `onCardClick`, `placeCardInSlot`, `wireSlotTargets`, keyboard tap-to-place — is shared verbatim.

---

## 2. Data structures

Extends the existing model. Nothing is duplicated and no existing record shape changed.

```js
p:<key>  = { name, empId, pin, createdAt, createdBy,
             hackathon: {                       // null / absent when unassigned
               enabled, fromDate, toDate, scenarioId, assignedBy, assignedAt
             } }

a:<key>  = { items: [...] }                     // UNCHANGED

h:<key>  = { attempts: [ {                      // NEW
               ts, submittedAt, scenarioId, scenarioTitle, attemptNo, completed,
               response:   [{kind, index, placedCardId}],
               earned, maximum, percentage,
               breakdown:  [{label, section, importance, weight, result, points}],
               distractors:[{label, section, importance, weight, result, points}]
             } ] }
```

**Why the assignment sits on `p:`** — login and the roster already read that record, so the access decision costs zero extra queries. Only employees ever write it.

**Why attempts sit in `h:`** — they carry the full response and breakdown, which would bloat `p:`; keeping them out of `a:` means practice statistics are mathematically unaffected; and one row per participant preserves the collision-free write model.

---

## 3. Access control

```js
getHackathonState(rec, attempts, now)  // NOT_ASSIGNED | SCHEDULED | ACTIVE | COMPLETED | EXPIRED
isHackathonActive(rec, attempts, now)
getParticipantMode(rec, attempts, now) // 'hackathon' | 'practice'
blockedByHackathon()                   // guard used by the three entry points
```

State is derived on every call, never stored. Evaluation order matters: **an existing submission wins over everything else**, so if an employee disables an assignment after someone has submitted, the result stays visible as COMPLETED rather than collapsing to NOT_ASSIGNED. COMPLETED is not ACTIVE, so the participant still gets practice back.

**This is not a hidden dropdown.** `loadProblem()`, `submitSolution()` and `enterParticipant()` each return at their first statement when the guard fires. Calling `loadProblem('disk-util')` from the browser console pushes the user back to the Hackathon screen and builds nothing — verified in the test suite.

**Refresh is inherently safe.** The app has no session persistence; init always calls `showScreen('login')`. A refresh therefore forces a re-login, which re-evaluates mode from stored state. There is no cached session to exploit, and none was added.

### Dates

The employee picks date-only values. Stored as `2026-09-16T00:00:00` and `2026-09-20T23:59:59` — no `Z`, no offset. `new Date()` parses these in the browser's local zone, so the window means what the instructor sees. No UTC conversion happens anywhere. A same-day From/To gives a valid one-day window (tested).

---

## 4. Scoring

```js
calculateHackathonScore(scenario, response) → {earned, maximum, percentage, breakdown, distractors}
```

Pure function — no DOM, no globals, asserted by test. The browser submits **placements only**; the score is recomputed from `HACK_SCENARIOS`. A score posted from the client would be ignored.

| Section | Rule |
|---|---|
| Pre-checks | order-sensitive |
| Resolve | order-sensitive |
| Escalate | presence only, position ignored |

| Result | Meaning | Points |
|---|---|---|
| `CORRECT` | right step, right place | full weight |
| `MISPLACED` | right step, wrong position | **40%** of weight |
| `MISSING` | step never placed | 0 |
| `INCORRECT` | distractor placed in a slot | 0, listed separately |

Weight lives on each step, not derived from importance — `hack-cpu` weights a CRITICAL step at 25, `hack-disk` at 24 and 22, `hack-shutdown` at 20/18/16/15. Importance is a report label only.

Worked example, `hack-cpu` with two pre-checks transposed: weights 5 and 10 both become MISPLACED, paying 2 and 4. Score **91/100**, not 0 — partial credit throughout.

Weights total 100 per scenario, so the employee reads a clean "72 / 100".

---

## 5. Participant experience

Login → active assignment → Hackathon screen. The practice screen is never rendered.

Problem statement, then the same flow skeleton and parts tray, same drag/drop and tap-to-place, same visual language. Distractors are drawn **only** from other Hackathon scenarios — `HACK_CARD_INDEX` is separate from `CARD_INDEX`, so practice steps never appear in a Hackathon tray and Hackathon steps never appear as practice distractors.

Submission is confirmed, then one-way. Afterwards the participant sees:

> ✓ **Submission recorded**
> Your Hackathon submission has been recorded successfully. Your results will be reviewed by your training administrator.

No score, no percentage, no breakdown, no marked slots, no answer key. `submitHackathonSolution()` never calls `renderAnswerKey()`.

If the write fails, the submission is rolled back — cards unlock, the button re-enables, a toast explains. The participant is not locked out by a network blip.

---

## 6. Employee experience

One column in the existing roster showing a state badge, plus an **Assign** / **Edit** button. The button stops event propagation so it doesn't toggle the detail row.

The modal takes: enable checkbox, From date, To date, scenario dropdown (all ten). Validation rejects a missing date, a missing scenario, and From-after-To. **Remove assignment** clears it.

Expanding a participant splits into two clearly-headed blocks — **Normal practice** and **Hackathon** — so the two are never conflated. The Hackathon block renders the full step-level table: step, section, importance, weight, result, points, plus the total and a footer documenting the rules. Two new stat chips count active and completed.

This breakdown exists nowhere in the participant-facing path.

---

## 7. Testing

**97 assertions, 97 passed.** Driven headlessly through the real DOM against a simulated Supabase.

All 20 of your listed edge cases are covered. Highlights:

| | |
|---|---|
| Five states (none / scheduled / active / expired / completed) | correct, including same-day boundary |
| Console call to `loadProblem('disk-util')` during Hackathon | blocked, returns to Hackathon screen |
| `enterParticipant()` / `submitSolution()` during Hackathon | blocked |
| `data-correct` in Hackathon DOM | **absent** — slot datasets carry no answer |
| weight / importance in participant HTML | absent |
| Participant result text | no score, no percentage, no breakdown |
| Double submit | second call is a no-op, one attempt row |
| Refresh mid-Hackathon | returns to login, re-evaluates, still Hackathon |
| Refresh after submission | practice restored |
| Expired / scheduled windows | practice, as expected |
| Employee disables after submission | result still visible, participant back on practice |
| 20 participants, 10 different scenarios, simultaneous submit | 20 rows, 20 attempts, no cross-writes |
| Supabase failure during submit | not marked submitted, button re-enabled, retry succeeds |

**Regression on the existing app:** 19 practice scenarios intact, 157 practice cards with zero collisions, no duplicate DOM IDs, practice grading unchanged, answer key still works, attempts still land in `a:`, no `h:` row created by practice.

**The original 45-user suite, re-run on this build:** 45 logins created, 45 trainees from 45 separate browsers, 135 concurrent submissions with **zero loss**; burst test 180/180 at 20/80/200 ms, zero errors.

**Excel alignment, re-verified:** 19/19 tickets, 100% of 66.5 hours, 0 cross-file conflicts.

### One bug the tests caught

`getHackathonState()` originally checked the `enabled` flag before checking for attempts. Disabling an assignment after a participant had submitted made their result disappear from the dashboard. Fixed by evaluating attempts first.

---

## 8. Known limitations

**The answer key is in the page source.** `HACK_SCENARIOS` ships in the JavaScript. Anyone who opens View Source or DevTools can read every step, weight and correct order. This is inherent to a static client-side application and cannot be fixed within it.

What *is* enforced:

1. Nothing about the answer enters participant-facing HTML — no `data-correct`, no weights, no importance.
2. Expected answers live only in `hackSlots[]` in memory.
3. Hackathon and practice card pools are separate indexes.
4. The score is computed from the authoritative config; a client-supplied score would be ignored.
5. The participant path never calls `renderAnswerKey()` and never renders a breakdown.

**The database policy is permissive.** `using (true) with check (true)` plus a publishable key in a public repo means a determined participant could write directly to their own `h:` row via the REST API and fabricate a result.

**Treat this as a training exercise, not a proctored examination.** It is robust against a participant clicking around the UI. It is not robust against a participant who opens DevTools. Given the audience — consultants learning runbook discipline — that is likely the right trade. But do not use these scores for anything with consequences attached without the hardening below.

Other notes: one submission per participant (the structure supports a future `maxAttempts`); two employees editing the same participant simultaneously is last-write-wins, unchanged from existing behaviour; if an assigned `scenarioId` no longer exists the participant sees a clear "not configured" message rather than an error.

---

## OPTIONAL FUTURE SECURITY HARDENING

Not required for this implementation, and deliberately not built.

To make the Hackathon a genuine assessment, the answer key has to leave the browser:

```
Hackathon Answer Key
        ↓
Supabase Edge Function  (holds HACK_SCENARIOS server-side)
        ↓
Server-side Scoring     (receives placements, computes score)
        ↓
Participant receives only "recorded"
```

**Shape of the change**

1. Move `HACK_SCENARIOS` into an Edge Function. The client keeps only `{id, title, problem, trigger, decision, close}` and an **unordered, unlabelled** card list — enough to render the board, not enough to know the answer.
2. Add `GET /hackathon-scenario?id=` returning that reduced payload.
3. Add `POST /hackathon-submit` taking `{participantKey, scenarioId, response}`, scoring it server-side with `calculateHackathonScore` (which is already pure and portable — it moves unchanged), writing `h:<key>` with the service-role key, and returning `{recorded: true}` and nothing else.
4. Tighten RLS so the anon role cannot write `h:` rows at all; only the function's service-role key can.
5. Optionally issue a short-lived submission token at login so a participant cannot submit outside their window.

`calculateHackathonScore` was written as a pure function with this move in mind — it takes config plus response and returns a result, with no DOM or global access. Steps 1, 3 and 4 are the substantive work; the participant UI barely changes.
