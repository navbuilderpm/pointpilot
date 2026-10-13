# PointPilot

**An award-travel agent that finds the best-value flight your credit-card points can actually book — and never moves a point without your approval.**

A capstone prototype for the Agentic AI Capstone, live at **https://navbuilderpm.github.io/pointpilot/**

---

## What it does

Points-savvy travelers waste hours comparing transfer partners, ratios, and award prices — and still sometimes transfer points that can't be undone, only to find the seat gone. PointPilot runs one agent loop, with every stage labeled on screen:

1. **Input** — a one-way trip search (departure city, destination, cabin, date window), or one of 14 demo trips.
2. **Context** — the app gathers every matching award seat with its cash fare, cents-per-point value, account status, and each card's rounded transfer amount, plus two policy files.
3. **Decision** — the agent (Claude, via the Anthropic API) includes, flags, or excludes each seat with a reason and a policy citation, then ranks, escalates, or refuses.
4. **Output** — a ranked list of bookable awards; unbookable seats are shown separately with the reason.
5. **Human review** — you pick the paying card and approve one gate that names the exact points leaving your card and the cash surcharge. Only then does the app transfer, re-check the seat, and book — or halt with recovery options if anything changed.

All figures are computed by deterministic code; the agent makes the judgment calls and explains them.

## Try it

1. Open the live link above (desktop browser).
2. Click your profile (top right) and paste your own **Anthropic API key**. It is stored only in your browser (localStorage) — never in this repository and never sent anywhere except Anthropic's API.
3. Click a deal card ("Best award deals from Charlotte") or run a search.

Without a key you can still browse the whole interface; the agent step shows a message asking for a key. That is by design: the public site shows the product, and the agent runs for whoever brings their own key.

**Worth trying:** Tokyo (happy path, booked end to end) · Seoul (boundary — the agent refuses) · Rome (the seat disappears after the transfer; watch the recovery) · the **Evals** view (8 test cases with recorded verdicts) · the **Customer view** toggle.

## What's in this repository

| Path | What it is |
|---|---|
| `index.html` | The whole app — one file, plain HTML/CSS/JavaScript, no build step. Data, styles, and photos are embedded so it also runs offline by double-click. |
| `data/` | The synthetic data the app is built from: cards, transfer partners, award seats, cash fares, trip requests, eval cases. These files are the source; the app carries a built-in copy. |
| `policies/` | The redemption rules (RP-1 to RP-11) and escalation triggers (EP-1 to EP-5) the agent cites. |
| `docs/1-discovery.md` | Discovery PRD — the problem and the workflow. |
| `docs/2-design.md` | Design PRD — the agent's role, context, boundaries, outputs, and evals. |
| `docs/3-develop.md` | Develop PRD — what was built, eval results, the improvement made, known limitations. |

## Honest limits

- **Prototype, not production.** Transfers and bookings are simulated in memory; no real accounts, airlines, or money are involved.
- **All data is synthetic** — fictional banks, cards, airlines, and traveler. Airport codes are real; everything else is made up. A real pilot would require a formal privacy review first.
- One-way trips, one traveler, from six US cities; the agent reviews one search at a time.
- Full list: *Known limitations* in `docs/3-develop.md` and at the bottom of the app's **Evals** view.

Destination photos are from Wikimedia Commons under their stated licenses; credits are in the app ("Photo Credits" under the deal cards).
