# Anchor Framework Patterns

Reference implementations for correct Anchor usage. Each pattern shows the recommended way to implement a common Solana program construct. Skills compare target program code against these patterns to determine conformance.

---

## Pattern: Account Initialization

**Name:** `account-initialization`

Correct account initialization with automatic space calculation and PDA seeds.

```rust
#[account]
#[derive(InitSpace)]
pub struct Vault {
    pub authority: Pubkey,    // 32
    pub balance: u64,         // 8
    pub is_active: bool,      // 1
    pub bump: u8,             // 1
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Vault::INIT_SPACE,
        seeds = [b"vault", authority.key().as_ref()],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    
    #[account(mut)]
    pub authority: Signer<'info>,
    
    pub system_program: Program<'info, System>,
}
```

**Key points:**
- `#[derive(InitSpace)]` for automatic space calculation
- `8 +` prefix accounts for the Anchor discriminator
- `seeds` define deterministic PDA derivation
- `bump` is auto-computed on init, store it in the account for reuse
- `payer` must be `Signer` and `mut`

---

## Pattern: PDA Derivation

**Name:** `pda-derivation`

Deterministic PDA derivation with canonical bump storage.

```rust
// On-chain: store bump at init
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    let vault = &mut ctx.accounts.vault;
    vault.authority = ctx.accounts.authority.key();
    vault.bump = ctx.bumps.vault;  // store canonical bump
    Ok(())
}

// On-chain: reuse stored bump on subsequent access
#[account(
    seeds = [b"vault", authority.key().as_ref()],
    bump = vault.bump,  // ~0 CU vs ~1,500 CU for re-derivation
)]
pub vault: Account<'info, Vault>,

// Client-side: derive PDA
let (vaultPda, bump) = Pubkey::find_program_address(
    &[b"vault", owner.as_ref()],
    program_id,
);
```

**Key points:**
- Store canonical bump in account data at init
- Reference stored bump with `bump = account.bump` on access
- Never hardcode bump values
- Client derivation must use identical seeds

---

## Pattern: CPI Invocation

**Name:** `cpi-invocation`

Safe cross-program invocation using CpiContext and typed program accounts.

```rust
// Unsigned CPI (user is signer)
let cpi_ctx = CpiContext::new(
    ctx.accounts.token_program.to_account_info(),
    Transfer {
        from: ctx.accounts.user_token.to_account_info(),
        to: ctx.accounts.vault_token.to_account_info(),
        authority: ctx.accounts.authority.to_account_info(),
    },
);
token::transfer(cpi_ctx, amount)?;

// PDA-signed CPI (program is signer via PDA)
let seeds = &[b"vault", ctx.accounts.authority.key.as_ref(), &[ctx.accounts.vault.bump]];
let signer_seeds = &[&seeds[..]];

let cpi_ctx = CpiContext::new_with_signer(
    ctx.accounts.token_program.to_account_info(),
    Transfer {
        from: ctx.accounts.vault_token.to_account_info(),
        to: ctx.accounts.user_token.to_account_info(),
        authority: ctx.accounts.vault.to_account_info(),
    },
    signer_seeds,
);
token::transfer(cpi_ctx, amount)?;
```

**Key points:**
- Always use `Program<'info, T>` for the program account (never `AccountInfo`)
- Use `CpiContext::new_with_signer` for PDA-signed CPIs
- Signer seeds must exactly match PDA derivation seeds
- Bump in signer seeds comes from stored account data

---

## Pattern: Access Control

**Name:** `access-control`

Authority validation using `has_one` and constraint macros.

```rust
#[derive(Accounts)]
pub struct UpdateConfig<'info> {
    #[account(
        mut,
        has_one = authority,
        seeds = [b"config"],
        bump = config.bump,
    )]
    pub config: Account<'info, Config>,
    
    pub authority: Signer<'info>,
}

// Multiple authority levels
#[derive(Accounts)]
pub struct AdminAction<'info> {
    #[account(
        mut,
        seeds = [b"protocol"],
        bump = protocol.bump,
        constraint = protocol.admin == admin.key() @ ErrorCode::Unauthorized,
    )]
    pub protocol: Account<'info, Protocol>,
    
    pub admin: Signer<'info>,
}
```

**Key points:**
- `has_one = field` checks that account.field == referenced_account.key()
- `constraint = expr @ Error` for custom validation logic
- Authority account must be `Signer<'info>` (not `AccountInfo`)
- Use `@` syntax to attach specific error codes to constraints

