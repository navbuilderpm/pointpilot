# PointPilot — Discovery PRD Answers

**Project:** PointPilot, an award-travel redemption agent
**Phase:** Discovery (revised to align with Design PRD v2.2)

---

## Workflow summary

An award-travel redemption workflow where the agent takes a user's trip request, finds award seats on the user's transfer partners, filters out anything the user can't actually fund or book, computes cents-per-point value against cash fares, and ranks the feasible options. After one explicit human approval that names the exact points to transfer and the cash surcharge to be charged, it executes the points transfer and the booking in a simulated environment. It halts instead of ticketing if either figure changes before the booking goes through.

---

## 1. User

A points-savvy, middle-class leisure traveler who holds 3 travel credit cards across 2 issuers and has accumulated roughly 450,000 points from everyday spending. They hold accounts with some, but not all, of those cards' airline transfer partners, and they fly from a regular home airport. They take 1–2 international long-haul trips per year plus occasional domestic trips. They want to "pointsmax" (get the most value out of their points), but they don't have time to check many loyalty programs by hand, and they can't afford to make an irreversible transfer mistake.

## 2. Workflow

Award-travel redemption planning and execution, in two passes.

**Recommendation pass (read-only).** Given a trip request, the agent:

1. Searches award inventory for seats that match the route, cabin, and stated date window.
2. Discards any option the user can't actually use. An option survives only if the user holds an account with that transfer partner and a card balance covers the award after the transfer ratio and minimum-increment rounding are applied.
3. Benchmarks each survivor against the cash fare and computes cents-per-point (cpp) after surcharges.
4. Ranks the survivors by cpp. Options below the 1.0 cpp card-portal floor are reported but never recommended.
5. Escalates instead of returning an empty list when nothing survives.

**Execution pass (state-changing).**

1. The user selects an option.
2. The agent presents a single approval gate that states the exact points to transfer after rounding, the stranded residual, the revised cpp, and the cash surcharge.
3. On approval, the agent executes the simulated transfer.
4. It re-verifies that the seat is still available at the approved figures.
5. It completes the simulated booking and issues a receipt showing the cash avoided.

## 3. Trigger

The user submits a trip request through a form-driven web UI:

- **Origin:** defaults to the user's saved home airport.
- **Destinations:** one or more destination airports.
- **Departure:** a target departure date and a flexibility window (± days).
- **Trip type:** one-way or round-trip. Round-trip adds a return date and flexibility window, and the calendar blocks a return date before the departure date.
- **Cabin:** economy, business, or first.

Every input comes from a constrained control. There is no free-text box, so the UI makes ambiguous or contradictory requests impossible to submit.

## 4. Current process

Today the user:

1. Manually looks up which airlines are transfer partners of each card.
2. Often creates loyalty program accounts just to see award prices.
3. Navigates several hard-to-use airline websites to check availability, points cost, and carrier surcharges.
4. Works out the transfer ratio and the partner's minimum transfer increment to see how many points actually have to move.
5. Calculates cents-per-point against the cash fare by hand.
6. Logs into the card issuer's portal to transfer the points. The transfer can't be undone.
7. Goes back to the airline site and hopes the seat is still there at the same price before completing the booking.

A single trip can take well over 30 minutes of cross-referencing across many sites.

## 5. Pain points

- Transfer-partner mappings are hard to track across multiple cards.
- Award prices are hidden behind loyalty accounts and poor site UX.
- Cpp math is done by hand, is error-prone, and often ignores surcharges.
- Minimum transfer increments force the user to move more points than the award needs. The leftover points are stranded in the partner program.
- Transfers can't be reversed. If the seat disappears or reprices between the transfer and the booking, the points are stuck in a program the user may not want.
- The transfer and booking steps happen in separate portals, with nothing checking that the approved terms still hold between them.
- Good redemptions are missed because checking every program is too tedious.

## 6. Agent opportunity

The agent runs a perceive → decide → act → check loop twice per trip: a read-only recommendation pass and a state-changing execution pass.

- **Deterministic tools do the arithmetic.** Seven simulated tools handle award search, cash fare lookup, balances, transfer rules, cpp calculation, transfer execution, and booking execution. Keeping all math in these tools means no figure drifts from one run to the next.
- **The model makes the judgment calls.** It sees the full candidate set, including rows with missing fares, sub-floor rows, and rows on partners the user has no account with. For each one it decides whether to include, flag, exclude, or escalate.
- **The check step catches the agent's own errors.** It confirms that every option is fundable, that nothing is priced against a missing fare, that the date window was never widened, and that the transfer landed in full before any booking is attempted.
- **The agent acts, visibly.** It executes the transfer and booking against the mock environment, so card balances drop, partner balances change, and a booking appears in My Trips.

Only 2 of the 7 tools (transfer and booking) change state, and both sit behind the human approval.

## 7. Synthetic data plan

All data is fictional and self-built.

**Facts**

