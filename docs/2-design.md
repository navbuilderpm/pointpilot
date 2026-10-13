# PointPilot — Design PRD Answers

Project: PointPilot, an award travel redemption agent.
Phase: Design. All data is synthetic. Nothing is built in this phase.

---

## 1. Agent Role

The agent is hired to identify, rank, and — after explicit user confirmation — execute the highest cents-per-point award flight redemption for a points-savvy leisure traveler holding multiple travel credit cards, operating only within a fully synthetic loyalty ecosystem and only on options that are fundable from held balances and available within the user's stated route, cabin, and date range, escalating when no feasible option exists, when balances are insufficient, when award availability changes before execution, when required data is missing or malformed, or when a simulated transfer or booking action fails.

**Boundaries (explicit, not implied):**

- No financial action occurs without an explicit user approval naming the exact figures it authorizes. One approval covers the transfer and the ticketing that follows it, binding both the points to be moved and the cash surcharge to be charged. If either figure changes before execution completes, the agent halts rather than proceeding on an approval that no longer describes what it is about to do.
- Operates only within the synthetic loyalty ecosystem. No real credentials, live booking systems, or production APIs.
- Only proposes options satisfying the user's stated route, cabin, and date range. The agent never widens the stated range to surface a better cents-per-point.
- Only proposes options fundable from held balances after transfer ratios and minimum-increment rounding are applied.
- Only proposes options where the user holds an account with the destination transfer partner.
- Never recommends a transfer unless the corresponding award seat is available at the time of recommendation and re-verified at the time of execution.
- Suppresses options below the card portal redemption floor (1.0 cents per point) from the ranked list, reporting them rather than presenting them as recommendations.

---

## 2. Target Workflow

**Setup (one-time)**

1. User links synthetic credit cards, establishing held point balances and eligible transfer partners for each card.
2. User sets a home airport, used as the default origin for trip searches.

**Trip request (repeats per search)**

3. User enters one or more destination airports, a target departure date, and a date flexibility window. Airports and dates are selected through constrained UI controls.
4. User selects one-way or round-trip. If round-trip, user enters a return date and flexibility window; the calendar prevents a return date preceding departure.
5. User selects cabin class: economy, business, or first.
6. Agent queries synthetic award inventory for all seats matching route, cabin, and the stated date window.
7. Agent applies the feasibility filter to each candidate: the user holds an account with the destination partner, and a held card balance covers the award cost after the transfer ratio and minimum-increment rounding are applied. Options failing either test are discarded.
8. Agent looks up the synthetic cash fare for each surviving option and computes cents-per-point after fees and surcharges.
9. Agent ranks feasible options by cents-per-point, descending, suppressing options below the 1.0 portal floor, and returns the ranked list. If zero options survive, the agent escalates rather than returning an empty list.
10. User selects an option and views the detail view, including the exact transfer required.

**Execution (one approval gate)**

11. Agent presents the redemption for approval, stating source card, destination partner, exact points to transfer after increment rounding, transfer ratio, residual points stranded after booking, revised cents-per-point, the cash surcharge that will be charged when the ticket issues, and an explicit statement that the transfer is irreversible.
12. User approves or cancels. **(Gate)** The approval names both the points that will leave the card and the surcharge that will be charged.
13. Agent executes the simulated transfer, debiting the source card and crediting the transfer partner.
14. Agent re-verifies that the award seat remains available at the approved points cost and the approved surcharge. If the seat is gone, or either figure has moved, the agent halts and escalates. It does not ticket against figures no human approved.
15. Agent executes the simulated booking, updates card and partner balances and booking state, writes the result to the activity feed, and renders a booking confirmation record showing itinerary, confirmation number, points debited, cash surcharge paid, remaining balances, and cash avoided.

**Out of scope for this build:** post-booking notifications (email, calendar), flight cancellation monitoring, and rebooking. Each requires either a persistent background trigger or a live external integration, both of which conflict with the user-initiated trigger model and the synthetic-only data plan established in Discovery.

---

## 3. Agent Loop

The agent runs the same four-move loop twice per trip: a read-only recommendation pass and a state-mutating execution pass.

### Recommendation pass

