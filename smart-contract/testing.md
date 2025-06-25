# Testing

Comprehensive testing is crucial for smart contract reliability and security. This section covers testing strategies, patterns, and implementation details for the charity dApp's Anchor program.

## Testing Architecture

### Test Framework Setup

```rust
// Cargo.toml - Test dependencies
[dev-dependencies]
anchor-client = "0.28.0"
solana-program-test = "1.16"
solana-sdk = "1.16"
tokio = { version = "1.0", features = ["macros", "rt-multi-thread"] }
spl-token = "4.0"
```

### Test Module Structure

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use anchor_client::solana_sdk::signature::{Keypair, Signer};
    use anchor_client::solana_sdk::system_program;
    use anchor_client::{Client, Cluster};
    use solana_program_test::*;
    use std::rc::Rc;

    // Test state and utilities
    struct TestContext {
        pub client: Client,
        pub program_id: Pubkey,
        pub payer: Rc<Keypair>,
    }

    impl TestContext {
        pub async fn new() -> Self {
            let program_test = ProgramTest::new(
                "charity",
                crate::ID,
                processor!(crate::entry),
            );
            
            let (mut banks_client, payer, recent_blockhash) = program_test.start().await;
            let client = Client::new_with_options(
                Cluster::Debug,
                Rc::clone(&payer),
                CommitmentConfig::processed(),
            );

            Self {
                client,
                program_id: crate::ID,
                payer,
            }
        }
    }
}
```

## Unit Tests

### Instruction Testing

```rust
#[tokio::test]
async fn test_create_charity() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = TestContext::new().await;
    let authority = Keypair::new();
    let name = "Test Charity".to_string();
    let description = "A test charity for unit testing".to_string();

    // Fund authority account
    ctx.client.request_airdrop(&authority.pubkey(), 1_000_000_000).await?;

    // Derive charity PDA
    let (charity_pda, _) = Pubkey::find_program_address(
        &[b"charity", authority.pubkey().as_ref(), name.as_bytes()],
        &ctx.program_id,
    );

    // Derive vault PDA
    let (vault_pda, _) = Pubkey::find_program_address(
        &[b"vault", charity_pda.as_ref()],
        &ctx.program_id,
    );

    // Execute create_charity instruction
    let tx = ctx.client
        .program(ctx.program_id)
        .request()
        .accounts(charity::accounts::CreateCharity {
            charity: charity_pda,
            vault: vault_pda,
            authority: authority.pubkey(),
            system_program: system_program::ID,
        })
        .args(charity::instruction::CreateCharity {
            name: name.clone(),
            description: description.clone(),
        })
        .signer(&authority)
        .send()
        .await?;

    // Verify charity account was created
    let charity_account: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity_pda)
        .await?;

    assert_eq!(charity_account.authority, authority.pubkey());
    assert_eq!(charity_account.name, name);
    assert_eq!(charity_account.description, description);
    assert_eq!(charity_account.donations_in_lamports, 0);
    assert_eq!(charity_account.donation_count, 0);
    assert!(!charity_account.paused);
    assert!(charity_account.deleted_at.is_none());

    Ok(())
}

