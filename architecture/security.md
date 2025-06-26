# Security Model

The charity dApp implements security through access control constraints that ensure only authorized users can perform specific actions.

## Access Control Overview

When creating contexts with new state, the smart contract implements specific constraints to restrict who can perform certain actions. For example, only the charity owner can update charity details or withdraw donations.

## Security Features

### Access Control

```rust
// Only charity authority can manage charity
#[account(
    mut,
    has_one = authority
)]
pub charity: Account<'info, Charity>,
```

### Input Validation

```rust
// Validate string lengths
require!(
    name.len() <= CHARITY_NAME_MAX_LEN,
    CustomError::InvalidNameLength
);

// Validate amounts
require!(
    amount > 0 && vault_balance >= amount,
    CustomError::InsufficientFunds
);
```

### Rent Protection

```rust
// Ensure rent-exempt balance remains
let min_rent = rent.minimum_balance(0);
require!(
    vault_balance.checked_sub(amount).unwrap_or(0) >= min_rent,
    CustomError::InsufficientFundsForRent
);
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