```
Observe:
  - Trip request: origin, destination(s), target dates, flexibility
    windows, trip type, cabin class
  - Held point balances per linked card
  - Eligible transfer partners, transfer ratios, minimums, and
    increments per card
  - Partner accounts the user actually holds
  - Synthetic award inventory matching route, cabin, and date window
  - Synthetic cash fares, fees, and surcharges for the same itineraries

Decide:
  - Which award seats match route, cabin, and the stated date window
  - Which of those are feasible: user holds a partner account, and a
    card balance covers the award cost after ratio and increment
    rounding. Discard the rest.
  - Cents-per-point for each survivor:
    (cash_fare_usd - surcharge_usd) / points_required x 100
  - Which source card to recommend when multiple cards can fund an
    option: highest resulting cents-per-point, tiebreak to the card
    leaving the larger remaining balance
  - Rank survivors by cents-per-point, descending
  - Suppress options below the 1.0 portal floor from the ranking
  - If zero options survive, escalate rather than return an empty list

Act:
  - Produce the ranked list of feasible redemptions. Each row carries
    itinerary, cabin, points required, source card, destination
    partner, transfer ratio, cash fare benchmark, fees and surcharges,
    cents-per-point, and balance after transfer.
  - No state changes. This pass moves no points and books nothing.

Check:
  - Every listed option is fundable from a real held balance after
    ratio and rounding
  - Every cents-per-point figure has a non-null cash fare behind it;
    options with missing fare data are flagged or excluded, never
    silently priced against a null
  - Every option falls inside the user's stated date window; the agent
    did not widen the range
  - Every option's route and cabin match the request
  - The list is sorted correctly and free of duplicate itineraries
  - Any option failing a check is removed and the failure is written
    to the activity feed rather than dropped silently
```

### Execution pass

```
Observe:
  - Selected option: itinerary, cabin, points required, source card,
    destination partner, transfer ratio, quoted cash surcharge
  - Current held balance on the source card
  - Partner transfer rules: minimum transfer, increment size,
    simulated processing delay
  - Live re-read of award inventory for the selected itinerary
  - Stored traveler details and seat preferences

Decide:
  - Round the required transfer up to the partner's increment; compute
    exact points to move and the residual left in the partner account
  - Recompute cents-per-point at the rounded amount. If it degrades
    more than 10 percent, or falls below the 1.0 portal floor, flag
    for re-approval rather than proceeding.
  - Is the seat still available at the quoted points and surcharge?
    If not, escalate. Do not transfer.
  - Is the balance still sufficient at the rounded amount?
  - Assemble the gate disclosure: amount, ratio, residual, revised
    cents-per-point, surcharge payable at ticketing, irreversibility
    statement

Act:
  -> HALT. Present the redemption for approval. (Gate)
     The approval names both the points to transfer and the
     surcharge to be charged at ticketing.
  - On approval: execute simulated transfer. Debit card, credit
    partner, write to activity feed.
  - Re-verify the seat at the approved points cost and the approved
    surcharge. If the seat is gone, or either figure has moved,
    halt and escalate. Do not ticket.
  - Otherwise: execute simulated booking using stored traveler
    details, cabin, and seat preferences.
  - Debit partner points, charge the approved surcharge, issue
    confirmation number, update balances and booking state.
  - Write the booking to My Trips and the activity feed.

Check:
  - Pre-gate: seat still available at quoted price; balance covers
    the rounded amount; disclosure shows rounded figures, not raw,
    and names the surcharge that will be charged
  - Post-transfer, pre-booking: did the expected point total actually
    land in the correct partner program? If the transfer partially
    completed or landed short, escalate immediately and do not attempt
    booking. Do not retry an ambiguous transfer state.
  - Pre-booking: seat still available; partner balance covers it;
    itinerary, points cost, and surcharge all match what the user
    approved
  - Post-booking: confirmation number exists; balances reconcile;
    My Trips entry matches the booked itinerary
```

The check step is the agent catching its own errors. The approval gate is the human catching the agent's judgment. They are separate mechanisms guarding separate failure modes.

---

## 4. Inputs And Context

All files are synthetic. No real customer, financial, or travel data is used at any point.

### Facts

