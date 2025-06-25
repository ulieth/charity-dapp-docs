# PDAs and Accounts

Program Derived Addresses (PDAs) are fundamental to Solana program security and deterministic account generation. This section covers PDA derivation, account relationships, and management patterns used in the charity dApp.

## PDA Fundamentals

### What are PDAs?

Program Derived Addresses are deterministic addresses generated from:
- **Program ID**: The on-chain program's address
- **Seeds**: Byte arrays that make addresses unique
- **Bump Seed**: Ensures address is off the Ed25519 curve

```rust
// PDA derivation pattern
let (pda_address, bump_seed) = Pubkey::find_program_address(
    &[seed1, seed2, seed3],
    &program_id
);
```

### Why Use PDAs?

1. **Deterministic**: Same seeds always generate same address
2. **No Private Key**: Program controls the account, not users
3. **Cross-Program Invocation**: Programs can sign for PDA accounts
4. **Security**: Prevents address collision attacks

## Charity PDA

### Address Derivation

```rust
// Seeds: [b"charity", authority.key(), name.as_bytes()]
let (charity_pda, charity_bump) = Pubkey::find_program_address(
    &[
        b"charity",                    // Static seed
        authority.key().as_ref(),      // Authority pubkey (32 bytes)
        name.as_bytes()                // Charity name (variable length)
    ],
    program_id
);
```

### Account Context

```rust
#[derive(Accounts)]
#[instruction(name: String, description: String)]
pub struct CreateCharity<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Charity::INIT_SPACE,
        seeds = [b"charity", authority.key().as_ref(), name.as_bytes()],
        bump
    )]
    pub charity: Account<'info, Charity>,
    
    #[account(
        init,
        payer = authority,
        space = 0,
        seeds = [b"vault", charity.key().as_ref()],
        bump
    )]
    pub vault: SystemAccount<'info>,
    
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### Uniqueness Guarantees

```rust
// Each authority can create charities with unique names
// authority1 + "Water Relief" = different PDA than authority2 + "Water Relief"
// authority1 + "Water Relief" = different PDA than authority1 + "Food Bank"

pub fn validate_charity_uniqueness(authority: &Pubkey, name: &str, program_id: &Pubkey) -> Pubkey {
    let (charity_pda, _) = Pubkey::find_program_address(
        &[
            b"charity",
            authority.as_ref(),
            name.as_bytes()
        ],
        program_id
    );
    charity_pda
}
```

## Vault PDA

### Address Derivation

```rust
// Seeds: [b"vault", charity_pda.key()]
let (vault_pda, vault_bump) = Pubkey::find_program_address(
    &[
        b"vault",                      // Static seed
        charity_pda.as_ref()           // Charity account address (32 bytes)
    ],
    program_id
);
```

### Account Relationship

```rust
#[account]
pub struct Charity {
    pub authority: Pubkey,
    // ... other fields
    pub vault_bump: u8,               // Store vault bump for later use
}

// Access vault from charity
impl Charity {
    pub fn vault_pda(&self, program_id: &Pubkey) -> (Pubkey, u8) {
        Pubkey::find_program_address(
            &[b"vault", self.key().as_ref()],
            program_id
        )
    }
    
    pub fn vault_seeds(&self) -> [&[u8]; 3] {
        [
            b"vault",
            self.key().as_ref(),
            &[self.vault_bump]
        ]
    }
}
```

### Cross-Program Invocation

```rust
pub fn withdraw_donations(ctx: Context<WithdrawDonations>, amount: u64) -> Result<()> {
    let charity = &ctx.accounts.charity;
    
    // Create signer seeds using stored bump
    let vault_seeds = &[
        b"vault",
        charity.key().as_ref(),
        &[charity.vault_bump]
    ];
    let signer_seeds = &[&vault_seeds[..]];
    
    // Transfer with PDA as signer
    let ix = anchor_lang::system_program::transfer(
        &ctx.accounts.vault.to_account_info(),
        &ctx.accounts.recipient.to_account_info(),
        amount,
    );
    
    anchor_lang::program::invoke_signed(
        &ix,
        &[
            ctx.accounts.vault.to_account_info(),
            ctx.accounts.recipient.to_account_info(),
            ctx.accounts.system_program.to_account_info(),
        ],
        signer_seeds
    )?;
    
    Ok(())
}
```

## Donation PDA

### Address Derivation

```rust
// Seeds: [b"donation", donor.key(), charity.key(), timestamp.to_le_bytes()]
let timestamp = Clock::get()?.unix_timestamp;
let (donation_pda, donation_bump) = Pubkey::find_program_address(
    &[
        b"donation",                   // Static seed
        donor.key().as_ref(),          // Donor wallet (32 bytes)
        charity.key().as_ref(),        // Charity account (32 bytes)
        timestamp.to_le_bytes().as_ref() // Unix timestamp (8 bytes)
    ],
    program_id
);
```

### Temporal Uniqueness

```rust
#[derive(Accounts)]
#[instruction(amount: u64)]
pub struct DonateSol<'info> {
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
    pub charity: Account<'info, Charity>,
    
    #[account(
        mut,
        seeds = [b"vault", charity.key().as_ref()],
        bump = charity.vault_bump
    )]
    pub vault: SystemAccount<'info>,
    
    #[account(mut)]
    pub donor: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### Collision Prevention

