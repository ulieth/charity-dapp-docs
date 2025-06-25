# Troubleshooting Guide

This comprehensive troubleshooting guide covers common issues you might encounter while developing, deploying, or using the Solana Charity dApp, along with their solutions.

## Development Environment Issues

### Solana CLI Issues

#### Command Not Found: `solana`

**Problem**: Terminal reports `command not found: solana`

**Solution**:
```bash
# Add Solana to PATH
export PATH="~/.local/share/solana/install/active_release/bin:$PATH"

# Add to shell profile for persistence
echo 'export PATH="~/.local/share/solana/install/active_release/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

#### Invalid Keypair Format

**Problem**: `Error: Invalid keypair file format`

**Solution**:
```bash
# Generate new keypair
solana-keygen new --outfile ~/.config/solana/id.json --force

# Verify keypair
solana-keygen verify ~/.config/solana/id.json
```

#### RPC Connection Failed

**Problem**: `Error: RPC request failed`

**Solution**:
```bash
# Check current configuration
solana config get

# Reset to working RPC endpoint
solana config set --url devnet

# Test connection
solana epoch-info
```

### Anchor Framework Issues

#### Anchor Command Not Found

**Problem**: `anchor: command not found`

**Solution**:
```bash
# Install Anchor via AVM
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force

# Install latest version
avm install latest
avm use latest

# Verify installation
anchor --version
```

#### Build Failed: Missing Dependencies

**Problem**: `error: failed to run custom build command for 'ring'`

**Solution**:
```bash
# Update Rust toolchain
rustup update

# Install build dependencies (Ubuntu/Debian)
sudo apt-get install pkg-config libudev-dev

# Install build dependencies (macOS)
xcode-select --install

# Clean and rebuild
anchor clean
anchor build
```

#### Program Size Too Large

**Problem**: `Error: Program size too large`

**Solution**:
```bash
# Optimize build for size
anchor build -- --features "no-entrypoint"

# Or edit Cargo.toml
[profile.release]
overflow-checks = true
lto = "fat"
codegen-units = 1
```

### Node.js and npm Issues

#### Node Version Incompatibility

**Problem**: `Error: Unsupported engine`

**Solution**:
```bash
# Check required Node version in package.json
cat package.json | grep engines

# Install correct Node version using nvm
nvm install 18
nvm use 18

# Verify version
node --version
```

#### Package Installation Failed

**Problem**: `npm ERR! peer dep missing`

**Solution**:
```bash
# Clear npm cache
npm cache clean --force

# Remove node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Or use legacy peer deps flag
npm install --legacy-peer-deps
```

## Smart Contract Issues

### Deployment Problems

#### Insufficient Funds for Deployment

**Problem**: `Error: Insufficient funds`

**Solution**:
```bash
# Check balance
solana balance

# Airdrop SOL (devnet only)
solana airdrop 5

# For mainnet, transfer SOL from another wallet
solana transfer RECIPIENT_ADDRESS AMOUNT --from SOURCE_KEYPAIR
```

#### Program Already Deployed

**Problem**: `Error: Program already deployed`

**Solution**:
```bash
# Use upgrade flag for redeployment
anchor upgrade target/deploy/charity.so --program-id PROGRAM_ID

# Or deploy with new program ID
anchor keys generate
anchor build
anchor deploy
```

#### Program ID Mismatch

**Problem**: Client code uses wrong program ID

**Solution**:
```bash
# Sync program IDs
anchor keys sync

# Rebuild client code
anchor build

# Update frontend environment variables
# NEXT_PUBLIC_PROGRAM_ID=NEW_PROGRAM_ID
```

### Transaction Failures

#### Transaction Simulation Failed

**Problem**: `Error: Transaction simulation failed`

**Solution**:
```bash
# Enable detailed logs
export RUST_LOG=debug
solana logs --url devnet

# Check account states
solana account ACCOUNT_ADDRESS --url devnet

# Verify program is deployed
solana program show PROGRAM_ID --url devnet
```

#### Account Not Found

**Problem**: `Error: Account not found`

**Solution**:
```typescript
// Check if account exists before using
const accountInfo = await connection.getAccountInfo(accountPublicKey);
if (!accountInfo) {
  throw new Error('Account not found');
}