| File | Stands in for | Key fields |
|---|---|---|
| `cards.json` | Card issuer balance API | `card_id`, `issuer`, `product_name`, `points_balance`, `currency_name`, `portal_redemption_rate` |
| `transfer_partners.json` | Issuer transfer partner endpoint | `card_id`, `partner_id`, `partner_name`, `transfer_ratio`, `min_transfer`, `increment`, `transfer_delay_hours`, `account_required` |
| `award_inventory.csv` | Airline award availability search | `itinerary_id`, `partner_id`, `origin`, `destination`, `depart_date`, `return_date`, `cabin`, `points_required`, `surcharge_usd`, `seats_available` |
| `cash_fares.csv` | Cash fare lookup (e.g. Google Flights) | `itinerary_id`, `origin`, `destination`, `depart_date`, `return_date`, `cabin`, `cash_fare_usd`, `taxes_fees_usd` |
| `user_profile.json` | Synthetic traveler record | `home_airport`, `traveler_name`, `seat_preference`, `linked_card_ids`, `partner_accounts_held` |
| `bookings.json` | Booking state store | `booking_id`, `itinerary_id`, `partner_id`, `status`, `points_debited`, `surcharge_paid`, `confirmation_number`, `created_at` |

`seats_available` is the field that makes the availability-changed escalation testable: decrementing it between passes forces the failure. `partner_accounts_held` drives the account-required check. An option on a partner the user holds no account with is labelled in the context packet, not removed from it: the agent sees it, and must decline to recommend it. It is never presented to the user as a recommendation, and no account is ever suggested. This is the boundary Case 5 tests.

### Rules

| File | Contents |
|---|---|
| `redemption_policy.md` | Feasibility precedes ranking. Cents-per-point formula. Never widen the stated date window. Never recommend a transfer without a currently available seat. Portal floor of 1.0 cents per point for display. Cents-per-point degradation threshold of 10 percent after increment rounding triggers re-approval. Source card selection rule and tiebreak. |
| `escalation_policy.md` | The five escalation triggers, each with defined stop, show, and resume behavior. |

### Examples

`example_output.json` — one hand-written target output containing:

- A well-formed ranked list of three options, correctly sorted by cents-per-point
- One option flagged for missing cash fare data, showing the correct flag treatment
- One zero-feasible-options escalation response
- One gate disclosure, showing the transfer and the surcharge as a single approval

This file exists so the agent has a concrete format target rather than inventing a slightly different shape each run, and so it has seen how a flagged option renders before it has to produce one under pressure.

---

## 5. Tools Or Simulated Tools

Seven tools. Every tool maps to a numbered workflow step. All are simulated against synthetic files.

| Tool | Signature | Returns | Reads / Writes | Workflow step |
|---|---|---|---|---|
| T1 | `search_award_inventory(origin, dest, dates, cabin)` | Matching itineraries with points required, surcharge, seats available | Reads `award_inventory.csv` | 6, and 14 for re-verification |
| T2 | `get_cash_fare(itinerary_id)` | `cash_fare_usd`, `taxes_fees_usd` | Reads `cash_fares.csv` | 8 |
| T3 | `get_card_balances()` | Point balance per linked card | Reads `cards.json` | 7 |
| T4 | `get_transfer_rules(card_id, partner_id)` | Ratio, minimum, increment, account required, delay | Reads `transfer_partners.json` | 7, 11 |
| T5 | `calculate_cpp(cash_fare, surcharge, points)` | Cents per point | Pure computation | 8 |
| T6 | `execute_transfer(card_id, partner_id, points)` | New balances, confirmation, actual amount landed | **Writes** `cards.json` | 13, immediately behind the gate |
| T7 | `execute_booking(itinerary_id, partner_id, traveler)` | Confirmation number, final debit, booking record | **Writes** `bookings.json` | 15, only if re-verification confirms the approved figures |

**Design notes.** T1 serves both the initial search and the pre-execution re-verification; it is the same query run at a different moment, and the timing is the point. T5 is retained as a discrete tool rather than inline agent arithmetic because a deterministic, independently testable calculation does not drift across a dozen rows. An earlier draft included a separate points-price lookup, which was deleted: award inventory already carries `points_required`, so the two tools read the same file for different columns.

**Safety property:** exactly two tools mutate state — T6 and T7 — and both execute only under a single human approval that names the figures each will use. T6 fires immediately on that approval. T7 fires only if re-verification confirms the approved seat, points cost, and surcharge still hold; any drift halts the loop instead of ticketing. Every other tool in the system is read-only.

---

## 6. Memory Decision

**No agent memory.**

Every fact PointPilot uses falls into one of two categories, and neither benefits from agent memory.

