---
name: token-analyzer
version: 1.0.3
description: Deep analysis of SPL Token and Token-2022 program interactions
author: parity-team
license: MIT
tags:
  - token
  - security
  - analysis
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: token_standard
    type: string
    default: both
    description: Token standard to analyze against (spl-token | token-2022 | both)
outputs:
  - name: findings
    type: Finding[]
  - name: score
    type: number
    range: 0-100
  - name: token_interactions
    type: object
    description: Map of all token operations discovered in the program
  - name: summary
    type: string
---

# Token Program Analyzer Skill

You are performing a specialized analysis of how a Solana program interacts with SPL Token and Token-2022 (Token Extensions) programs. This skill goes deeper than the general security-audit on token-specific attack surfaces: mint authority abuse, transfer hook bypasses, metadata extension mishandling, and token account lifecycle errors.

## Context Sources

Before starting analysis, fetch the following from the Parity repository:

| Source | Path | Contains |
|--------|------|----------|
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | Base vulnerability patterns including CPI safety and account validation |
| Curated Audit Findings | `programs/parity/src/context_engine.rs` > `CURATED_AUDIT_FINDINGS` | Real token-related vulnerabilities from OtterSec, Sec3, Neodyme |
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct CPI and account patterns for token operations |
| Security Audit Skill | `skills/security-audit/SKILL.md` | Base security checks that apply to all programs |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

## Step 1: Identify Token Operations

Scan the program for all token-related operations and catalog them:

**Mint operations:**
- `MintTo` / `mint_to` CPI calls
- Mint authority checks and transfers
- Supply cap enforcement (if any)

**Transfer operations:**
- `Transfer` / `transfer` CPI calls
- `TransferChecked` / `transfer_checked` usage (required for Token-2022)
- Transfer fee handling
- Transfer hook interactions

**Burn operations:**
- `Burn` / `burn` CPI calls
- Burn authority validation

**Account management:**
- `InitializeAccount` / token account creation
- `CloseAccount` / token account closure
- Associated Token Account (ATA) derivation and usage
- `FreezeAccount` / `ThawAccount` operations

**Token-2022 extensions (if applicable):**
- Transfer hooks
- Metadata / MetadataPointer
- Confidential transfers
- Transfer fees
- Permanent delegate
- Non-transferable tokens
- Interest-bearing tokens

Build a complete map of which instructions perform which token operations.

## Step 2: Mint Authority Validation (Critical)

For every mint operation, verify:

1. The mint authority is validated before CPI
2. Mint authority cannot be reassigned without proper governance
3. Supply is bounded if the protocol expects finite supply

**Vulnerable pattern:**
```rust
// VULNERABLE: mint authority not validated, any signer can mint
let cpi_ctx = CpiContext::new(
    ctx.accounts.token_program.to_account_info(),
    MintTo {
        mint: ctx.accounts.mint.to_account_info(),
        to: ctx.accounts.destination.to_account_info(),
        authority: ctx.accounts.mint_authority.to_account_info(), // unchecked
    },
);
```

**Secure pattern:**
```rust
// SECURE: PDA mint authority with seeds verification
let seeds = &[b"mint-authority", &[ctx.accounts.config.mint_auth_bump]];
let signer_seeds = &[&seeds[..]];

let cpi_ctx = CpiContext::new_with_signer(
    ctx.accounts.token_program.to_account_info(),
    MintTo {
        mint: ctx.accounts.mint.to_account_info(),
        to: ctx.accounts.destination.to_account_info(),
        authority: ctx.accounts.mint_authority.to_account_info(),
    },
    signer_seeds,
);

require!(
    ctx.accounts.mint.supply
        .checked_add(amount)
        .ok_or(ErrorCode::Overflow)? <= MAX_SUPPLY,
    ErrorCode::SupplyCapExceeded
);
```

**Finding if violated:**
- Severity: Critical
- Title: "Unvalidated mint authority allows unlimited token creation"

## Step 3: Transfer Safety (High)

For every transfer operation, check:

1. **TransferChecked vs Transfer**: Token-2022 requires `transfer_checked` with explicit decimals. Using `Transfer` on Token-2022 mints will fail silently or bypass transfer hooks.

2. **Transfer hook compliance**: If the mint has a transfer hook extension, verify the program passes the required extra accounts. Missing hook accounts causes the transfer to fail.

3. **Amount validation**: Transfer amounts are validated against available balance before CPI (avoid relying on the token program to reject, as error messages are opaque).

4. **Self-transfer**: Check whether the program guards against transferring to the same account (source == destination), which can be used to manipulate balance tracking in some protocols.

