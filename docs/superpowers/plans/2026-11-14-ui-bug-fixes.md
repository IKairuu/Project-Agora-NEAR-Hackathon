# AGORA UI Bug Fixes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 11 reported UI/UX bugs across the AGORA frontend, one task per issue, in the order listed.

**Architecture:** All fixes are in the React frontend (`frontend/src/`). Most are in `Overlays.tsx`, `agents.tsx` (Wallet section), `hire.tsx`, and `styles/agora.css`. No backend changes are needed.

**Tech Stack:** React 18, TypeScript, Vite, plain CSS (`agora.css`)

**Spec:** This file is the spec. Each task maps to one item from the bug list in the image.

## Global Constraints

- Do NOT refactor unrelated code while fixing a bug.
- Keep all existing class names unless a task specifically changes one.
- All new loading states must use `aria-busy` for accessibility.
- Modal `closeModal()` is imported from `../store/store`.
- `setState` is from `../store/store`.

## Review Focus

1. **Withdraw modal close** — after `liveWalletMove` resolves successfully, `closeModal()` is already called on line 219; the bug may be that `busy` state is never reset on a network error path, leaving the modal open with a spinner and no way to dismiss.
2. **Activity tab scroll** — tab panel `<div className="panel">` for activity items may have no `max-height` + `overflow-y: auto`, causing overflow without scroll.
3. **Budget label** — `demoJob()` in `hire.tsx` hardcodes `'under 40'` as the budget string regardless of actual `JOB_BUDGET`; needs to read the real value.
4. **Fraud/lure label** — `OffersCard` in `hire.tsx` has hardcoded copy "usual price 28 to 45 USDC"; needs to be a live computed range from the offer pool.
5. **Watching matching jobs** — in `agents.tsx`, the Bidding agent status text `'Watching matching jobs'` is shown for all sellers; for human sellers this is misleading since human sellers don't have a bidding agent.

---

## Task 1: Add loading spinner to Link Wallet, Add Funds, Withdraw modals

**Files:**
- Modify: `frontend/src/ui/Overlays.tsx`

**Root cause:** `LinkWallet` already has busy states (`'load'`, `'pick'`, `'sign'`). `WalletMove` has a `busy` boolean. However, neither modal shows a visible loading spinner/indicator while `busy === 'load'` in `LinkWallet`, and `WalletMove` does not show any loading feedback until the wallet opens. The button text changes but no spinner is shown while the wallet is being checked on mount.

**Interfaces:**
- Consumes: `busy` state (`'' | 'load' | 'pick' | 'sign'` in LinkWallet, `boolean` in WalletMove)
- Produces: visible `<span className="spin" />` inside buttons during all busy states

- [ ] **Step 1: In `LinkWallet`, wrap the initial loading state in a spinner**

  In the branch `busy === 'load'` (the mount check), currently the button reads `'Loading wallets…'`. Add a `<span className="spin" />` before the text so it matches the existing spinner pattern used elsewhere in the app.

  Target line ~171 in `Overlays.tsx`:
  ```tsx
  // BEFORE
  {busy === 'load' ? 'Loading wallets…' : busy === 'pick' ? 'Waiting for your wallet…' : 'Choose a wallet'}
  // AFTER
  {busy === 'load' ? <><span className="spin" />Loading wallets…</> : busy === 'pick' ? <><span className="spin" />Waiting for your wallet…</> : <><Icon name="wallet" />Choose a wallet</>}
  ```

- [ ] **Step 2: In `LinkWallet`, add a spinner to the "Sign and link" / "Sign and sign in" button**

  Target line ~167:
  ```tsx
  // BEFORE
  {busy === 'sign' ? 'Waiting for your wallet…' : signIn ? 'Sign and sign in' : 'Sign and link'}
  // AFTER
  {busy === 'sign' ? <><span className="spin" />Waiting for your wallet…</> : signIn ? 'Sign and sign in' : 'Sign and link'}
  ```

- [ ] **Step 3: In `WalletMove`, add a spinner to the confirm button**

  Target line ~228:
  ```tsx
  // BEFORE
  {busy ? 'Waiting for your wallet…' : 'Confirm in your wallet'}
  // AFTER
  {busy ? <><span className="spin" />Waiting for your wallet…</> : 'Confirm in your wallet'}
  ```

