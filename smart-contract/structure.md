# Program Structure

This section provides a detailed overview of how the Anchor program is organized, including file structure, module organization, and code architecture patterns.

## Why Program Structure Matters

A well-organized smart contract structure is crucial for several reasons:

**Security**: Clear separation of concerns makes it easier to audit and verify that each component handles only its intended responsibilities, reducing attack surfaces.

**Maintainability**: Modular code organization allows developers to understand, modify, and extend specific functionality without affecting other parts of the system.

**Reusability**: Common utilities, error handling, and validation logic can be shared across different instructions, reducing code duplication and potential bugs.

**Clarity**: When dealing with financial transactions and user funds, every component's purpose must be immediately clear to prevent costly mistakes.

## Directory Structure

The program follows a clean modular structure that separates different concerns:

```
anchor/programs/charity/src/
├── lib.rs                    # Main program entry point and exports
├── instructions/             # All program instructions
│   ├── mod.rs               # Module exports
│   ├── create_charity.rs    # Create charity instruction
│   ├── update_charity.rs    # Update charity instruction
│   ├── donate_sol.rs        # Donation instruction
│   ├── withdraw_donations.rs # Withdrawal instruction
│   ├── pause_donations.rs   # Pause/unpause instruction
│   └── delete_charity.rs    # Delete charity instruction
├── state/                   # Account structures
│   ├── mod.rs              # State module exports
│   ├── charity.rs          # Charity account structure
│   └── donation.rs         # Donation record structure
└── common/                 # Shared utilities
    ├── mod.rs             # Common module exports
    ├── constants.rs       # Program constants
    ├── errors.rs          # Custom error definitions
    └── events.rs          # Event definitions
```

### Structure Rationale

**`lib.rs`** - Serves as the program's main entry point where all instructions are exposed. This centralized approach ensures all program functionality is accessible through a single interface, making it easier for clients to interact with the contract.

**`instructions/`** - Contains individual instruction handlers, each in its own file. This separation allows each instruction to be developed, tested, and audited independently, while the module system ensures clean imports and exports.

**`state/`** - Defines the data structures that persist on the blockchain. Keeping state definitions separate makes it clear what data the program manages and how it's structured, which is crucial for understanding storage costs and account relationships.

**`common/`** - Houses shared utilities like constants, errors, and events. This prevents code duplication and ensures consistent behavior across all instructions, particularly important for error handling and validation logic.

## Main Program Entry Point (`lib.rs`)

The main entry point consolidates all program functionality and handles the critical business logic for each instruction. Rather than delegating to separate handler functions, all instruction logic is implemented directly here for simplicity and clarity.

