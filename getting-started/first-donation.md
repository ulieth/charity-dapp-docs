# Your First Donation

This detailed walkthrough will guide you through making your first donation on the Solana Charity dApp, explaining each step and what happens behind the scenes.

## Before You Start

Ensure you have:
- ✅ The application running at `http://localhost:3000`
- ✅ Local Solana validator running
- ✅ Wallet connected with test SOL
- ✅ At least one charity created (or use existing ones)

If you need help with any of these, check the [Quick Start](quick-start.md) guide.

## Step-by-Step Donation Process

### Step 1: Navigate to Charities

1. Open `http://localhost:3000` in your browser
2. Click **"Charities"** in the navigation menu
3. You should see a list of available charities

**What you'll see:**
```
🏠 My First Charity
Description: Testing the charity platform
💰 Total Raised: 0 SOL | 👥 Donations: 0
[Donate] [View Details]
```

### Step 2: Select a Charity

Click **"Donate"** on any charity. This opens the donation modal.

**What happens:**
- The app checks if donations are paused (they shouldn't be for new charities)
- It fetches the current charity state from the blockchain
- The donation form is displayed

### Step 3: Enter Donation Amount

In the donation modal:

1. **Amount field**: Enter `0.1` (for 0.1 SOL)
2. **Review details**: 
   - Charity name and description
   - Your wallet address
   - Current SOL balance

**Behind the scenes:**
- The app validates you have sufficient balance
- It calculates transaction fees (~0.00025 SOL)
- Checks minimum donation amount (if any)

### Step 4: Submit the Donation

Click **"Donate"** button:

1. **Transaction preparation**: The app creates a donation transaction
2. **Wallet prompt**: Your wallet extension opens for approval
3. **Review transaction**: Check details in your wallet
4. **Approve**: Click "Approve" in your wallet

**Transaction details you'll see:**
```
Transaction Type: Program Interaction
Program: charity (YOUR_PROGRAM_ID)
Instruction: donate_sol
Amount: 0.1 SOL + ~0.00025 SOL (fees)
```

### Step 5: Transaction Confirmation

After approving:

1. **Submitting**: The transaction is sent to the blockchain
2. **Confirmation toast**: "Transaction submitted" appears
3. **Waiting**: The app waits for blockchain confirmation
4. **Success toast**: "Donation made successfully!" appears

**What happens on-chain:**
1. SOL transfers from your wallet to the charity's vault
2. Charity's donation count increases by 1
3. Charity's total raised amount increases by 0.1 SOL
4. A donation record is created with your details
5. An event is emitted for transaction logging

## Verifying Your Donation

### Check the Charity Page

Refresh the charities page. You should see updated information:

```
🏠 My First Charity
Description: Testing the charity platform
💰 Total Raised: 0.1 SOL | 👥 Donations: 1
[Donate] [View Details]
```

### View Donation Details

Click **"View Details"** on the charity to see:

- **Donation History**: List of all donations
- **Your Donation**: Shows your wallet address, amount, and timestamp
- **Vault Balance**: Current SOL held by the charity

### Check Your Account

Navigate to `/account` to see:

- **Your Balance**: Reduced by 0.1 + fees
- **Donation History**: Your donation listed
- **Transaction**: Link to view on Solana explorer (if available)

## Understanding the Transaction

### What Just Happened?

Your donation involved several blockchain operations:

1. **Balance Check**: Verified sufficient funds
2. **PDA Creation**: Found the charity's vault address
3. **Transfer**: Moved SOL from your wallet to the vault
4. **State Update**: Updated charity and donation records
5. **Event Emission**: Logged the donation for transparency

### The Smart Contract Code

Here's what executed in the Rust program:

```rust
pub fn donate_sol(ctx: Context<DonateSol>, amount: u64) -> Result<()> {
    // 1. Verify donations aren't paused
    require!(!charity.paused, CustomError::DonationsPaused);
    
    // 2. Transfer SOL to vault
    let transfer_instruction = system_instruction::transfer(
        donor.key, vault.key, amount
    );
    
    // 3. Update charity stats
    charity.donation_count += 1;
    charity.donations_in_lamports += amount;
    
    // 4. Record donation history
    donation.donor_key = donor.key();
    donation.amount_in_lamports = amount;
    
    // 5. Emit event
    emit!(MakeDonationEvent { ... });
}
```

## Try Different Scenarios

### Make Multiple Donations

Try making several donations to see:
- How donation count increases
- How total amount accumulates
- How donation history builds up

### Donate to Different Charities

If multiple charities exist:
- Donate to different ones
- Compare their stats
- Check your donation history

### Test Error Cases

Try these to understand error handling:

1. **Insufficient funds**: Try donating more SOL than you have
2. **Invalid amount**: Try donating 0 SOL
3. **Paused charity**: If you own a charity, pause it then try donating

## Donation Best Practices

### Amount Considerations

- **Minimum**: Most charities accept any positive amount
- **Fees**: Remember transaction fees (~0.00025 SOL)
- **Testing**: Use small amounts (0.01-0.1 SOL) for testing

### Security Notes

- **Verify charity**: Check the charity details before donating
- **Double-check amount**: Ensure you're donating the intended amount
- **Wallet security**: Only approve transactions you understand

## What's Next?

Now that you've made a donation, try:

1. **[Create your own charity](../smart-contract/instructions.md#create-charity)**
2. **[Explore charity management](../frontend/components.md#charity-management)**
3. **[Understand the architecture](../architecture/README.md)**
4. **[Learn about PDAs](../smart-contract/pdas.md)**

## Troubleshooting

### Transaction Failed

If your donation fails:

```bash
# Check your balance
solana balance

# View recent transactions
solana confirm -v TRANSACTION_SIGNATURE

# Check validator logs
solana logs
```

### Donation Not Showing

If the donation doesn't appear:

1. **Wait for confirmation**: Transactions take a few seconds
2. **Refresh the page**: Force reload the charity data
3. **Check network**: Ensure you're on the correct network (localhost)

### Wallet Issues

Common wallet problems:

- **Connection lost**: Reconnect your wallet
- **Wrong network**: Switch to localhost in wallet settings
- **Insufficient SOL**: Run `npm run airdrop` for more test SOL

Congratulations! You've successfully made your first donation and learned how the entire process works. 🎉

The next step is understanding how all the pieces fit together. Continue to the [Architecture Overview](../architecture/README.md) to learn about the system design.