- [ ] **Step 4: Verify visually** — Start the dev server (`pnpm dev` in `frontend/`), open Wallet page, click Add funds and Withdraw, confirm spinner shows.

- [ ] **Step 5: Commit**
  ```bash
  git add frontend/src/ui/Overlays.tsx
  git commit -m "fix: add loading spinners to LinkWallet and WalletMove modals"
  ```

---

## Task 2: Wallet UI — Replace text links with buttons for Link Wallet, Add Funds, Withdraw

**Files:**
- Modify: `frontend/src/views/agents.tsx` (lines ~221-238)

**Root cause:** In `Wallet()`, the "Add funds" and "Withdraw" actions use custom `<button className="pri">` and `<button>` elements styled in the wallet card. The "Link your own NEAR wallet" is a `<button className="lk small">` (text-link style). These need to look like proper buttons consistently.

**Interfaces:**
- Consumes: `act.addfunds`, `act.withdrawme`, `act.send`, `act.receive` from `actions.tsx`
- Produces: visually consistent `<button className="btn">` / `<button className="btn primary">` elements

- [ ] **Step 1: Change "Add funds" to `btn primary` and "Withdraw" to `btn`**

  In `agents.tsx` around line 220-223, the `wal-acts` div:
  ```tsx
  // BEFORE
  <button className="pri" onClick={act.addfunds}><i><Icon name="plus" /></i>Add funds</button>
  <button onClick={act.withdrawme}><i><Icon name="out" /></i>Withdraw</button>
  // AFTER
  <button className="btn primary" onClick={act.addfunds}><Icon name="plus" />Add funds</button>
  <button className="btn" onClick={act.withdrawme}><Icon name="out" />Withdraw</button>
  ```

- [ ] **Step 2: Change "Link your own NEAR wallet" text-link to a button**

  Around line 235:
  ```tsx
  // BEFORE
  {s.live && <button className="lk small" onClick={() => setState({ modal: { kind: 'linkwallet' } })}><Icon name="wallet" />{s.walletBal ? 'Link a different wallet' : 'Link your own NEAR wallet'}</button>}
  // AFTER
  {s.live && <button className="btn" onClick={() => setState({ modal: { kind: 'linkwallet' } })}><Icon name="wallet" />{s.walletBal ? 'Link a different wallet' : 'Link your own NEAR wallet'}</button>}
  ```

- [ ] **Step 3: Verify** — wallet page shows proper buttons, not text links.

- [ ] **Step 4: Commit**
  ```bash
  git add frontend/src/views/agents.tsx
  git commit -m "fix: use proper buttons for wallet actions instead of text links"
  ```

---

## Task 3: Withdraw modal does not close after success

**Files:**
- Modify: `frontend/src/ui/Overlays.tsx` (WalletMove function, lines ~210-231)

**Root cause (Phase 1 investigation):** `liveWalletMove` returns `true` on success, and on that path `closeModal()` is called. However if the wallet popup is dismissed or times out, `liveWalletMove` returns `false` and `setBusy(false)` is called — this is correct. The issue is likely that `closeModal` is not called in the `Frame` click-outside handler after a pending `busy=true` state, OR the `WalletMove` modal does not use the `Frame` wrapper's Escape key handler correctly when busy.

**Fix:** Ensure the Cancel button is never disabled (so the user can always escape), and that the `Frame` backdrop click still fires `closeModal` even when `busy=true`.

- [ ] **Step 1: Remove `disabled` from Cancel button in `WalletMove` so it always closes**

  Line ~228:
  ```tsx
  // BEFORE
  <div className="acts"><Cancel /><button className="btn primary" disabled={busy} aria-busy={busy} onClick={() => void go()}>...</button></div>
  // AFTER — Cancel is never disabled; primary action is disabled while busy
  <div className="acts"><Cancel /><button className="btn primary" disabled={busy} aria-busy={busy} onClick={() => void go()}>...</button></div>
  ```
  (`Cancel` already calls `closeModal` and has no `disabled` prop — verify `Cancel` component has no disabled prop propagated.)

- [ ] **Step 2: Verify `Frame` scrim click closes even during busy — it should since `Frame` closes on backdrop click unconditionally**

  Read the `Frame` component (lines ~16-26 of `Overlays.tsx`) — confirm `onClick` on `.scrim` calls `closeModal()` without checking `busy`. If it does, no change needed here.

