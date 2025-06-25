# Deployment Guide

This comprehensive guide covers deploying the Solana Charity dApp from local development to production environments. Whether you're deploying to Solana's testnet for testing or mainnet for production, this guide has you covered.

## Deployment Overview

The Solana Charity dApp consists of two main components that require deployment:

1. **Smart Contract (Anchor Program)**: Deployed to Solana blockchain
2. **Frontend Application**: Deployed to web hosting platform

## Smart Contract Deployment

### Testnet Deployment

#### 1. Configure for Devnet

```bash
# Set Solana CLI to devnet
solana config set --url devnet
solana config set --keypair ~/.config/solana/id.json

# Verify configuration
solana config get
```

#### 2. Ensure Sufficient Balance

```bash
# Check balance
solana balance

# Airdrop SOL if needed (devnet only)
solana airdrop 5

# Verify you have enough for deployment (~2-3 SOL recommended)
```

#### 3. Build and Deploy Program

```bash
# Build the program
npm run anchor::build

# Deploy to devnet
npm run anchor::deploy
```

#### 4. Verify Deployment

```bash
# Check program account
solana program show $(cat anchor/target/deploy/charity-keypair.json | jq -r '.[0:32] | map(tostring) | join("")' | xxd -r -p | base58)

# Verify program is executable
anchor test --skip-local-validator
```

### Mainnet Deployment

⚠️ **Warning**: Mainnet deployment costs real SOL and deploys to production. Ensure thorough testing on devnet first.

#### 1. Configure for Mainnet

```bash
# Set Solana CLI to mainnet-beta
solana config set --url mainnet-beta

# Use a mainnet keypair with sufficient balance
solana config set --keypair ~/.config/solana/mainnet-keypair.json
```

#### 2. Security Checklist

Before mainnet deployment:

- [ ] All tests pass on devnet
- [ ] Security audit completed
- [ ] Code review completed
- [ ] Error handling tested
- [ ] Emergency procedures documented

#### 3. Deploy to Mainnet

```bash
# Final build
npm run anchor::build

# Deploy (this costs real SOL)
anchor deploy --provider.cluster mainnet-beta

# Immediately verify deployment
solana program show PROGRAM_ID
```

#### 4. Post-Deployment Verification

```bash
# Test basic functionality
anchor test --provider.cluster mainnet-beta --skip-deploy

# Monitor initial transactions
solana logs --url mainnet-beta
```

## Frontend Deployment

### Environment Configuration

#### Development Environment

```bash
# .env.local
NEXT_PUBLIC_SOLANA_NETWORK=localhost
NEXT_PUBLIC_SOLANA_RPC_URL=http://localhost:8899
NEXT_PUBLIC_PROGRAM_ID=YOUR_LOCAL_PROGRAM_ID
```

#### Testnet Environment

```bash
# .env.staging
NEXT_PUBLIC_SOLANA_NETWORK=devnet
NEXT_PUBLIC_SOLANA_RPC_URL=https://api.devnet.solana.com
NEXT_PUBLIC_PROGRAM_ID=YOUR_DEVNET_PROGRAM_ID
```

#### Production Environment

```bash
# .env.production
NEXT_PUBLIC_SOLANA_NETWORK=mainnet-beta
NEXT_PUBLIC_SOLANA_RPC_URL=https://api.mainnet-beta.solana.com
NEXT_PUBLIC_PROGRAM_ID=YOUR_MAINNET_PROGRAM_ID
```

### Vercel Deployment

#### 1. Prepare for Deployment

```bash
# Install Vercel CLI
npm install -g vercel

# Build the application
npm run build

# Test the build locally
npm run start
```

#### 2. Deploy to Vercel

```bash
# Initialize Vercel project
vercel

# Configure environment variables in Vercel dashboard
# or use CLI:
vercel env add NEXT_PUBLIC_SOLANA_NETWORK
vercel env add NEXT_PUBLIC_SOLANA_RPC_URL
vercel env add NEXT_PUBLIC_PROGRAM_ID
```

#### 3. Configure vercel.json

```json
{
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "env": {
    "NEXT_PUBLIC_SOLANA_NETWORK": "@next_public_solana_network",
    "NEXT_PUBLIC_SOLANA_RPC_URL": "@next_public_solana_rpc_url",
    "NEXT_PUBLIC_PROGRAM_ID": "@next_public_program_id"
  }
}
```

### Netlify Deployment

#### 1. Build Configuration

Create `netlify.toml`:

```toml
[build]
  command = "npm run build"
  publish = ".next"

[build.environment]
  NODE_VERSION = "18"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

#### 2. Deploy

```bash
# Install Netlify CLI
npm install -g netlify-cli

# Build and deploy
netlify build
netlify deploy --prod
```

### Custom Server Deployment

#### Docker Deployment

Create `Dockerfile`:

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# Copy application code
COPY . .

# Build the application
RUN npm run build

# Expose port
EXPOSE 3000

# Start the application
CMD ["npm", "start"]
```