```rust
use anchor_lang::prelude::*;

// Program imports
pub mod instructions;
pub mod state;
pub mod common;

// Re-exports for cleaner imports
pub use instructions::*;
pub use state::*;
pub use common::*;

// Program ID
declare_id!("9MipEJLetsngpXJuyCLsSu3qTJrHQ6E6W1rZ1GrG68am");

#[program]
pub mod charity {
    use super::*;

    /// Create a new charity account
    pub fn create_charity(
        ctx: Context<CreateCharity>,
        name: String,
        description: String,
    ) -> Result<()> {
        // Validate input lengths
        require!(
            name.len() <= CHARITY_NAME_MAX_LEN,
            CustomError::InvalidNameLength
        );
        require!(
            description.len() <= CHARITY_DESCRIPTION_MAX_LEN,
            CustomError::InvalidDescriptionLength
        );

        let current_time = Clock::get()?.unix_timestamp;

        *ctx.accounts.charity = Charity {
            authority: ctx.accounts.authority.key(),
            name,
            description,
            donations_in_lamports: 0,
            donation_count: 0,
            paused: false,
            created_at: current_time,
            updated_at: current_time,
            deleted_at: None,
            withdrawn_at: None,
            vault_bump: ctx.bumps.vault,
        };

        emit!(CreateCharityEvent {
            charity_key: ctx.accounts.charity.key(),
            charity_name: ctx.accounts.charity.name.clone(),
            description: ctx.accounts.charity.description.clone(),
            created_at: current_time,
        });

        Ok(())
    }

    /// Update charity description
    pub fn update_charity(
        ctx: Context<UpdateCharity>,
        description: String,
    ) -> Result<()> {
        let charity = &mut ctx.accounts.charity;
        let current_time = Clock::get()?.unix_timestamp;

        require!(
            !description.eq(&charity.description),
            CustomError::InvalidDescription
        );

        require!(
            description.len() <= CHARITY_DESCRIPTION_MAX_LEN,
            CustomError::InvalidDescriptionLength
        );

        charity.description = description;
        charity.updated_at = current_time;

        emit!(UpdateCharityEvent {
            charity_key: charity.key(),
            charity_name: charity.name.clone(),
            description: charity.description.clone(),
            updated_at: current_time,
        });

        Ok(())
    }

    /// Donate SOL to a charity
    pub fn donate_sol(
        ctx: Context<DonateSol>,
        amount: u64,
    ) -> Result<()> {
        let donor = &mut ctx.accounts.donor;
        let charity = &mut ctx.accounts.charity;
        let vault = &mut ctx.accounts.vault;
        let donation = &mut ctx.accounts.donation;
        let current_time = Clock::get()?.unix_timestamp;

        // Check that donations are not paused
        require!(!charity.paused, CustomError::DonationsPaused);

        // Transfer SOL to vault
        let transfer_instruction = system_instruction::transfer(donor.key, vault.key, amount);
        anchor_lang::solana_program::program::invoke(
            &transfer_instruction,
            &[
                donor.to_account_info(),
                vault.to_account_info(),
                ctx.accounts.system_program.to_account_info(),
            ],
        )?;

        // Update charity state
        charity.donation_count = charity.donation_count.checked_add(1).ok_or(error!(CustomError::Overflow))?;
        charity.donations_in_lamports = charity.donations_in_lamports.checked_add(amount).ok_or(error!(CustomError::Overflow))?;

        // Record donation history
        donation.donor_key = donor.key();
        donation.charity_key = charity.key();
        donation.charity_name = charity.name.clone();
        donation.amount_in_lamports = amount;
        donation.created_at = current_time;

        emit!(MakeDonationEvent {
            donor_key: donor.key(),
            charity_key: charity.key(),
            charity_name: charity.name.clone(),
            amount,
            created_at: current_time,
        });

        Ok(())
    }

    /// Withdraw donations from charity vault
    pub fn withdraw_donations(
        ctx: Context<WithdrawDonations>,
        amount: u64,
    ) -> Result<()> {
        let recipient = &mut ctx.accounts.recipient;
        let charity = &mut ctx.accounts.charity;
        let vault = &mut ctx.accounts.vault;
        let current_time = Clock::get()?.unix_timestamp;

        // Ensure there are enough lamports to withdraw
        let vault_balance = **vault.lamports.borrow();
        require!(
            amount > 0 && vault_balance >= amount,
            CustomError::InsufficientFunds
        );

        let rent = Rent::get()?;
        let min_rent = rent.minimum_balance(0);

        // Ensure we maintain rent-exemption threshold
        require!(
            vault_balance.checked_sub(amount).unwrap_or(0) >= min_rent,
            CustomError::InsufficientFundsForRent
        );

        // Transfer lamports from vault to recipient
        **vault.lamports.borrow_mut() = vault_balance.checked_sub(amount).unwrap();
        **recipient.lamports.borrow_mut() = recipient
            .lamports()
            .checked_add(amount)
            .ok_or(error!(CustomError::Overflow))?;

        // Update charity state
        charity.donations_in_lamports = charity
            .donations_in_lamports
            .checked_sub(amount)
            .ok_or(error!(CustomError::Overflow))?;
        charity.withdrawn_at = Some(current_time);

        emit!(WithdrawCharitySolEvent {
            charity_key: charity.key(),
            charity_name: charity.name.clone(),
            donations_in_lamports: charity.donations_in_lamports,
            donation_count: charity.donation_count,
            withdrawn_at: current_time,
        });

        Ok(())
    }

    /// Pause or unpause donations for a charity
    pub fn pause_donations(
        ctx: Context<PauseDonations>,
        paused: bool,
    ) -> Result<()> {
        let charity = &mut ctx.accounts.charity;
        let current_time = Clock::get()?.unix_timestamp;

        charity.paused = paused;
        charity.updated_at = current_time;

        emit!(PauseDonationsEvent {
            charity_key: charity.key(),
            charity_name: charity.name.clone(),
            paused,
            updated_at: current_time,
        });

        Ok(())
    }

    /// Delete a charity and withdraw remaining funds
    pub fn delete_charity(
        ctx: Context<DeleteCharity>,
    ) -> Result<()> {
        let charity = &mut ctx.accounts.charity;
        let vault = &mut ctx.accounts.vault;
        let authority = &mut ctx.accounts.authority;
        let recipient = &mut ctx.accounts.recipient;
        let current_time = Clock::get()?.unix_timestamp;

        let vault_balance = **vault.lamports.borrow();
        let mut withdrawn_to_recipient = false;

        if vault_balance > 0 {
            if recipient.lamports() > 0 {
                // Transfer to recipient
                **vault.lamports.borrow_mut() = vault_balance.checked_sub(vault_balance).unwrap();
                **recipient.lamports.borrow_mut() = recipient
                    .lamports()
                    .checked_add(vault_balance)
                    .ok_or(error!(CustomError::Overflow))?;
                withdrawn_to_recipient = true;
            } else {
                // Transfer to authority
                **vault.lamports.borrow_mut() = vault_balance.checked_sub(vault_balance).unwrap();
                **authority.lamports.borrow_mut() = authority
                    .lamports()
                    .checked_add(vault_balance)
                    .ok_or(error!(CustomError::Overflow))?;
            }
        }

        emit!(DeleteCharityEvent {
            charity_key: charity.key(),
            charity_name: charity.name.clone(),
            deleted_at: current_time,
            withdrawn_to_recipient,
        });

        Ok(())
    }
}
```

