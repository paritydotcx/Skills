---
name: security-audit
version: 1.2.0
description: Comprehensive Solana program security analysis with structured finding output
author: parity
license: MIT
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: framework
    type: string
    default: anchor
    description: Program framework (anchor | native)
  - name: severity_threshold
    type: string
    default: info
    description: Minimum severity to include in output (critical | high | medium | info)
outputs:
  - name: findings
    type: Finding[]
  - name: score
    type: number
    range: 0-100
  - name: summary
    type: string
chains_to: null
---

# Security Audit Skill

You are performing a security audit of a Solana smart contract. Your goal is to identify vulnerabilities that could lead to loss of funds, unauthorized access, or protocol manipulation. You must produce structured findings with severity, location, description, and remediation for each issue.

## Context

Solana programs operate under a different security model than EVM contracts. The runtime does not enforce access control by default. Every account passed to an instruction is caller-controlled unless the program explicitly validates ownership, signing authority, and derivation. Anchor provides constraint macros that handle common validations, but misconfiguration or omission of these constraints is the primary source of exploitable vulnerabilities.

According to aggregated audit data from OtterSec, Sec3, and Neodyme across 163 public Solana audits, 1,669 vulnerabilities were identified with an average of 1.4 critical or high-severity findings per engagement.

## Context Sources

Before starting analysis, fetch the following pattern databases from the Parity repository. These contain the detection rules, severity classifications, and fix patterns you must apply during each step.

| Source | Path | Contains |
|--------|------|----------|
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | Pattern IDs, severity levels, pattern types, descriptions, and detection hints for each vulnerability class |
| Curated Audit Findings | `programs/parity/src/context_engine.rs` > `CURATED_AUDIT_FINDINGS` | Real vulnerability-and-fix pairs from OtterSec, Sec3, and Neodyme public audit reports |
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct Anchor usage patterns (init, PDA, CPI, access control, close, events, checked math) with example code |
| Scoring Function | `programs/parity/src/context_engine.rs` > `calculate_risk_score` | Severity-to-penalty mapping used for the final score |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

Load `VULNERABILITY_RULES` to get the `detection_hint` for each pattern type. These hints tell you exactly what syntax or constraint to search for in the target program. Load `CURATED_AUDIT_FINDINGS` to cross-reference any discovered vulnerability against known real-world instances and their documented fix patterns. Load `ANCHOR_PATTERNS` to compare the target program's code against known-correct implementations.

## Step 1: Parse Program Structure

Read the entire program source. Identify and catalog:

- All `#[derive(Accounts)]` structs (these define the account inputs for each instruction)
- All `#[account]` and `#[program]` blocks
- Every instruction handler function (functions inside the `#[program]` module)
- All CPI calls (`invoke`, `invoke_signed`, `CpiContext::new`, `CpiContext::new_with_signer`)
- All PDA derivation calls (`Pubkey::find_program_address`, `Pubkey::create_program_address`, `seeds` constraints)
- All arithmetic operations on token amounts, balances, or user-supplied values
- All account close operations (`close = target` constraint or manual lamport transfer)

Build a mental map of the program's instruction flow: which accounts each instruction touches, what state it mutates, and what external programs it invokes.

## Step 2: Signer Validation (Critical)

For every instruction that modifies state, check that the authority account is constrained as a signer.

**What to look for:**
- `Signer<'info>` type on authority/admin/owner accounts
- `has_one = authority` constraint linking a state account to its authorized signer
- `constraint = some_account.authority == signer.key()` for manual authority checks

**Vulnerable pattern:**
```rust
// VULNERABLE: authority is AccountInfo, not Signer
pub authority: AccountInfo<'info>,
```

**Secure pattern:**
```rust
// SECURE: authority must sign the transaction
pub authority: Signer<'info>,

// SECURE: state account validates its authority
#[account(mut, has_one = authority)]
pub config: Account<'info, Config>,
```

**Finding if violated:**
- Severity: Critical
- Pattern: `missing-signer-check`
- Description: "Instruction does not verify that the authority account has signed the transaction. Any account can be passed as authority, allowing unauthorized state modification."