- [ ] **Step 3: If `closeModal` is not called on success path, add explicit call**

  In `WalletMove.go()`, currently:
  ```tsx
  if (await liveWalletMove(op, n)) closeModal(); else setBusy(false);
  ```
  This looks correct. If the modal is not closing, the issue is that `liveWalletMove` never resolves `true` in the mock/test mode. Add a fallback: also set `busy=false` on success before `closeModal` to avoid a lingering spinner flash.
  ```tsx
  const result = await liveWalletMove(op, n);
  if (result) { setBusy(false); closeModal(); } else { setBusy(false); }
  ```

- [ ] **Step 4: Verify** — trigger a mock withdraw, confirm modal closes.

- [ ] **Step 5: Commit**
  ```bash
  git add frontend/src/ui/Overlays.tsx
  git commit -m "fix: ensure withdraw modal closes after successful operation"
  ```

---

## Task 4: Activity Tab must be scrollable

**Files:**
- Modify: `frontend/src/styles/agora.css`

**Root cause:** The Activity tab panel `<div className="panel">` for wallet items has no constrained height, so when items overflow the page it stretches instead of scrolling.

- [ ] **Step 1: Find the `.wal` and `.panel` selectors in `agora.css`**
  ```bash
  Select-String -Path "frontend/src/styles/agora.css" -Pattern "\.wal|walTab|activity"
  ```

- [ ] **Step 2: Add a max-height and overflow-y to the activity panel**

  After confirming the structure, add a scoped rule. The activity tab content is wrapped in `<div className="panel">` inside `<div className="wal">`. Add:
  ```css
  .wal .panel { max-height: 480px; overflow-y: auto; }
  ```
  If a more specific selector is needed (to not affect Assets tab), add a wrapper class or use the `:has()` selector — but the simplest approach is a `.wal .panel` rule since both tabs have the same outer wrapper.

- [ ] **Step 3: Verify** — with many wallet items, the Activity tab scrolls without overflowing the page.

- [ ] **Step 4: Commit**
  ```bash
  git add frontend/src/styles/agora.css
  git commit -m "fix: make wallet activity tab scrollable"
  ```

---

## Task 5: Issue — clicking something opens Lumenlabs (wrong link)

**Files:**
- Modify: Wherever the lumenlabs link/navigation occurs (requires grep to identify exact location)

**Root cause (investigation needed):**

- [ ] **Step 1: Grep the codebase for "lumenlabs"**
  ```bash
  Select-String -Path "frontend/src" -Recurse -Pattern "lumenlabs|lumen" | Select-Object Filename, LineNumber, Line
  ```

- [ ] **Step 2: Identify what component or link triggers the lumenlabs open**

  Could be in `parts.tsx`, a data file, or an `<a href>` somewhere.

- [ ] **Step 3: Fix the link** — either remove it, change it to an internal route via `<RLink>`, or replace with the correct URL.

- [ ] **Step 4: Verify** — the action that previously opened lumenlabs now does the correct thing.

- [ ] **Step 5: Commit**
  ```bash
  git add <affected file>
  git commit -m "fix: remove/correct lumenlabs link that opened wrong site"
  ```

---

## Task 6: "Watching matching jobs" label is misleading for a human seller

**Files:**
- Modify: `frontend/src/views/agents.tsx` (Agents function, the `A` array, line ~38)

**Root cause:** The Bidding agent row in the `A` array always shows `'Watching matching jobs'` as status when `s.seller` is set, regardless of whether the seller is a human or an AI agent. For a human seller, the bidding agent is a tool they manually use — "watching" implies automated polling which misleads human sellers.

- [ ] **Step 1: Check how `s.seller` distinguishes human vs agent sellers**

  In `store.ts`, find the `seller` field type to see if it carries a `kind` field.

- [ ] **Step 2: Update the status string for the Bidding agent row**

  ```tsx
  // BEFORE (line ~38 in agents.tsx)
  ['Bidding agent', 'Seller side. Bids on jobs that match your skills.', s.seller ? 'Function-call key, bid and deliver' : 'Off while you only buy', s.seller ? 'Watching matching jobs' : 'Off', false],
  // AFTER
  ['Bidding agent', 'Seller side. Bids on jobs that match your skills.', s.seller ? 'Function-call key, bid and deliver' : 'Off while you only buy', s.seller ? 'Ready to bid on matching jobs' : 'Off', false],
  ```

  "Ready to bid" is accurate for both humans and agents — humans bid manually, agents bid automatically, but both are in a ready state.