#[tokio::test]
async fn test_donate_sol() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = TestContext::new().await;
    let authority = Keypair::new();
    let donor = Keypair::new();
    let donation_amount = 1_000_000_000; // 1 SOL

    // Setup charity first
    let charity_pda = setup_charity(&ctx, &authority, "Test Charity", "Test Description").await?;
    
    // Fund donor account
    ctx.client.request_airdrop(&donor.pubkey(), 2_000_000_000).await?;

    let (vault_pda, _) = Pubkey::find_program_address(
        &[b"vault", charity_pda.as_ref()],
        &ctx.program_id,
    );

    let timestamp = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)?
        .as_secs() as i64;

    let (donation_pda, _) = Pubkey::find_program_address(
        &[
            b"donation",
            donor.pubkey().as_ref(),
            charity_pda.as_ref(),
            timestamp.to_le_bytes().as_ref(),
        ],
        &ctx.program_id,
    );

    // Execute donation
    let tx = ctx.client
        .program(ctx.program_id)
        .request()
        .accounts(charity::accounts::DonateSol {
            charity: charity_pda,
            vault: vault_pda,
            donation: donation_pda,
            donor: donor.pubkey(),
            system_program: system_program::ID,
        })
        .args(charity::instruction::DonateSol {
            amount: donation_amount,
        })
        .signer(&donor)
        .send()
        .await?;

    // Verify charity state updated
    let charity_account: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity_pda)
        .await?;

    assert_eq!(charity_account.donations_in_lamports, donation_amount);
    assert_eq!(charity_account.donation_count, 1);

    // Verify donation record created
    let donation_account: charity::Donation = ctx.client
        .program(ctx.program_id)
        .account(donation_pda)
        .await?;

    assert_eq!(donation_account.donor_key, donor.pubkey());
    assert_eq!(donation_account.charity_key, charity_pda);
    assert_eq!(donation_account.amount_in_lamports, donation_amount);

    // Verify vault received funds
    let vault_account = ctx.client.get_account(&vault_pda).await?;
    assert!(vault_account.lamports >= donation_amount);

    Ok(())
}
```

### Error Condition Testing

```rust
#[tokio::test]
async fn test_donate_to_paused_charity() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = TestContext::new().await;
    let authority = Keypair::new();
    let donor = Keypair::new();

    // Setup and pause charity
    let charity_pda = setup_charity(&ctx, &authority, "Test Charity", "Test Description").await?;
    pause_charity(&ctx, &authority, charity_pda).await?;

    // Fund donor
    ctx.client.request_airdrop(&donor.pubkey(), 1_000_000_000).await?;

    let (vault_pda, _) = Pubkey::find_program_address(
        &[b"vault", charity_pda.as_ref()],
        &ctx.program_id,
    );

    let timestamp = get_current_timestamp();
    let (donation_pda, _) = Pubkey::find_program_address(
        &[
            b"donation",
            donor.pubkey().as_ref(),
            charity_pda.as_ref(),
            timestamp.to_le_bytes().as_ref(),
        ],
        &ctx.program_id,
    );

    // Attempt donation to paused charity
    let result = ctx.client
        .program(ctx.program_id)
        .request()
        .accounts(charity::accounts::DonateSol {
            charity: charity_pda,
            vault: vault_pda,
            donation: donation_pda,
            donor: donor.pubkey(),
            system_program: system_program::ID,
        })
        .args(charity::instruction::DonateSol { amount: 1_000_000 })
        .signer(&donor)
        .send()
        .await;

    // Verify error occurred
    assert!(result.is_err());
    let error = result.unwrap_err();
    assert!(error.to_string().contains("DonationsPaused"));

    Ok(())
}