### Key Design Decisions in lib.rs

**Direct Implementation**: Each instruction contains its complete logic rather than calling external handlers. This approach makes the code more transparent and easier to audit, as all critical operations are visible in one place.

**Input Validation**: Every instruction begins with thorough validation of inputs (name length, description length, amounts) to prevent invalid state and potential exploits.

**Safe Arithmetic**: All mathematical operations use checked arithmetic to prevent overflow attacks, which could manipulate donation amounts or counters.

**Event Emission**: Each state-changing operation emits events for transparency and off-chain monitoring, providing an audit trail of all contract activities.

**State Consistency**: Operations that modify multiple fields (like donation tracking) ensure all related data is updated atomically to maintain consistency.

## Instruction Organization

The instruction organization separates each operation into its own context definition and account validation rules.

### Instruction Module (`instructions/mod.rs`)

The instruction module defines the account contexts that each instruction requires. These contexts specify exactly which accounts each instruction needs, their required permissions, and validation constraints.

#### Why Separate Context Definitions?

**Security**: Each instruction declares its exact account requirements upfront, making it impossible for malicious actors to pass unexpected accounts or gain unauthorized access.

**Validation**: Account constraints are checked automatically by Anchor before instruction execution, preventing many common attack vectors.

**Clarity**: Developers can immediately understand what accounts and permissions each instruction needs without reading through the implementation.

```rust
// Public instruction handlers
pub mod create_charity;
pub mod update_charity;
pub mod donate_sol;
pub mod withdraw_donations;
pub mod pause_donations;
pub mod delete_charity;

// Re-export instruction contexts
pub use create_charity::*;
pub use update_charity::*;
pub use donate_sol::*;
pub use withdraw_donations::*;
pub use pause_donations::*;
pub use delete_charity::*;
```

### Example Instruction Structure (`instructions/create_charity.rs`)

This example demonstrates how instruction contexts define security boundaries and account relationships. The `CreateCharity` context shows several important patterns used throughout the program.

#### Understanding the CreateCharity Context

**PDA (Program Derived Address) Usage**: The charity account uses a PDA derived from the authority's public key and charity name. This ensures that each authority can only create one charity with a given name, preventing duplicate names and establishing clear ownership.

**Vault Separation**: The vault PDA is separate from the charity account. This design pattern separates data storage from fund storage, which is crucial for security - if the charity account somehow becomes corrupted, the funds remain safe in the vault.

**Initialization Safety**: Both accounts use `init` constraints, meaning they must not already exist. This prevents overwriting existing data and ensures clean account creation.

**Rent Considerations**: The `payer = authority` constraint ensures the charity creator pays for account rent, aligning costs with ownership.

