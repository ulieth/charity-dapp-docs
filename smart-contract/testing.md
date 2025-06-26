# Testing

The charity dApp uses comprehensive testing to ensure smart contract reliability and security.

## Testing Architecture

Tests are organized using Rust's testing framework with Anchor's testing utilities. The main test framework includes:

- **Test Framework**: `anchor-client`, `solana-program-test`, `tokio` for async testing
- **Test Context**: Centralized test environment setup with client, program ID, and payer keypair
- **Utilities**: Helper functions for common operations like charity creation and donations

## Test Categories

### Unit Tests

**Instruction Testing**
- `test_create_charity`: Verifies charity creation with proper PDA derivation and account initialization
- `test_donate_sol`: Tests SOL donations, vault transfers, and donation record creation
- Tests cover all main instructions: create, donate, withdraw, update, pause/unpause, delete

**Error Condition Testing**  
- `test_donate_to_paused_charity`: Ensures donations fail when charity is paused
- `test_unauthorized_withdrawal`: Verifies only charity authority can withdraw funds
- Tests cover validation errors, authorization failures, and edge cases

### Integration Tests

**End-to-End Workflow Testing**
- `test_complete_charity_lifecycle`: Full charity workflow from creation to deletion
- Tests multi-step processes: create → donate → withdraw → update → pause → unpause → final withdrawal

**Multi-Charity Testing**
- `test_multiple_charities_same_authority`: Verifies same authority can manage multiple charities
- Tests PDA uniqueness and isolated charity states

### Property-Based Testing

**Fuzzing Tests**
- `test_donation_amount_properties`: Tests donation behavior with random amounts and balances
- `test_pda_uniqueness`: Verifies PDA generation is deterministic and unique
- Uses `proptest` crate for property-based testing

### Performance Testing

**Load Testing**
- `test_high_volume_donations`: Tests concurrent donations from multiple donors
- Measures transaction throughput and success rates
- Verifies state consistency under load

## Test Organization

### Helper Functions
- `setup_charity()`: Creates test charity with proper funding
- `donate_to_charity()`: Executes donation with PDA derivation
- `CharityBuilder`: Builder pattern for test data creation

### Test Utilities
- Centralized timestamp generation
- Account funding helpers  
- Error assertion utilities
- Builder patterns for complex test scenarios

## Continuous Integration

Tests run automatically via GitHub Actions on push/PR with:
- Rust toolchain setup
- Solana CLI installation  
- Anchor framework setup
- Dependency caching
- Security audit with `cargo audit`

The testing approach ensures reliability through comprehensive coverage of happy paths, error conditions, edge cases, and performance scenarios.