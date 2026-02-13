---
name: defi-review
version: 0.9.0
description: Systematic review of DeFi protocols on Solana covering AMM invariants, lending safety, oracle manipulation, flash loan vectors, and liquidation logic
author: parity-team
license: MIT
tags:
  - defi
  - security
  - oracle
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: protocol_type
    type: string
    required: false
    description: Protocol category (amm | lending | staking | vault | perpetuals)
  - name: framework
    type: string
    default: anchor
outputs:
  - name: findings
    type: Finding[]
  - name: score
    type: number
    range: 0-100
  - name: defi_surface
    type: object
    description: Identified DeFi attack surface map
  - name: summary
    type: string
---

# DeFi Protocol Review Skill

You are performing a specialized security review of a DeFi protocol on Solana. This skill extends the base security-audit with domain-specific checks for financial primitives: AMM constant-product invariants, lending collateral ratios, oracle price manipulation vectors, flash loan attack surfaces, and liquidation logic correctness.

## Context Sources

| Source | Path | Contains |
|--------|------|----------|
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | Base pattern database |
| Curated Audit Findings | `programs/parity/src/context_engine.rs` > `CURATED_AUDIT_FINDINGS` | Real DeFi vulnerabilities from public audits |
| Security Audit Skill | `skills/security-audit/SKILL.md` | Base security checks (run these first) |
| Token Analyzer Skill | `skills/token-analyzer/SKILL.md` | Token-specific checks for SPL/Token-2022 interactions |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

## Step 1: Protocol Classification

Identify the protocol type by analyzing instruction handlers and account structures:

- **AMM/DEX**: Look for `swap`, `add_liquidity`, `remove_liquidity` instructions, pool accounts with reserve balances, LP token mints
- **Lending**: Look for `deposit`, `borrow`, `repay`, `liquidate` instructions, collateral accounts, interest rate calculations
- **Staking**: Look for `stake`, `unstake`, `claim_rewards` instructions, stake pool accounts, reward distribution logic
- **Vault/Yield**: Look for `deposit`, `withdraw` with share-based accounting, strategy execution
- **Perpetuals**: Look for `open_position`, `close_position`, `add_margin`, oracle price feeds, funding rate calculations

Classify the protocol and apply the relevant domain-specific checks below.

## Step 2: Oracle Price Feed Validation (Critical)

If the protocol uses external price data, verify:

1. **Oracle source validation**: The oracle account is verified against a known program ID (Pyth, Switchboard, Chainlink). The account is not a raw `AccountInfo` that could be substituted with attacker-controlled data.

2. **Staleness check**: The oracle price has a freshness constraint. Prices older than N seconds (typically 30-60s) must be rejected.

3. **Confidence interval**: For Pyth oracles, the confidence interval is checked. A wide confidence band indicates unreliable pricing. Protocols should reject prices where `confidence / price > threshold` (typically 2-5%).

4. **Price manipulation window**: Check whether the protocol reads the price and acts on it in the same transaction without guards. Flash loan-funded oracle manipulation is possible if:
   - Price is read from an AMM pool's reserve ratio (not an oracle)
   - No TWAP or multi-block averaging is used
   - The price influences a high-value operation (liquidation, borrowing, swap)

**Vulnerable pattern:**
```rust
// VULNERABLE: reads pool reserves as price, manipulable via flash loan
let price = pool.reserve_a as u128 * PRECISION / pool.reserve_b as u128;
```

**Secure pattern:**
```rust
// SECURE: Pyth oracle with staleness and confidence checks
let price_feed = ctx.accounts.oracle.load_price_feed()?;
let current_price = price_feed.get_price_no_older_than(clock.unix_timestamp, MAX_ORACLE_AGE)?;

require!(
    current_price.conf as u64 * 100 / current_price.price.unsigned_abs() < MAX_CONFIDENCE_PCT,
    ErrorCode::OraclePriceUnreliable
);
```

**Finding if violated:**
- Severity: Critical
- Title: "Oracle price feed lacks staleness or confidence validation"

## Step 3: AMM Invariant Checks (Critical)

For AMM/DEX protocols, verify:

1. **Constant product**: `k = x * y` is maintained after every swap. The new reserves must satisfy `new_x * new_y >= old_x * old_y` (allowing for fees).

2. **Slippage protection**: Swap instructions accept a `minimum_amount_out` parameter. Without it, sandwich attacks extract value from every swap.

3. **LP share calculation**: Liquidity provider shares are calculated proportionally. First depositor attack: check that the pool cannot be initialized with dust amounts that give the first LP disproportionate shares.