#[tokio::test]
async fn test_unauthorized_withdrawal() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = TestContext::new().await;
    let authority = Keypair::new();
    let unauthorized_user = Keypair::new();

    // Setup charity with donations
    let charity_pda = setup_charity_with_donations(&ctx, &authority, 1_000_000_000).await?;

    let (vault_pda, _) = Pubkey::find_program_address(
        &[b"vault", charity_pda.as_ref()],
        &ctx.program_id,
    );

    // Fund unauthorized user
    ctx.client.request_airdrop(&unauthorized_user.pubkey(), 1_000_000_000).await?;

    // Attempt unauthorized withdrawal
    let result = ctx.client
        .program(ctx.program_id)
        .request()
        .accounts(charity::accounts::WithdrawDonations {
            charity: charity_pda,
            vault: vault_pda,
            recipient: unauthorized_user.pubkey(),
            authority: unauthorized_user.pubkey(),
            system_program: system_program::ID,
        })
        .args(charity::instruction::WithdrawDonations { amount: 500_000_000 })
        .signer(&unauthorized_user)
        .send()
        .await;

    // Verify authorization error
    assert!(result.is_err());
    let error = result.unwrap_err();
    assert!(error.to_string().contains("Unauthorized"));

    Ok(())
}
```

## Integration Tests

### End-to-End Workflow Testing

```rust
#[tokio::test]
async fn test_complete_charity_lifecycle() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = TestContext::new().await;
    let authority = Keypair::new();
    let donor1 = Keypair::new();
    let donor2 = Keypair::new();
    let recipient = Keypair::new();

    // Fund all accounts
    ctx.client.request_airdrop(&authority.pubkey(), 1_000_000_000).await?;
    ctx.client.request_airdrop(&donor1.pubkey(), 2_000_000_000).await?;
    ctx.client.request_airdrop(&donor2.pubkey(), 2_000_000_000).await?;

    // 1. Create charity
    let charity_name = "Disaster Relief Fund";
    let charity_description = "Emergency aid for natural disaster victims";
    let charity_pda = create_charity(
        &ctx,
        &authority,
        charity_name,
        charity_description,
    ).await?;

    // 2. Multiple donations
    let donation1_amount = 500_000_000; // 0.5 SOL
    let donation2_amount = 1_000_000_000; // 1 SOL
    
    donate_to_charity(&ctx, &donor1, charity_pda, donation1_amount).await?;
    donate_to_charity(&ctx, &donor2, charity_pda, donation2_amount).await?;

    // 3. Verify charity state
    let charity_account: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity_pda)
        .await?;

    assert_eq!(
        charity_account.donations_in_lamports,
        donation1_amount + donation2_amount
    );
    assert_eq!(charity_account.donation_count, 2);

    // 4. Partial withdrawal
    let withdrawal_amount = 750_000_000; // 0.75 SOL
    withdraw_donations(
        &ctx,
        &authority,
        charity_pda,
        recipient.pubkey(),
        withdrawal_amount,
    ).await?;

    // 5. Verify remaining balance
    let (vault_pda, _) = Pubkey::find_program_address(
        &[b"vault", charity_pda.as_ref()],
        &ctx.program_id,
    );
    
    let vault_account = ctx.client.get_account(&vault_pda).await?;
    let expected_remaining = donation1_amount + donation2_amount - withdrawal_amount;
    assert!(vault_account.lamports >= expected_remaining);

    // 6. Update charity description
    let new_description = "Emergency aid for natural disaster victims - Updated";
    update_charity(&ctx, &authority, charity_pda, new_description).await?;

    // 7. Pause donations temporarily
    pause_charity(&ctx, &authority, charity_pda).await?;

    // 8. Verify paused donation fails
    let result = donate_to_charity(&ctx, &donor1, charity_pda, 100_000_000).await;
    assert!(result.is_err());

    // 9. Unpause and make final donation
    unpause_charity(&ctx, &authority, charity_pda).await?;
    donate_to_charity(&ctx, &donor1, charity_pda, 250_000_000).await?;

    // 10. Final withdrawal and cleanup
    let final_charity: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity_pda)
        .await?;
    
    withdraw_donations(
        &ctx,
        &authority,
        charity_pda,
        recipient.pubkey(),
        final_charity.donations_in_lamports - withdrawal_amount,
    ).await?;

    Ok(())
}
```

### Multi-Charity Testing

```rust
#[tokio::test]
async fn test_multiple_charities_same_authority() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = TestContext::new().await;
    let authority = Keypair::new();
    let donor = Keypair::new();

    ctx.client.request_airdrop(&authority.pubkey(), 2_000_000_000).await?;
    ctx.client.request_airdrop(&donor.pubkey(), 3_000_000_000).await?;

    // Create multiple charities with same authority
    let charity1 = create_charity(
        &ctx,
        &authority,
        "Food Bank",
        "Local food assistance program",
    ).await?;

    let charity2 = create_charity(
        &ctx,
        &authority,
        "Education Fund",
        "Scholarships for underprivileged students",
    ).await?;

    let charity3 = create_charity(
        &ctx,
        &authority,
        "Medical Aid",
        "Healthcare support for remote communities",
    ).await?;

    // Donate to each charity
    donate_to_charity(&ctx, &donor, charity1, 500_000_000).await?;
    donate_to_charity(&ctx, &donor, charity2, 750_000_000).await?;
    donate_to_charity(&ctx, &donor, charity3, 1_000_000_000).await?;

    // Verify each charity received correct amount
    let charity1_account: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity1)
        .await?;
    let charity2_account: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity2)
        .await?;
    let charity3_account: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity3)
        .await?;

    assert_eq!(charity1_account.donations_in_lamports, 500_000_000);
    assert_eq!(charity2_account.donations_in_lamports, 750_000_000);
    assert_eq!(charity3_account.donations_in_lamports, 1_000_000_000);

    // Verify PDAs are unique
    assert_ne!(charity1, charity2);
    assert_ne!(charity2, charity3);
    assert_ne!(charity1, charity3);

    Ok(())
}
```

## Property-Based Testing

### Fuzzing Donation Amounts

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_donation_amount_properties(
        amount in 1u64..=u64::MAX,
        donor_balance in 1_000_000_000u64..=10_000_000_000u64
    ) {
        let rt = tokio::runtime::Runtime::new().unwrap();
        rt.block_on(async {
            let ctx = TestContext::new().await;
            let authority = Keypair::new();
            let donor = Keypair::new();

            // Setup
            ctx.client.request_airdrop(&authority.pubkey(), 1_000_000_000).await.unwrap();
            ctx.client.request_airdrop(&donor.pubkey(), donor_balance).await.unwrap();

            let charity_pda = setup_charity(
                &ctx,
                &authority,
                "Fuzz Test Charity",
                "Testing with random amounts"
            ).await.unwrap();

            // Property: Donation should succeed if amount <= donor balance
            let result = donate_to_charity(&ctx, &donor, charity_pda, amount).await;
            
            if amount <= donor_balance {
                prop_assert!(result.is_ok(), "Donation should succeed with sufficient balance");
                
                // Verify charity state updated correctly
                let charity: charity::Charity = ctx.client
                    .program(ctx.program_id)
                    .account(charity_pda)
                    .await
                    .unwrap();
                
                prop_assert_eq!(charity.donations_in_lamports, amount);
                prop_assert_eq!(charity.donation_count, 1);
            } else {
                prop_assert!(result.is_err(), "Donation should fail with insufficient balance");
            }
        });
    }
}
```