// Or use try-catch for account fetch
try {
  const account = await program.account.charity.fetch(charityKey);
} catch (error) {
  console.log('Charity not found:', error);
}
```

#### Custom Error: Unauthorized

**Problem**: `Error: AnchorError caused by account: charity. Error Code: Unauthorized`

**Solution**:
```typescript
// Ensure wallet is connected and is the charity authority
const { publicKey } = useWallet();
if (!publicKey) {
  throw new Error('Wallet not connected');
}

// Verify authority matches
if (!charity.authority.equals(publicKey)) {
  throw new Error('Not authorized for this charity');
}
```

## Frontend Issues

### Wallet Connection Problems

#### Wallet Not Detected

**Problem**: Wallet adapter doesn't detect installed wallet

**Solution**:
```typescript
// Check if wallet is properly installed
const wallets = useMemo(() => [
  new PhantomWalletAdapter(),
  new SolflareWalletAdapter(),
], []);

// Add wallet detection
useEffect(() => {
  const phantom = window.solana;
  if (phantom?.isPhantom) {
    console.log('Phantom wallet detected');
  }
}, []);
```

#### Connection Timeout

**Problem**: Wallet connection times out

**Solution**:
```typescript
// Increase timeout in wallet adapter
const walletConfig = useMemo(() => ({
  autoConnect: true,
  timeout: 30000, // 30 seconds
}), []);

// Add retry logic
const connectWithRetry = async (maxRetries = 3) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await connect();
      break;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
};
```

#### Wrong Network

**Problem**: Wallet connected to wrong network

**Solution**:
```typescript
// Check and prompt network switch
const checkNetwork = async () => {
  const cluster = connection.rpcEndpoint;
  if (cluster.includes('mainnet') && process.env.NODE_ENV !== 'production') {
    alert('Please switch to devnet for development');
  }
};

// Add network detection
useEffect(() => {
  if (connected) {
    checkNetwork();
  }
}, [connected]);
```

### State Management Issues

#### Stale Data

**Problem**: UI shows outdated information

**Solution**:
```typescript
// Force refetch after mutations
const { mutate: createCharity } = useMutation({
  mutationFn: createCharityFn,
  onSuccess: () => {
    // Invalidate related queries
    queryClient.invalidateQueries(['charity']);
    queryClient.invalidateQueries(['myCharities']);
  },
});

// Use shorter stale time for real-time data
const charityQuery = useQuery({
  queryKey: ['charity', charityId],
  queryFn: fetchCharity,
  staleTime: 10000, // 10 seconds
  refetchInterval: 30000, // Refetch every 30 seconds
});
```

#### Memory Leaks

**Problem**: Component state updates after unmount

**Solution**:
```typescript
// Use cleanup in useEffect
useEffect(() => {
  let cancelled = false;
  
  const fetchData = async () => {
    const data = await getData();
    if (!cancelled) {
      setData(data);
    }
  };
  
  fetchData();
  
  return () => {
    cancelled = true;
  };
}, []);

// Use AbortController for fetch requests
useEffect(() => {
  const controller = new AbortController();
  
  fetch('/api/data', { signal: controller.signal })
    .then(response => response.json())
    .then(data => setData(data))
    .catch(error => {
      if (error.name !== 'AbortError') {
        console.error('Fetch error:', error);
      }
    });
    
  return () => controller.abort();
}, []);
```

### Build and Deployment Issues

#### Build Failed: Type Errors

**Problem**: TypeScript compilation errors

**Solution**:
```bash
# Check for type errors
npm run type-check

# Fix common issues
# 1. Update @types packages
npm update @types/node @types/react

# 2. Check tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true
  }
}
```

#### Bundle Size Too Large

**Problem**: JavaScript bundle exceeds size limits

**Solution**:
```javascript
// next.config.js - Add bundle analyzer
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

// Optimize imports
import { Button } from '@/components/ui/button'; // Good
import * as UI from '@/components/ui'; // Bad - imports everything

// Use dynamic imports for large components
const LargeComponent = dynamic(() => import('./LargeComponent'), {
  loading: () => <Loading />,
});
```

#### Environment Variables Not Found

**Problem**: `process.env.NEXT_PUBLIC_PROGRAM_ID is undefined`

**Solution**:
```bash
# Check environment file exists
ls -la .env*

