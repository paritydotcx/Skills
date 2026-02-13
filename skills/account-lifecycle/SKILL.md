---
name: account-lifecycle
version: 0.8.0
description: Manages the full lifecycle of Solana accounts including creation, initialization, resize, migration, and closing with rent-exemption validation and lamport drainage safety
author: parity-team
license: MIT
tags:
  - anchor
  - tooling
  - security
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: framework
    type: string
    default: anchor
outputs:
  - name: findings
    type: Finding[]
  - name: lifecycle_map
    type: object
    description: Account lifecycle state diagram per account type
  - name: score
    type: number
    range: 0-100
  - name: summary
    type: string
---

# Account Lifecycle Manager Skill

You are analyzing how a Solana program manages the full lifecycle of its accounts: creation, initialization, mutation, reallocation, migration, and closure. Account lifecycle errors are a major source of vulnerabilities. Improper initialization allows reinitialization attacks. Improper closure leaves stale data or loses lamports. Missing rent checks cause accounts to be garbage-collected.

## Context Sources

| Source | Path | Contains |
|--------|------|----------|
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | `reinitialization-attack`, `close-account-drain`, `rent-exemption` patterns |
| Curated Audit Findings | `programs/parity/src/context_engine.rs` > `CURATED_AUDIT_FINDINGS` | Real close-account and state management vulnerabilities |
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct account-initialization, close-account patterns |
| Security Audit Skill | `skills/security-audit/SKILL.md` | Reinitialization (Step 7) and close safety (Step 8) checks |
| PDA Helper Skill | `skills/pda-helper/SKILL.md` | PDA creation and bump storage patterns |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

## Step 1: Account Type Inventory

For each `#[account]` type, determine its lifecycle:

1. **Creation point**: which instruction creates (inits) this account type
2. **Mutation points**: which instructions modify this account's data
3. **Reallocation points**: which instructions resize this account
4. **Closure point**: which instruction closes (destroys) this account
5. **Migration points**: any instructions that change the account's data layout (version upgrades)

Build a state diagram:
```
[uninitialized] --init--> [active] --update--> [active] --close--> [closed]
                                    --resize--> [active]
                                    --migrate-> [active_v2]
```

If an account type has no close instruction, flag it (lamports are permanently locked).

## Step 2: Initialization Safety (Critical)

For every `init` constraint:

1. **Space calculation**: `space = 8 + T::INIT_SPACE` is present and correct. Manual byte counting is fragile.

2. **Payer validation**: The `payer` account is a `Signer` and `mut` (will be debited for rent).

3. **Reinitialization guard**: Verify that calling the init instruction on an already-initialized account fails. Anchor's `init` constraint handles this by checking the discriminator, but `init_if_needed` does not.

4. **`init_if_needed` review**: If used, verify the instruction handles both the "new account" and "existing account" code paths correctly. The handler must check whether the account was just created or already existed.

```rust
// SAFE: init fails if account already exists
#[account(init, payer = user, space = 8 + Vault::INIT_SPACE)]

// CAUTION: init_if_needed allows re-entry into init logic
#[account(init_if_needed, payer = user, space = 8 + Vault::INIT_SPACE)]
```

**Finding if violated:**
- Severity: Critical
- Pattern: `reinitialization-attack`
- Title: "Account can be reinitialized via init_if_needed without guard"

## Step 3: Rent-Exemption Validation (Medium)

For every account created or resized:

1. The account will be rent-exempt after creation (Anchor's `init` handles this automatically)
2. For manual account creation via `create_account`, verify space and lamport calculations include the rent-exempt minimum
3. After `realloc`, verify the account still meets rent-exemption threshold

```rust
// Manual rent calculation (if not using Anchor init)
let rent = Rent::get()?;
let required_lamports = rent.minimum_balance(account_space);
```

**Finding if violated:**
- Severity: Medium
- Pattern: `rent-exemption`
- Title: "Account may not be rent-exempt after resize"

## Step 4: Reallocation Safety (High)

If the program uses `realloc`:

1. **Authorization**: Only authorized accounts (owner, admin) can trigger realloc
2. **Size bounds**: Realloc has a maximum size and does not allow unbounded growth
3. **Rent top-up**: If the account grows, additional lamports for rent are transferred from the payer
4. **Zero-init**: New bytes from realloc are not automatically zeroed in all cases. Verify the program does not read from newly allocated space without initializing it.
5. **Shrink safety**: If the account shrinks, verify lamports are returned to the payer

```rust
#[account(
    mut,
    realloc = 8 + new_size,
    realloc::payer = authority,
    realloc::zero = true, // zero-init new bytes
)]
```

**Finding if violated:**
- Severity: High
- Title: "Account reallocation without size bounds"

## Step 5: Close Account Safety (High)

For every close operation:

**Anchor `close` constraint:**
```rust
#[account(mut, close = destination, has_one = authority)]
pub target: Account<'info, SomeAccount>,
```
Anchor handles: lamport drain to destination, data zeroing, discriminator clearing.

**Manual close (if not using Anchor constraint):**
Verify all three steps are performed:
1. Transfer all lamports to destination
2. Zero all account data bytes
3. Assign account to System Program (optional but recommended)

```rust
// VULNERABLE: data not zeroed
let dest = ctx.accounts.destination.to_account_info();
let target = ctx.accounts.target.to_account_info();
**dest.try_borrow_mut_lamports()? += target.lamports();
**target.try_borrow_mut_lamports()? = 0;
// Missing: target.data.borrow_mut().fill(0);
```

**Checks:**
1. Close destination is validated (not attacker-controlled)
2. Authority check exists on close instruction
3. Account data is zeroed (prevents stale data reads)
4. If the account is a token account, its balance is zero before close

**Finding if violated:**
- Severity: High
- Pattern: `close-account-drain`
- Title: "Account close does not zero data"

## Step 6: Account Migration (Medium)

If the program supports account data migration (version upgrades):

1. Old and new layouts are compatible or have explicit migration logic
2. Migration instruction is access-controlled (admin only)
3. Space is recalculated for the new layout
4. All existing fields are preserved or explicitly handled
5. Version field exists to prevent double-migration

**Finding if applicable:**
- Severity: Medium
- Title: "Account migration does not validate version field"

## Step 7: Orphan Account Detection (Info)

Check for account types that:
- Are created but never closed (permanent lamport lock)
- Are created but never read after initialization (dead state)
- Have constraints referencing accounts that may not exist

**Finding if applicable:**
- Severity: Info
- Title: "Account type has no close instruction (lamports permanently locked)"

## Scoring

| Severity | Penalty |
|----------|---------|
| Critical | -25 |
| High | -15 |
| Medium | -8 |
| Info | -3 |

## Output Format

```json
{
  "score": 72,
  "lifecycle_map": {
    "Vault": {
      "init": "initialize",
      "mutations": ["deposit", "withdraw"],
      "realloc": null,
      "close": "close_vault",
      "migration": null,
      "rent_safe": true,
      "close_safe": true,
      "reinit_safe": true
    },
    "Config": {
      "init": "initialize_config",
      "mutations": ["update_config"],
      "realloc": null,
      "close": null,
      "migration": null,
      "rent_safe": true,
      "close_safe": null,
      "reinit_safe": true,
      "issues": ["no close instruction, lamports locked"]
    }
  },
  "findings": [...],
  "summary": "3 account types analyzed. 1 high (missing data zero on close), 1 info (Config has no close instruction). Init safety: all accounts protected against reinitialization."
}
```
