# Architecture

> Design phase. Nothing here is implemented yet. Types are written in TypeScript for readability; the implementation language is an open question (see the end of this document).

## The one idea

A language model is good at turning *"sell a third of my AAPL in the Roth if it closes below the 50-day"* into structure, and bad at being trusted with money. So the system is split in two by a **trust boundary**:

```
          MODEL ZONE — can be wrong              │            CODE ZONE — deterministic
                                                 │
 you type ──▶ compile ──▶ resolve ──▶ OrderIntent│──▶ risk gate ──▶ ticket ──▶ adapter ──▶ journal
                 ▲                               │                    ▲
                 │ reads news, filings, web      │                 you approve
                 │ (data only — cannot emit)     │
```

Everything left of the boundary produces a *proposal*. Everything right of it is ordinary code, and nothing routes without an approval record.

## Components

| Component | Zone | Responsibility |
|---|---|---|
| Front-end | — | Terminal first; chat, voice and Telegram later. Collects user text and shows tickets. |
| Compiler | Model | Turns user text into a draft `OrderIntent`. Uses the user's own model key or a local model. No broker access. |
| Resolver | Model + user | Replaces every ambiguous term with an explicit field, or asks the user. Records each resolution. |
| Risk gate | Code | Evaluates the intent against the user's rules. Pure function: intent + rules + positions → pass/fail with reasons. |
| Ticket | Code | Renders the resolved intent and gate result. The only place an `ApprovedTicket` can be constructed. |
| Broker adapter | Code | Talks to one broker's API. Accepts only `ApprovedTicket`. |
| Journal | Code | Append-only record of every intent, resolution, gate result, approval, acknowledgement and fill. |
| Agent runtime | Code | Evaluates standing-instruction triggers and produces intents through the same pipeline. |

## Two channels

The single most important rule. There are two kinds of text in the system:

- **Instructions** — typed by the user, directly, in a front-end.
- **Content** — anything retrieved: news, filings, transcripts, emails, web pages, tool output.

Content can inform an answer. It can never become an order. This is enforced structurally, not by asking the model to behave:

```ts
type IntentSource = "user";            // the only accepted value

type OrderIntent = {
  source: IntentSource;
  text: string;                        // the exact user text that produced this intent
  // ...
};
```

The front-end is the only component that constructs `source: "user"`, and the compiler's retrieval context is kept in a separate call that returns analysis, never an intent.

## OrderIntent

```ts
type OrderIntent = {
  id: string;                          // ULID
  source: "user";
  text: string;
  account: AccountRef;                 // never inferred silently — resolved or asked
  instrument:
    | { kind: "equity"; symbol: string }
    | { kind: "option"; underlying: string; right: "call" | "put"; expiry: string; strike: number };
  side: "buy" | "sell" | "sell_short" | "buy_to_cover" | "buy_to_open" | "sell_to_close";
  quantity:
    | { kind: "shares"; value: number }
    | { kind: "notional"; usd: number }
    | { kind: "fraction_of_position"; value: number };   // must be resolved to shares before approval
  order:
    | { type: "market" }
    | { type: "limit"; limit: number }
    | { type: "stop"; stop: number }
    | { type: "stop_limit"; stop: number; limit: number };
  timing: { tif: "day" | "gtc"; session: "regular" | "extended" };
  trigger?: Condition;                 // absent = route on approval
  resolutions: Resolution[];
};

type Resolution = {
  term: string;                        // "a third"
  readings: string[];                  // ["⅓ of shares in Roth", "⅓ of shares across all accounts", "⅓ of position value"]
  chosen: string;
  by: "user" | "default";              // defaults are always shown on the ticket
};
```

## Risk gate

Rules are data the user owns, not prompts:

```yaml
accounts:
  roth:
    execution: true
    max_order_pct_of_account: 25
    max_orders_per_day: 5
    instruments: [equity]
  joint:
    execution: false            # read-only: tickets render, nothing routes
global:
  require_approval: always
  halt: false                   # `desk halt` flips this
```

The gate is a pure function. Given the same intent, rules and positions it returns the same result, with a reason for every failure. No model is involved.

## Adapter interface

```ts
interface BrokerAdapter {
  id: string;                                            // "schwab", "ibkr", "alpaca", …
  capabilities(): Capabilities;                          // what this broker can actually do
  authStatus(): Promise<AuthStatus>;                     // expired → hard stop, never cached fallback
  accounts(): Promise<Account[]>;
  positions(account: AccountRef): Promise<Position[]>;
  preview(intent: ResolvedIntent): Promise<BrokerPreview>;  // broker-side validation, routes nothing
  place(ticket: ApprovedTicket): Promise<OrderAck>;
}
```

Seven members. `place` accepts only an `ApprovedTicket`, a type that can be constructed in exactly one place — the approval step — which makes invariant 1 in [`SECURITY.md`](../SECURITY.md) checkable at compile time rather than by review.

`capabilities()` exists so the front-end can refuse, up front, an instruction a broker can't execute, instead of discovering it at routing time.

## Agents

A standing instruction is an `OrderIntent` template plus a trigger, with a lifecycle:

```
paused ──arm──▶ armed
  │               │
  └──preview──▶ preview ──arm──▶ armed
                                  │
                        halt / fire-limit / auth failure ──▶ paused
```

- **paused** — does nothing.
- **preview** — evaluates its trigger and journals what it *would* have done. Routes nothing.
- **armed** — produces intents through the full pipeline, within its caps.

New agents and imported agents always start `paused`. An agent can never change its own state.

## Journal

One append-only record per order attempt:

```
text → intent → resolutions → gate result → approval → broker preview → ack → fills
```

Approval records and user text are required fields. An order without both is a bug.

## Open questions

- **Implementation language.** Python has the broadest broker SDK and market-data ecosystem; TypeScript makes the type-level guarantees above easier to enforce.
- **Triggers on a machine that sleeps.** Self-hosted means standing instructions only evaluate while the machine is awake. Options: place simple triggers as native conditional orders where the broker supports them, run on a small always-on device, or accept and surface the gap on every agent.
- **Model default.** Bring-your-own API key, a local model, or both — and what the minimum capable model is for reliable compilation.
- **Options scope for v0.2.** Single-leg only, or defined-risk spreads from the start.
- **License.** To be chosen before the repository is made public.
