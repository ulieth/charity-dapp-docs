# Anchor Program Overview

The heart of the Solana Charity dApp is a sophisticated smart contract built with the Anchor framework. This program handles all on-chain logic for charity management, donations, and fund withdrawals with enterprise-grade security and efficiency.

## Program Architecture

### Core Philosophy

The program is designed around three key principles:

1. **Security First**: All operations are validated, access-controlled, and error-handled
2. **Gas Efficiency**: Optimized for minimal transaction costs
3. **Transparency**: Every action is logged via events for complete auditability

### Program Structure

```
anchor/programs/charity/src/
├── lib.rs                    # Main program entry point
├── instructions/             # All program instructions
│   ├── create_charity.rs
│   ├── update_charity.rs
│   ├── donate_sol.rs
│   ├── withdraw_donations.rs
│   ├── pause_donations.rs
│   └── delete_charity.rs
├── state/                    # Account structures
│   ├── charity.rs
│   └── donation.rs
└── common/                   # Shared utilities
    ├── constants.rs
    ├── errors.rs
    └── events.rs
```

## Program Instructions

### 1. Create Charity (`create_charity`)

Creates a new charity account with associated vault for donations.

**Function signature for creating a new charity:**
```rust
pub fn create_charity(
    ctx: Context<CreateCharity>,
    name: String,
    description: String,
) -> Result<()>
```

**What it does**:
- Validates input length constraints
- Creates charity account with authority
- Initializes vault PDA for donations
- Emits creation event for transparency

### 2. Donate SOL (`donate_sol`)

Transfers SOL from donor to charity vault.

**Function signature for making a donation:**
```rust
pub fn donate_sol(
    ctx: Context<DonateSol>,
    amount: u64
) -> Result<()>
```

**What it does**:
- Checks if donations are paused
- Transfers SOL to vault PDA
- Updates charity statistics
- Records donation history
- Emits donation event

### 3. Withdraw Donations (`withdraw_donations`)

Allows charity authority to withdraw funds.

**Function signature for withdrawing donations:**
```rust
pub fn withdraw_donations(
    ctx: Context<WithdrawDonations>,
    amount: u64
) -> Result<()>
```

**What it does**:
- Validates authority ownership
- Ensures sufficient vault balance
- Maintains rent-exempt balance
- Transfers funds to recipient
- Updates charity state

### 4. Pause/Unpause Donations (`pause_donations`)

Toggles donation acceptance for emergency situations.

**Function signature for pausing/unpausing donations:**
```rust
pub fn pause_donations(
    ctx: Context<PauseDonations>,
    paused: bool
) -> Result<()>
```

### 5. Update Charity (`update_charity`)

Allows charity authority to update description.

**Function signature for updating charity information:**
```rust
pub fn update_charity(
    ctx: Context<UpdateCharity>,
    description: String
) -> Result<()>
```

### 6. Delete Charity (`delete_charity`)

Removes charity and withdraws remaining funds.

**Function signature for deleting a charity:**
```rust
pub fn delete_charity(
    ctx: Context<DeleteCharity>
) -> Result<()>
```

## Account Architecture

### Charity Account

The main account storing charity information:

**Charity account structure with all fields and their sizes:**
```rust
#[account]
pub struct Charity {
    pub authority: Pubkey,              // 32 bytes - Charity owner
    pub name: String,                   // Variable - Charity name
    pub description: String,            // Variable - Description
    pub donations_in_lamports: u64,     // 8 bytes - Total donations
    pub donation_count: u32,            // 4 bytes - Number of donations
    pub paused: bool,                   // 1 byte - Donation status
    pub created_at: i64,                // 8 bytes - Creation timestamp
    pub updated_at: i64,                // 8 bytes - Last update
    pub deleted_at: Option<i64>,        // 9 bytes - Deletion timestamp
    pub withdrawn_at: Option<i64>,      // 9 bytes - Last withdrawal
    pub vault_bump: u8,                 // 1 byte - PDA bump seed
}
```

### Donation Account

Records individual donation transactions:

**Donation account structure tracking individual contributions:**
```rust
#[account]
pub struct Donation {
    pub donor_key: Pubkey,              // 32 bytes - Donor wallet
    pub charity_key: Pubkey,            // 32 bytes - Target charity
    pub charity_name: String,           // Variable - Charity name
    pub amount_in_lamports: u64,        // 8 bytes - Donation amount
    pub created_at: i64,                // 8 bytes - Donation timestamp
}
```

## Program Derived Addresses (PDAs)

### Charity PDA

Deterministic address for charity accounts:

**PDA derivation using authority and charity name as seeds:**
```rust
// Seeds: [b"charity", authority.key(), name.as_bytes()]
let (charity_pda, bump) = Pubkey::find_program_address(
    &[
        b"charity",
        authority.key().as_ref(),
        name.as_bytes()
    ],
    program_id
);
```

### Vault PDA

Secure storage for charity donations:

**Vault PDA derived from charity address for secure fund storage:**
```rust
// Seeds: [b"vault", charity_pda.key()]
let (vault_pda, bump) = Pubkey::find_program_address(
    &[
        b"vault",
        charity_pda.as_ref()
    ],
    program_id
);
```

### Donation PDA

Individual donation records:

**Donation PDA using donor, charity, and count for unique addresses:**
```rust
// Seeds: [b"donation", donor.key(), charity.key(), timestamp.to_le_bytes()]
let (donation_pda, bump) = Pubkey::find_program_address(
    &[
        b"donation",
        donor.key().as_ref(),
        charity.key().as_ref(),
        &charity.donation_count.to_le_bytes(),
    ],
    program_id
);
```

## Error Handling

### Custom Error Types

**Enum defining all possible program errors with descriptive messages:**
```rust
#[error_code]
pub enum CustomError {
    #[msg("Unauthorized: Only charity authority can perform this action")]
    Unauthorized,

    #[msg("Invalid name length: Name must be between 1 and 50 characters")]
    InvalidNameLength,

    #[msg("Donations are currently paused for this charity")]
    DonationsPaused,

    #[msg("Insufficient funds for this operation")]
    InsufficientFunds,
}
```

### Error Propagation

**Example of using require! macro for error checking:**
```rust
// Errors automatically bubble up through the call stack
pub fn donate_sol(ctx: Context<DonateSol>, amount: u64) -> Result<()> {
    require!(!charity.paused, CustomError::DonationsPaused);
    // ... rest of function
    Ok(())
}
```

## Event System

### Event Definitions

**Struct definitions for events emitted by the program:**
```rust
#[event]
pub struct CreateCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub description: String,
    pub created_at: i64,
}

#[event]
pub struct MakeDonationEvent {
    pub donor_key: Pubkey,
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub amount: u64,
    pub created_at: i64,
}
```

### Event Emission

**Example of emitting an event for off-chain indexing:**
```rust
// Emit events for off-chain indexing
emit!(CreateCharityEvent {
    charity_key: ctx.accounts.charity.key(),
    charity_name: ctx.accounts.charity.name.clone(),
    description: ctx.accounts.charity.description.clone(),
    created_at: current_time,
});
```

## Testing Architecture

### Test Structure

**Basic test module structure using Tokio async runtime:**
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use anchor_lang::prelude::*;
    use solana_program_test::*;

    #[tokio::test]
    async fn test_create_charity() {
        // Test implementation
    }
}
```

### Testing Patterns

- **Setup**: Initialize program and accounts
- **Execute**: Call program instruction
- **Verify**: Check account states and events
- **Cleanup**: Reset for next test

## Performance Optimizations

### Account Sizing

**Account initialization with precise space calculation:**
```rust
// Precise account sizing to minimize rent
#[account(
    init,
    payer = authority,
    space = 8 + Charity::INIT_SPACE,  // Discriminator + account data
    seeds = [b"charity", authority.key().as_ref(), name.as_bytes()],
    bump
)]
pub charity: Account<'info, Charity>,
```

### Efficient Operations

- **Bulk Updates**: Multiple state changes in single instruction
- **Optimized PDAs**: Minimal seeds for faster derivation
- **Direct Lamport Manipulation**: Avoids system program calls

## Integration Points

### TypeScript Client

Anchor automatically generates TypeScript client code:

**TypeScript client method call using generated program interface:**
```typescript
// Generated client usage
const tx = await program.methods
  .createCharity(name, description)
  .accounts({
    authority: wallet.publicKey,
  })
  .rpc();
```

### IDL (Interface Definition Language)

**JSON structure defining program interface for client generation:**
```json
{
  "instructions": [
    {
      "name": "createCharity",
      "accounts": [...],
      "args": [
        {"name": "name", "type": "string"},
        {"name": "description", "type": "string"}
      ]
    }
  ]
}
```

## Next Topics

Explore specific aspects of the smart contract:

- **[Program Structure](structure.md)** - Detailed code organization
- **[Instructions](instructions.md)** - Deep dive into each instruction
- **[State Management](state.md)** - Account lifecycle and relationships
- **[PDAs and Accounts](pdas.md)** - Address derivation and account management
- **[Testing](testing.md)** - Comprehensive testing strategies

The Anchor program demonstrates production-ready Solana development with proper security, error handling, and optimization techniques.
