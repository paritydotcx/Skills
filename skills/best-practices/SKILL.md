---
name: best-practices
version: 1.2.0
description: Solana and Anchor coding standards analysis with convention scoring
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
outputs:
  - name: findings
    type: Finding[]
  - name: score
    type: number
    range: 0-100
  - name: summary
    type: string
---

# Best Practices Skill

You are evaluating a Solana smart contract against Anchor and Solana development best practices. Unlike the security-audit skill, this analysis focuses on code quality, maintainability, and adherence to framework conventions rather than exploitable vulnerabilities. Findings from this skill are typically medium or info severity but indicate code that is harder to maintain, more error-prone, or deviates from community standards.

## Context Sources

Before starting analysis, fetch the following from the Parity repository to use as reference patterns:

| Source | Path | Contains |
|--------|------|----------|
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct Anchor patterns for account init, PDA derivation, CPI, access control, error handling, close, events, and checked math |
| Skill Definition | `programs/parity/src/skills.rs` > `BUILTIN_SKILLS` (best-practices entry) | Canonical step list and input/output spec for this skill |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

Compare the target program's patterns against `ANCHOR_PATTERNS` to determine conformance. Each pattern entry has `pattern_name`, `description`, and `example_code` showing the correct implementation.

## Step 1: Parse Program Structure

Read the entire program source and catalog:

- All `#[derive(Accounts)]` structs and their constraints
- All `#[account]` definitions (stored account types)
- All instruction handler functions
- Error enum definitions (`#[error_code]`)
- Event definitions (`#[event]`)
- Use of `msg!` logging macro
- Module structure and imports

## Step 2: Account Space Calculation

Check whether stored account types use `InitSpace` derive for automatic space calculation.

**What to look for:**
- `#[derive(InitSpace)]` on all `#[account]` structs
- Hardcoded `space` values in `init` constraints (fragile, breaks when fields change)
- `8 + MyAccount::INIT_SPACE` pattern in init constraints (correct usage)

**Suboptimal pattern:**
```rust
// BAD: hardcoded space breaks silently when fields are added
#[account(init, payer = user, space = 8 + 32 + 8 + 1)]
```

**Recommended pattern:**
```rust
#[derive(InitSpace)]
pub struct MyAccount {
    pub authority: Pubkey,   // 32
    pub balance: u64,        // 8
    pub is_active: bool,     // 1
}

#[account(init, payer = user, space = 8 + MyAccount::INIT_SPACE)]
```

**Finding if violated:**
- Severity: Medium
- Title: "Hardcoded account space calculation"
- Description: "Account initialization uses hardcoded byte count instead of InitSpace derive. Adding or removing fields will silently allocate incorrect space, potentially truncating data or wasting rent."

## Step 3: Error Definitions

Check that the program defines descriptive custom errors.

**What to look for:**
- `#[error_code]` enum with named variants
- Error messages that describe the condition, not just the error name
- Use of `require!` or `require_keys_eq!` macros instead of if-else with raw error returns
- Instruction handlers that return `Ok(())` without validating input preconditions

**Suboptimal pattern:**
```rust
// BAD: error name alone is not informative in transaction logs
#[error_code]
pub enum MyError {
    Unauthorized,
    InvalidInput,
}
```

**Recommended pattern:**
```rust
#[error_code]
pub enum MyError {
    #[msg("Caller is not the program authority")]
    Unauthorized,
    #[msg("Deposit amount must be greater than zero")]
    InvalidDepositAmount,
    #[msg("Withdrawal exceeds available balance")]
    InsufficientBalance,
}
```

**Finding if violated:**
- Severity: Info
- Title: "Error variants missing descriptive messages"
- Description: "Custom error variants do not include #[msg()] annotations. Transaction explorers and debugging tools will only show the variant name, making on-chain errors harder to diagnose."

## Step 4: Event Emission

Every state-changing instruction should emit an event for off-chain indexing.

**What to look for:**
- `#[event]` struct definitions for significant state changes (deposits, withdrawals, transfers, config updates, account creation/closure)
- `emit!()` calls at the end of instruction handlers
- Events that include relevant context (who, what, how much, when)