4. **Remove liquidity**: LP token burn amount correctly corresponds to the share of reserves returned. Rounding favors the pool (rounds down the amount returned), not the withdrawer.

5. **Fee accounting**: Fees are deducted before the invariant check, not after. Fee recipient is validated.

**Finding if violated:**
- Severity: Critical
- Title: "AMM swap does not enforce constant product invariant"

## Step 4: Lending Protocol Checks (Critical)

For lending protocols, verify:

1. **Collateral ratio enforcement**: Every borrow instruction checks that the user's collateral value exceeds the borrow value by the required ratio (e.g., 150%).

2. **Interest accrual**: Interest compounds correctly. The accrual function handles time gaps (what happens if no one interacts for hours or days). Check for time-based manipulation where an attacker triggers accrual at a favorable moment.

3. **Liquidation threshold**: Liquidation is triggered at the correct threshold. The liquidator cannot liquidate healthy positions. The liquidation bonus does not exceed what makes the protocol insolvent.

4. **Bad debt handling**: What happens when collateral value drops below debt value between oracle updates. The protocol should have a bad debt socialization mechanism or insurance fund.

5. **Borrow cap**: Maximum borrow per user and per market is enforced.

**Finding if violated:**
- Severity: Critical
- Title: "Lending borrow instruction does not validate collateral ratio"

## Step 5: Flash Loan Attack Surface (Critical)

Analyze whether any operation can be exploited within a single transaction:

1. **Atomicity abuse**: Identify operations where reading a value and acting on it in the same instruction/transaction allows manipulation. Common vectors:
   - Read pool reserves, manipulate with flash loan, execute swap at manipulated price, repay flash loan
   - Read collateral value, inflate via flash loan deposit, borrow against inflated value, withdraw collateral

2. **Reentrancy via CPI**: Check whether the protocol's state is updated before or after CPI calls. Pre-CPI state updates prevent reentrancy.

3. **Multi-instruction transactions**: Check whether the protocol assumes operations happen in separate transactions. On Solana, multiple instructions in one transaction share the same account state.

**Finding if applicable:**
- Severity: Critical
- Title: "Protocol vulnerable to flash loan price manipulation"

## Step 6: Liquidation Logic (High)

If the protocol has liquidation:

1. **Liquidation incentive**: The bonus must be large enough to incentivize liquidators but small enough to not drain borrower equity excessively.

2. **Partial liquidation**: Check whether liquidators can liquidate more than necessary. A 100% liquidation when only 10% is needed harms borrowers.

3. **Self-liquidation**: Check whether a user can liquidate their own position to extract the liquidation bonus.

4. **Cascading liquidation**: If liquidating one user affects another user's collateral ratio, the protocol should handle cascading liquidations correctly.

**Finding if violated:**
- Severity: High
- Title: "Liquidation allows full position closure when partial would suffice"

## Step 7: Economic Invariants (High)

Check protocol-wide invariants that must always hold:

1. **Total deposits >= total borrows** (lending)
2. **LP token supply * share price = total reserves** (AMM)
3. **Staked amount + pending rewards = total pool balance** (staking)
4. **Sum of all user shares = total shares outstanding** (vault)

Identify whether these invariants are enforced on-chain or only implicitly assumed.

**Finding if violated:**
- Severity: High
- Title: "Protocol does not enforce global economic invariant on-chain"

## Scoring

| Severity | Penalty per finding |
|----------|-------------------|
| Critical | -25 |
| High | -15 |
| Medium | -8 |
| Info | -3 |

Start at 100. Subtract penalties. Floor at 0.

## Output Format

```json
{
  "score": 38,
  "defi_surface": {
    "protocol_type": "lending",
    "oracle_type": "pyth",
    "has_flash_loan": false,
    "has_liquidation": true,
    "token_standard": "spl-token",
    "attack_vectors_identified": ["oracle_staleness", "partial_liquidation_missing"]
  },
  "findings": [
    {
      "severity": "critical",
      "title": "Oracle price staleness not enforced in borrow instruction",
      "location": { "file": "lib.rs", "line": 134, "instruction": "borrow" },
      "description": "The Pyth price feed is read without a staleness check. An attacker could borrow against a stale price during high volatility.",
      "recommendation": "Add get_price_no_older_than(clock.unix_timestamp, 30) to reject prices older than 30 seconds."
    }
  ],
  "summary": "Lending protocol with 2 critical (oracle, collateral ratio), 1 high (liquidation). Flash loan surface: low (no pool-based pricing). Primary risk: oracle manipulation during borrow."
}
```