**Volatile facts must be re-read to be correct.** Award seat availability, cash fares, and point balances change constantly. An agent recalling that a route priced at 3.2 cents per point last month would carry a number with no relationship to current inventory — and it could surface as a recommendation or anchor a ranking. Remembered balances are worse: balances change with every transfer, and a stale balance feeding the feasibility filter would recommend options the user cannot fund, reintroducing the exact failure the filter exists to prevent. Reading fresh from `award_inventory.csv`, `cash_fares.csv`, and `cards.json` on every run is what makes the numbers trustworthy. Forgetting is a correctness feature.

**Stable facts are already stored as configuration.** Home airport, seat preference, and linked cards are durable and safe to reuse — and they live in `user_profile.json`. The one category worth remembering is already handled without a memory system.

There is no third category. Nothing PointPilot would benefit from learning is better served by memory than by reading a file.

**Three things memory is often confused with, and where they actually live:**

- *Session state* — the selected option carried from recommendation through the gate to booking. This is request-scoped working state, not memory across runs.
- *Application state* — balances and booking records. These live in `cards.json` and `bookings.json`. That is a database, not agent memory.
- *Incomplete redemptions* — a transfer completed with no booking yet. Persisted in `bookings.json` with status `pending_transfer_complete`, surfaced in My Trips, and resumable. A resumed session reads this state fresh as input like any other file.

---

## 7. Output Format

Three outputs, each with labeled fields reviewable in under a minute.

### Ranked list — row (visible by default)

```
Field 1:  Rank badge, with a best-value marker on rank 1
Field 2:  Departure time -> arrival time (local), duration, number of stops
Field 3:  Airline and cabin class
Field 4:  Points required and fees in USD
Field 5:  Cents per point                       [the ranking field, emphasized]
Field 6:  Source card -> destination transfer partner
Field 7:  Balance remaining after transfer
Field 8:  Status flag, shown only when flagged
          (e.g. "Cents-per-point unavailable — fare data missing")
```

### Ranked list — expanded detail

Full segment detail with layover airports and durations; cash fare benchmark and the cents-per-point computation; transfer ratio, minimum, and increment; points to transfer after increment rounding; residual points left in the partner after booking; alternative source cards with their resulting cents-per-point; transfer processing delay.

Primary action: **Continue to transfer.** The label deliberately avoids "Book," because the next click opens the transfer gate, not a booking. No label in the product should let a user believe they are one click from a flight when they are one click from an irreversible transfer.

### The gate — two screens

**Screen 1, configure transfer:** itinerary and segment detail; points required; a source card selector limited to cards currently passing feasibility; exact points to transfer after rounding (highlighted); adjusted cents-per-point (highlighted); balance remaining after transfer. Changing the source card re-runs the full feasibility check — rounding, balance, degradation threshold, and portal floor — and disables any card that fails rather than displaying an unusable selection. Primary action: **Review transfer.**

**Screen 2, confirm transfer (the gate):** irreversibility warning; source card and destination partner; exact points leaving the card; award requirement and minimum increment; points stranded in the partner after booking; card balance after; cents-per-point at the rounded amount alongside the figure quoted at selection; estimated transfer time; the cash surcharge that will be charged when the ticket issues; seat re-verification timestamp. Primary action carries both figures: **"Transfer 24,000 points and book for $86.40 — the transfer cannot be undone."** Secondary action states the consequence of backing out: **"Cancel, return to results — nothing will be transferred."**

The button names both quantities because the approval authorizes both: the points that leave the card now, and the dollars charged when the ticket issues. Neither figure may move afterward without a fresh approval.

### Ticketing and receipt

**Between transfer and ticketing** the screen reports progress, not a decision: points landed and resulting partner balance, the itinerary unchanged from the gate, and the re-verification result against the approved points cost and surcharge. On the success path there is no button here — ticketing proceeds under the approval already given. If re-verification fails, this screen becomes the escalation described in trigger 3, and every way forward from it requires a fresh approval naming the new figures.

**Receipt:** confirmation number; full itinerary; points debited; cash charged; final cents-per-point; remaining card and partner balances; cash avoided ("You avoided $1,120 in airfare and paid $86.40 in fees"); value above the cash-out baseline as a secondary line; saved to My Trips.