Build and run:

```bash
# Build Docker image
docker build -t charity-dapp .

# Run container
docker run -p 3000:3000 -e NEXT_PUBLIC_SOLANA_NETWORK=devnet charity-dapp
```

## Monitoring and Maintenance

### Performance Monitoring

#### Web Vitals Tracking

```typescript
// pages/_app.tsx
export function reportWebVitals(metric: NextWebVitalsMetric) {
  console.log(metric);
  
  // Send to analytics service
  if (metric.label === 'web-vital') {
    analytics.track('Web Vital', {
      name: metric.name,
      value: metric.value,
      id: metric.id,
    });
  }
}
```

#### Error Tracking

```typescript
// utils/error-tracking.ts
export function trackError(error: Error, context?: any) {
  console.error('Application error:', error, context);
  
  // Send to error tracking service
  errorTracker.captureException(error, {
    extra: context,
    tags: {
      component: 'charity-dapp',
      environment: process.env.NODE_ENV,
    },
  });
}
```

### Blockchain Monitoring

#### Transaction Monitoring

```bash
# Monitor program transactions
solana logs --url devnet PROGRAM_ID

# Watch for specific events
solana logs --url devnet | grep "CreateCharityEvent"
```

#### Account Monitoring

```typescript
// Monitor charity account changes
const connection = new Connection('https://api.devnet.solana.com');

connection.onAccountChange(
  charityPublicKey,
  (accountInfo) => {
    console.log('Charity account updated:', accountInfo);
    // Update local cache
    queryClient.invalidateQueries(['charity', charityPublicKey.toString()]);
  },
  'confirmed'
);
```

### Health Checks

#### Frontend Health Check

```typescript
// pages/api/health.ts
export default function handler(req: NextApiRequest, res: NextApiResponse) {
  const healthCheck = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version,
    environment: process.env.NODE_ENV,
  };
  
  res.status(200).json(healthCheck);
}
```

#### Program Health Check

```bash
# Check program is still deployed and executable
solana program show PROGRAM_ID --url devnet
```

## Rollback Procedures

### Frontend Rollback

#### Vercel Rollback

```bash
# List deployments
vercel list

# Rollback to previous deployment
vercel rollback DEPLOYMENT_URL
```

#### Manual Rollback

```bash
# Deploy previous Git commit
git checkout PREVIOUS_COMMIT_HASH
vercel --prod
```

### Smart Contract Rollback

⚠️ **Important**: Solana programs cannot be rolled back once deployed. Plan carefully:

1. **Deploy new program**: Deploy fixed version with new program ID
2. **Update frontend**: Point frontend to new program ID
3. **Migrate data**: If needed, create migration instructions
4. **Communicate**: Inform users of the program update

## Security Considerations

### Frontend Security

- **Environment Variables**: Never expose private keys in frontend
- **HTTPS Only**: Ensure all production traffic uses HTTPS
- **Content Security Policy**: Implement CSP headers
- **Regular Updates**: Keep dependencies updated

### Smart Contract Security

- **Immutability**: Once deployed, programs cannot be changed
- **Upgrade Authority**: Consider upgrade authority implications
- **Access Controls**: Verify all access controls work correctly
- **Audit Trail**: Maintain audit logs of all deployments

## Troubleshooting

### Common Deployment Issues

#### Program ID Mismatch

```bash
# Update program ID in client code
anchor keys sync

# Rebuild and redeploy frontend
npm run build
```

#### Insufficient Balance

```bash
# Check balance
solana balance

# Add more SOL (testnet)
solana airdrop 5

# For mainnet, transfer SOL from another wallet
```

#### Build Failures

```bash
# Clear caches
rm -rf .next node_modules anchor/target

# Reinstall and rebuild
npm install
npm run anchor::build
npm run build
```

### Performance Issues

#### Slow RPC Responses

```typescript
// Use faster RPC endpoints
const connection = new Connection(
  'https://solana-api.projectserum.com', // Faster endpoint
  'confirmed'
);
```

#### Bundle Size Optimization

```javascript
// next.config.js
const nextConfig = {
  experimental: {
    optimizePackageImports: ['@solana/wallet-adapter-react'],
  },
  webpack: (config) => {
    config.resolve.fallback = {
      fs: false,
      net: false,
      tls: false,
    };
    return config;
  },
};
```

## Next Steps

After successful deployment:

- **[Monitoring](monitoring.md)** - Set up comprehensive monitoring
- **[Performance](../advanced/performance.md)** - Optimize application performance
- **[Security](../advanced/security.md)** - Implement security best practices
- **[Scaling](../advanced/scalability.md)** - Plan for growth and scaling

Congratulations! Your Solana Charity dApp is now live and ready to accept donations on the blockchain. 🎉