```rust
use anchor_lang::prelude::*;
use crate::{state::Charity, common::*};

/// Account context for creating a charity
#[derive(Accounts)]
#[instruction(name: String)]
pub struct CreateCharity<'info> {
    /// Charity account to be created
    #[account(
        init,
        payer = authority,
        space = 8 + Charity::INIT_SPACE,
        seeds = [b"charity", authority.key().as_ref(), name.as_bytes()],
        bump
    )]
    pub charity: Account<'info, Charity>,

    /// Vault PDA for storing donations
    #[account(
        init,
        payer = authority,
        space = 0,
        seeds = [b"vault", charity.key().as_ref()],
        bump
    )]
    pub vault: SystemAccount<'info>,

    /// Authority (signer) creating the charity
    #[account(mut)]
    pub authority: Signer<'info>,

    /// System program for account creation
    pub system_program: Program<'info, System>,
}

/// Handler function for create_charity instruction
pub fn handler(
    ctx: Context<CreateCharity>,
    name: String,
    description: String,
) -> Result<()> {
    // Input validation
    require!(
        name.len() >= MIN_NAME_LEN && name.len() <= MAX_NAME_LEN,
        CustomError::InvalidNameLength
    );
    require!(
        description.len() >= MIN_DESCRIPTION_LEN && description.len() <= MAX_DESCRIPTION_LEN,
        CustomError::InvalidDescriptionLength
    );

    // Get current timestamp
    let current_time = Clock::get()?.unix_timestamp;

    // Initialize charity account
    let charity = &mut ctx.accounts.charity;
    charity.authority = ctx.accounts.authority.key();
    charity.name = name.clone();
    charity.description = description.clone();
    charity.donations_in_lamports = 0;
    charity.donation_count = 0;
    charity.paused = false;
    charity.created_at = current_time;
    charity.updated_at = current_time;
    charity.deleted_at = None;
    charity.withdrawn_at = None;
    charity.vault_bump = ctx.bumps.vault;

    // Emit creation event
    emit!(CreateCharityEvent {
        charity_key: charity.key(),
        charity_name: name,
        description,
        authority: ctx.accounts.authority.key(),
        created_at: current_time,
    });

    msg!("Charity '{}' created successfully", charity.name);
    Ok(())
}
```

#### The Vault Design Pattern

The vault pattern used in this instruction is worth understanding in detail:

**Why Not Store SOL in the Charity Account?**
- Mixing data and funds in one account creates complexity
- If the charity account needs to be modified (realloc), it could affect fund storage
- Separate concerns make the system more robust and easier to audit

**Security Benefits:**
- Vault is owned by the program, not by users
- Funds can only be moved through program instructions
- Clear separation between metadata and actual assets

**Gas Efficiency:**
- Vault account has zero data space, minimizing rent costs
- All metadata is stored efficiently in the charity account

## State Organization

State organization defines how data persists on the blockchain and the relationships between different account types.

### State Module (`state/mod.rs`)

The state module centralizes all account structure definitions, making it easy to understand what data the program manages and how it's organized.

#### State Design Philosophy

**Immutable History**: Donation records are never modified after creation, providing an immutable audit trail of all transactions.

**Efficient Storage**: Field ordering and data types are chosen to minimize account size and rent costs while maintaining necessary functionality.

**Clear Relationships**: Each account type has clear relationships to others (charity ↔ vault, charity ↔ donations) that are enforced at the program level.

```rust
pub mod charity;
pub mod donation;

// Re-export state structures
pub use charity::*;
pub use donation::*;
```

### Charity State Structure (`state/charity.rs`)

The charity account is the core data structure that maintains all metadata about a charity organization. Each field serves a specific purpose in the overall system design.

#### Field-by-Field Rationale

**authority**: Establishes ownership and control. Only this address can modify the charity or withdraw funds.

**name & description**: Limited to 30 and 100 characters respectively to balance expressiveness with storage costs.

**donations_in_lamports & donation_count**: Track financial metrics. Using lamports (smallest SOL unit) avoids floating-point precision issues.

**paused**: Allows charity owners to temporarily stop accepting donations during emergencies or maintenance.

**timestamps**: created_at, updated_at, deleted_at, withdrawn_at provide a complete audit trail of charity lifecycle events.

**vault_bump**: Stores the PDA bump seed for the associated vault, enabling efficient vault address derivation without additional computation.