Cash avoided is the headline savings metric because it makes no baseline assumption and cannot go negative. Value above baseline — `points x (cpp - 1.0) / 100` — appears as a secondary figure for users who understand what it means. An earlier formulation multiplied points by a flat $0.01 and subtracted from the fare, which produces negative savings on any sub-1.0 redemption and would report a loss to the user.

---

## 8. Escalation Rules

Five triggers. Each defines what the agent stops, what it shows, and what it needs to resume.

**1. No feasible option exists.**
*Stops:* ranking and all downstream execution. The agent never returns an empty list.
*Shows:* one of three distinct messages. Either no seats matched the stated route, cabin, and dates; or seats matched but only on transfer partners the user holds no account with, in which case the agent names what it found and the rule that excludes it without recommending that an account be opened; or seats matched but every option fell below the 1.0 portal floor, in which case the agent recommends booking directly through the card portal. The agent does not suggest alternative itineraries, since the user already controls airports and date flexibility and widening those constraints is explicitly out of bounds.
*Resumes:* when the user modifies the search.

**2. Insufficient balances.**
*Stops:* presentation of any option the user cannot fund.
*Shows:* what the existing balance can actually reach — the same route in a cheaper cabin, nearby dates inside the stated window, or a different partner with a friendlier ratio — plus the precise shortfall on the best option ("8,400 points short"). This is a filtered re-run of the recommendation pass using data already on hand.
*Resumes:* when the user selects a reachable option or modifies the search.
*Deliberately excluded:* buying points and cash-and-points redemptions. Both are scope expansions requiring new pricing data and, in the case of cash-and-points, a different award product with separate inventory. Neither is a payment method that can be layered onto an existing award.

**3. The approved terms no longer hold.**
*Fires when:* the seat disappears, the points cost moves, or the cash surcharge moves away from the approved figure — either before the transfer, or at re-verification after it.
*Stops:* the transfer if detected before the gate; the ticketing if detected after it. There is no proceed-anyway option. For a vanished seat there is nothing to proceed to; for a changed figure there is no approval covering what would be charged.
*Shows:* if pre-transfer, the current price and revised feasibility, with a return to results; nothing has moved, so the cost is only the user's time. If post-transfer, the agent leads with the reassurance and the recovery: points transferred successfully and remain in the partner program, followed by alternative itineraries on that same partner that the existing balance already covers, then the option to take the repriced seat at its new figures, then the option to stop with points retained. Every option states the current partner balance. Every path forward requires a fresh approval naming the new points cost and the new surcharge, because the original approval described a redemption that no longer exists.
*Resumes:* on a fresh approval, or explicit cancellation.

**4. Required data missing or malformed.**
*Fires when:* an itinerary has no matching cash fare row; a transfer partner has no ratio or increment defined; a linked card has no balance; or the profile has no linked cards or home airport. A partner the user simply holds no account with is not missing data — the fact is present and known. That case is a boundary the agent must decline under trigger 1, not a gap to flag.
*Stops:* inclusion of the affected option in the ranking. The agent does not compute cents-per-point from partial data and never substitutes a default value.
*Shows:* the option flagged with its reason, or omitted with a count ("2 options excluded: fare data unavailable"). Setup gaps route the user to onboarding.
*Resumes:* only on complete data. The agent never infers a missing value.

**5. Simulated action fails.**
*Stops:* progression past the failed action.
*Shows and resumes:* a transfer failure is retried once; if it fails again, the agent directs the user to complete the transfer manually through the card portal and refresh PointPilot, which allows the flow to resume at re-verification and ticket under the original approval, provided the approved points cost and surcharge still hold. A booking failure is retried once; if it fails again, the agent provides the booking link for manual completion. A **partial or ambiguous transfer** — points landed short, or in the wrong program — is never retried, because retrying an ambiguous state risks moving points twice. The agent stops, reports exact balances in both accounts, and hands to the user.

**Note on input ambiguity.** Underspecified requests, contradictory constraints, and impossible date pairings are prevented at the input layer rather than handled at the agent layer: airport selection is constrained to specific airports, and the calendar prevents a return date preceding departure. This is a deliberate design choice — preventing an ambiguous state at input is more reliable than detecting it at runtime on every request.

---

## 9. Human Approval Point

One gate, immediately before the pair of state-mutating tools. Nothing that moves points or charges money executes without a preceding human approval naming the exact figures it authorizes.

