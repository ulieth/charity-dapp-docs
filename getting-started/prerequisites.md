# Prerequisites

Before you can run the Solana Charity dApp, you'll need to install several tools and dependencies. This page will guide you through setting up your development environment.

## Required Tools

### 1. Node.js and npm

The frontend is built with Next.js, which requires Node.js.

```bash
# Install Node.js (version 18 or higher recommended)
# Download from: https://nodejs.org/

# Verify installation
node --version
npm --version
```

### 2. Rust

The smart contract is written in Rust using the Anchor framework.

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Configure PATH
source ~/.cargo/env

# Verify installation
rustc --version
cargo --version
```

### 3. Solana CLI

The Solana CLI tools are essential for blockchain development.

```bash
# Install Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/v1.18.22/install)"

# Add to PATH (add to your shell profile)
export PATH="~/.local/share/solana/install/active_release/bin:$PATH"

# Verify installation
solana --version
```

### 4. Anchor Framework

Anchor simplifies Solana program development.

```bash
# Install Anchor using AVM (Anchor Version Manager)
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force

# Install Anchor version 0.30.1 (used by this project)
avm install 0.30.1
avm use 0.30.1

# Verify installation
anchor --version
```

## Solana Wallet Setup

### 5. Create a Local Wallet

You'll need a Solana wallet for testing.

```bash
# Generate a new keypair
solana-keygen new --outfile ~/.config/solana/id.json

# Set as default keypair
solana config set --keypair ~/.config/solana/id.json

# Verify configuration
solana config get
```

### 6. Browser Wallet (Optional)

For the full user experience, install a browser wallet:

- **[Phantom](https://phantom.app/)** - Popular and user-friendly
- **[Solflare](https://solflare.com/)** - Feature-rich wallet
- **[Backpack](https://backpack.app/)** - Modern wallet with great UX

## Next Steps

Once all prerequisites are installed, continue to the [Installation](installation.md) guide to clone and set up the project.