```rust
use anchor_lang::prelude::*;

/// Main charity account structure
#[account]
pub struct Charity {
    /// Authority (owner) of the charity
    pub authority: Pubkey,              // 32 bytes
    
    /// Name of the charity (max 30 characters)
    pub name: String,                   // 4 + len bytes
    
    /// Description of the charity (max 100 characters)
    pub description: String,            // 4 + len bytes
    
    /// Total donations received in lamports
    pub donations_in_lamports: u64,     // 8 bytes
    
    /// Number of individual donations
    pub donation_count: u64,            // 8 bytes
    
    /// Whether donations are currently paused
    pub paused: bool,                   // 1 byte
    
    /// Timestamp when charity was created
    pub created_at: i64,                // 8 bytes
    
    /// Timestamp when charity was last updated
    pub updated_at: i64,                // 8 bytes
    
    /// Timestamp when charity was deleted (if applicable)
    pub deleted_at: Option<i64>,        // 9 bytes
    
    /// Timestamp of last withdrawal (if applicable)
    pub withdrawn_at: Option<i64>,      // 9 bytes
    
    /// Bump seed for vault PDA
    pub vault_bump: u8,                 // 1 byte
}

impl Space for Charity {
    const INIT_SPACE: usize = 
        32 +                    // authority
        4 + 30 +               // name (String with length prefix) - max 30 chars
        4 + 100 +              // description - max 100 chars
        8 +                     // donations_in_lamports
        8 +                     // donation_count (u64)
        1 +                     // paused
        8 +                     // created_at
        8 +                     // updated_at
        1 + 8 +                 // deleted_at (Option<i64>)
        1 + 8 +                 // withdrawn_at (Option<i64>)
        1;                      // vault_bump
}

impl Charity {
    /// Check if charity is active (not deleted and not paused)
    pub fn is_active(&self) -> bool {
        self.deleted_at.is_none() && !self.paused
    }
    
    /// Check if charity can accept donations
    pub fn can_accept_donations(&self) -> bool {
        self.is_active()
    }
    
    /// Get charity age in seconds
    pub fn age_seconds(&self) -> Result<i64> {
        let current_time = Clock::get()?.unix_timestamp;
        Ok(current_time - self.created_at)
    }
    
    /// Calculate average donation amount
    pub fn average_donation(&self) -> u64 {
        if self.donation_count == 0 {
            0
        } else {
            self.donations_in_lamports / self.donation_count
        }
    }
}
```

### Donation Record Structure (`state/donation.rs`)

```rust
use anchor_lang::prelude::*;

/// Individual donation record
#[account]
pub struct Donation {
    /// Public key of the donor
    pub donor_key: Pubkey,              // 32 bytes
    
    /// Public key of the charity receiving the donation
    pub charity_key: Pubkey,            // 32 bytes
    
    /// Name of the charity (for easy reference)
    pub charity_name: String,           // 4 + len bytes
    
    /// Donation amount in lamports
    pub amount_in_lamports: u64,        // 8 bytes
    
    /// Timestamp when donation was made
    pub created_at: i64,                // 8 bytes
}

impl Space for Donation {
    const INIT_SPACE: usize = 
        32 +                    // donor_key
        32 +                    // charity_key
        4 + 30 +               // charity_name (max 30 chars)
        8 +                     // amount_in_lamports
        8;                      // created_at
}

impl Donation {
    /// Convert lamports to SOL for display
    pub fn amount_in_sol(&self) -> f64 {
        self.amount_in_lamports as f64 / 1_000_000_000.0
    }
    
    /// Get donation age in seconds
    pub fn age_seconds(&self) -> Result<i64> {
        let current_time = Clock::get()?.unix_timestamp;
        Ok(current_time - self.created_at)
    }
}
```

### Space Optimization Strategy

The space calculation demonstrates careful optimization for Solana's rent model:
- Fixed-size fields (Pubkey, u64, i64, bool) have predictable costs
- Variable strings use length prefixes (4 bytes) plus maximum content
- Optional fields add 1 discriminator byte plus the wrapped type
- Total space is calculated upfront to ensure rent-exemption

This approach minimizes ongoing costs while ensuring accounts remain rent-exempt indefinitely.

### Helper Methods Rationale

The implementation includes several helper methods that encapsulate common business logic:

