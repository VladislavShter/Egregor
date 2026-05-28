# 🛡️ Real Audit Case Study — Smart Contract Review by Egregor

> A documented, reproducible result: vulnerabilities that single frontier AI models
> missed when working alone — and which Egregor's multi-AI Code Review Pipeline found
> when the same models worked together as a structured consilium.

[![Confidence](https://img.shields.io/badge/Consilium_Confidence-4%2F5-green?style=flat-square)](#)
[![Status](https://img.shields.io/badge/contract_status-not_deployed-blue?style=flat-square)](#)
[![Critical Issues](https://img.shields.io/badge/critical_findings-4%2B-red?style=flat-square)](#)

---

## 📌 TL;DR

The production smart contracts of **SovereignBank Web3** were reviewed for security
vulnerabilities. We compared two approaches on the **same code**:

| Reviewer | Critical issues found |
| -------- | --------------------- |
| Claude Opus — single, direct prompt | **0** |
| Gemini Pro — single, direct prompt | **0** |
| **Egregor Code Review Pipeline (5-step consilium)** | **4 critical + additional flagged scenarios** |

The individual models that found **nothing on their own** found **multiple critical
issues** when orchestrated into a structured pipeline with a dedicated Security role,
an independent reviewer, a comparison step, and a final synthesis with a Confidence Map.

> **Why this is credible:** the consilium also openly **declared what it did NOT check**,
> marking several functions as *"not deeply audited — requires a separate pass."*
> A system that tells you the boundaries of its own analysis is a system you can trust.

---

## ⚠️ Responsible Disclosure

All findings below are being **remediated before deployment**. The SovereignBank Web3
contract is **not yet live on any mainnet**. This document describes **classes** of
vulnerabilities and their fixes — it is a security improvement record, not an attack guide.

---

## 🔬 Methodology

**Target:** `SovereignBankPro.sol`, `SovereignAutoPay.sol`, `SovereignVaults.sol` and
related modules (modular architecture: Vaults, AutoPay, Cashback, Math).

**The Egregor Code Review Pipeline** runs the code through a strict sequential chain,
where each AI acts as a specialist with a defined role:

| Step | Role | Model | Function |
| ---- | ---- | ----- | -------- |
| 1 | ✍️ Reasoning | DeepSeek R1 | Deep architectural analysis *(failed this run — fetch error)* |
| 2 | 🛡️ Security | GPT-4o mini | Vulnerability hunt |
| 3 | 🔧 Alternative | DeepSeek R1 0528 | Rewriting problem zones with fixed code |
| 4 | ⚖️ Comparison | Comparison model | Original vs alternative, impartial verdict |
| 5 | 👑 Final Verdict | Synthesis model | Structured report + Confidence Map |

**Transparency note:** Step 1 did not complete due to a `fetch` error. The consilium
proceeded with steps 2–5 and **reported this gap in the final verdict** rather than
hiding it. This is the Anti-Groupthink principle in practice — the system does not
pretend to a completeness it didn't achieve.

---

## 🤝 Consensus (where the AIs agreed)

The consilium participants converged on three core points:

1. **`executeAutoPay` in `SovereignBankPro.sol`** carries residual **reentrancy risk**
   even *with* the `nonReentrant` modifier, because external `safeTransfer` calls sit
   next to state changes. Requires isolation of external transfers and balance-invariant
   checks.

2. **`createStandingOrder` in `SovereignAutoPay.sol`** does **not validate input
   parameters** (`_amount`, `_interval`, `_recipient`), opening a **DoS vector** and
   allowing creation of non-functional orders.

3. **Initial role configuration** in `initialize`/constructor leaves the **deployer with
   permanent privileges** — an architectural risk requiring mandatory delegation through
   `SovereignTimelock` and auto-revocation of temporary roles.

There was also agreement on `initialize`: a `try/catch` around `decimals()` **masks real
token-compatibility errors** and must be replaced with strict verification within an
allowed range.

---

## ⚔️ Disagreements (Anti-Groupthink in action)

This is where the multi-AI architecture proved its value:

- **Step 2 (GPT-4o mini)** proposed a broader but shallower list (10 items, including
  `receive`, `emergencyWithdrawFull`, `claimInheritance`, `cancelRecovery`) — but
  **without concrete code and without confirming the findings** by reading the actual
  functions. Several were essentially hypotheses.

- **Step 3 (DeepSeek R1 0528)** concentrated on 4 items but delivered **ready-to-use code**.

- **Step 4 (Comparison)** effectively **discarded half of Step 2's findings as
  unconfirmed**, and flagged `emergencyWithdrawFull`/`claimInheritance` as
  *"missed cases requiring further work."*

A single model would have either drowned you in 10 unverified hypotheses, or given you
4 findings with no cross-check. The consilium **separated confirmed findings from noise** —
and honestly labeled what remained unaudited.

---

## 🔴 Critical Findings (must fix)

### 1. `executeAutoPay` — reentrancy via external transfers
Separate the "internal recipient" path (storage balance changes) from the "external
recipient" path (`safeTransfer`). All state changes **before** the external call; after
the call, verify the invariant
`stablecoin.balanceOf(address(this)) == contractBalance - netAmount`, else revert. Apply
the same pattern to `processPayment`, `withdraw`, `processRelayedPayment`.

### 2. `createStandingOrder` — missing input validation
Add checks: `_recipient != address(0)`, `_amount > 0`, `_interval` within sane bounds
(e.g. `1 days ≤ _interval ≤ 365 days`), and a pre-check `users[msg.sender].balance >= _amount`.
Without these, zero-value order spam and unexecutable orders are possible.

### 3. `initialize` — weak stablecoin verification
Replace the swallowing `try/catch` with strict verification: `decimals` must fall within
6–18, else revert with an explicit error. Prevents integration with counterfeit tokens
and non-standard ERC-20s.

### 4. `initialize`/constructor — permanent deployer privileges
Delegate all admin roles (`DEFAULT_ADMIN_ROLE`, `UPGRADER_ROLE`, `GOVERNANCE_ROLE`) to
`SovereignTimelock` at initialization; `renounceRole` for the deployer. Operational roles
(`OPERATOR_ROLE`, `PAUSER_ROLE`) with an auto-revocation or strict rotation mechanism.

### 5. Unaudited scenarios — require a separate pass
`emergencyWithdrawFull`, `claimInheritance`, `finalizeRecovery` and the related
`_liquidateUserVaults` in `SovereignVaults.sol` were **not deeply checked** by the
pipeline — yet they touch mass balance changes and the invariant
`actualBalance >= obligations`. A dedicated audit of `totalUserFunds`,
`totalVaultBalances`, `totalCollectedFees` updates and the `_checkSolvency` call after
each such operation is mandatory.

---

## 🟡 Recommended Changes

1. **`createVault` — check ordering.** Check user existence first, then blacklist.
   Gas savings on legitimate calls; minor readability improvement.
2. **Universal `safeTransferWithCheck` wrapper** for all ERC-20 operations: verify the
   contract's and recipient's balance delta after transfer. Protects against
   fee-on-transfer and rebasing tokens.
3. **`_checkSolvency`** — call after *every* mutation of
   `totalUserFunds`/`totalVaultBalances`/`totalCollectedFees`, not selectively. Make the
   invariant a hard contract condition, not an optional check.
4. **Agents (`approveAgent`/`revokeAgent`).** Tighten permission scope and add
   per-agent amount/operation limits.

---

## 🟢 Optional Improvements

1. Custom errors instead of string reverts throughout the contract.
2. Events for all administrative changes (`setFeeBasisPoints`, `setLimits`, `setTreasury`)
   with old and new values.
3. NatSpec documentation on public `SovereignBankPro` functions — simplifies future
   external audit.
4. `_recipient != msg.sender` check in `createStandingOrder` (if business logic forbids
   self-transfers).

---

## ✅ What the Consilium Approved (keep as-is)

- **Modular architecture** (`SovereignVaults`, `SovereignAutoPay`, `SovereignCashback`,
  `SovereignMath`) — simplifies audit and reuse.
- **UUPS + `SovereignTimelock`** upgrade mechanism — correct choice for a long-lived
  DeFi contract.
- **`nonReentrant` modifiers** on critical functions — correct base protection
  (needs strengthening, not removal).
- **`SovereignMath`** as a separate arithmetic library — good decision even with
  Solidity 0.8+ built-in protection.
- **`AccessControlUpgradeable` and `SafeERC20`** from OpenZeppelin — industry standard.
- **`MockERC20`** isolated from production logic — correct.

---

## 💡 Key Fixes (code)

```solidity
// SovereignAutoPay.sol
function createStandingOrder(
    address _recipient,
    uint256 _amount,
    uint256 _interval
) external {
    if (_recipient == address(0)) revert ZeroAddress();
    if (_amount == 0) revert ZeroAmount();
    if (_interval < 1 days || _interval > 365 days) revert InvalidInterval();
    if (users[msg.sender].balance < _amount) revert InsufficientBalance();

    uint256 newOrderId = ++autoPayCount[msg.sender];
    standingOrders[msg.sender][newOrderId] = StandingOrder({
        recipient: _recipient,
        amount: _amount,
        interval: _interval,
        lastPayment: block.timestamp,
        isActive: true
    });
    emit AutoPayCreated(msg.sender, newOrderId, _recipient, _amount, _interval);
}

// SovereignBankPro.sol::initialize (fragment)
uint8 d;
try IERC20Metadata(_stablecoin).decimals() returns (uint8 _d) {
    d = _d;
} catch {
    revert DecimalsCheckFailure();
}
if (d < 6 || d > 18) revert DecimalsOutOfRange(d);

_grantRole(DEFAULT_ADMIN_ROLE, address(timelock));
_grantRole(UPGRADER_ROLE, address(timelock));
// deployer receives no permanent admin rights

// SovereignBankPro.sol::executeAutoPay (structure)
function executeAutoPay(address _user, uint256 _orderId)
    external onlyRole(OPERATOR_ROLE) nonReentrant whenNotPaused
{
    if (blacklisted[_user]) revert BlacklistedAddress();
    StandingOrder storage order = standingOrders[_user][_orderId];
    User storage u = users[_user];
    if (!u.exists) revert UserNotRegistered();
    if (!_checkAutoPayReady(_user, _orderId)) revert OrderNotReady();
    if (u.balance < order.amount) revert InsufficientBalance();

    uint256 fee = (order.amount * feeBasisPoints) / 10000;
    uint256 net = order.amount - fee;

    if (users[order.recipient].exists) {
        u.balance -= order.amount;
        users[order.recipient].balance += net;
        totalUserFunds -= fee;
        totalCollectedFees += fee;
    } else {
        uint256 preBal = stablecoin.balanceOf(address(this));
        u.balance -= order.amount;
        totalUserFunds -= order.amount;
        totalCollectedFees += fee;
        SafeERC20.safeTransfer(stablecoin, order.recipient, net);
        if (stablecoin.balanceOf(address(this)) != preBal - net) revert TransferInvariant();
    }

    _markAutoPayExecuted(_user, _orderId);
    _checkSolvency();
    emit AutoPayExecuted(_user, _orderId, order.amount);
}
```

---

## 🎯 The Takeaway

The contract has a **solid modular foundation** but required mandatory fixes to the
reentrancy pattern in `executeAutoPay`, input validation in `createStandingOrder`, strict
stablecoin verification in `initialize`, and delegation of all admin roles to
`SovereignTimelock`.

But the real lesson is about **method**. The same frontier models that found nothing
alone found four critical issues when forced to work as a structured, adversarial team —
and were honest about the scenarios they didn't reach.

**That is the entire thesis of Egregor:** the failures of individual AI models overlap
far less than you'd expect. Make them debate, cross-check, and synthesize — and you catch
what any one of them, no matter how advanced, would miss alone.

> A single AI gives you an answer. A consilium gives you an answer **plus the map of its
> own uncertainty.**

---

### 🔗 Want to run audits like this?

Egregor's Code Review Pipeline runs a full smart-contract audit for **$0.30–2.00**
(three of the five pipeline models are free). See the [main README](./README.md) for
setup, features, and the full Egregor manual.

*Smart contract audits for the price of a coffee, not the price of a car.*
