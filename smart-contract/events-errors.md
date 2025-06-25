# Events and Errors

The Anchor program implements comprehensive event emission and error handling to ensure transparency, debugging capability, and proper error communication. This section covers the event system and error management patterns.

## Why Events Matter in Blockchain

Events serve a fundamentally different purpose in blockchain applications compared to traditional software:

**Immutable Audit Trail**: Events provide a permanent, tamper-proof record of all significant actions that occurred in the program.

**Off-Chain Integration**: Events enable off-chain applications (websites, mobile apps, analytics tools) to track and respond to on-chain activity without constantly polling account states.

**Cost-Effective Monitoring**: Events are much cheaper to emit than storing data in accounts, making them ideal for logging detailed information.

**User Transparency**: Anyone can verify what happened by examining the event logs, providing complete transparency for charitable activities.

## Error Handling Philosophy

Smart contract error handling must be more rigorous than traditional applications because:

**Irreversible Transactions**: Once a transaction is committed to the blockchain, it cannot be undone. Errors must be caught before state changes occur.

**Financial Implications**: Errors in financial smart contracts can result in permanent loss of funds.

**Gas Costs**: Users pay transaction fees even for failed transactions, so clear error messages help users avoid costly mistakes.

**Security**: Proper error handling prevents attacks that might exploit edge cases or unexpected conditions.

## Event System

### Event Architecture

Events in Anchor provide a way to emit structured data that can be consumed by off-chain applications, indexers, and monitoring systems. All events are automatically logged to the blockchain transaction logs.

#### Event Design Principles

**Complete Coverage**: Every state-changing operation emits an event, ensuring no important actions go unrecorded.

**Structured Data**: Events use typed structures rather than free-form text, enabling reliable parsing and indexing.

**Self-Contained**: Each event contains all relevant information about the action, minimizing the need for additional lookups.

**Forward Compatibility**: Event structures are designed to be extensible while maintaining backward compatibility.

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

### Event Emission Benefits

**Real-Time Notifications**: Off-chain applications can listen for events to provide immediate feedback to users about transaction results.

**Analytics and Reporting**: Event data enables comprehensive analytics about charity performance, donation patterns, and system usage.

**Integration Points**: Events provide clean integration points for external systems like accounting software, reporting tools, and user interfaces.

**Debugging and Monitoring**: Events help developers and operators understand system behavior and diagnose issues.

## Core Events

Understanding each event type helps clarify what information is captured and why it's important for transparency and functionality.

### Charity Management Events

These events track the lifecycle of charity organizations from creation to deletion.

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

#### Event Analysis

**CreateCharityEvent**: Records the birth of a new charity organization with essential identification information.

**UpdateCharityEvent**: Tracks changes to charity information, maintaining an audit trail of modifications.

**DeleteCharityEvent**: Documents the permanent closure of a charity and how remaining funds were handled.

**PauseDonationsEvent**: Logs when charities are paused or unpaused, critical for understanding donation availability.

### Donation Events

These events track the flow of funds into and out of charity organizations.

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

#### Financial Event Analysis

**MakeDonationEvent**: Captures every donation with donor identity, recipient, amount, and timing. This creates a complete record of charitable giving.

**WithdrawCharitySolEvent**: Records when charities access their funds, including remaining balances for transparency about fund usage.

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

### Event Usage in Practice

Events enable several critical capabilities:

**Financial Transparency**: Donors can verify their donations reached the intended charity and track how funds are used.

**Regulatory Compliance**: Complete event logs provide audit trails for regulatory reporting and compliance verification.

**User Experience**: Real-time event monitoring enables responsive user interfaces that immediately reflect transaction results.

**Analytics**: Aggregated event data enables insights into donation patterns, charity performance, and system health.

## Error Handling

Error handling in the charity program is designed to prevent financial loss and provide clear feedback to users.

### Error Handling Strategy

**Prevention Over Recovery**: The program focuses on preventing errors rather than recovering from them, since blockchain transactions cannot be reversed.

**Clear Communication**: Error messages are written to be understandable by end users, helping them correct issues and retry transactions successfully.

**Security Focus**: Error conditions are designed to fail safely, ensuring that partial state changes cannot leave the system in an inconsistent state.

### Custom Error Types

Each error type addresses specific failure modes that users or attackers might encounter.

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

#### Error Categories Analysis

**Authorization Errors** (`Unauthorized`): Prevents unauthorized access to charity management functions.

**Validation Errors** (`InvalidNameLength`, `InvalidDescriptionLength`, `InvalidDescription`): Ensures data quality and prevents abuse.

**Operational Errors** (`DonationsPaused`, `InsufficientFunds`): Handles business logic constraints and operational limitations.

**Technical Errors** (`Overflow`, `InvalidVaultAccount`): Protects against technical failures and potential attacks.

**Financial Protection** (`InsufficientFundsForRent`): Prevents operations that would compromise account viability.

### Error Usage Patterns

The program implements consistent patterns for different types of validation and error handling.

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

### Validation Pattern Benefits

These validation patterns provide multiple benefits:

**User Protection**: Clear, early validation helps users correct mistakes before paying transaction fees.

**System Integrity**: Consistent validation prevents invalid data from entering the system.

**Attack Prevention**: Input validation prevents many common attack vectors that exploit edge cases.

**Cost Efficiency**: Early validation failures consume minimal compute units, reducing costs for failed transactions.

## Monitoring and Alerting

Event and error monitoring enables proactive system management and user support.

### Monitoring Strategy

**Real-Time Detection**: Monitor event streams for unusual patterns or high error rates that might indicate problems.

**Historical Analysis**: Analyze error patterns over time to identify systemic issues or user experience problems.

**User Support**: Error logs help support teams quickly diagnose and resolve user issues.

**Security Monitoring**: Unusual error patterns can indicate attempted attacks or system vulnerabilities.

### Critical Error Detection

Certain errors require immediate attention because they may indicate security issues or system problems.

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

Event data enables comprehensive system analytics and performance monitoring:

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

### Metrics and Analytics Benefits

**System Health**: Metrics help operators understand system usage, performance, and growth patterns.

**User Insights**: Analytics reveal user behavior patterns, helping improve the user experience.

**Financial Tracking**: Complete financial metrics enable accurate reporting and compliance monitoring.

**Performance Optimization**: Event analysis helps identify bottlenecks and optimization opportunities.

## Summary

The comprehensive event and error system ensures full transparency, proper error handling, and monitoring capabilities for the charity dApp's smart contract operations. This system provides:

- **Complete Transparency** through comprehensive event logging
- **Robust Error Handling** that protects users and system integrity  
- **Monitoring Capabilities** for system health and security
- **User-Friendly Feedback** through clear error messages
- **Analytics Foundation** for system optimization and insights

Together, these capabilities create a trustworthy, transparent, and maintainable charity platform that users can depend on for secure charitable giving.