**is_active()**: Provides a single source of truth for determining if a charity can accept donations.

**average_donation()**: Calculates metrics safely, handling the division-by-zero case.

**age_seconds()**: Enables time-based queries and analytics without exposing raw timestamp calculations.

These methods abstract complex logic into simple, reusable functions that maintain consistency across the application.

## Common Utilities

The common utilities module centralizes shared functionality that multiple instructions need, preventing code duplication and ensuring consistent behavior across the program.

### Constants (`common/constants.rs`)

Constants define the operational parameters of the charity program. These values represent careful trade-offs between functionality and cost efficiency.

#### Rationale for Chosen Limits

**CHARITY_NAME_MAX_LEN: 30 characters** - Allows meaningful charity names while keeping storage costs reasonable. Most charity names fit comfortably within this limit.

**CHARITY_DESCRIPTION_MAX_LEN: 100 characters** - Provides enough space for a brief mission statement while preventing abuse through excessive storage usage.

These limits can be adjusted if needed, but changes would require program upgrades and migration of existing data.

```rust
/// Maximum length for charity names
pub const CHARITY_NAME_MAX_LEN: usize = 30;

/// Maximum length for charity descriptions
pub const CHARITY_DESCRIPTION_MAX_LEN: usize = 100;

/// Minimum length for charity descriptions
pub const MIN_DESCRIPTION_LEN: usize = 1;

/// Maximum donation amount (100 SOL)
pub const MAX_DONATION_AMOUNT: u64 = 100_000_000_000;

/// Minimum donation amount (0.001 SOL)
pub const MIN_DONATION_AMOUNT: u64 = 1_000_000;

/// Lamports per SOL
pub const LAMPORTS_PER_SOL: u64 = 1_000_000_000;

/// Seeds for PDA derivation
pub const CHARITY_SEED: &[u8] = b"charity";
pub const VAULT_SEED: &[u8] = b"vault";
pub const DONATION_SEED: &[u8] = b"donation";
```

### Error Definitions (`common/errors.rs`)

```rust
use anchor_lang::prelude::*;

/// Custom error codes for the charity program
#[error_code]
pub enum CustomError {
    #[msg("Unauthorized: Only charity authority can perform this action")]
    Unauthorized,
    
    #[msg("Invalid name length: Name must be between 1 and 50 characters")]
    InvalidNameLength,
    
    #[msg("Invalid description length: Description must be between 1 and 200 characters")]
    InvalidDescriptionLength,
    
    #[msg("Donations are currently paused for this charity")]
    DonationsPaused,
    
    #[msg("Invalid donation amount: Amount must be greater than 0")]
    InvalidAmount,
    
    #[msg("Donation amount exceeds maximum allowed")]
    ExcessiveDonation,
    
    #[msg("Donation amount below minimum required")]
    InsufficientDonation,
    
    #[msg("Insufficient funds for this operation")]
    InsufficientFunds,
    
    #[msg("Insufficient funds to maintain rent exemption")]
    InsufficientFundsForRent,
    
    #[msg("Withdrawal amount exceeds available balance")]
    ExcessiveWithdrawal,
    
    #[msg("Arithmetic overflow occurred")]
    Overflow,
    
    #[msg("Charity has been deleted")]
    CharityDeleted,
    
    #[msg("Charity not found")]
    CharityNotFound,
    
    #[msg("Invalid recipient address")]
    InvalidRecipient,
    
    #[msg("Operation would violate rent exemption")]
    RentViolation,
}
```

### Event Definitions (`common/events.rs`)

```rust
use anchor_lang::prelude::*;

/// Event emitted when a charity is created
#[event]
pub struct CreateCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub description: String,
    pub authority: Pubkey,
    pub created_at: i64,
}

/// Event emitted when a donation is made
#[event]
pub struct MakeDonationEvent {
    pub donor_key: Pubkey,
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub amount: u64,
    pub total_donations: u64,
    pub donation_count: u32,
    pub created_at: i64,
}

/// Event emitted when donations are withdrawn
#[event]
pub struct WithdrawDonationsEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub authority: Pubkey,
    pub recipient: Pubkey,
    pub amount: u64,
    pub remaining_balance: u64,
    pub withdrawn_at: i64,
}

/// Event emitted when charity details are updated
#[event]
pub struct UpdateCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub old_description: String,
    pub new_description: String,
    pub authority: Pubkey,
    pub updated_at: i64,
}

/// Event emitted when donations are paused/unpaused
#[event]
pub struct PauseDonationsEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub authority: Pubkey,
    pub paused: bool,
    pub timestamp: i64,
}

/// Event emitted when a charity is deleted
#[event]
pub struct DeleteCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub authority: Pubkey,
    pub final_balance: u64,
    pub total_donations_received: u64,
    pub total_donation_count: u32,
    pub deleted_at: i64,
}
```