### PDA Generation Testing

```rust
proptest! {
    #[test]
    fn test_pda_uniqueness(
        authority_seed in any::<[u8; 32]>(),
        name in "[a-zA-Z0-9 ]{1,50}"
    ) {
        let authority = Pubkey::new_from_array(authority_seed);
        let program_id = crate::ID;

        let (charity_pda, charity_bump) = Pubkey::find_program_address(
            &[b"charity", authority.as_ref(), name.as_bytes()],
            &program_id,
        );

        let (vault_pda, vault_bump) = Pubkey::find_program_address(
            &[b"vault", charity_pda.as_ref()],
            &program_id,
        );

        // Properties
        prop_assert_ne!(charity_pda, vault_pda, "Charity and vault PDAs must be different");
        prop_assert!(charity_bump <= 255, "Bump seed must be valid");
        prop_assert!(vault_bump <= 255, "Vault bump seed must be valid");

        // Verify deterministic generation
        let (charity_pda2, charity_bump2) = Pubkey::find_program_address(
            &[b"charity", authority.as_ref(), name.as_bytes()],
            &program_id,
        );

        prop_assert_eq!(charity_pda, charity_pda2, "PDA generation must be deterministic");
        prop_assert_eq!(charity_bump, charity_bump2, "Bump generation must be deterministic");
    }
}
```

## Performance Testing

### Load Testing

```rust
#[tokio::test]
async fn test_high_volume_donations() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = TestContext::new().await;
    let authority = Keypair::new();
    let num_donors = 100;
    let donation_amount = 10_000_000; // 0.01 SOL per donation

    // Setup charity
    ctx.client.request_airdrop(&authority.pubkey(), 1_000_000_000).await?;
    let charity_pda = setup_charity(
        &ctx,
        &authority,
        "Load Test Charity",
        "Testing high volume donations",
    ).await?;

    // Create and fund multiple donor accounts
    let mut donors = Vec::new();
    for i in 0..num_donors {
        let donor = Keypair::new();
        ctx.client.request_airdrop(&donor.pubkey(), 100_000_000).await?;
        donors.push(donor);
    }

    // Execute donations concurrently
    let start_time = std::time::Instant::now();
    
    let donation_tasks: Vec<_> = donors.iter().enumerate().map(|(i, donor)| {
        let ctx = &ctx;
        let charity_pda = charity_pda;
        
        async move {
            donate_to_charity(ctx, donor, charity_pda, donation_amount).await
        }
    }).collect();

    let results = futures::future::join_all(donation_tasks).await;
    let duration = start_time.elapsed();

    // Analyze results
    let successful_donations = results.iter().filter(|r| r.is_ok()).count();
    let failed_donations = results.len() - successful_donations;

    println!("Load test results:");
    println!("  Total donations: {}", num_donors);
    println!("  Successful: {}", successful_donations);
    println!("  Failed: {}", failed_donations);
    println!("  Duration: {:?}", duration);
    println!("  TPS: {:.2}", num_donors as f64 / duration.as_secs_f64());

    // Verify final state
    let final_charity: charity::Charity = ctx.client
        .program(ctx.program_id)
        .account(charity_pda)
        .await?;

    assert_eq!(
        final_charity.donation_count as usize,
        successful_donations,
        "Donation count should match successful donations"
    );

    assert_eq!(
        final_charity.donations_in_lamports,
        (successful_donations as u64) * donation_amount,
        "Total amount should match successful donations * amount"
    );

    Ok(())
}
```