## Step 3: Arithmetic Safety (High)

Check every arithmetic operation (`+`, `-`, `*`, `/`, `%`) on values that could be user-influenced or that represent token amounts.

**What to look for:**
- Direct use of `+`, `-`, `*` operators instead of `checked_add`, `checked_sub`, `checked_mul`
- Division without zero-check on divisor
- Casting between integer sizes (`as u64`, `as u128`) without bounds validation
- Exponentiation or compound calculations without overflow guards

**Vulnerable pattern:**
```rust
// VULNERABLE: overflow on large deposits
let shares = deposit_amount * total_shares / total_deposits;
```

**Secure pattern:**
```rust
// SECURE: checked arithmetic with explicit error
let shares = deposit_amount
    .checked_mul(total_shares)
    .ok_or(ErrorCode::Overflow)?
    .checked_div(total_deposits)
    .ok_or(ErrorCode::DivisionByZero)?;
```

**Finding if violated:**
- Severity: High
- Pattern: `unchecked-arithmetic`
- Description: "Arithmetic operation may overflow or underflow. On large values this could result in incorrect token amounts, share calculations, or fee computations."

## Step 4: PDA Validation (Critical)

For every PDA used in the program, verify that:

1. The seeds are deterministic and not attacker-controlled
2. The bump is stored and reused (canonical bump), not re-derived on every call
3. PDA accounts have `seeds` and `bump` constraints in their `#[derive(Accounts)]` struct

**What to look for:**
- `seeds = [...]` constraint present on all PDA accounts
- `bump = account.bump` using stored canonical bump
- No user-supplied strings or unbounded byte arrays in seeds without length validation
- `find_program_address` calls where seeds include attacker-controlled data

**Vulnerable pattern:**
```rust
// VULNERABLE: user_input has no length limit, can collide with other PDAs
seeds = [b"vault", user_input.as_bytes()]
```

**Secure pattern:**
```rust
// SECURE: deterministic seeds with stored bump
#[account(
    seeds = [b"vault", owner.key().as_ref()],
    bump = vault.bump,
)]
pub vault: Account<'info, Vault>,
```

**Finding if violated:**
- Severity: Critical
- Pattern: `unvalidated-pda`
- Description: "PDA derivation uses attacker-controlled seeds without validation. An attacker could derive a different PDA than intended, accessing or overwriting arbitrary program state."

## Step 5: CPI Security (Critical)

Every cross-program invocation must verify the target program ID.

**What to look for:**
- CPI calls using `invoke` or `invoke_signed` with `AccountInfo` for the program account instead of a typed `Program<'info, T>` wrapper
- Missing program ID validation before CPI
- CPI to token programs without using `anchor_spl` typed accounts

**Vulnerable pattern:**
```rust
// VULNERABLE: program account is unverified AccountInfo
pub token_program: AccountInfo<'info>,
```

**Secure pattern:**
```rust
// SECURE: program type is enforced at deserialization
pub token_program: Program<'info, Token>,
```

**Finding if violated:**
- Severity: Critical
- Pattern: `insecure-cpi`
- Description: "Cross-program invocation does not verify the target program ID. An attacker could substitute a malicious program that mimics the expected interface but executes arbitrary logic."

## Step 6: Account Type Safety (High)

Every account deserialized as input must use typed wrappers that verify the Anchor discriminator.

**What to look for:**
- Raw `AccountInfo<'info>` used where `Account<'info, T>` should be used
- Manual deserialization with `try_from_slice` without discriminator check
- Accounts where different types could be substituted (type cosplay)

**Vulnerable pattern:**
```rust
// VULNERABLE: no discriminator check, any account data matches
pub user_account: AccountInfo<'info>,
```

**Secure pattern:**
```rust
// SECURE: Account<> verifies discriminator on deserialization
pub user_account: Account<'info, UserAccount>,
```

**Finding if violated:**
- Severity: High
- Pattern: `type-cosplay`
- Description: "Account can be substituted with a different account type due to missing discriminator check. An attacker could pass a Token account where a Config account is expected."