**Vulnerable pattern:**
```rust
// VULNERABLE: uses Transfer instead of TransferChecked
// Will fail on Token-2022 mints with transfer hooks
token::transfer(cpi_ctx, amount)?;
```

**Secure pattern:**
```rust
// SECURE: uses TransferChecked, compatible with both SPL Token and Token-2022
token::transfer_checked(cpi_ctx, amount, decimals)?;
```

**Finding if violated:**
- Severity: High
- Title: "Transfer operation incompatible with Token-2022"

## Step 4: Token Account Lifecycle (High)

Verify token account creation and closure follows safe patterns:

1. **Creation**: Token accounts are initialized with the correct mint and owner. Associated Token Accounts (ATAs) are preferred over manual initialization.

2. **Closure**: When closing token accounts:
   - Balance must be zero before close (or explicitly transferred out)
   - Lamport destination must be validated
   - The `close_authority` must be the token account owner or a validated delegate

3. **Rent recovery**: Closed token accounts should return rent to a controlled address, not an arbitrary one.

**Vulnerable pattern:**
```rust
// VULNERABLE: closing token account without checking balance
// Remaining tokens are burned, potentially losing user funds
token::close_account(cpi_ctx)?;
```

**Secure pattern:**
```rust
// SECURE: verify balance is zero before closing
require!(
    ctx.accounts.token_account.amount == 0,
    ErrorCode::TokenAccountNotEmpty
);
token::close_account(cpi_ctx)?;
```

**Finding if violated:**
- Severity: High
- Title: "Token account closed with non-zero balance"

## Step 5: Decimal Handling (High)

Check that the program correctly handles token decimals:

1. All user-facing amounts must account for the mint's decimal precision
2. Conversions between raw amounts and UI amounts use the correct power of 10
3. Division operations on token amounts handle remainder correctly (no silent truncation)
4. Cross-mint operations (swaps, LPs) normalize decimals before comparison

**Vulnerable pattern:**
```rust
// VULNERABLE: assumes 6 decimals, breaks for mints with different precision
let ui_amount = raw_amount / 1_000_000;
```

**Secure pattern:**
```rust
// SECURE: reads decimals from mint account
let decimals = ctx.accounts.mint.decimals;
let factor = 10u64.checked_pow(decimals as u32).ok_or(ErrorCode::Overflow)?;
```

**Finding if violated:**
- Severity: High
- Title: "Hardcoded decimal precision"

## Step 6: Token-2022 Extension Compatibility (Medium)

If the program is designed to work with Token-2022, verify:

1. **Transfer fees**: If the mint charges transfer fees, the program accounts for the fee in its logic. The received amount is less than the sent amount.

2. **Permanent delegate**: If the mint has a permanent delegate, the delegate can transfer tokens without the owner's signature. Programs that assume only the owner can move tokens are vulnerable.

3. **Non-transferable**: Programs must check this extension before attempting transfers.

4. **Confidential transfers**: If used, programs cannot read amounts from the account directly.

5. **Metadata**: Programs reading metadata must handle both legacy Metaplex metadata and Token-2022 on-chain metadata.

**Finding if applicable:**
- Severity: Medium
- Title: "Program does not account for Token-2022 transfer fee extension"

## Step 7: Associated Token Account Derivation (Medium)

Verify ATA usage:

1. ATAs are derived using `get_associated_token_address` or equivalent
2. The program does not assume a specific token account exists without checking
3. ATA creation uses `create_associated_token_account` or `init_if_needed` with proper constraints

**Finding if violated:**
- Severity: Medium
- Title: "Token account address not derived as ATA"

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
  "score": 62,
  "token_interactions": {
    "mints": ["initialize_mint@L23", "mint_tokens@L67"],
    "transfers": ["deposit@L45", "withdraw@L89"],
    "burns": [],
    "closures": ["close_vault@L112"],
    "extensions_used": ["transfer_fee", "metadata_pointer"]
  },
  "findings": [
    {
      "severity": "critical",
      "title": "Unvalidated mint authority in mint_tokens instruction",
      "location": { "file": "lib.rs", "line": 67, "instruction": "mint_tokens" },
      "description": "The mint_authority account is not validated against the config. Any account can be passed as mint authority.",
      "recommendation": "Add has_one = mint_authority constraint on the config account, or use PDA-derived mint authority with seeds verification."
    }
  ],
  "summary": "1 critical, 2 high, 1 medium. Primary risk: unauthorized minting. Token-2022 compatibility: partial (missing transfer fee handling)."
}
```