- [ ] **Step 3: Verify** — Agents page with seller profile shows "Ready to bid on matching jobs".

- [ ] **Step 4: Commit**
  ```bash
  git add frontend/src/views/agents.tsx
  git commit -m "fix: use accurate bidding agent status label for human sellers"
  ```

---

## Task 7 (DONE): Spec writer not updating page on navigation — marked complete in the issue list, skip.

---

## Task 8: Inputting GitHub repo link uses a local input field (not expected)

**Files:**
- Modify: `frontend/src/ui/Overlays.tsx` or wherever the repo attachment modal/input is defined

**Root cause (investigation needed):**

- [ ] **Step 1: Search for the repo input field**
  ```bash
  Select-String -Path "frontend/src" -Recurse -Pattern "attachrepo|repo.*input|github.*input|Add repository" | Select-Object Filename, LineNumber, Line
  ```

- [ ] **Step 2: Find the `attachrepo` modal or inline field**

  In `actions.tsx`, `act.attachrepo` is called. Find what modal or state it sets.

- [ ] **Step 3: If the input is `type="text"` or `type="file"` (local), change it to accept a URL**

  Ensure the placeholder text reads e.g. `https://github.com/owner/repo` and input type is `url` or `text` with URL validation.

- [ ] **Step 4: Verify** — clicking "Add repository" shows an input that accepts a GitHub URL.

- [ ] **Step 5: Commit**
  ```bash
  git add <affected file>
  git commit -m "fix: repo input field accepts GitHub URL not local path"
  ```

---

## Task 9: Card says "Budget under 40 USDC" but you set 10 (hardcoded demo text)

**Files:**
- Modify: `frontend/src/views/hire.tsx` (line ~384 in `Dash` component)
- Modify: `frontend/src/views/hire.tsx` (line ~293 in `demoJob` function)

**Root cause:** Two places hardcode budget display:

1. `demoJob()` on line ~293: `budget: s.stage >= 5 ? usd(hPrice(s)) : 'under 40'` — the `'under 40'` is a hardcoded string.
2. `Dash` component line ~384: `<span>Budget under 40 USDC</span>` — fully hardcoded.

Both should use `JOB_BUDGET.label` which is imported from `'../lib/live'` and already used in `LiveSpecCard`.

- [ ] **Step 1: Check what `JOB_BUDGET.label` resolves to**

  Search `live.ts` for `JOB_BUDGET`:
  ```bash
  Select-String -Path "frontend/src/lib/live.ts" -Pattern "JOB_BUDGET"
  ```

- [ ] **Step 2: Fix `demoJob()` in `hire.tsx`**

  ```tsx
  // BEFORE (~line 293)
  budget: s.stage >= 5 ? usd(hPrice(s)) : 'under 40',
  // AFTER
  budget: s.stage >= 5 ? usd(hPrice(s)) : (JOB_BUDGET.label || 'under 40'),
  ```
  Ensure `JOB_BUDGET` is imported (it already is at line 4).

- [ ] **Step 3: Fix the hardcoded "Budget under 40 USDC" in `Dash`**

  Line ~384:
  ```tsx
  // BEFORE
  <span>Budget under 40 USDC</span>
  // AFTER
  <span>Budget {JOB_BUDGET.label || 'under 40'} USDC</span>
  ```

- [ ] **Step 4: Verify** — set a budget of 10 in the demo, the card shows "Budget under 10 USDC" or the actual set value.

- [ ] **Step 5: Commit**
  ```bash
  git add frontend/src/views/hire.tsx
  git commit -m "fix: show actual budget value on job card instead of hardcoded 'under 40'"
  ```

---

## Task 10: Seller Dashboard GitHub UI redesign

**Files:**
- Investigate: `frontend/src/views/agents.tsx` (Limits/Keys section) and/or seller dashboard views

**Root cause (investigation needed):** This issue references a "Seller Dashboard GitHub UI redesign." Need to clarify what the current GitHub UI looks like vs. what is expected.