### Error Handling Philosophy

The error definitions implement a comprehensive but focused approach to error handling:

**Specific Error Types**: Each error corresponds to a specific failure mode, making debugging and user communication clearer.

**Security-First**: Errors like `Unauthorized` and `InsufficientFunds` protect against common attack vectors.

**User-Friendly**: Error messages are written to be understandable by end users, not just developers.

**Fail-Fast**: The error system encourages early validation to catch problems before state changes occur.

### Event System Design

Events provide transparency and enable off-chain applications to track program activity:

**Complete Coverage**: Every state-changing operation emits an event, creating a complete audit trail.

**Structured Data**: Events include all relevant information needed for indexing and display in user interfaces.

**Immutable Log**: Once emitted, events cannot be modified, providing trustworthy historical records.

**Efficient Indexing**: Event structure is designed to enable efficient querying by various criteria (donor, charity, time period).

## Code Organization Patterns

The overall code organization follows several consistent patterns that enhance security, maintainability, and clarity.

### Handler Pattern

Each instruction follows a consistent pattern that ensures security and maintainability:

1. **Context Definition**: Account validation and constraints
2. **Handler Function**: Business logic implementation  
3. **Input Validation**: Parameter checking and sanitization
4. **State Updates**: Account data modifications
5. **Event Emission**: Audit trail and notification

#### Why This Pattern Works

**Predictability**: Developers know exactly where to find validation, business logic, and state changes in any instruction.

**Security**: The pattern enforces validation-first approach, preventing invalid operations from affecting state.

**Auditability**: Each step is clearly separated, making security audits more systematic and thorough.

**Debugging**: When issues arise, the structured pattern makes it easy to identify whether the problem is in validation, logic, or state management.

### Error Handling Pattern

The error handling pattern prioritizes safety and clarity through consistent validation and meaningful error messages.

#### Error Handling Philosophy

**Fail Early**: Validate all inputs and preconditions before making any state changes.

**Specific Errors**: Use descriptive error types that help users understand what went wrong.

**Safe Propagation**: Use the `?` operator to propagate errors without losing context.

```rust
// Consistent error handling throughout the program
pub fn example_handler(ctx: Context<ExampleContext>) -> Result<()> {
    // Validate preconditions
    require!(condition1, CustomError::SpecificError1);
    require!(condition2, CustomError::SpecificError2);
    
    // Perform operations with error propagation
    let result = risky_operation()?;
    
    // Update state
    ctx.accounts.account.field = result;
    
    Ok(())
}
```

### PDA Derivation Pattern

Program Derived Addresses (PDAs) create deterministic, unique addresses that the program controls. The consistent seed patterns ensure predictable address generation.

#### PDA Design Rationale

**Deterministic**: Same inputs always produce the same address, enabling clients to derive addresses independently.

**Unique**: The combination of program ID, authority, and name ensures no address collisions.

**Secure**: Only the program can sign for PDA addresses, preventing unauthorized access.

**Hierarchical**: The seed structure creates logical relationships (charity → vault, charity → donations).

```rust
// Consistent PDA seed patterns
pub fn derive_charity_pda(authority: &Pubkey, name: &str) -> (Pubkey, u8) {
    Pubkey::find_program_address(
        &[CHARITY_SEED, authority.as_ref(), name.as_bytes()],
        &crate::ID,
    )
}
```

## Summary

This structured approach ensures maintainable, secure, and efficient program development while following Anchor framework best practices. Each organizational choice serves specific purposes:

- **Security** through clear separation of concerns and validation
- **Efficiency** through optimized data structures and minimal rent costs  
- **Transparency** through comprehensive event emission and error handling
- **Maintainability** through consistent patterns and modular organization

The result is a robust charity platform that handles user funds safely while providing clear interfaces for interaction and monitoring.