# Example: "Add money" story on multi-currency-wallet

This is unedited output from one real run of `/kspp-analysis:analyze-story`, apart from replacing a local folder path with the repo link.

- **Story:** [stories/add-money.md](stories/add-money.md)
- **Codebase:** [sybilla11/multi-currency-wallet](https://github.com/sybilla11/multi-currency-wallet) at commit `d72919e`
- **Command:** `/kspp-analysis:analyze-story examples/stories/add-money.md`, run from the wallet repo's root
- **Check afterwards:** every file path the report cites exists in the repo, and every line number is within that file. Files the report proposes creating are marked new.

---

# Pre-Implementation Analysis: Add Money to a Wallet (Including Cross-Currency)

## 1. Story summary

As a wallet holder, I want to fund any wallet — including a foreign-currency one — from my linked bank account, seeing the converted amount and fee before confirming, with the FX quote locked for 30 seconds.

**Verdict: Needs answers before starting.** The codebase ([multi-currency-wallet](https://github.com/sybilla11/multi-currency-wallet)) is a UI-showcase wallet app that has deliberately *not yet built* two things this story depends on: a linked-bank-account concept (planned as P3, not started) and any money-movement endpoint (planned as P2, not started — currently only `GET /api/wallets` and `GET /api/rates` exist). The project's own docs (`PROJECT_PLAN.md:59-61`, `ARCHITECTURE.md:132`) describe almost exactly this story as the next phase and already commit to several of its answers (quote id, rate, spread, 30s expiry, optimistic pending→settle), which is good grounding — but a few product decisions aren't settled and will change the schema and flow.

## 2. Gaps and open questions

| # | Question | Why it matters | Blocking? |
|---|---|---|---|
| 1 | Is "fee" a separate charge on top of the quoted FX rate, or is it the FX spread already built into the rate (just surfaced for transparency)? `ARCHITECTURE.md:132` says the quote carries "rate, spread and expiry," but the story asks to show "the converted amount **and** any fee" as if they're two line items. | Changes the quote schema (one numeric field vs. two), the deposit math (subtract a fee vs. just convert), and the confirmation copy. | Yes |
| 2 | Is "any of my wallets" limited to currencies the account already holds (today: USD, CAD, GBP — `src/mocks/db.ts:5-9`), or can adding money open a brand-new currency wallet? | Determines whether this story needs a "create wallet" path in addition to "top up existing wallet," and whether the destination picker is a fixed list or free currency entry. | Yes |
| 3 | When the wallet currency differs from the bank's, does the user type the amount in the bank's (source) currency or the wallet's (destination) currency? | Flips which side of `convert()` the entered amount anchors to, and what the confirm button's primary number means ("you send $500" vs "wallet receives €460"). | Yes |
| 4 | Linking a bank account is P3 ("Add bank"), not yet built (`PROJECT_PLAN.md:98-101` all unchecked, no bank entity anywhere in `src/mocks/`, `src/api/contracts.ts`, or `src/types/market.ts`). Does this story assume a bank is already linked (stub one in mocks) and defer the real linking UI to P3, or does it need to build minimal bank-linking too? | Without a decision, "Add money" has no bank to select from — the happy path can't be demoed. | Yes |
| 5 | Can a user have more than one linked bank, possibly in different currencies? | `src/lib/placeholderData.ts:24-56` sample activity shows three different bank names (Chase/RBC/Barclays) against three currencies, which reads as one bank per market rather than multiple per user. | No (see Assumption A1) |
| 6 | Is there a deposit minimum/maximum or a daily limit? | Affects validation rules and error-path tests. | No (Assumption A2) |
| 7 | Can a bank-side deposit be declined for "insufficient funds," given banks have no modeled balance today? `PROJECT_PLAN.md:96` explicitly lists "insufficient funds" as a P2 error path. | Requires adding a mock bank balance/limit to make that error reachable at all. | No (Assumption A3) |
| 8 | If the user takes no action before the 30s quote expires, does the UI silently re-quote, or show an explicit "quote expired, get a new one" state? | Changes whether expiry is a dead-end requiring a restart or a seamless refresh. | No (Assumption A4) |

**Assumptions** (used to keep the rest of this report concrete; flag to the team before/while building):
- **A1** — Exactly one linked bank account exists, implicitly tied to the current market's domestic currency (one bank per market, consistent with P3's "one form… Linked bank appears in Add money/Withdraw," singular). Multi-bank-per-user is out of scope for this story.
- **A2** — No minimum/maximum beyond "amount must be a positive, non-zero integer number of minor units valid for the wallet's currency exponent" (reusing `money()`'s existing validation, `src/lib/money.ts:38-43`).
- **A3** — The mock bank gets a simulated available balance (new mock data) so "insufficient funds" is a reachable, testable error, per `PROJECT_PLAN.md:96`.
- **A4** — On expiry, the preview is replaced with an explicit "This quote has expired" message and a "Get a new quote" action; the expired quote id is rejected by the mock server if submitted anyway (server-authoritative, per `ARCHITECTURE.md:132`).
- **A5 (resolves Q1)** — "Fee" = the FX spread, surfaced as a single transparent line (e.g., "includes 0.6% spread") rather than a separate deducted charge; no additional flat fee. *(Flag this explicitly to the team — it's the most consequential assumption and easy to get wrong.)*
- **A6 (resolves Q2)** — Add money is restricted to the wallets already in the account (USD/CAD/GBP today); opening a new-currency wallet is out of scope for this story.
- **A7 (resolves Q3)** — The user enters the amount in the **destination wallet's currency** ("how much do you want to add to this wallet"), and the preview shows what that costs/converts to from the bank side. *(This is the more common BaaS UX, but Q3 should be confirmed — it inverts which side of the quote the user edits.)*

## 3. Acceptance criteria

### Happy path
1. **Given** a wallet holder with a domestic wallet (e.g., USD) and a linked bank in the same currency, **when** they add money and confirm, **then** no conversion preview is shown, the amount is added as a pending transaction immediately, and it settles to the wallet balance shortly after (mirrors the "optimistic pending → settle" pattern already named in `ARCHITECTURE.md:132`).
2. **Given** a wallet holder with a foreign-currency wallet (e.g., GBP) and a bank in a different currency (USD), **when** they enter an amount and the system fetches a quote, **then** they see the converted amount, the spread/fee (assumes A5), the locked rate, and an expiry countdown before any confirm action is enabled.
3. **Given** a visible, not-yet-expired quote, **when** the user confirms, **then** the client submits the quote id (not its own computed amount) and the server is authoritative on the final rate/amount (per `ARCHITECTURE.md:132`).
4. **Given** a successful confirm, **when** the deposit is created, **then** a pending transaction appears immediately in the UI and transitions to completed once the mock settles it.
5. **Given** the rate-lock countdown is active, **when** 30 seconds have not yet elapsed, **then** the displayed rate, converted amount, and fee do not change even if the underlying mid-market rate moves in the background.

### Validation and errors
6. **Given** an amount field left blank or zero, **when** the user tries to proceed, **then** the action is blocked with an accessible validation message (no request is sent).
7. **Given** an amount with more decimal places than the currency's exponent allows (e.g., JPY has 0 decimals per `currencyExponent()`, `src/lib/money.ts:29-36`), **when** entered, **then** the input is rejected or rounded with a visible message — behavior must be explicit, not silently truncated.
8. **Given** an amount exceeding the simulated bank balance (assumes A3), **when** the user confirms, **then** an "insufficient funds" error is shown and no transaction is created.
9. **Given** a quote that has expired (30s elapsed) when the user clicks confirm, **then** the submission is rejected (assumes A4) with a message prompting a fresh quote, and the stale quote id cannot be replayed to lock in an old rate.
10. **Given** the rates/quote service returns an error (mirrring the existing `rates-error` scenario pattern in `src/mocks/handlers.ts:30-32`), **when** the user opens Add Money for a foreign-currency wallet, **then** the flow shows a retryable error instead of a broken or zero preview.
11. **Given** the deposit submission fails after an optimistic pending transaction was shown, **when** the mock reports failure, **then** the pending transaction is rolled back / marked failed rather than silently disappearing (per `ARCHITECTURE.md:132`'s "roll back on failure").

### Edge cases
12. **Given** a wallet currency identical to the bank currency, **when** the user opens Add Money, **then** no quote is fetched and no countdown is shown (confirms assumption that preview only appears cross-currency, per the story's conditional "If the wallet currency differs…").
13. **Given** a currency with a non-2 exponent (e.g., a hypothetical 0- or 3-decimal currency), **when** converting for the preview, **then** rounding uses the existing half-even `convert()` rule (`src/lib/money.ts:122-132`) with no float drift.
14. **Given** the user re-opens or re-triggers the amount field after a quote was already fetched, **when** the amount changes, **then** a new quote is requested and the previous 30s timer is discarded (no stale lock reused for a different amount).

### Permissions
15. **Given** a wallet holder, **when** they attempt to add money to a wallet that is not theirs — *not currently applicable*: this app has no multi-user auth model (`PROJECT_PLAN.md:32` lists "real auth" as explicitly out of scope). No criterion needed unless auth is introduced.

### Non-functional
16. **Accessibility:** the countdown is announced via `aria-live` (consistent with existing rate-status pattern at `src/components/dashboard/BalanceCard.tsx:39`), and the modal/dialog traps focus and is dismissible via Escape (no existing modal primitive to confirm against — new pattern).
17. **i18n/locale:** the converted amount, fee, and rate all format through `Intl.NumberFormat` via `format()` / the existing `formatRate` pattern (`src/components/dashboard/WalletList.tsx:10-12`), not hardcoded `$`/`£` strings.

## 4. Codebase impact map

| Area | File(s) | Change | Evidence |
|---|---|---|---|
| Wire contracts | `src/api/contracts.ts` | Add `quoteRequestSchema`/`quoteResponseSchema` (rate, spread/fee, expiry, quote id) and `depositRequestSchema`/`depositResponseSchema` | Existing pattern: `moneySchema`, `ratesResponseSchema` at lines 11-30 |
| API client | `src/api/client.ts` | Add a `postJson` (or similar) helper — only `getJson` exists today | `getJson` is GET-only, `src/api/client.ts:14-24` |
| Query hooks | `src/api/queries.ts` | Add `useCreateQuoteMutation`, `useDepositMutation` (TanStack Query mutations) | Existing `useQuery` hooks only, `src/api/queries.ts:10-25` |
| Mock data | `src/mocks/db.ts` | Add a mock linked bank (currency, simulated balance per A1/A3) and an in-memory quote store | `WALLETS`, `USD_REFERENCE` as the existing in-memory "server" state, lines 5-16 |
| Mock handlers | `src/mocks/handlers.ts` | Add `POST /api/quotes` and `POST /api/deposits`, including expiry/insufficient-funds responses | Existing `http.get` handler pattern, lines 14-39 |
| Scenario switch | `src/mocks/scenario.ts` | Possibly extend `Scenario` union for a `quote-error` / deposit-failure demo state | Existing `Scenario` type and URL-driven switch, lines 5-22 |
| Money math | `src/lib/money.ts` | Likely no change — `convert()`, `parseRatio()`, `invert()` already do exactly the conversion math needed | `convert()` at lines 122-132, already used by `buildPortfolio` |
| View model | `src/features/wallets/` (new: `addMoney/` subfolder) | New pure function (e.g., `buildConversionPreview`) following `buildPortfolio`'s pure-function style | `buildPortfolio()` pattern, `src/features/wallets/portfolio.ts:29-63` |
| UI entry point | `src/components/dashboard/BalanceCard.tsx` | Wire the disabled "Add money" button (line 78-93, currently `disabled title="Coming soon"`) to open the new flow | `ACTIONS` array and disabled button, lines 7-11, 76-93 |
| New UI | `src/components/dashboard/AddMoneyModal.tsx` or `src/components/addMoney/*` (new) | Modal/dialog, amount input, conversion preview, countdown, confirm | No existing modal/dialog primitive in `src/components/` — new pattern |
| Bank model | `src/types/market.ts` (extend) or new `src/types/bank.ts` | A `BankAccount` type doesn't exist yet | `BankFieldSpec`/`MarketConfig` only describe per-country *form fields*, not an actual linked account, `src/types/market.ts:5-21` |
| Tests (unit) | `src/features/addMoney/*.test.ts` (new) | Pure conversion-preview and fee math | Style of `src/features/wallets/portfolio.test.ts` |
| Tests (contract) | `src/mocks/handlers.test.ts` (extend) | New quote/deposit endpoints honor contracts | Existing pattern, lines 1-36 |
| Tests (integration) | `src/App.test.tsx` (extend) | End-to-end flow against mock service | Existing `renderApp()` harness, lines 9-19 |

**Patterns to follow**
- Pure view-model-then-thin-component split, as in `buildPortfolio()` (`src/features/wallets/portfolio.ts`) / `usePortfolio()` (`src/features/wallets/usePortfolio.ts`) — a new `buildConversionPreview` should be pure and independently unit-testable, with a thin hook joining the quote mutation.
- Money/rate handling must go through `src/lib/money.ts` (`convert`, `parseRatio`, `invert`) — never raw float math, matching the project's core architectural rule (`ARCHITECTURE.md:45-63`).
- Async states (loading/error/empty) should reuse `Skeleton`, `ErrorPanel`, `EmptyPanel` from `src/components/dashboard/AsyncStates.tsx` rather than new one-off markup.
- Contract-first: define Zod schemas before wiring the mock handler and the client call, as every existing endpoint does (`src/api/contracts.ts` → `client.ts` → `queries.ts` → mock `handlers.ts`).
- `?scenario=` URL-driven state injection (`src/mocks/scenario.ts`) is the project's established way to make error/edge states demoable and testable without hand-editing code — extend it rather than bypassing it.

**Existing tests to extend:** `src/mocks/handlers.test.ts` (contract tests for new endpoints), `src/App.test.tsx` (integration flow through `renderApp()`).

## 5. TDD test plan

1. **Unit** — `src/lib/money.test.ts` (extend if a fee/spread helper is added to `money.ts`): `"applies a spread on top of the mid-market rate without float drift"` — only needed if fee computation lives in `money.ts` rather than the mock server. Covers AC1/AC13 foundation.
2. **Unit** — new `src/features/addMoney/conversionPreview.test.ts`: `"buildConversionPreview returns the destination amount and fee for a cross-currency quote"` — AC2.
3. **Unit** — same file: `"buildConversionPreview returns no preview when wallet and bank currency match"` — AC12.
4. **Unit** — same file: `"buildConversionPreview rejects a zero or negative amount"` — AC6.
5. **Unit** — same file: `"buildConversionPreview respects the currency's minor-unit exponent (no extra decimals)"` — AC7.
6. **Contract** — `src/mocks/handlers.test.ts`: `"POST /api/quotes returns a quote id, rate, fee and a 30s expiry for a supported currency pair"` — AC2, AC5.
7. **Contract** — same file: `"POST /api/quotes rejects an unsupported currency with 400"` — mirrors existing `/api/rates` 400 test (line 23-25) — AC10.
8. **Contract** — same file: `"POST /api/deposits succeeds for a same-currency amount with no quote id"` — AC1.
9. **Contract** — same file: `"POST /api/deposits with a valid, unexpired quote id creates a pending transaction"` — AC3, AC4.
10. **Contract** — same file: `"POST /api/deposits rejects an expired quote id"` — AC9.
11. **Contract** — same file: `"POST /api/deposits rejects an amount exceeding the simulated bank balance with an insufficient-funds error"` — AC8.
12. **Integration** — `src/App.test.tsx` (extend): `"opens Add Money, shows no conversion preview, and completes a same-currency deposit"` — AC1, AC12.
13. **Integration** — same file: `"opens Add Money for a foreign wallet, shows the converted amount, fee and a live countdown before confirm is enabled"` — AC2, AC5, AC16.
14. **Integration** — same file: `"confirming after the quote expires shows an error and does not submit the stale quote"` (use Vitest fake timers to fast-forward 30s) — AC9.
15. **Integration** — same file: `"a failed deposit rolls back the optimistic pending transaction"` — AC11.
16. **Integration** — same file: `"insufficient funds blocks the deposit with a visible message"` — AC8.

**Coverage check:** All acceptance criteria (1–14, 16–17) map to at least one test above except **AC15** (permissions), which is explicitly not applicable given the app's current no-auth, single-account model, and **AC17** (i18n formatting), which is better covered by a lightweight unit test on whatever new `format`-wrapping helper is introduced for the fee/rate line (add once that helper exists) rather than a dedicated scenario above.

## 6. Risks and dependencies

- **Hard dependency on unbuilt P3 ("Add bank").** This story cannot demo its headline scenario (bank → foreign wallet) without at least a stub linked bank account. Building a minimal stub now risks duplicating work when P3 lands properly; building full P3 first is safer but expands scope well beyond this story.
- **No mutation support exists yet** (`getJson` is the only client helper — `src/api/client.ts`). Introducing POST requires new client plumbing that every subsequent money-moving feature (Withdraw, Add Bank) will also depend on — get the shape right the first time.
- **No modal/dialog primitive exists** in `src/components/`. `PROJECT_PLAN.md:36` named Radix UI for this, but it isn't in `package.json` dependencies. Decide whether to add a dependency or hand-roll a minimal accessible dialog, consistent with the project's stated preference for hand-rolled, fully-tested primitives over adopting a library wholesale (as was done for money.ts vs. dinero.js, `ARCHITECTURE.md:61`).
- **Timer-based testing risk.** The 30-second lock requires fake-timer-based tests (Vitest `vi.useFakeTimers()`); none of the current tests manipulate time, so this is a new test-infrastructure pattern to introduce carefully to avoid flaking the existing suite.
- **Fee semantics ambiguity (Q1)** is the single biggest risk to get wrong quietly — it changes both the money math and what number the user is shown as "what they'll pay," which is compliance-sensitive even in a demo app that models itself on BaaS semantics.
- **Optimistic UI rollback** is new: existing code has no precedent for TanStack Query mutations with optimistic updates + rollback, so this needs its own careful test coverage (AC11) rather than copying an existing pattern.

## 7. Size and slicing

**Size: L.** This touches contracts, client, mocks, a new pure view-model, a new modal UI component, and the first write/mutation path in a previously read-only app — comparable to the ~8–10h the project's own plan allocates to all of P2 (`PROJECT_PLAN.md:59-61`), and only after the bank-account question is resolved.

Proposed vertical slices, each independently shippable and demoable via the existing `?scenario=` pattern:

1. **Slice 1 — Same-currency add money.** Wallet holder adds money from a (stubbed) linked bank in the same currency as the wallet: no quote, no conversion, pending→settled, insufficient-funds error. Unlocks the write path, the modal UI, and the optimistic-update pattern without FX complexity. Covers AC1, AC4, AC6, AC8, AC11, AC12.
2. **Slice 2 — Cross-currency conversion preview + 30s locked quote.** Adds the quote endpoint, converted-amount/fee display, countdown, and expiry handling for a foreign-currency wallet. This is the story's actual headline ask. Covers AC2, AC3, AC5, AC9, AC10, AC14.
3. **Slice 3 (stretch) — Edge-case and accessibility hardening.** Exponent edge cases (0/3-decimal currencies), focus-trap/ARIA polish on the modal, re-quote-on-amount-change behavior. Covers AC7, AC13, AC16, AC17.

If the bank-linking dependency (Q4) resolves to "build minimal linking first," that becomes a Slice 0 before Slice 1.
