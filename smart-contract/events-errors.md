# Events and Errors

The Anchor program implements comprehensive event emission and error handling to ensure transparency, debugging capability, and proper error communication. This section covers the event system and error management patterns.

## Event System

### Event Architecture

Events in Anchor provide a way to emit structured data that can be consumed by off-chain applications, indexers, and monitoring systems. All events are automatically logged to the blockchain transaction logs.

```rust
use anchor_lang::prelude::*;

// Event definitions
#[event]
pub struct CreateCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub description: String,
    pub authority: Pubkey,
    pub vault_key: Pubkey,
    pub created_at: i64,
}

#[event]
pub struct MakeDonationEvent {
    pub donor_key: Pubkey,
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub amount: u64,
    pub donation_key: Pubkey,
    pub created_at: i64,
}
```

### Event Emission

```rust
// Emit events within instruction handlers
pub fn create_charity(
    ctx: Context<CreateCharity>,
    name: String,
    description: String,
) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    let current_time = Clock::get()?.unix_timestamp;
    
    // Set charity data
    charity.authority = ctx.accounts.authority.key();
    charity.name = name.clone();
    charity.description = description.clone();
    charity.created_at = current_time;
    charity.vault_bump = ctx.bumps.vault;
    
    // Emit creation event
    emit!(CreateCharityEvent {
        charity_key: charity.key(),
        charity_name: name,
        description,
        authority: charity.authority,
        vault_key: ctx.accounts.vault.key(),
        created_at: current_time,
    });
    
    Ok(())
}
```

## Core Events

### Charity Management Events

```rust
#[event]
pub struct CreateCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub description: String,
    pub created_at: i64,
}

#[event]
pub struct UpdateCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub description: String,
    pub updated_at: i64,
}

#[event]
pub struct DeleteCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub deleted_at: i64,
    pub withdrawn_to_recipient: bool,
}

#[event]
pub struct PauseDonationsEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub paused: bool,
    pub updated_at: i64,
}
```

### Donation Events

```rust
#[event]
pub struct MakeDonationEvent {
    pub donor_key: Pubkey,
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub amount: u64,
    pub created_at: i64,
}

#[event]
pub struct WithdrawCharitySolEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub donations_in_lamports: u64,
    pub donation_count: u64,
    pub withdrawn_at: i64,
}
```

### System Events

```rust
#[event]
pub struct EmergencyPauseEvent {
    pub authority: Pubkey,
    pub reason: String,
    pub paused_at: i64,
}

#[event]
pub struct ProgramUpgradeEvent {
    pub old_program_id: Pubkey,
    pub new_program_id: Pubkey,
    pub upgrade_authority: Pubkey,
    pub upgraded_at: i64,
}
```

## Event Usage Patterns

### Event Emission with Error Handling

```rust
pub fn donate_sol(ctx: Context<DonateSol>, amount: u64) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    let donation = &mut ctx.accounts.donation;
    let current_time = Clock::get()?.unix_timestamp;
    
    // Validate state before processing
    require!(!charity.paused, CustomError::DonationsPaused);
    require!(charity.deleted_at.is_none(), CustomError::CharityDeleted);
    
    // Process donation
    let old_total = charity.donations_in_lamports;
    charity.donations_in_lamports = charity.donations_in_lamports
        .checked_add(amount)
        .ok_or(CustomError::OverflowError)?;
    charity.donation_count = charity.donation_count
        .checked_add(1)
        .ok_or(CustomError::OverflowError)?;
    
    // Transfer SOL
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
    
    // Initialize donation record
    donation.donor_key = ctx.accounts.donor.key();
    donation.charity_key = charity.key();
    donation.charity_name = charity.name.clone();
    donation.amount_in_lamports = amount;
    donation.created_at = current_time;
    
    // Emit success event
    emit!(MakeDonationEvent {
        donor_key: donation.donor_key,
        charity_key: donation.charity_key,
        charity_name: donation.charity_name.clone(),
        amount,
        donation_key: donation.key(),
        total_donations: charity.donations_in_lamports,
        donation_count: charity.donation_count,
        created_at: current_time,
    });
    
    Ok(())
}
```

### Conditional Event Emission

```rust
pub fn update_charity(
    ctx: Context<UpdateCharity>,
    description: String,
) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    let old_description = charity.description.clone();
    let current_time = Clock::get()?.unix_timestamp;
    
    // Only emit event if description actually changed
    if old_description != description {
        charity.description = description.clone();
        charity.updated_at = current_time;
        
        emit!(UpdateCharityEvent {
            charity_key: charity.key(),
            charity_name: charity.name.clone(),
            old_description,
            new_description: description,
            authority: charity.authority,
            updated_at: current_time,
        });
    }
    
    Ok(())
}
```

## Error Handling

### Custom Error Types