```rust
// Multiple donations from same donor to same charity are differentiated by timestamp
// donor1 + charity1 + timestamp1 = unique donation
// donor1 + charity1 + timestamp2 = different unique donation

pub fn generate_donation_seeds(
    donor: &Pubkey,
    charity: &Pubkey,
    timestamp: i64
) -> [&[u8]; 4] {
    [
        b"donation",
        donor.as_ref(),
        charity.as_ref(),
        timestamp.to_le_bytes().as_ref()
    ]
}
```

## Account Size Management

### Static vs Dynamic Sizing

```rust
// Static size accounts (preferred for gas efficiency)
#[account]
pub struct Charity {
    pub authority: Pubkey,              // 32 bytes
    pub name: String,                   // 4 + max_len bytes
    pub description: String,            // 4 + max_len bytes
    // ... fixed-size fields
}

impl Space for Charity {
    const INIT_SPACE: usize = 
        32 +                    // authority
        4 + CHARITY_NAME_MAX_LEN +     // name
        4 + CHARITY_DESC_MAX_LEN +     // description
        8 +                     // donations_in_lamports
        4 +                     // donation_count
        1 +                     // paused
        8 +                     // created_at
        8 +                     // updated_at
        1 + 8 +                // deleted_at
        1 + 8 +                // withdrawn_at
        1;                     // vault_bump
}
```

### Dynamic Reallocation

```rust
// Reallocate account when content grows
#[derive(Accounts)]
#[instruction(new_description: String)]
pub struct UpdateCharityDescription<'info> {
    #[account(
        mut,
        realloc = 8 + Charity::space_for_description(&new_description),
        realloc::payer = authority,
        realloc::zero = false,
        has_one = authority,
        seeds = [b"charity", authority.key().as_ref(), charity.name.as_bytes()],
        bump
    )]
    pub charity: Account<'info, Charity>,
    
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

impl Charity {
    pub fn space_for_description(description: &str) -> usize {
        Self::INIT_SPACE - CHARITY_DESC_MAX_LEN + description.len()
    }
}
```

## Account Constraints

### Authority Validation

```rust
#[derive(Accounts)]
pub struct UpdateCharity<'info> {
    #[account(
        mut,
        has_one = authority @ CustomError::Unauthorized,
        seeds = [b"charity", authority.key().as_ref(), charity.name.as_bytes()],
        bump
    )]
    pub charity: Account<'info, Charity>,
    pub authority: Signer<'info>,
}
```

### State Validation

```rust
#[derive(Accounts)]
pub struct DonateSol<'info> {
    #[account(
        mut,
        constraint = !charity.paused @ CustomError::DonationsPaused,
        constraint = charity.deleted_at.is_none() @ CustomError::CharityDeleted,
        seeds = [b"charity", charity.authority.as_ref(), charity.name.as_bytes()],
        bump
    )]
    pub charity: Account<'info, Charity>,
    
    // Vault must belong to this charity
    #[account(
        mut,
        seeds = [b"vault", charity.key().as_ref()],
        bump = charity.vault_bump
    )]
    pub vault: SystemAccount<'info>,
}
```

### Cross-Account Validation

```rust
#[derive(Accounts)]
pub struct WithdrawDonations<'info> {
    #[account(
        mut,
        has_one = authority,
        seeds = [b"charity", authority.key().as_ref(), charity.name.as_bytes()],
        bump
    )]
    pub charity: Account<'info, Charity>,
    
    #[account(
        mut,
        seeds = [b"vault", charity.key().as_ref()],
        bump = charity.vault_bump
    )]
    pub vault: SystemAccount<'info>,
    
    /// CHECK: Can be any account to receive funds
    #[account(mut)]
    pub recipient: UncheckedAccount<'info>,
    
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

## Advanced PDA Patterns

### Hierarchical PDAs

```rust
// Organization -> Department -> Charity hierarchy
let (org_pda, _) = Pubkey::find_program_address(
    &[b"organization", org_authority.as_ref(), org_name.as_bytes()],
    program_id
);

let (dept_pda, _) = Pubkey::find_program_address(
    &[b"department", org_pda.as_ref(), dept_name.as_bytes()],
    program_id
);

