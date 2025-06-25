# State Management

The Anchor program implements a sophisticated state management system that tracks charity information, donations, and program state across the Solana blockchain. This section covers account structures, lifecycle management, and state relationships.

## Understanding Blockchain State

Unlike traditional databases, blockchain state management has unique characteristics that shape how we design data structures:

**Immutability**: Once written to the blockchain, data cannot be modified arbitrarily. Changes must be explicit and go through program instructions.

**Cost Awareness**: Every byte stored on-chain incurs ongoing rent costs. Efficient data structures directly impact user costs.

**Concurrent Safety**: Multiple users may interact with the program simultaneously. State transitions must be atomic and safe.

**Transparency**: All state changes are publicly visible and auditable, making data integrity paramount.

## Design Philosophy

Our state management follows several key principles:

**Separation of Concerns**: Financial data (SOL) is stored separately from metadata to minimize complexity and risk.

**Audit Trail**: Critical events are timestamped and preserved, creating an immutable history of operations.

**Predictable Costs**: Account sizes are calculated upfront to ensure rent-exemption and predictable user costs.

**Security First**: State transitions are validated at every step to prevent invalid or malicious modifications.

## Account Overview

The program manages three primary account types, each serving a specific purpose in the charity ecosystem:

1. **Charity Account** - Core charity information and metadata
2. **Donation Account** - Individual donation transaction records  
3. **Vault Account** - SOL storage for charity funds

### Account Relationship Model

```
┌─────────────────┐    owns    ┌─────────────────┐
│   Authority     │─────────────│ Charity Account │
│   (Wallet)      │             │   (Metadata)    │
└─────────────────┘             └─────────────────┘
                                        │
                                   derives│
                                        ▼
                                ┌─────────────────┐
                                │  Vault Account  │
                                │   (SOL Funds)   │
                                └─────────────────┘
                                        ▲
                                receives│
                                        │
┌─────────────────┐   creates   ┌─────────────────┐
│    Donors       │─────────────│ Donation Records│
│   (Wallets)     │             │   (History)     │
└─────────────────┘             └─────────────────┘
```

This separation ensures that:
- Metadata and funds are isolated for security
- Donation history is preserved immutably  
- Each account type can be optimized for its specific purpose

## Charity Account Structure

The charity account serves as the central metadata store for each charity organization. It contains all the information needed to identify, manage, and track the charity without storing actual funds.

### Design Rationale

**Why Separate Metadata from Funds?**
- Reduces complexity when updating charity information
- Minimizes risk to funds during metadata operations  
- Allows for more efficient account space usage
- Enables atomic operations on either metadata or funds independently

**Field Selection Rationale:**
Each field in the charity account was chosen to serve specific operational or security needs while minimizing storage costs.

### Account Definition

```rust
#[account]
pub struct Charity {
    pub authority: Pubkey,              // 32 bytes - Charity owner/manager
    pub name: String,                   // Variable - Charity display name
    pub description: String,            // Variable - Charity description
    pub donations_in_lamports: u64,     // 8 bytes - Total donations received
    pub donation_count: u32,            // 4 bytes - Number of donations
    pub paused: bool,                   // 1 byte - Donation acceptance status
    pub created_at: i64,                // 8 bytes - Unix timestamp of creation
    pub updated_at: i64,                // 8 bytes - Last modification timestamp
    pub deleted_at: Option<i64>,        // 9 bytes - Deletion timestamp (if deleted)
    pub withdrawn_at: Option<i64>,      // 9 bytes - Last withdrawal timestamp
    pub vault_bump: u8,                 // 1 byte - PDA bump seed for vault
}
```

### Field-by-Field Analysis

**authority (32 bytes)**: The Solana public key that owns and controls this charity. This establishes clear ownership and authorization for all charity operations.

**name (4 + max 30 bytes)**: The charity's display name with a length prefix. Limited to 30 characters to balance expressiveness with storage efficiency.

