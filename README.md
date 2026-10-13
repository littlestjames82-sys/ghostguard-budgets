# GhostGuard Budgets + Kill Switch

**Stateful budgets, velocity rules, and a kill switch — on top of GhostGuard.**

GhostGuard's policy engine judges one action at a time. That's the
right core, but real incidents are cumulative: no single purchase was
over the line, the *day's total* was; no single delete was suspicious,
the *third one in an hour* was. This add-on (GhostGuard more-ideas #1
+ #4, built Oct 10, 2026) adds the stateful layer and the big red
button, without touching the core package — it imports the engine
from `../ghostguard/dist`, exactly like the MCP adapter does.

By **Ghost Developer Studio**. Private, unpublished. Zero runtime
dependencies; Node ≥ 18; ESM.

## The one guarantee that shapes everything

**The stateful layer can only make a verdict stricter, never looser.**

The stateless policy decides first. Budgets/velocity then compute a
candidate verdict from recorded history, and the final verdict is the
*stricter* of the two by rank (`deny > hand_off > ask > pre_approved >
allow`). A budget can never grant, widen, or resurrect anything: a
locked `no-spend-ever` deny stays a deny no matter how large the
configured budget is, and a config that tries to set
`onExceed: "allow"` is clamped to `"deny"` at load time (and reported
in `guard.warnings`). Both properties are covered by tests.

## Quick start

```js
import { createStatefulGuard } from "./src/index.mjs";

const guard = createStatefulGuard({
  policy: myGhostGuardPolicyDoc,        // or a compiled policy
  configPath: "./budgets.config.json",   // or config: { ... }
  statePath: "./ghostguard-budgets-state.json",
  freezePath: "./ghostguard.freeze.json",
  recorder: myActionRecorder,           // optional — GhostGuard record contract
  // now: () => Date.now(),              // injectable clock (tests use this)
});

const { decision, record, stateful } = guard.decide({
  kind: "shop.buy",
  permission_class: "SPEND",
  actor: "checkout-agent",
  project_id: "storefront",
  initiator: "routine",
  payload: { amount: 129.99, item: "…" },
});

if (decision.verdict === "allow" || decision.verdict === "pre_approved") {
  runTheAction(); // execute only on a cleared verdict — as in GhostGuard core
}
```

`guard.status({ actor, project_id })` returns the freeze state,
per-budget used/remaining, and per-rule window counts.

## Config format

```json
{
  "spendBudgets": [
    { "id": "daily-spend-cap", "scope": "actor_project",
      "window": "24h", "limit": 500,
      "amountField": "amount", "classes": ["SPEND"], "onExceed": "deny" }
  ],
  "velocityRules": [
    { "id": "third-delete-asks", "scope": "actor",
      "window": "1h", "match": { "permission_class": "DELETE" },
      "maxInWindow": 2, "onExceed": "ask" }
  ]
}
```

**Windows** — `"window"` accepts `"30s" | "15m" | "1h" | "24h" | "7d"`
(or `windowMs`, `windowHours`, `windowDays`, …). Windows are rolling,
computed from recorded event timestamps: an event counts while
`now - window < ts <= now`.

**Scopes** — `actor` (per proposing agent), `project` (per project),
`actor_project` (per pair; the default), `global`.

**Spend budgets** — sum the `amountField` (default `payload.amount`)
of cleared SPEND events in the window; a new spend that would push
the sum over `limit` gets `onExceed` (default `deny`). A spend whose
amount is missing or non-numeric escalates to `ask` — the guard
cannot verify what it cannot see. Spending exactly up to the limit
is allowed; the next unit over is not.

**Velocity rules** — count cleared events matching `match`
(`kind` / `kinds` / `permission_class` / `permission_classes`) in the
window. With `maxInWindow: 2`, the 1st and 2nd matching actions flow
normally and the 3rd gets `onExceed` — escalation fires on the Nth
action, not before.

**Counting semantics** — only actions whose final verdict is
`allow` / `pre_approved` are counted and persisted. Attempts that
were denied or asked do not consume budget; an asked 3rd delete does
not make the 4th request "more denied."

## Kill switch

```sh
node src/cli.mjs freeze   --freeze-file ./ghostguard.freeze.json --by ryan --reason "agent acting weird"
node src/cli.mjs status   --freeze-file ./ghostguard.freeze.json --state ./ghostguard-budgets-state.json --config ./budgets.config.json
node src/cli.mjs unfreeze --freeze-file ./ghostguard.freeze.json
```

While the freeze file exists, **every consequential class — all
except `READ` — is denied with `rule_id: "kill-switch-freeze"`.**
`READ` still flows through policy + budgets normally. The guard
re-reads the freeze file on *every* decision, so flipping it mid-run
takes effect on the very next action; unfreezing restores flow the
same way. A freeze file whose body is `{"frozen": false}` is not a
freeze; a freeze file that exists but cannot be parsed **is** a
freeze (fail closed — a torn write must never silently unfreeze).

## Failure behaviour (fail closed, everywhere)

- **State file corrupt / unreadable / wrong shape** → every decision
  is `deny`, rule `state-store-unavailable`, READ included. Only a
  *missing* state file means "empty state."
- **State persist fails after a cleared verdict** → the verdict is
  overridden to `deny` (`state-store-unavailable`) before it is
  returned. If the guard can't record an action against its budgets,
  the action doesn't run.
- **Recorder throws** → the throw propagates from `decide()`; the
  caller executes nothing. Same contract as GhostGuard core.

## State file

Plain JSON: `{ "version": 1, "events": [{ ts, actor, project_id,
kind, permission_class, amount }] }`. Writes are atomic-ish (temp
file + rename in the same directory). Events older than the largest
configured window are pruned on each counted write.

## Honest limits

- **Single-node file state.** One JSON file, loaded and rewritten per
  counted action. That serialises writers on one machine and does not
  scale across hosts or processes racing the same file — the rename
  keeps the file valid, but two guards on two machines have two
  budgets, not one. A shared store (GhostGuard Cloud / a real DB) is
  the distributed answer; this add-on is the local one.
- **The kill switch is a file.** Anything with write access to the
  freeze file can freeze or unfreeze the system. Protect it like a
  credential; the audit record shows *that* a freeze decided, and the
  freeze file records who/reason — it cannot prove who really wrote it.
- **Amounts are trusted as classified.** The budget sums the
  payload's amount field as the integrator classified it; a caller
  that understates `payload.amount` understates its budget use.
  Classification discipline is load-bearing, as in GhostGuard core.
- **Not a compliance tool.** This add-on gates effects and keeps
  state about them. It certifies nothing.

## Tests

```sh
npm test        # = npm run verify — 39 checks, all injected-clock, no sleeps
```

Covering: under/over-cap budgets and counting semantics, per-actor
scope isolation, window expiry for budgets and velocity, Nth-action
escalation timing, amount-missing escalation, stricter-only vs a huge
budget and vs a hostile `onExceed: "allow"` config, kill-switch
mid-run flips (SPEND/DELETE/COMMUNICATE blocked, READ flows, unfreeze
restores, corrupt freeze fails closed), corrupt/wrong-shape state
fails closed, persistence across guard instances, recorder records,
and the CLI freeze/status/unfreeze round-trip.

Package layout: `src/guard.mjs` (decision flow), `src/config.mjs`
(normalisation), `src/state.mjs` (store), `src/freeze.mjs` (kill
switch), `src/cli.mjs` (freeze/unfreeze/status), `config.example.json`.

## Repository layout note

This add-on imports the GhostGuard core engine from a **sibling checkout**:
`../../ghostguard/dist/index.js` (its source and/or tests do; nothing in the
core is modified). To run it, clone the core next to this repo so both folders
share a parent directory:

```sh
git clone https://github.com/littlestjames82-sys/ghostguard
git clone https://github.com/littlestjames82-sys/ghostguard-budgets
# build the core once so dist/ exists:
cd ghostguard && npm run build && cd ..
cd ghostguard-budgets && npm test
```

Part of the GhostGuard family by **Ghost Developer Studio**.