```rust
#[error_code]
pub enum CustomError {
    #[msg("Unauthorized access")]
    Unauthorized,
    
    #[msg("Invalid name length")]
    InvalidNameLength,
    
    #[msg("Invalid description length")]
    InvalidDescriptionLength,

    #[msg("Invalid description")]
    InvalidDescription,
    
    #[msg("Donations for this charity are paused")]
    DonationsPaused,
    
    #[msg("Charity has been deleted and cannot accept donations")]
    CharityDeleted,
    
    #[msg("Insufficient funds for withdrawal")]
    InsufficientFunds,
    
    #[msg("Cannot withdraw below rent-exemption threshold")]
    InsufficientFundsForRent,
    
    #[msg("Donation amount must be greater than 0")]
    InvalidDonationAmount,
    
    #[msg("Donation amount exceeds maximum allowed")]
    DonationAmountTooLarge,
    
    #[msg("Math overflow occurred")]
    Overflow,
    
    #[msg("PDA derivation failed")]
    InvalidPDA,
    
    #[msg("Invalid charity PDA")]
    InvalidCharityPDA,
    
    #[msg("Invalid vault account")]
    InvalidVaultAccount,
    
    #[msg("Invalid donation PDA")]
    InvalidDonationPDA,
    
    #[msg("Account is not owned by the expected program")]
    InvalidPDAOwner,
    
    #[msg("Vault must be empty before charity deletion")]
    VaultNotEmpty,
    
    #[msg("Charity not found")]
    CharityNotFound,
    
    #[msg("Donation not found")]
    DonationNotFound,
    
    #[msg("Invalid timestamp")]
    InvalidTimestamp,
    
    #[msg("Program is paused for maintenance")]
    ProgramPaused,
}
```

### Error Usage Patterns

#### Input Validation

```rust
pub fn validate_charity_input(name: &str, description: &str) -> Result<()> {
    // Name validation
    require!(
        !name.is_empty() && name.len() <= CHARITY_NAME_MAX_LEN,
        CustomError::InvalidNameLength
    );
    
    // Description validation
    require!(
        !description.is_empty() && description.len() <= CHARITY_DESC_MAX_LEN,
        CustomError::InvalidDescriptionLength
    );
    
    // Additional validation
    require!(
        !name.chars().all(|c| c.is_whitespace()),
        CustomError::InvalidNameLength
    );
    
    Ok(())
}
```

#### State Validation

```rust
pub fn validate_charity_state(charity: &Charity) -> Result<()> {
    require!(!charity.paused, CustomError::DonationsPaused);
    require!(charity.deleted_at.is_none(), CustomError::CharityDeleted);
    Ok(())
}

pub fn validate_donation_amount(amount: u64) -> Result<()> {
    require!(amount > 0, CustomError::InvalidDonationAmount);
    require!(amount <= MAX_DONATION_AMOUNT, CustomError::DonationAmountTooLarge);
    Ok(())
}
```

#### Arithmetic Safety

```rust
pub fn safe_add_donation(charity: &mut Charity, amount: u64) -> Result<()> {
    // Safe addition with overflow check
    charity.donations_in_lamports = charity.donations_in_lamports
        .checked_add(amount)
        .ok_or(CustomError::OverflowError)?;
    
    charity.donation_count = charity.donation_count
        .checked_add(1)
        .ok_or(CustomError::OverflowError)?;
    
    Ok(())
}

pub fn safe_withdraw_funds(vault_balance: u64, amount: u64, min_rent: u64) -> Result<()> {
    require!(vault_balance >= amount, CustomError::InsufficientFunds);
    
    let remaining = vault_balance
        .checked_sub(amount)
        .ok_or(CustomError::OverflowError)?;
    
    require!(remaining >= min_rent, CustomError::InsufficientFundsForRent);
    
    Ok(())
}
```

### Error Propagation

```rust
// Errors automatically bubble up through the call stack
pub fn create_charity(
    ctx: Context<CreateCharity>,
    name: String,
    description: String,
) -> Result<()> {
    // Input validation (can return error)
    validate_charity_input(&name, &description)?;
    
    // PDA validation (can return error)
    validate_pda_derivation(&ctx)?;
    
    // State initialization (can return error)
    initialize_charity_state(&mut ctx.accounts.charity, &name, &description)?;
    
    // Event emission (rarely fails)
    emit_charity_creation_event(&ctx.accounts.charity)?;
    
    Ok(())
}
```

## Event Indexing

### Event Parsing

```rust
// Off-chain event parsing
pub fn parse_charity_events(transaction_logs: &[String]) -> Vec<CharityEvent> {
    let mut events = Vec::new();
    
    for log in transaction_logs {
        if log.starts_with("Program data: ") {
            if let Ok(event) = parse_anchor_event::<CreateCharityEvent>(log) {
                events.push(CharityEvent::Created(event));
            } else if let Ok(event) = parse_anchor_event::<MakeDonationEvent>(log) {
                events.push(CharityEvent::Donation(event));
            }
            // ... other event types
        }
    }
    
    events
}

#[derive(Debug)]
pub enum CharityEvent {
    Created(CreateCharityEvent),
    Updated(UpdateCharityEvent),
    Deleted(DeleteCharityEvent),
    Donation(MakeDonationEvent),
    Withdrawal(WithdrawDonationsEvent),
    Paused(PauseDonationsEvent),
}
```