**description (4 + max 100 bytes)**: A brief mission statement. 100 characters provide enough space for meaningful descriptions while controlling costs.

**donations_in_lamports (8 bytes)**: Total donations received in lamports (Solana's smallest unit). Using lamports avoids floating-point precision issues in financial calculations.

**donation_count (8 bytes)**: Number of individual donations received. Helps calculate average donation size and provides usage metrics.

**paused (1 byte)**: Emergency stop mechanism. Allows charity owners to temporarily halt donations during maintenance or emergencies.

**created_at (8 bytes)**: Unix timestamp of charity creation. Enables age calculations and temporal queries.

**updated_at (8 bytes)**: Unix timestamp of last modification. Tracks when charity information was last changed.

**deleted_at (9 bytes)**: Optional timestamp of deletion. Soft deletion preserves historical data while marking the charity as inactive.

**withdrawn_at (9 bytes)**: Optional timestamp of last withdrawal. Tracks when funds were last accessed by the charity.

**vault_bump (1 byte)**: PDA bump seed for the associated vault. Enables efficient vault address derivation without recomputation.

### Space Calculation

Understanding space calculation is crucial for rent optimization and cost predictability.

```rust
impl Space for Charity {
    const INIT_SPACE: usize = 
        32 +                    // authority
        4 + 50 +               // name (4 bytes length + max 50 chars)
        4 + 500 +              // description (4 bytes length + max 500 chars)
        8 +                    // donations_in_lamports
        4 +                    // donation_count
        1 +                    // paused
        8 +                    // created_at
        8 +                    // updated_at
        1 + 8 +                // deleted_at (Option<i64>)
        1 + 8 +                // withdrawn_at (Option<i64>)
        1;                     // vault_bump
}
```

### State Lifecycle

```rust
// Creation State
pub fn create_charity(ctx: Context<CreateCharity>, name: String, description: String) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    let current_time = Clock::get()?.unix_timestamp;
    
    charity.authority = ctx.accounts.authority.key();
    charity.name = name;
    charity.description = description;
    charity.donations_in_lamports = 0;
    charity.donation_count = 0;
    charity.paused = false;
    charity.created_at = current_time;
    charity.updated_at = current_time;
    charity.deleted_at = None;
    charity.withdrawn_at = None;
    charity.vault_bump = ctx.bumps.vault;
    
    Ok(())
}

// Update State
pub fn update_charity(ctx: Context<UpdateCharity>, description: String) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    charity.description = description;
    charity.updated_at = Clock::get()?.unix_timestamp;
    Ok(())
}

// Soft Delete State
pub fn delete_charity(ctx: Context<DeleteCharity>) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    charity.deleted_at = Some(Clock::get()?.unix_timestamp);
    charity.updated_at = Clock::get()?.unix_timestamp;
    Ok(())
}
```

## Donation Account Structure

### Account Definition

```rust
#[account]
pub struct Donation {
    pub donor_key: Pubkey,              // 32 bytes - Wallet that made donation
    pub charity_key: Pubkey,            // 32 bytes - Target charity account
    pub charity_name: String,           // Variable - Charity name at time of donation
    pub amount_in_lamports: u64,        // 8 bytes - Donation amount
    pub created_at: i64,                // 8 bytes - Donation timestamp
}
```

### Space Calculation

```rust
impl Space for Donation {
    const INIT_SPACE: usize = 
        32 +                    // donor_key
        32 +                    // charity_key
        4 + 50 +               // charity_name (4 bytes length + max 50 chars)
        8 +                    // amount_in_lamports
        8;                     // created_at
}
```

### Donation State Management

```rust
pub fn donate_sol(ctx: Context<DonateSol>, amount: u64) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    let donation = &mut ctx.accounts.donation;
    let current_time = Clock::get()?.unix_timestamp;
    
    // Update charity state
    charity.donations_in_lamports = charity.donations_in_lamports
        .checked_add(amount)
        .ok_or(CustomError::OverflowError)?;
    charity.donation_count = charity.donation_count
        .checked_add(1)
        .ok_or(CustomError::OverflowError)?;
    charity.updated_at = current_time;
    
    // Initialize donation record
    donation.donor_key = ctx.accounts.donor.key();
    donation.charity_key = charity.key();
    donation.charity_name = charity.name.clone();
    donation.amount_in_lamports = amount;
    donation.created_at = current_time;
    
    Ok(())
}
```

## Vault Account Management

### Vault Operations

```rust
// Vault is a system account (not Anchor account)
// SOL transfers directly modify lamport balance

pub fn donate_sol(ctx: Context<DonateSol>, amount: u64) -> Result<()> {
    // Transfer SOL from donor to vault
    let ix = anchor_lang::system_program::transfer(
        &ctx.accounts.donor.to_account_info(),
        &ctx.accounts.vault.to_account_info(),
        amount,
    );
    
    anchor_lang::program::invoke(
        &ix,
        &[
            ctx.accounts.donor.to_account_info(),
            ctx.accounts.vault.to_account_info(),
        ],
    )?;
    
    Ok(())
}

pub fn withdraw_donations(ctx: Context<WithdrawDonations>, amount: u64) -> Result<()> {
    let charity = &ctx.accounts.charity;
    let vault = &ctx.accounts.vault;
    
    // Validate withdrawal amount
    let vault_balance = vault.lamports();
    let min_rent = Rent::get()?.minimum_balance(0);
    
    require!(
        vault_balance >= amount,
        CustomError::InsufficientFunds
    );
    
    require!(
        vault_balance.checked_sub(amount).unwrap_or(0) >= min_rent,
        CustomError::InsufficientFundsForRent
    );
    
    // Transfer from vault to recipient
    **vault.try_borrow_mut_lamports()? -= amount;
    **ctx.accounts.recipient.try_borrow_mut_lamports()? += amount;
    
    // Update charity state
    let charity = &mut ctx.accounts.charity;
    charity.withdrawn_at = Some(Clock::get()?.unix_timestamp);
    charity.updated_at = Clock::get()?.unix_timestamp;
    
    Ok(())
}
```

## State Validation

### Input Validation

```rust
// String length validation
pub fn validate_charity_name(name: &str) -> Result<()> {
    require!(
        !name.is_empty() && name.len() <= CHARITY_NAME_MAX_LEN,
        CustomError::InvalidNameLength
    );
    Ok(())
}

// Amount validation
pub fn validate_donation_amount(amount: u64) -> Result<()> {
    require!(
        amount > 0 && amount <= MAX_DONATION_AMOUNT,
        CustomError::InvalidDonationAmount
    );
    Ok(())
}

// State validation
pub fn validate_charity_state(charity: &Charity) -> Result<()> {
    require!(!charity.paused, CustomError::DonationsPaused);
    require!(charity.deleted_at.is_none(), CustomError::CharityDeleted);
    Ok(())
}
```

### Access Control

```rust
// Authority validation
#[derive(Accounts)]
pub struct UpdateCharity<'info> {
    #[account(
        mut,
        has_one = authority @ CustomError::Unauthorized
    )]
    pub charity: Account<'info, Charity>,
    pub authority: Signer<'info>,
}

// Cross-account validation
#[derive(Accounts)]
#[instruction(amount: u64)]
pub struct DonateSol<'info> {
    #[account(
        mut,
        constraint = !charity.paused @ CustomError::DonationsPaused,
        constraint = charity.deleted_at.is_none() @ CustomError::CharityDeleted
    )]
    pub charity: Account<'info, Charity>,
    
    #[account(
        mut,
        seeds = [b"vault", charity.key().as_ref()],
        bump = charity.vault_bump,
    )]
    pub vault: SystemAccount<'info>,
    
    #[account(
        init,
        payer = donor,
        space = 8 + Donation::INIT_SPACE,
        seeds = [
            b"donation",
            donor.key().as_ref(),
            charity.key().as_ref(),
            &Clock::get()?.unix_timestamp.to_le_bytes()
        ],
        bump
    )]
    pub donation: Account<'info, Donation>,
    
    #[account(mut)]
    pub donor: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

## State Queries

### Account Filtering

```rust
// Get all charities by authority
let charities = program.account::<charity::Charity>().all()
    .await?
    .into_iter()
    .filter(|(_, charity)| charity.authority == authority_pubkey)
    .collect::<Vec<_>>();

// Get active charities (not deleted)
let active_charities = program.account::<charity::Charity>().all()
    .await?
    .into_iter()
    .filter(|(_, charity)| charity.deleted_at.is_none())
    .collect::<Vec<_>>();

// Get donations by donor
let donations = program.account::<donation::Donation>().all()
    .await?
    .into_iter()
    .filter(|(_, donation)| donation.donor_key == donor_pubkey)
    .collect::<Vec<_>>();
```

### State Aggregation

```rust
// Calculate charity statistics
impl Charity {
    pub fn average_donation_amount(&self) -> u64 {
        if self.donation_count == 0 {
            0
        } else {
            self.donations_in_lamports / self.donation_count as u64
        }
    }
    
    pub fn is_active(&self) -> bool {
        self.deleted_at.is_none() && !self.paused
    }
    
    pub fn days_since_creation(&self) -> i64 {
        let now = Clock::get().unwrap().unix_timestamp;
        (now - self.created_at) / 86400 // seconds per day
    }
}
```

## State Synchronization

### Event-Driven Updates

```rust
// State changes emit events for off-chain synchronization
pub fn update_charity(ctx: Context<UpdateCharity>, description: String) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    let old_description = charity.description.clone();
    
    charity.description = description.clone();
    charity.updated_at = Clock::get()?.unix_timestamp;
    
    emit!(UpdateCharityEvent {
        charity_key: charity.key(),
        old_description,
        new_description: description,
        updated_at: charity.updated_at,
    });
    
    Ok(())
}
```

### Batch Operations

```rust
// Multiple state updates in single transaction
pub fn batch_pause_charities(ctx: Context<BatchPauseCharities>, charity_keys: Vec<Pubkey>) -> Result<()> {
    for charity_key in charity_keys {
        let charity_account = ctx.remaining_accounts
            .iter()
            .find(|acc| acc.key() == charity_key)
            .ok_or(CustomError::CharityNotFound)?;
            
        let mut charity: Account<Charity> = Account::try_from(charity_account)?;
        charity.paused = true;
        charity.updated_at = Clock::get()?.unix_timestamp;
        charity.exit(&ctx.program_id)?;
    }
    Ok(())
}
```

## Memory Management

### Account Reallocation

```rust
// Reallocate account space when needed
#[derive(Accounts)]
pub struct UpdateCharityDescription<'info> {
    #[account(
        mut,
        realloc = 8 + Charity::space_with_description(&new_description),
        realloc::payer = authority,
        realloc::zero = false,
        has_one = authority
    )]
    pub charity: Account<'info, Charity>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

impl Charity {
    pub fn space_with_description(description: &str) -> usize {
        Self::INIT_SPACE - 504 + (4 + description.len()) // Adjust for new description length
    }
}
```

### Cleanup Operations

```rust
// Clean up expired donation records
pub fn cleanup_old_donations(ctx: Context<CleanupDonations>) -> Result<()> {
    let current_time = Clock::get()?.unix_timestamp;
    let cutoff_time = current_time - (365 * 24 * 60 * 60); // 1 year ago
    
    for donation_account in ctx.remaining_accounts.iter() {
        let donation: Account<Donation> = Account::try_from(donation_account)?;
        if donation.created_at < cutoff_time {
            // Close account and reclaim rent
            donation.close(ctx.accounts.rent_collector.to_account_info())?;
        }
    }
    Ok(())
}
```

The state management system ensures data integrity, proper access control, and efficient storage while maintaining full transparency through event emission and comprehensive validation.