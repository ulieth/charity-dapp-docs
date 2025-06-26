# Security Model

The charity dApp implements security through access control constraints that ensure only authorized users can perform specific actions.

## Access Control Overview

When creating contexts with new state, the smart contract implements specific constraints to restrict who can perform certain actions. For example, only the charity owner can update charity details or withdraw donations.

## Authority-Based Access Control

```rust
// Only charity owners can update their charity
#[derive(Accounts)]
pub struct UpdateCharity<'info> {
    #[account(
        mut,
        has_one = authority @ CustomError::Unauthorized,
        seeds = [b"charity", authority.key().as_ref(), charity.name.as_bytes()],
        bump = charity.vault_bump
    )]
    pub charity: Account<'info, Charity>,

    pub authority: Signer<'info>,
}

pub fn update_charity(
    ctx: Context<UpdateCharity>,
    description: String,
) -> Result<()> {
    let charity = &mut ctx.accounts.charity;

    // Only the charity owner (authority) can update
    charity.description = description;
    charity.updated_at = Clock::get()?.unix_timestamp;

    Ok(())
}
```

## Withdrawal Constraints

```rust
// Only charity owners can withdraw donations
pub fn withdraw_donations(
    ctx: Context<WithdrawDonations>,
    amount: u64,
) -> Result<()> {
    let charity = &mut ctx.accounts.charity;
    let vault = &mut ctx.accounts.vault;

    // Authority constraint ensures only owner can withdraw
    **vault.to_account_info().try_borrow_mut_lamports()? -= amount;
    **ctx.accounts.authority.to_account_info().try_borrow_mut_lamports()? += amount;

    charity.withdrawn_at = Some(Clock::get()?.unix_timestamp);

    Ok(())
}
```

## Key Security Features

### Access Control Matrix

| Operation | Authority | Donor | Public |
|-----------|-----------|--------|---------|
| Create Charity | ✅ | ❌ | ❌ |
| Update Charity | ✅ (own) | ❌ | ❌ |
| Delete Charity | ✅ (own) | ❌ | ❌ |
| Pause Donations | ✅ (own) | ❌ | ❌ |
| Withdraw Funds | ✅ (own) | ❌ | ❌ |
| Donate | ✅ | ✅ | ✅ |
| View Charity | ✅ | ✅ | ✅ |

The system uses ownership verification, input validation, state protection, and transaction signing to ensure secure operations.

This access control system ensures that sensitive operations like updating charity information or withdrawing funds can only be performed by the legitimate charity owner.