# Verify variable names start with NEXT_PUBLIC_
# .env.local
NEXT_PUBLIC_PROGRAM_ID=9MipEJLetsngpXJuyCLsSu3qTJrHQ6E6W1rZ1GrG68am

# Restart development server
npm run dev
```

## Performance Issues

### Slow RPC Responses

**Problem**: Blockchain queries take too long

**Solution**:
```typescript
// Use faster RPC endpoints
const FAST_RPC_ENDPOINTS = {
  devnet: 'https://rpc.helius.xyz/?api-key=YOUR_KEY',
  mainnet: 'https://solana-api.projectserum.com',
};

// Implement caching
const connection = new Connection(endpoint, {
  commitment: 'confirmed',
  confirmTransactionInitialTimeout: 30000,
});

// Batch RPC calls
const [charity1, charity2] = await Promise.all([
  program.account.charity.fetch(charity1Key),
  program.account.charity.fetch(charity2Key),
]);
```

### High Memory Usage

**Problem**: Application consumes too much memory

**Solution**:
```typescript
// Implement proper cleanup
useEffect(() => {
  const subscription = connection.onAccountChange(
    accountKey,
    callback,
    'confirmed'
  );
  
  return () => {
    connection.removeAccountChangeListener(subscription);
  };
}, []);

// Limit query cache size
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      cacheTime: 300000, // 5 minutes
      staleTime: 60000,  // 1 minute
    },
  },
});
```

## Security Issues

### Private Key Exposure

**Problem**: Accidentally committing private keys

**Solution**:
```bash
# Remove from git history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch path/to/secret/file' \
  --prune-empty --tag-name-filter cat -- --all

# Add to .gitignore
echo "*.json" >> .gitignore
echo ".env*" >> .gitignore
echo "!.env.example" >> .gitignore

# Use environment variables
export PRIVATE_KEY="your-private-key"
```

### Transaction Replay Attacks

**Problem**: Transactions being replayed

**Solution**:
```rust
// Use recent blockhash
let recent_blockhash = ctx.accounts.recent_blockhash.recent_blockhash;

// Add nonce accounts for durable transactions
#[account(mut)]
pub nonce_account: Account<'info, Nonce>,
```

## Testing Issues

### Tests Failing Randomly

**Problem**: Tests pass sometimes but fail other times

**Solution**:
```typescript
// Use deterministic test data
const TEST_CHARITY_NAME = `test-charity-${Date.now()}`;

// Add proper cleanup
afterEach(async () => {
  // Clean up test accounts
  await cleanupTestAccounts();
});

// Use transaction confirmation
await provider.connection.confirmTransaction(signature, 'confirmed');
```

### Bankrun Issues

**Problem**: Bankrun tests not working

**Solution**:
```typescript
// Ensure proper bankrun setup
import { startAnchor } from "solana-bankrun";

const context = await startAnchor("", [
  { name: "charity", programId: PROGRAM_ID }
], []);

// Use correct program instance
const program = new Program(IDL, PROGRAM_ID, new BankrunProvider(context));
```

## Getting Help

### Log Analysis

```bash
# Enable detailed Solana logs
export RUST_LOG=solana_runtime::system_instruction_processor=trace,solana_runtime::message_processor=debug,solana_bpf_loader=debug,solana_rbpf=debug

# Monitor program logs
solana logs --url devnet PROGRAM_ID

# Check transaction details
solana confirm TRANSACTION_SIGNATURE --url devnet -v
```

### Debug Information Collection

When seeking help, provide:

1. **Environment details**:
   ```bash
   solana --version
   anchor --version
   node --version
   npm --version
   ```

2. **Error messages**: Full error text and stack traces

3. **Transaction signatures**: For failed transactions

4. **Account addresses**: For account-related issues

5. **Code snippets**: Minimal reproducible example

### Community Resources

- **Solana Discord**: [https://discord.gg/solana](https://discord.gg/solana)
- **Anchor Discord**: [https://discord.gg/ZCHmqvXgDw](https://discord.gg/ZCHmqvXgDw)
- **Solana Stack Exchange**: [https://solana.stackexchange.com](https://solana.stackexchange.com)
- **GitHub Issues**: [https://github.com/coral-xyz/anchor/issues](https://github.com/coral-xyz/anchor/issues)

Remember: Always search existing issues before creating new ones, and provide as much detail as possible when asking for help!