## Test Utilities

### Helper Functions

```rust
// Test utility functions
async fn setup_charity(
    ctx: &TestContext,
    authority: &Keypair,
    name: &str,
    description: &str,
) -> Result<Pubkey, Box<dyn std::error::Error>> {
    let (charity_pda, _) = Pubkey::find_program_address(
        &[b"charity", authority.pubkey().as_ref(), name.as_bytes()],
        &ctx.program_id,
    );

    let (vault_pda, _) = Pubkey::find_program_address(
        &[b"vault", charity_pda.as_ref()],
        &ctx.program_id,
    );

    ctx.client
        .program(ctx.program_id)
        .request()
        .accounts(charity::accounts::CreateCharity {
            charity: charity_pda,
            vault: vault_pda,
            authority: authority.pubkey(),
            system_program: system_program::ID,
        })
        .args(charity::instruction::CreateCharity {
            name: name.to_string(),
            description: description.to_string(),
        })
        .signer(authority)
        .send()
        .await?;

    Ok(charity_pda)
}

async fn donate_to_charity(
    ctx: &TestContext,
    donor: &Keypair,
    charity_pda: Pubkey,
    amount: u64,
) -> Result<(), Box<dyn std::error::Error>> {
    let (vault_pda, _) = Pubkey::find_program_address(
        &[b"vault", charity_pda.as_ref()],
        &ctx.program_id,
    );

    let timestamp = get_current_timestamp();
    let (donation_pda, _) = Pubkey::find_program_address(
        &[
            b"donation",
            donor.pubkey().as_ref(),
            charity_pda.as_ref(),
            timestamp.to_le_bytes().as_ref(),
        ],
        &ctx.program_id,
    );

    ctx.client
        .program(ctx.program_id)
        .request()
        .accounts(charity::accounts::DonateSol {
            charity: charity_pda,
            vault: vault_pda,
            donation: donation_pda,
            donor: donor.pubkey(),
            system_program: system_program::ID,
        })
        .args(charity::instruction::DonateSol { amount })
        .signer(donor)
        .send()
        .await?;

    Ok(())
}

fn get_current_timestamp() -> i64 {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs() as i64
}

// Test data builders
struct CharityBuilder {
    name: String,
    description: String,
    authority: Option<Keypair>,
}

impl CharityBuilder {
    pub fn new(name: &str) -> Self {
        Self {
            name: name.to_string(),
            description: format!("Test charity: {}", name),
            authority: None,
        }
    }

    pub fn description(mut self, description: &str) -> Self {
        self.description = description.to_string();
        self
    }

    pub fn authority(mut self, authority: Keypair) -> Self {
        self.authority = Some(authority);
        self
    }

    pub async fn build(self, ctx: &TestContext) -> Result<(Pubkey, Keypair), Box<dyn std::error::Error>> {
        let authority = self.authority.unwrap_or_else(|| Keypair::new());
        ctx.client.request_airdrop(&authority.pubkey(), 1_000_000_000).await?;
        
        let charity_pda = setup_charity(ctx, &authority, &self.name, &self.description).await?;
        Ok((charity_pda, authority))
    }
}
```

## Continuous Integration

### GitHub Actions Testing

```yaml
# .github/workflows/test.yml
name: Test Smart Contract

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Install Rust
      uses: actions-rs/toolchain@v1
      with:
        toolchain: stable
        profile: minimal
        override: true
    
    - name: Install Solana CLI
      run: |
        sh -c "$(curl -sSfL https://release.solana.com/v1.16.0/install)"
        echo "$HOME/.local/share/solana/install/active_release/bin" >> $GITHUB_PATH
    
    - name: Install Anchor
      run: |
        cargo install --git https://github.com/coral-xyz/anchor avm --locked --force
        avm install latest
        avm use latest
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/registry
          ~/.cargo/git
          target/
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Run tests
      run: |
        cd anchor
        anchor test
    
    - name: Run security audit
      run: cargo audit
```

This comprehensive testing approach ensures the charity dApp's smart contract is reliable, secure, and performs well under various conditions.