**Suboptimal pattern:**
```rust
// BAD: state changes with no event emission
pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
    ctx.accounts.vault.balance += amount;
    Ok(())
}
```

**Recommended pattern:**
```rust
pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
    ctx.accounts.vault.balance = ctx.accounts.vault.balance
        .checked_add(amount)
        .ok_or(ErrorCode::Overflow)?;
    
    emit!(DepositEvent {
        user: ctx.accounts.user.key(),
        vault: ctx.accounts.vault.key(),
        amount,
        new_balance: ctx.accounts.vault.balance,
    });
    
    Ok(())
}
```

**Finding if violated:**
- Severity: Medium
- Title: "Missing event emission on state change"
- Description: "Instruction modifies on-chain state without emitting an event. Off-chain indexers, analytics, and UIs will not be able to track this operation without polling account data directly."

## Step 5: Typed Account Wrappers

Check that all account references use typed Anchor wrappers instead of raw `AccountInfo`.

**What to look for:**
- `AccountInfo<'info>` used where `Account<'info, T>`, `Program<'info, T>`, `Signer<'info>`, or `SystemAccount<'info>` would be appropriate
- `UncheckedAccount` used without a documented reason (comment explaining why)

**Finding if violated:**
- Severity: Medium
- Title: "Raw AccountInfo used instead of typed wrapper"
- Description: "Account is declared as AccountInfo without type validation. Using Account<'info, T> enforces discriminator checks, owner validation, and deserialization safety. Raw AccountInfo should only be used when explicitly needed with a documented reason."

## Step 6: Instruction Handler Complexity

Evaluate whether instruction handlers follow the single-responsibility principle.

**What to look for:**
- Instruction handlers longer than 50 lines (suggest extracting helper functions)
- Multiple unrelated state mutations in a single instruction
- Business logic mixed with account validation logic (Anchor handles validation in the Accounts struct, business logic belongs in the handler)

**Finding if violated:**
- Severity: Info
- Title: "Instruction handler exceeds recommended complexity"
- Description: "Handler function is over 50 lines and performs multiple conceptually distinct operations. Consider extracting helper functions or splitting into separate instructions for readability and testability."

## Step 7: Logging

Check for diagnostic `msg!` logging in instruction handlers.

**What to look for:**
- `msg!` calls that log the instruction name and key parameters at entry
- Absence of logging in error branches (makes debugging failed transactions difficult)
- Avoid logging sensitive data (private keys, seeds) in production

**Finding if violated:**
- Severity: Info
- Title: "No diagnostic logging in instruction handler"
- Description: "Instruction handler does not include msg! logging. Adding entry-point logging with key parameters aids debugging via Solana transaction logs."

## Step 8: Constant Definitions

Check that magic numbers and repeated values are extracted to named constants.

**What to look for:**
- Hardcoded numeric values in logic (fee percentages, thresholds, timeouts)
- Repeated string literals used as PDA seeds without a shared constant
- Anchor discriminator size (8 bytes) referenced as a raw number instead of a named constant

**Finding if violated:**
- Severity: Info
- Title: "Magic number in program logic"
- Description: "Numeric literal is used directly in logic without a named constant. This makes the value's purpose unclear and increases risk of inconsistency if the same value appears elsewhere."

## Scoring

Calculate a best-practices score from 0 to 100:

| Severity | Penalty per finding |
|----------|-------------------|
| Medium | -8 |
| Info | -3 |

Start at 100. Subtract penalties. Floor at 0.

## Output Format

Return results as structured JSON:

```json
{
  "score": 82,
  "findings": [
    {
      "severity": "medium",
      "title": "Missing event emission on withdraw",
      "location": {
        "file": "lib.rs",
        "line": 73,
        "instruction": "withdraw"
      },
      "description": "The withdraw instruction modifies vault balance without emitting an event.",
      "recommendation": "Add a WithdrawEvent struct and emit it with user, vault, amount, and remaining balance fields."
    }
  ],
  "summary": "0 critical, 0 high, 2 medium, 3 info findings. Code is functional but would benefit from event emission and typed wrappers."
}
```