## Step 7: Reinitialization Protection (Critical)

Accounts that use `init` must not be re-initializable.

**What to look for:**
- `init` constraint without `init_if_needed` flag on accounts that should only be created once
- Missing `is_initialized` flag or equivalent check
- Accounts where calling the init instruction again overwrites existing state

**Finding if violated:**
- Severity: Critical
- Pattern: `reinitialization-attack`
- Description: "Account can be re-initialized by calling the init instruction multiple times. An attacker could reset protocol state, overwrite ownership, or drain accumulated funds."

## Step 8: Account Close Safety (High)

If the program closes accounts (returns lamports and deallocates), verify:

1. Lamports are drained to a specified destination
2. Account data is zeroed after drain
3. The close operation has proper authority checks

**What to look for:**
- `close = destination` constraint in Anchor (handles both drain and zero)
- Manual close logic that transfers lamports but does not zero data
- Close instructions without signer validation

**Vulnerable pattern:**
```rust
// VULNERABLE: data not zeroed, stale data remains readable
**account.to_account_info().try_borrow_mut_lamports()? = 0;
```

**Secure pattern:**
```rust
// SECURE: Anchor handles lamport drain and data zeroing
#[account(mut, close = destination, has_one = authority)]
pub target: Account<'info, SomeAccount>,
```

**Finding if violated:**
- Severity: High
- Pattern: `close-account-drain`
- Description: "Close account instruction does not properly drain lamports and zero data. Stale data could be read by subsequent transactions or the account could be re-occupied."

## Step 9: Owner Validation (High)

Accounts passed to instructions must have their owner field validated if the program relies on cross-program account data.

**What to look for:**
- Accounts from other programs (SPL Token accounts, Associated Token accounts) without owner checks
- `owner = expected_program` constraint missing on foreign program accounts
- Manual owner checks that compare `account.owner` incorrectly

**Finding if violated:**
- Severity: High
- Pattern: `owner-check`
- Description: "Account owner is not validated. An attacker could inject an account owned by a different program with crafted data matching the expected layout."

## Step 10: Reentrancy via CPI (Critical)

If the program performs state updates after CPI calls, check for reentrancy risk.

**What to look for:**
- State mutation (balance updates, counter increments, flag changes) that occurs AFTER a CPI call rather than BEFORE
- CPI calls to programs that could callback into the originating program

**Secure pattern:**
```
// Checks-effects-interactions:
1. Validate inputs (checks)
2. Update state (effects)
3. Perform CPI (interactions)
```

**Finding if violated:**
- Severity: Critical
- Pattern: `reentrancy-via-cpi`
- Description: "State update occurs after CPI call. If the target program performs a callback, the state can be read in an inconsistent state, enabling reentrancy attacks."

## Scoring

Calculate a security score from 0 to 100 using the following penalty model:

| Severity | Penalty per finding |
|----------|-------------------|
| Critical | -25 |
| High | -15 |
| Medium | -8 |
| Info | -3 |

Start at 100. Subtract penalties for each finding. Floor at 0. A program with no findings scores 100. A program with two critical findings and one high finding scores 100 - 50 - 15 = 35.

## Output Format

Return results as a structured JSON object:

```json
{
  "score": 35,
  "findings": [
    {
      "severity": "critical",
      "pattern": "missing-signer-check",
      "title": "Missing signer validation on withdraw instruction",
      "location": {
        "file": "lib.rs",
        "line": 47,
        "instruction": "withdraw"
      },
      "description": "The withdraw instruction accepts an authority account as AccountInfo without Signer constraint. Any address can call withdraw and drain the vault.",
      "recommendation": "Change authority field to Signer<'info> and add has_one = authority constraint on the vault account."
    }
  ],
  "summary": "2 critical, 1 high, 0 medium, 0 info findings. Primary risk: unauthorized fund withdrawal via missing signer check."
}
```

Each finding must include the exact line number, the instruction name where it occurs, a specific description of the vulnerability (not generic), and a concrete remediation with code-level guidance.
