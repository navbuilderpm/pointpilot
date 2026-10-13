# PointPilot Escalation Policy

Synthetic policy for the PointPilot prototype. Cite sections by number (for example "EP-3"). Each trigger defines what stops, what is shown, and what resumes.

## EP-1 No feasible option exists
- **Stops:** ranking and all downstream execution. Never return an empty list.
- **Shows exactly one of three messages:**
  - (a) No seats matched the stated route, cabin, and dates.
  - (b) Seats matched, but only on transfer partners the user holds no account with. Name what was found and the rule that excludes it (RP-2a). Do not recommend opening an account (RP-11).
  - (c) Seats matched, but every option fell below the 1.0 portal floor. Recommend booking directly through the card portal.
- Do not suggest alternative airports or dates. The user controls those.
- **Resumes:** when the user modifies the search.

## EP-2 Insufficient balances
- **Stops:** presentation of any option the user cannot fund.
- **Shows:** what the existing balances can reach — the same route in a cheaper cabin, other dates inside the stated window, or a different partner — plus the exact shortfall on the best option (for example "8,400 points short").
- Buying points and cash-and-points redemptions are out of scope.
- **Resumes:** when the user selects a reachable option or modifies the search.

## EP-3 The approved terms no longer hold
- **Fires when:** the seat disappears, the points cost moves, or the cash surcharge moves away from the approved figure. This applies before the transfer or at re-verification after it.
- **Stops:** the transfer, if detected before it. The ticketing, if detected after the transfer. There is no proceed-anyway option.
- **Shows before the transfer:** the current price and revised feasibility, with a return to results. Nothing has moved.
- **Shows after the transfer:**
  1. Lead with the fact that the points transferred successfully and remain in the partner program, with the exact partner balance.
  2. Then list alternative itineraries on the same partner that the balance already covers.
  3. Then offer the repriced seat at its new figures.
  4. Then offer to stop with the points retained.
- Never state or imply that a transfer can be reversed. Record the redemption as `pending_transfer_complete`.
- **Resumes:** only on a fresh approval that names the new points cost and surcharge, or on explicit cancellation.

## EP-4 Required data missing or malformed
- **Fires when:**
  - an itinerary has no cash fare row;
  - a partner has no ratio or increment;
  - a linked card has no balance; or
  - the profile has no linked cards or home airport.
- A partner the user holds no account with is **not** missing data. That case falls under EP-1(b).
- **Stops:** inclusion of the affected option in the ranking. Never compute from partial data. Never substitute a default.
- **Shows:** the option flagged with its reason, or omitted with a count (for example "1 option excluded: fare data unavailable"). Setup gaps route to onboarding.
- **Resumes:** only on complete data.

## EP-5 Simulated action fails
- **Stops:** progression past the failed action.
- **Transfer failure:** retry once. If it fails again, direct the user to complete the transfer in the card portal, then resume at re-verification under the original approval if the approved figures still hold.
- **Booking failure:** retry once. If it fails again, provide the booking link for manual completion.
- **Partial or ambiguous transfer** (points landed short, or in the wrong program): never retry. Stop, report the exact balances in both accounts, and hand control to the user.
