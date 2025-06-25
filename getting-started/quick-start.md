# Quick Start

Get the Solana Charity dApp running in just a few minutes! This guide assumes you've completed the [Prerequisites](prerequisites.md) and [Installation](installation.md) steps.

## Start the Application

### 1. Start Local Solana Validator

In your first terminal:

```bash
# Navigate to project directory
cd charity-dapp

# Start the local Solana validator
npm run local-validator
```

Keep this terminal open. You should see output like:
```
Ledger location: test-ledger
Log: test-ledger/validator.log
⠁ Initializing...
✅ RPC URL: http://localhost:8899
✅ WebSocket URL: ws://localhost:8900
```

### 2. Deploy the Program

In a second terminal:

```bash
# Navigate to project directory
cd charity-dapp

# Deploy the Anchor program
npm run anchor::deploy
```

You should see:
```
Deploying workspace: http://localhost:8899
Upgrade authority: YOUR_KEYPAIR_ADDRESS
Deploying program "charity"...
Program path: anchor/target/deploy/charity.so
✅ Program deployed successfully
Program Id: PROGRAM_ID_HERE
```

### 3. Start the Frontend

```bash
# Start the Next.js development server
npm run dev
```

The application will start at `http://localhost:3000`.

## First Look at the App

### 1. Open the Application

Navigate to `http://localhost:3000` in your browser. You should see:

- **Header**: Application title and wallet connection button
- **Navigation**: Links to different sections (Charities, Account, Clusters)
- **Main Content**: Welcome page with project information

### 2. Connect Your Wallet

Click the **"Connect Wallet"** button in the header:

- If you have Phantom or another wallet installed, select it
- For development, you can also use the "Local Keypair" option
- Approve the connection when prompted

### 3. Get Test SOL

If using a local keypair, get some SOL for testing:

```bash
# In your terminal
npm run airdrop
```

Your wallet balance should now show ~10 SOL.

## Create Your First Charity

### 1. Navigate to Charities

Click **"Charities"** in the navigation menu.

### 2. Create a New Charity

Click the **"Create Charity"** button and fill out:

- **Name**: "My First Charity"
- **Description**: "Testing the charity platform"

Click **"Create Charity"** and approve the transaction.

### 3. View Your Charity

After the transaction confirms, you should see your charity listed with:
- Name and description
- Current donation amount (0 SOL)
- Number of donations (0)
- Management buttons (Pause, Withdraw, Delete)

## Make Your First Donation

### 1. Find a Charity

Browse the charities list or use the one you just created.

### 2. Make a Donation

Click **"Donate"** on any charity:

- Enter an amount (e.g., 0.1 SOL)
- Click **"Donate"**
- Approve the transaction in your wallet

### 3. Verify the Donation

After confirmation, you should see:
- Updated donation amount on the charity
- Increased donation count
- Your donation in the transaction history

## Explore Features

### Charity Management (if you own a charity)

- **Pause Donations**: Temporarily stop accepting donations
- **Withdraw Funds**: Transfer donations to your wallet
- **Update Description**: Modify charity information
- **Delete Charity**: Remove the charity (withdraws remaining funds)

### Account Page

Visit `/account` to see:
- Your wallet address and SOL balance
- Charities you've created
- Donations you've made
- Transaction history

### Cluster Information

Visit `/clusters` to see:
- Current Solana network (localhost)
- RPC endpoint information
- Program deployment details

## Quick Test Scenario

Here's a complete test flow to verify everything works:

```bash
# 1. Start services (in separate terminals)
npm run local-validator
npm run anchor::deploy
npm run dev

# 2. Airdrop SOL
npm run airdrop

# 3. In browser at localhost:3000:
# - Connect wallet
# - Create a charity
# - Make a donation to your charity
# - Withdraw funds
# - Check account page for history
```

## Common Issues

### Wallet Connection Failed

- Ensure you have a wallet extension installed
- Try refreshing the page
- Check browser console for errors

### Transaction Failed

```bash
# Check validator logs
solana logs

# Verify wallet has SOL
solana balance

# Airdrop more if needed
npm run airdrop
```

### Program Not Found

```bash
# Redeploy the program
npm run anchor::deploy

# Check program is deployed
solana program show $(cat anchor/target/deploy/charity-keypair.json | jq -r '.[0:32] | @base64')
```

### Frontend Build Errors

```bash
# Clear Next.js cache
rm -rf .next

# Reinstall dependencies
rm -rf node_modules
npm install

# Restart dev server
npm run dev
```

## Development Workflow

For ongoing development:

1. **Make changes** to smart contract (`anchor/programs/charity/src/`)
2. **Rebuild**: `npm run anchor::build`
3. **Test changes**: `npm run anchor::test`
4. **Redeploy**: `npm run anchor::deploy`
5. **Frontend changes** auto-reload via hot reload

## Next Steps

Now that you have the app running:

- **[Make your first donation](first-donation.md)** - Detailed walkthrough
- **[Architecture Overview](../architecture/README.md)** - Understand how it works
- **[Smart Contract Deep Dive](../smart-contract/README.md)** - Explore the Rust code

Great job! You now have a fully functional charity dApp running locally. 🎉