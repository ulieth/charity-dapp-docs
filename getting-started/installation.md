# Installation

Now that you have all the prerequisites installed, let's clone the project and set it up for development.

## Clone the Repository

```bash
# Clone the repository
git clone https://github.com/ulieth/charity_dapp.git
cd charity_dapp

# Or if you forked it
git clone https://github.com/YOUR_USERNAME/charity_dapp.git
cd charity_dapp
```

## Install Dependencies

### Install Node.js Dependencies

```bash
# Install all frontend dependencies
npm install
```

This will install all the required packages including:
- Next.js and React for the frontend
- Solana web3.js and wallet adapter
- Anchor client libraries
- UI components (Tailwind CSS, DaisyUI)
- Development tools (TypeScript, ESLint)

### Build the Anchor Program

```bash
# Build the Solana program
npm run anchor::build
```

This command:
1. Compiles the Rust program in `anchor/programs/charity/`
2. Generates TypeScript client code
3. Creates the program IDL (Interface Definition Language)

### Verify Installation

```bash
# Check that everything built successfully
ls -la anchor/target/deploy/
```

You should see:
- `charity.so` - The compiled program
- `charity-keypair.json` - Program keypair

## Configure Solana Environment

### Set Up Local Development

```bash
# Configure Solana for local development
solana config set --url localhost

# Set the keypair (if not already set)
solana config set --keypair ~/.config/solana/id.json

# Verify configuration
solana config get
```

Expected output:
```
Config File: ~/.config/solana/cli/config.yml
RPC URL: http://localhost:8899
WebSocket URL: ws://localhost:8900/
Keypair Path: ~/.config/solana/id.json
Commitment: confirmed
```

### Start Local Validator

Open a new terminal and start the Solana test validator:

```bash
# Start local Solana validator
npm run local-validator
```

Keep this terminal open - it will run the local blockchain for development.

### Deploy to Local Network

In your main terminal:

```bash
# Deploy the program to localnet
npm run anchor::deploy
```

This will:
1. Deploy the compiled program to your local validator
2. Update the program ID in the client code
3. Make the program available for testing

## Verify Deployment

### Check Program Deployment

```bash
# Verify the program is deployed
solana program show PROGRAM_ID
```

### Airdrop SOL for Testing

```bash
# Give yourself some SOL for testing
npm run airdrop

# Or manually airdrop
solana airdrop 10
```

### Run Tests

```bash
# Run the Anchor program tests
npm run anchor::test
```

All tests should pass, confirming your setup is correct.

## Project Structure Overview

After successful installation, your project structure should look like:

```
charity-dapp/
├── anchor/
│   ├── programs/charity/src/     # Rust smart contract source
│   ├── target/deploy/            # Compiled program files
│   ├── tests/                    # Program tests
│   └── Anchor.toml               # Anchor configuration
├── src/
│   ├── app/                     # Next.js app pages
│   ├── components/              # React components
│   └── lib/                     # Utility functions
├── scripts/                     # Build and utility scripts
├── node_modules/                # Node.js dependencies
└── package.json                 # Project configuration
```

## Troubleshooting

### Build Failures

If the Anchor build fails:

```bash
# Clean and rebuild
anchor clean
npm run anchor::build
```

### Deployment Issues

If deployment fails:

```bash
# Check validator is running
solana logs

# Restart validator if needed
# (Ctrl+C in validator terminal, then restart)
npm run local-validator
```

### Port Conflicts

If you get port conflicts:

```bash
# Kill processes on Solana ports
lsof -ti:8899 | xargs kill -9
lsof -ti:8900 | xargs kill -9

# Restart validator
npm run local-validator
```

### Program ID Mismatch

If you get program ID errors:

```bash
# Update program ID
npm run checkProgramId

# Redeploy if needed
npm run anchor::deploy
```

## Environment Variables

Create a `.env.local` file for local development:

```bash
# Copy example environment file
cp .env.example .env.local
```

Edit `.env.local`:
```
NEXT_PUBLIC_SOLANA_NETWORK=localhost
NEXT_PUBLIC_SOLANA_RPC_URL=http://localhost:8899
```

## Next Steps

Great! Your development environment is now set up. Continue to the [Quick Start](quick-start.md) guide to run the application and see it in action.