| File | Stands in for | Contents |
|---|---|---|
| `cards.json` | Card issuer balance API | 3 fictional cards, issuers, point balances, portal redemption rate |
| `transfer_partners.json` | Issuer transfer partner endpoint | Card-to-partner mappings, transfer ratios, minimum transfer, increment size, transfer delay, account-required flag |
| `award_inventory.csv` | Airline award availability search | Itineraries by partner, route, dates, and cabin, with points required, surcharge, and seats available |
| `cash_fares.csv` | Cash fare lookup | Matching cash fares and taxes/fees for cpp benchmarking |
| `user_profile.json` | Traveler record | Home airport, traveler name, seat preference, linked cards, partner accounts held |
| `bookings.json` | Booking state store | Booking records and status, including `pending_transfer_complete` for transfers not yet ticketed |

**Rules:** `redemption_policy.md` (feasibility before ranking, cpp formula, 1.0 portal floor, 10 percent degradation threshold, source-card tiebreak) and `escalation_policy.md` (five escalation triggers).

**Example:** `example_output.json`, a hand-written target output that shows a correctly ranked list, a flagged option, an escalation, and a gate disclosure.

**Eval scenarios:** built on the same files, and anchored on seven designed cases:

1. Happy path
2. Missing cash fare
3. Seat gone after the transfer
4. All options below the portal floor
5. High-value seat on a partner the user has no account with
6. Rounding that pushes an option below the floor
7. Surcharge changing after the transfer

Edge cases can be triggered by editing a single field, for example decrementing `seats_available` or deleting a fare row.

No real accounts, credentials, personal data, or live airline or bank systems are used. Card "linking" is a mock selection screen with no credential entry.

## 8. Human boundary

**One approval gate sits before any points move or money is charged.** The approval binds three figures:

1. The points leaving the card after increment rounding
2. The award's points cost at ticketing
3. The cash surcharge

The approval button names both the points and the dollar amount, and states that the transfer can't be undone. That one approval authorizes the transfer and the ticketing that follows it. If any of the three figures moves before ticketing, the agent halts and asks for a fresh approval. It never tickets against a number no human saw.

The gate is structural. There is no setting that turns it off, no free-text input, and no free-entry amount field. Every figure the user approves was computed by a deterministic function. The only way to change the terms is the source-card selector, which is limited to cards that pass feasibility.

The agent also may never:

- Widen the user's stated route, cabin, or date range to find better value
- Propose an option the user's balances can't fund after rounding
- Propose an option on a partner the user holds no account with, or suggest opening one
- Recommend a transfer without a currently available seat
- Retry an ambiguous or partial transfer

All actions occur only in the simulated environment.

## 9. Success metric

- **Primary: recommendation accuracy.** The percentage of eval scenarios where the agent's top recommendation matches the annotated best feasible redemption.
- **Secondary: option recall.** The percentage of all feasible, above-floor redemptions the agent surfaces per scenario, so no savings opportunity is lost.
- **Guardrail: boundary and escalation correctness.** The percentage of edge and boundary scenarios where the agent does the right thing: flags or excludes instead of fabricating a value, refuses instead of recommending, and halts instead of ticketing. The failure this system must guard against is confident, well-formatted, wrong output, not crashes.
- **Supporting: time benchmark.** A small-sample comparison of manual search (roughly 30+ minutes per trip, measured on 2–3 scenarios) against the agent's end-to-end run time.
- **User-facing value: cash avoided.** Shown on every receipt, for example "You avoided $1,120 in airfare and paid $86.40 in fees." Value above the 1.0 cpp baseline appears as a secondary line.

## 10. Initial demo idea

A web UI walkthrough of one trip, end to end:

1. **Setup.** Select 3 fictional cards and set a home airport. Balances, transfer partners, and held partner accounts appear.
2. **Trip request.** Submit the form: for example, CLT to NRT, business class, one-way, ±3 days.
3. **Live agent activity feed.** Each step streams visibly: searching award space, applying the feasibility filter, fetching cash fares, computing cpp, and logging exclusions.
4. **Ranked list.** Feasible options sorted by cpp, with the best value marked. Each row shows points and fees, cpp, source card → partner, and the balance after transfer. Any flagged options show their reason.
5. **Detail view.** Shows the rounded transfer amount, the stranded residual, and alternative source cards. The primary action is "Continue to transfer," never "Book."
6. **The gate.** First, configure the source card. Then confirm with a single button such as "Transfer 24,000 points and book for $86.40 — the transfer cannot be undone."
7. **Execution.** The card balance visibly drops and the points land in the partner program. Re-verification confirms the approved figures, and ticketing proceeds.
8. **Receipt and My Trips.** Confirmation number, itinerary, points debited, surcharge paid, final cpp, remaining balances, and cash avoided. This proves the simulated world changed.

**Optional second run:** a boundary case, such as a 2.8 cpp seat on a partner the user has no account with. The demo shows the agent refusing to recommend it and escalating instead.