---

## Pattern: Error Handling

**Name:** `error-handling`

Custom error definitions with `require!` macro for validation.

```rust
#[error_code]
pub enum ErrorCode {
    #[msg("Deposit amount must be greater than zero")]
    InvalidAmount,
    #[msg("Arithmetic overflow in balance calculation")]
    Overflow,
    #[msg("Withdrawal exceeds available balance")]
    InsufficientBalance,
    #[msg("Caller is not the program authority")]
    Unauthorized,
    #[msg("Account is not in active state")]
    AccountInactive,
}

// Usage with require! macro
pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
    require!(amount > 0, ErrorCode::InvalidAmount);
    require!(ctx.accounts.vault.is_active, ErrorCode::AccountInactive);
    
    // require_keys_eq! for pubkey comparison
    require_keys_eq!(
        ctx.accounts.vault.authority,
        ctx.accounts.authority.key(),
        ErrorCode::Unauthorized
    );
    
    Ok(())
}
```

**Key points:**
- Every `#[msg()]` should describe the condition, not just name the error
- Use `require!` instead of if-else with manual error returns
- `require_keys_eq!` for Pubkey comparisons (cleaner than manual)
- Error codes start at 6000 in Anchor (0-5999 reserved)

---

## Pattern: Close Account

**Name:** `close-account`

Safe account closure with lamport drain and data zeroing.

```rust
#[derive(Accounts)]
pub struct CloseVault<'info> {
    #[account(
        mut,
        close = destination,
        has_one = authority,
        seeds = [b"vault", authority.key().as_ref()],
        bump = vault.bump,
    )]
    pub vault: Account<'info, Vault>,
    
    #[account(mut)]
    /// CHECK: receives lamports from closed account
    pub destination: SystemAccount<'info>,
    
    pub authority: Signer<'info>,
}
```

**Key points:**
- `close = destination` handles lamport drain AND data zeroing
- `has_one = authority` ensures only the owner can close
- Destination should be `SystemAccount` or `Signer` (not raw `AccountInfo`)
- For token accounts: verify balance is zero before close

---

## Pattern: Event Emission

**Name:** `event-emission`

Structured event emission for off-chain indexing.

```rust
#[event]
pub struct DepositEvent {
    pub authority: Pubkey,
    pub vault: Pubkey,
    pub amount: u64,
    pub new_balance: u64,
    pub timestamp: i64,
}

pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
    let vault = &mut ctx.accounts.vault;
    vault.balance = vault.balance
        .checked_add(amount)
        .ok_or(ErrorCode::Overflow)?;
    
    let clock = Clock::get()?;
    
    emit!(DepositEvent {
        authority: ctx.accounts.authority.key(),
        vault: vault.key(),
        amount,
        new_balance: vault.balance,
        timestamp: clock.unix_timestamp,
    });
    
    Ok(())
}
```

**Key points:**
- Emit events at the end of every state-changing instruction
- Include who (signer), what (accounts affected), how much (values), and when (timestamp)
- Events are indexed by Anchor's event parser and RPC providers
- No `msg!` needed alongside events (events serve as structured logging)

---

## Pattern: Checked Math

**Name:** `checked-math`

Overflow-safe arithmetic using checked operations.

```rust
// Basic checked operations
let result = a.checked_add(b).ok_or(ErrorCode::Overflow)?;
let result = a.checked_sub(b).ok_or(ErrorCode::Underflow)?;
let result = a.checked_mul(b).ok_or(ErrorCode::Overflow)?;
let result = a.checked_div(b).ok_or(ErrorCode::DivisionByZero)?;

// Compound calculation with u128 upcast
let shares = (deposit as u128)
    .checked_mul(total_shares as u128)
    .ok_or(ErrorCode::Overflow)?
    .checked_div(total_deposits as u128)
    .ok_or(ErrorCode::DivisionByZero)? as u64;

// Saturating operations (for non-critical values like counters)
let counter = counter.saturating_add(1);

// Power/exponent
let factor = 10u64.checked_pow(decimals as u32).ok_or(ErrorCode::Overflow)?;
```

**Key points:**
- Use `checked_*` for all calculations involving user values, balances, and fees
- Upcast to `u128` before multiplying two `u64` values to prevent intermediate overflow
- `saturating_*` is acceptable for non-financial counters where capping is safe
- Always provide a specific error code (not generic ProgramError)