### Event Filtering

```rust
// Filter events by charity
pub fn filter_charity_events(events: &[CharityEvent], charity_key: &Pubkey) -> Vec<&CharityEvent> {
    events.iter()
        .filter(|event| match event {
            CharityEvent::Created(e) => e.charity_key == *charity_key,
            CharityEvent::Updated(e) => e.charity_key == *charity_key,
            CharityEvent::Deleted(e) => e.charity_key == *charity_key,
            CharityEvent::Donation(e) => e.charity_key == *charity_key,
            CharityEvent::Withdrawal(e) => e.charity_key == *charity_key,
            CharityEvent::Paused(e) => e.charity_key == *charity_key,
        })
        .collect()
}

// Filter events by donor
pub fn filter_donor_events(events: &[CharityEvent], donor_key: &Pubkey) -> Vec<&CharityEvent> {
    events.iter()
        .filter(|event| match event {
            CharityEvent::Donation(e) => e.donor_key == *donor_key,
            _ => false,
        })
        .collect()
}
```

## Error Recovery

### Graceful Error Handling

```rust
pub fn handle_donation_error(error: &CustomError) -> DonationResult {
    match error {
        CustomError::DonationsPaused => DonationResult::Paused {
            message: "Donations are temporarily paused for this charity".to_string(),
            retry_after: Some(3600), // Retry after 1 hour
        },
        CustomError::CharityDeleted => DonationResult::Deleted {
            message: "This charity is no longer accepting donations".to_string(),
            alternatives: suggest_similar_charities(),
        },
        CustomError::InsufficientFunds => DonationResult::InsufficientFunds {
            message: "Insufficient funds in wallet".to_string(),
            required_amount: 0, // Would be calculated
        },
        _ => DonationResult::UnknownError {
            message: "An unexpected error occurred".to_string(),
            error_code: format!("{:?}", error),
        },
    }
}

#[derive(Debug)]
pub enum DonationResult {
    Success { transaction_id: String },
    Paused { message: String, retry_after: Option<u64> },
    Deleted { message: String, alternatives: Vec<String> },
    InsufficientFunds { message: String, required_amount: u64 },
    UnknownError { message: String, error_code: String },
}
```

### Event-Based Error Tracking

```rust
#[event]
pub struct ErrorEvent {
    pub error_code: u32,
    pub error_message: String,
    pub instruction: String,
    pub authority: Option<Pubkey>,
    pub context: String,
    pub timestamp: i64,
}

// Emit error events for monitoring
pub fn emit_error_event(error: &CustomError, context: &str) -> Result<()> {
    emit!(ErrorEvent {
        error_code: error.error_code(),
        error_message: error.to_string(),
        instruction: context.to_string(),
        authority: None, // Could be set based on context
        context: context.to_string(),
        timestamp: Clock::get()?.unix_timestamp,
    });
    Ok(())
}
```

## Monitoring and Alerting

### Critical Error Detection

```rust
// Monitor for critical errors in off-chain systems
pub fn is_critical_error(error: &CustomError) -> bool {
    matches!(error,
        CustomError::OverflowError |
        CustomError::InvalidPDA |
        CustomError::InvalidPDAOwner |
        CustomError::ProgramPaused
    )
}

// Alert on high error rates
pub fn should_alert(error_count: u32, time_window: u64) -> bool {
    let error_rate = error_count as f64 / time_window as f64;
    error_rate > 0.1 // Alert if more than 10% error rate
}
```

### Event-Based Metrics

```rust
// Calculate metrics from events
pub fn calculate_charity_metrics(events: &[CharityEvent]) -> CharityMetrics {
    let mut metrics = CharityMetrics::default();
    
    for event in events {
        match event {
            CharityEvent::Created(_) => metrics.total_charities += 1,
            CharityEvent::Donation(e) => {
                metrics.total_donations += 1;
                metrics.total_amount += e.amount;
            },
            CharityEvent::Withdrawal(e) => {
                metrics.total_withdrawals += 1;
                metrics.total_withdrawn += e.amount;
            },
            // ... other events
        }
    }
    
    metrics
}

#[derive(Default)]
pub struct CharityMetrics {
    pub total_charities: u32,
    pub total_donations: u32,
    pub total_amount: u64,
    pub total_withdrawals: u32,
    pub total_withdrawn: u64,
}
```

The comprehensive event and error system ensures full transparency, proper error handling, and monitoring capabilities for the charity dApp's smart contract operations.