let (charity_pda, _) = Pubkey::find_program_address(
    &[b"charity", dept_pda.as_ref(), charity_name.as_bytes()],
    program_id
);
```

### Counter-Based PDAs

```rust
// For scenarios requiring sequential numbering
#[account]
pub struct CharityCounter {
    pub authority: Pubkey,
    pub count: u64,
}

pub fn create_numbered_charity(ctx: Context<CreateNumberedCharity>) -> Result<()> {
    let counter = &mut ctx.accounts.counter;
    counter.count += 1;
    
    // Use counter in PDA derivation
    let expected_charity_pda = Pubkey::find_program_address(
        &[
            b"charity",
            counter.authority.as_ref(),
            counter.count.to_le_bytes().as_ref()
        ],
        ctx.program_id
    ).0;
    
    require_keys_eq!(
        ctx.accounts.charity.key(),
        expected_charity_pda,
        CustomError::InvalidPDA
    );
    
    Ok(())
}
```

### Multi-Seed Validation

```rust
// Validate multiple PDA relationships simultaneously
pub fn validate_pda_family(
    charity_key: &Pubkey,
    vault_key: &Pubkey,
    authority: &Pubkey,
    name: &str,
    program_id: &Pubkey
) -> Result<()> {
    // Validate charity PDA
    let (expected_charity, _) = Pubkey::find_program_address(
        &[b"charity", authority.as_ref(), name.as_bytes()],
        program_id
    );
    require_keys_eq!(*charity_key, expected_charity, CustomError::InvalidCharityPDA);
    
    // Validate vault PDA
    let (expected_vault, _) = Pubkey::find_program_address(
        &[b"vault", charity_key.as_ref()],
        program_id
    );
    require_keys_eq!(*vault_key, expected_vault, CustomError::InvalidVaultPDA);
    
    Ok(())
}
```

## Account Lifecycle

### Creation Flow

```rust
pub fn create_charity_with_vault(
    ctx: Context<CreateCharityWithVault>,
    name: String,
    description: String
) -> Result<()> {
    // 1. Initialize charity account
    let charity = &mut ctx.accounts.charity;
    charity.authority = ctx.accounts.authority.key();
    charity.name = name;
    charity.description = description;
    charity.vault_bump = ctx.bumps.vault;
    charity.created_at = Clock::get()?.unix_timestamp;
    
    // 2. Vault is automatically initialized as SystemAccount
    // 3. No explicit initialization needed for system accounts
    
    // 4. Emit creation event
    emit!(CreateCharityEvent {
        charity_key: charity.key(),
        vault_key: ctx.accounts.vault.key(),
        authority: charity.authority,
        name: charity.name.clone(),
    });
    
    Ok(())
}
```

### Cleanup Flow

```rust
pub fn close_charity(ctx: Context<CloseCharity>) -> Result<()> {
    let charity = &ctx.accounts.charity;
    let vault = &ctx.accounts.vault;
    
    // 1. Ensure vault is empty
    require_eq!(vault.lamports(), 0, CustomError::VaultNotEmpty);
    
    // 2. Transfer any remaining rent from vault to authority
    if vault.lamports() > 0 {
        **vault.try_borrow_mut_lamports()? = 0;
        **ctx.accounts.authority.try_borrow_mut_lamports()? += vault.lamports();
    }
    
    // 3. Close charity account (returns rent to authority)
    charity.close(ctx.accounts.authority.to_account_info())?;
    
    Ok(())
}
```

## Security Considerations

### Seed Injection Prevention

```rust
// BAD: User-controlled seeds can cause collision
let (pda, _) = Pubkey::find_program_address(
    &[user_input.as_bytes()], // DON'T DO THIS
    program_id
);

// GOOD: Mix static and validated dynamic seeds
let (pda, _) = Pubkey::find_program_address(
    &[
        b"charity",                    // Static prefix
        authority.as_ref(),            // Validated pubkey
        validated_name.as_bytes()      // Length-checked string
    ],
    program_id
);
```

### Bump Seed Storage

```rust
// Store bump seeds in account to avoid recomputation
#[account]
pub struct Charity {
    // ... other fields
    pub vault_bump: u8,               // Store for efficiency
}

// Use stored bump for CPI
let vault_seeds = [
    b"vault",
    charity.key().as_ref(),
    &[charity.vault_bump]           // Use stored bump
];
```

### PDA Ownership Verification

```rust
// Always verify PDA is owned by expected program
pub fn verify_pda_ownership(account: &AccountInfo, expected_owner: &Pubkey) -> Result<()> {
    require_keys_eq!(
        *account.owner,
        *expected_owner,
        CustomError::InvalidPDAOwner
    );
    Ok(())
}
```

PDAs provide the foundation for secure, deterministic account management in the charity dApp, enabling complex relationships while maintaining security and efficiency.