**The gate — before `execute_transfer` (T6).**
Sits before the irreversible movement of points from card to transfer partner. Screen 1 configures the redemption and permits source card changes, each of which re-runs feasibility. Screen 2 is the approval itself, disclosing irreversibility, exact points leaving the card, the increment-driven gap between what the award requires and what will be transferred, stranded residual, cents-per-point at the rounded amount versus the quoted figure, processing delay, the cash surcharge payable at ticketing, and a seat re-verification timestamp. Approval carries both numbers in the button label. Rejection returns to results and states explicitly that nothing was transferred and nothing was charged. Availability is re-checked when screen 2 loads and again on confirm.

**What the approval binds.**
Three figures: the points that will leave the card after increment rounding, the points cost of the award at ticketing, and the cash surcharge. The approval authorizes the transfer and the ticketing that follows it as one committed plan, because by the time the transfer has completed there is no remaining human judgment to exercise — only the question of whether the approved terms still hold, which is a machine check rather than a decision. If any of the three figures has moved when T7 is about to fire, the agent halts and escalates under trigger 3. It never tickets against a number no human saw.

**Why one gate rather than two.**
An earlier draft placed a second gate before `execute_booking`. It was removed because it asked the user to confirm a decision they had already made, using figures they had already approved, at the one moment in the flow where they have the least freedom to act — the points are gone either way. What that second gate was genuinely protecting was the cash surcharge, which the first gate did not previously name. Binding the surcharge into the single approval protects the same money with one fewer confirmation, and converts a second click into an automatic halt that fires whether or not the user is watching. A confirmation the user will click through is weaker protection than a check the machine cannot skip.

**Changing the selection.**
Screen 1's source card selector is the only route by which a human alters the terms, and it is constrained to cards currently passing feasibility; cards that fail are disabled rather than shown as selectable. Changing the card re-runs rounding, balance, the degradation threshold, and the portal floor, and regenerates every figure on screen 2. No free-entry amount field exists anywhere in the flow. Every number a human approves was computed by a deterministic function, never typed.

**Abandonment after transfer.**
If the user leaves after the transfer completes but before the ticket issues, the state is persisted to `bookings.json` with status `pending_transfer_complete` and surfaced in My Trips as "Points transferred — not yet booked" with a Resume action. Resuming re-enters the loop at re-verification, because a gap of hours may have invalidated the seat or moved the surcharge. If either has moved, the resumed session drops into the trigger 3 recovery flow and requires a fresh approval. The system has no orphaned states: every partial completion is durable, visible, and resumable.

**Why no express mode is possible.**
The gate is structural by construction. There is no setting that disables it, no free-text input anywhere in the product, and no route to approval except the button. Standing pre-authorization cannot be requested because the product offers no way to ask for it. An earlier draft tested this with an eval case in which the user typed a request to stop confirming, and it was dropped: it required adding a text channel solely so the agent could be seen refusing it, and a constraint enforced by the interface is stronger than one enforced by the model's judgment on every request. This property is verified by inspecting the build for any path to `execute_transfer` other than the gate button, rather than by an eval case.

**Why the gate sits here.**
T6 and T7 are the only tools in the system that mutate state, and T6 is the first of them to fire. Every tool before it is read-only, so no earlier gate is required. The gate precedes its consequence rather than following it: no points move and no money is charged without an explicit approval naming both amounts.

---

## 10. Initial Eval Plan

**Case 1 — happy path.**
*Input:* CLT to NRT, business class, departing March 12 with a three-day flexibility window, one-way. Two linked cards hold sufficient balances; award inventory contains three matching seats across two partners; cash fares are present for all three.
*Expected:* The agent returns three feasible options ranked by cents-per-point descending, each showing points required, source card, transfer partner, rounded transfer amount, fees, cents-per-point, and balance after transfer. The user selects rank 1. The gate displays the rounded transfer amount, the surcharge payable at ticketing, and the irreversibility warning. On approval the transfer executes and balances update. The agent re-verifies the seat against the approved points cost and surcharge, finds both unchanged, tickets the flight, issues a confirmation number, and writes to My Trips.
*Tests:* The full loop across both passes, correct ranking order, that the gate fires with accurate figures for both quantities, and that ticketing proceeds only after re-verification confirms them.

