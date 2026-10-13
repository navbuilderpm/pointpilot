# PointPilot Redemption Policy

Synthetic policy for the PointPilot prototype. Cite sections by number (for example "RP-4").

## RP-1 Stated search only
Propose only options that match the trip request's origin, destination(s), and cabin, and whose departure date falls inside the stated window: `depart_date` plus or minus `depart_flex_days`. Never widen the route, cabin, or date window to surface better value.

## RP-2 Feasibility precedes ranking
An option may be ranked only if all three hold:
- (a) The user holds an account with the destination partner (`partner_accounts_held` in `user_profile.json`).
- (b) At least one linked card that transfers to that partner can fund the transfer amount under RP-3.
- (c) The seat is currently available (`seats_available` of 1 or more).

Options that fail are not ranked. They are reported with the failed test.

## RP-3 Transfer amount and rounding
- Card points needed = `points_required` divided by `transfer_ratio`, rounded up to a whole point.
- The transfer amount is that figure rounded up to the partner's `increment`, and never below `min_transfer`.
- Residual (stranded) points = (transfer amount × `transfer_ratio`) minus `points_required`.

## RP-4 Cents-per-point formula
cents-per-point = (`cash_fare_usd` − `surcharge_usd`) ÷ card points needed × 100.

"Card points needed" is the unrounded figure from RP-3. At a 1:1 ratio it equals `points_required`. Report cents-per-point to two decimals.

## RP-5 No price without a fare
If an itinerary has no row in `cash_fares.csv`, its cents-per-point is unavailable. Do not compute it, estimate it, or substitute a default fare. Flag or exclude the option with the reason "Cents-per-point unavailable — fare data missing" (see EP-4).

## RP-6 Portal floor
The card portal redeems at 1.0 cents per point (`portal_redemption_rate`). Options below 1.0 are suppressed from the ranked list and reported as below the floor. They are never recommended.

## RP-7 Source card selection
When more than one card can fund an option, recommend the card that gives the highest cents-per-point. If cents-per-point ties, choose the card that leaves the larger remaining balance.

## RP-8 Ranking
Rank feasible, above-floor options by cents-per-point, highest first. Mark rank 1 as best value. List each itinerary once.

## RP-9 Rounding degradation
Recompute cents-per-point at the rounded transfer amount:

(`cash_fare_usd` − `surcharge_usd`) ÷ transfer amount × 100

If the result is more than 10 percent below the quoted figure, or falls below the 1.0 floor, flag the option for re-approval. Do not proceed.

## RP-10 One approval, three bound figures
One human approval authorizes the transfer and the ticketing that follows it. It binds three figures: the points leaving the card, the award's points cost, and the cash surcharge. If any of the three changes before ticketing, halt under EP-3. Never ticket against a figure no human approved.

## RP-11 No account suggestions
Never suggest opening a partner account. An option on a partner the user holds no account with is not presented as a recommendation or as a missed deal.