- [ ] **Step 1: Identify the GitHub-related UI in the seller dashboard**
  ```bash
  Select-String -Path "frontend/src" -Recurse -Pattern "github|GitHub|repo|seller.*dash" -CaseSensitive:$false | Select-Object Filename, LineNumber, Line | Select-Object -First 30
  ```

- [ ] **Step 2: Assess the current state** — look at what renders for sellers in the dashboard view, specifically any GitHub repo display.

- [ ] **Step 3: Implement the redesign** — based on findings, update the component to match the intended design (cards, icons, link styling per the existing design system).

- [ ] **Step 4: Verify** — seller dashboard GitHub section renders correctly.

- [ ] **Step 5: Commit**
  ```bash
  git add <affected files>
  git commit -m "fix: seller dashboard GitHub UI redesign"
  ```

---

## Task 11: "Tests locked" message is misleading / Fraud and lure price range is hardcoded

**Files:**
- Modify: `frontend/src/views/hire.tsx` (OffersCard, line ~177; buyer agent message line ~237)

**Root cause — two sub-issues:**

**A. "Tests locked" message (line ~237):** The buyer agent message reads: *"Tests locked. I sent the job privately to 9 sellers and bidding agents whose skills match."* The number "9" is hardcoded. This is misleading if the actual number differs.

**B. Fraud and lure price range (line ~177):** The footer reads: *"Price check by the fraud and lure agent: usual price 28 to 45 USDC"* — both numbers are hardcoded regardless of actual offer prices.

- [ ] **Step 1: Fix the hardcoded "9 sellers" count**

  Line ~237:
  ```tsx
  // BEFORE
  <p>Tests locked. I sent the job privately to 9 sellers and bidding agents whose skills match. Their offers appear below as they arrive, lowest price first.</p>
  // AFTER — use a constant or config value; check if ARRIVE.length or similar gives the actual count
  ```
  Search for where `9` sellers is defined:
  ```bash
  Select-String -Path "frontend/src/data/data.ts" -Pattern "ARRIVE|sellers.*match|9" | Select-Object LineNumber, Line | Select-Object -First 10
  ```
  Replace `9` with `{ARRIVE.length}` or the appropriate data constant.

- [ ] **Step 2: Fix the hardcoded price range in the fraud/lure footer**

  Line ~177:
  ```tsx
  // BEFORE
  <span className="muted xsmall">Price check by the fraud and lure agent: usual price 28 to 45 USDC{valid.length >= 3 ? ', median valid offer 35.50' : ''}.</span>
  // AFTER — compute from the offer pool
  ```
  Compute min/max/median from `valid` rows:
  ```tsx
  const prices = valid.map(r => r.price).sort((a, b) => a - b);
  const pMin = prices[0] ?? 0, pMax = prices[prices.length - 1] ?? 0;
  const med = prices.length >= 3 ? prices[Math.floor(prices.length / 2)] : null;
  ```
  Then render:
  ```tsx
  <span className="muted xsmall">
    {prices.length >= 2
      ? `Price check by the fraud and lure agent: usual price ${usd(pMin)} to ${usd(pMax)} USDC${med ? `, median valid offer ${usd(med)}` : ''}.`
      : 'Price check by the fraud and lure agent: gathering offers…'}
  </span>
  ```

- [ ] **Step 3: Verify** — with 3+ offers visible, the range reflects actual offer prices. With <3 offers, shows "gathering offers".

- [ ] **Step 4: Commit**
  ```bash
  git add frontend/src/views/hire.tsx
  git commit -m "fix: use dynamic seller count and price range in buyer agent messages"
  ```

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-11-14-ui-bug-fixes.md`. Please review the plan.

**Which execution approach would you prefer?**

- **Subagent-driven** — A fresh subagent implements each task and a fresh reviewer checks it before the next one starts, then a whole-branch review at the end. Most thorough; costs a fresh context per task.
- **Native** — I implement every task myself in this session, one by one, then one fresh reviewer checks the whole branch.

**I recommend Native** for this plan, because all 11 tasks are small isolated frontend edits in 3–4 files, and the tasks don't depend on each other's interfaces — native execution avoids the per-task context overhead. Does the plan capture what you want, and which approach should we use?