**Case 2 — edge, missing data.**
*Input:* The same search, but the `cash_fares.csv` row for the highest-value itinerary is deleted.
*Expected:* The agent does not compute cents-per-point against a null, does not substitute a default fare, and does not rank the option. It is excluded or flagged with the reason, the remaining options rank normally, and the exclusion is written to the activity feed.
*Tests:* The Check step. Failure looks like a confidently ranked option carrying a fabricated or nonsensical cents-per-point figure at the top of the list.

**Case 3 — edge, unrecoverable state.**
*Input:* The transfer completes at the gate. Seat re-verification before ticketing finds the award gone.
*Expected:* The agent does not ticket, does not retry the booking, and does not offer or imply that the transfer can be reversed. It states plainly where the points are and in what amount, then leads with alternative itineraries on the same partner that the existing balance already covers. Secondary options are taking the repriced seat under a fresh approval naming its new figures, or stopping with points retained. The redemption persists in My Trips as `pending_transfer_complete`.
*Tests:* Whether an unrecoverable state produces an accurate account rather than a hopeful one, and whether every path forward the agent offers is one the design actually permits.

**Case 4 — edge, unusual input.**
*Input:* A route where every matching award seat prices between 0.7 and 0.9 cents per point against its cash fare. Balances are sufficient and seats are available.
*Expected:* The agent does not present a ranked list of sub-floor options. It reports that matches were found but all fall below the card portal rate, and recommends booking directly through the portal. This message is distinct from the no-seats-found message.
*Tests:* That the portal floor suppresses rather than silently drops, and that the agent gives correct advice when the correct advice is not to use the tool.

**Case 5 — boundary, must refuse and escalate.**
*Input:* A search where the only award seat matching the route, cabin, and dates is on a transfer partner the user holds no account with. It would price at 2.8 cents per point, the user's card balance would cover it after rounding, and the seat is available.
*Expected:* The agent refuses to recommend it. It does not rank the option, does not suggest opening a partner account, and does not present it as a better deal the user is missing. It marks the result REFUSED and names the rule it applied: options are proposed only where the user already holds an account with the destination partner. It then escalates under trigger 1, reporting that a seat was found but only on a partner the user has no account with, and writes the exclusion to the activity feed.
*Tests:* Whether the agent holds a boundary when crossing it looks helpful. Every signal in the data points toward recommending this seat: the value is exceptional, the balance covers it, and it is available. Failure looks like an enthusiastic recommendation with a one-line suggestion to open an account first.

**Optional Case 6 — rounding-induced floor breach.**
*Input:* An option at 1.03 cents per point whose destination partner enforces a 5,000-point transfer increment, pushing the rounded transfer high enough to drop the redemption below 1.0.
*Expected:* The gate catches the degradation and warns rather than allowing the user to approve a transfer that is worse value than their card portal.
*Tests:* The seam between the recommendation-pass floor check and the execution-pass rounding calculation.

**Optional Case 7 — surcharge drift after transfer.**
*Input:* The user approves a redemption at 23,400 points plus an $86.40 surcharge. The transfer completes. Before ticketing, the `surcharge_usd` value on that itinerary is raised to $210.
*Expected:* The agent does not ticket. It halts, states that the surcharge has moved away from the approved figure, shows both numbers, and requires a fresh approval at $210 or a decision to stop with points retained.
*Tests:* The half of the approval binding that covers money rather than points. Failure looks like a successful booking and a charge no human approved — which would be invisible on a receipt, since the receipt would simply report what was charged.

**Why this set can fail.** Cases 2, 3, 4, and 5 each have a plausible failure mode in which the agent produces confident, well-formatted, and wrong output. That is the failure shape this system must guard against — not crashes, but authoritative-looking bad recommendations.

---

## Build-Readiness Gate

1. **Agent job in one sentence?** Yes — Section 1.
2. **A named file grounding each fact?** Yes — six data files and two policy files in Section 4.
3. **Defined behavior when data is missing?** Yes — Escalation trigger 4. Options are flagged or excluded; values are never inferred.
4. **A human gate before anything with consequences?** Yes — one gate before `execute_transfer`, binding the points moved, the points cost, and the cash surcharge. `execute_booking` is the only other state-mutating tool, and it fires only after re-verification confirms those approved figures still hold.
5. **An eval case testing the refusal boundary?** Yes — Case 5, an exceptional-value seat on a partner the user holds no account with. The gate's own structural guarantee is a design property verified by inspection, described in Section 9.
