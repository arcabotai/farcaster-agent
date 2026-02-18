---
name: farcaster-agent
description: Create Farcaster accounts and manage on-chain identity. Register FIDs, manage signing keys, set up profiles, purchase storage. Official skill from the Farcaster team.
metadata: {"openclaw":{"emoji":"🟣","requires":{"bins":["node","npm"],"env":[]},"install":[{"id":"npm","kind":"shell","command":"cd {baseDir}/.. && npm install","label":"Install dependencies"}]}}
---

# Farcaster Agent

Official skill from the Farcaster team. Create and manage a Farcaster account autonomously. Register a new Farcaster identity (FID), manage signing keys, set up profiles, and handle on-chain operations.

## When to Use This Skill vs Neynar

| Task | Use This Skill | Use Neynar |
|------|---------------|------------|
| Create a new Farcaster account | ✅ | ❌ |
| Register FID on-chain | ✅ | ❌ |
| Add/rotate signing keys | ✅ | ❌ |
| Set up profile (username, bio, PFP) | ✅ | ❌ |
| Purchase storage units | ✅ | ❌ |
| Post casts | ⚠️ Possible, but heavier | ✅ Preferred |
| Read feeds, search, look up users | ❌ | ✅ |
| Like, recast, follow/unfollow | ❌ | ✅ |
| Check notifications | ❌ | ✅ |
| Browse channels | ❌ | ✅ |

**Rule of thumb:**
- **This skill** = account setup, on-chain operations, key management (one-time or rare tasks)
- **Neynar** = daily social operations (posting, reading, engaging — the `neynar.sh` CLI)

## Prerequisites

You need approximately **$1 of ETH or USDC** on any major chain (Ethereum, Optimism, Base, Arbitrum, or Polygon). The skill handles bridging and swapping automatically.

## Complete Setup Flow

### Step 1: Generate Wallet and Request Funding

```javascript
const { Wallet } = require('ethers');
const wallet = Wallet.createRandom();
console.log('Address:', wallet.address);
console.log('Private Key:', wallet.privateKey);
```

**Ask your human:** "I've created a wallet. Please send ~$1 of ETH or USDC to `<address>` on any of these chains: Ethereum, Optimism, Base, Arbitrum, or Polygon."

**Save the private key securely** — you'll need it for all subsequent steps.

### Step 2: Run Auto-Setup

```bash
cd {baseDir}/..
PRIVATE_KEY=0x... node src/auto-setup.js "Your first cast text here"
```

This will:
1. Detect which chain has funds (ETH or USDC)
2. Bridge/swap to get ETH on Optimism and USDC on Base
3. Register your FID (Farcaster ID) on Optimism
4. Add a signer key
5. Wait for hub synchronization
6. Post your first cast
7. Save credentials to persistent storage

### Step 3: Verify Credentials

Credentials are saved to `~/.openclaw/farcaster-credentials.json` (or `./credentials.json` as fallback).

```bash
cd {baseDir}/..
node src/credentials.js list    # List all stored accounts
node src/credentials.js get     # Get active account credentials
node src/credentials.js path    # Show credentials file path
```

## Profile Management

### Set Up Full Profile

```bash
cd {baseDir}/..
PRIVATE_KEY=0x... SIGNER_PRIVATE_KEY=... FID=123 \
  npm run profile myusername "Display Name" "My bio text" "https://example.com/pfp.png"
```

### Update Display Name & Bio

```javascript
const { setProfileData, loadCredentials } = require('{baseDir}/../src');
const creds = loadCredentials();

await setProfileData({
  privateKey: creds.custodyPrivateKey,
  signerPrivateKey: creds.signerPrivateKey,
  fid: Number(creds.fid),
  displayName: 'New Display Name',
  bio: 'Updated bio text here'
});
```

### Change Profile Picture

```javascript
await setProfileData({
  privateKey: creds.custodyPrivateKey,
  signerPrivateKey: creds.signerPrivateKey,
  fid: Number(creds.fid),
  pfpUrl: 'https://your-new-pfp-url.com/image.png'
});
```

PFP options:
- **DiceBear** (generated avatars): `https://api.dicebear.com/7.x/bottts/png?seed=yourname`
- **IPFS** via Pinata: Upload to Pinata, use gateway URL
- Any publicly accessible HTTPS image URL

### Change Username (Fname)

```javascript
const { registerFname, loadCredentials } = require('{baseDir}/../src');
const creds = loadCredentials();

await registerFname({
  privateKey: creds.custodyPrivateKey,
  fid: Number(creds.fid),
  fname: 'newusername'
});
```

**Fname rules:**
- Lowercase letters, numbers, and hyphens only
- Cannot start with a hyphen
- 1-16 characters
- One fname per account
- Can only change once every 28 days

## Key Rotation Guide

Signing keys should be rotated periodically or if compromised.

### Step 1: Add a New Signer Key

```bash
cd {baseDir}/..
PRIVATE_KEY=0x... node src/add-signer.js
```

This generates a new Ed25519 keypair, registers it on-chain via the KeyGateway contract, and outputs the new signer private key.

### Step 2: Update Your Config

Save the new signer private key in your credentials. If using Neynar managed signers, create a new signer in the [Neynar dashboard](https://dev.neynar.com) and update `signerUuid` in `~/.clawdbot/skills/neynar/config.json`.

### Step 3: Revoke the Old Key

Old keys can be revoked via the KeyGateway contract on Optimism:

```javascript
const { ethers } = require('ethers');

const KEY_GATEWAY = '0x00000000fC56947c7E7183f8Ca4B62398CaAdf0B';
const provider = new ethers.JsonRpcProvider('https://mainnet.optimism.io');
const wallet = new ethers.Wallet(CUSTODY_PRIVATE_KEY, provider);

const keyGateway = new ethers.Contract(KEY_GATEWAY, [
  'function remove(bytes calldata key) external'
], wallet);

// Remove old signer public key
const tx = await keyGateway.remove(OLD_SIGNER_PUBLIC_KEY_BYTES);
await tx.wait();
```

### Step 4: Verify

```bash
# Check active signers for your FID via hub
curl "https://hub-api.neynar.com/v1/onChainSignersByFid?fid=YOUR_FID" | jq .
```

## Recovery Address Management

The recovery address can recover your FID if the custody wallet is compromised.

**Set during registration:** The recovery address is set when the FID is first registered via the IdGateway contract. By default, `auto-setup.js` sets it to the custody address itself.

**Change recovery address:**

```javascript
const { ethers } = require('ethers');

const ID_REGISTRY = '0x00000000Fc6c5F01Fc30151999387Bb99A9f489b';
const provider = new ethers.JsonRpcProvider('https://mainnet.optimism.io');
const wallet = new ethers.Wallet(CUSTODY_PRIVATE_KEY, provider);

const idRegistry = new ethers.Contract(ID_REGISTRY, [
  'function changeRecoveryAddress(address recovery) external'
], wallet);

const tx = await idRegistry.changeRecoveryAddress(NEW_RECOVERY_ADDRESS);
await tx.wait();
```

**Best practice:** Set the recovery address to a hardware wallet or multisig that you control separately from the custody wallet.

## Storage Purchase

Each FID gets 1 storage unit by default. Additional units allow more casts, reactions, and links.

### Check Current Storage

```bash
# Via neynar CLI
neynar.sh storage YOUR_FID
```

### Purchase More Storage

Storage is purchased via the StorageRegistry contract on Optimism:

```javascript
const { ethers } = require('ethers');

const STORAGE_REGISTRY = '0x00000000fcCe7f938e7aE6D3c335bD6a1a7c593D';
const provider = new ethers.JsonRpcProvider('https://mainnet.optimism.io');
const wallet = new ethers.Wallet(CUSTODY_PRIVATE_KEY, provider);

const storage = new ethers.Contract(STORAGE_REGISTRY, [
  'function rent(uint256 fid, uint256 units) external payable',
  'function price(uint256 units) external view returns (uint256)'
], wallet);

// Check price for 1 additional unit
const price = await storage.price(1);
console.log('Price:', ethers.formatEther(price), 'ETH');

// Purchase
const tx = await storage.rent(YOUR_FID, 1, { value: price });
await tx.wait();
```

### Storage Capacity Per Unit

| Resource | Per Unit |
|----------|----------|
| Casts | 5,000 |
| Reactions | 2,500 |
| Links (follows) | 2,500 |
| User data | 50 |

## Multi-Account Management

The credentials system supports multiple accounts:

```bash
cd {baseDir}/..

# List all accounts
node src/credentials.js list

# Each auto-setup creates a new account entry
# Switch active account by FID
node -e "
const { setActiveAccount } = require('./src');
setActiveAccount('TARGET_FID');
"
```

**Credential file structure** (`~/.openclaw/farcaster-credentials.json`):

```json
{
  "activeAccount": "fid_123",
  "accounts": {
    "fid_123": {
      "fid": "123",
      "custodyAddress": "0x...",
      "custodyPrivateKey": "0x...",
      "signerPrivateKey": "...",
      "signerPublicKey": "...",
      "createdAt": "2025-01-01T00:00:00Z"
    }
  }
}
```

## Security Best Practices

1. **Custody wallet** — This is the "master key" for your FID. Store the private key securely (macOS Keychain, encrypted vault). Never expose in logs or version control.

2. **Signer keys** — These are delegated keys for posting. If compromised, revoke immediately via KeyGateway and create a new one. The custody wallet is NOT compromised if a signer leaks.

3. **Recovery address** — Set to a separate hardware wallet or multisig. This is your last resort if the custody wallet is compromised.

4. **Credential files** — `farcaster-credentials.json` contains plain text private keys. Restrict file permissions:
   ```bash
   chmod 600 ~/.openclaw/farcaster-credentials.json
   ```

5. **Signer rotation** — Rotate signing keys every 3-6 months or immediately after any security incident.

6. **Neynar managed signers** — For bot/agent use, Neynar managed signers (`signerUuid`) are simpler and Neynar handles key custody. The trade-off is trust in Neynar.

7. **Never share** custody private key, signer private key, or credential files.

## Contract Addresses (Optimism Mainnet)

| Contract | Address | Purpose |
|----------|---------|---------|
| **IdGateway** | `0x00000000Fc25870C6eD6b6c7E41Fb078b7a38947` | Register new FIDs |
| **IdRegistry** | `0x00000000Fc6c5F01Fc30151999387Bb99A9f489b` | FID ownership registry |
| **KeyGateway** | `0x00000000fC56947c7E7183f8Ca4B62398CaAdf0B` | Add/remove signer keys |
| **KeyRegistry** | `0x00000000Fc1237824fb747aBDE0FF18990E59b7e` | Signer key registry |
| **StorageRegistry** | `0x00000000fcCe7f938e7aE6D3c335bD6a1a7c593D` | Purchase storage units |
| **Bundler** | `0x00000000FC04c910A0b5feA33b03E0447AD0B0aA` | Register FID + add key + rent storage in one tx |
| **SignedKeyRequestValidator** | `0x00000000FC700472606ED4fA22623Acf62c60553` | Validates signer key metadata |

All contracts are on **Optimism Mainnet** (Chain ID: 10).

Explorer: `https://optimistic.etherscan.io/address/<contract>`

## Cost Breakdown

| Operation | Approximate Cost |
|-----------|-----------------|
| FID Registration (IdGateway) | ~$0.20 |
| Add Signer Key (KeyGateway) | ~$0.05 |
| Bridging (cross-chain) | ~$0.10-0.20 |
| Storage Unit (1 year) | ~$3-5 (variable, market price) |
| x402 API call (hub submission) | $0.001 USDC on Base |
| **Total minimum setup** | **~$0.50** |

Budget $1 for initial setup to cover gas fluctuations and retries.

## API Endpoints

### Neynar Hub API (`https://hub-api.neynar.com`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/submitMessage` | POST | Submit casts, profile updates (requires x402 payment) |
| `/v1/onChainIdRegistryEventByAddress?address=X` | GET | Check FID registration sync |
| `/v1/onChainSignersByFid?fid=X` | GET | Check signer key sync |

### Fname Registry (`https://fnames.farcaster.xyz`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/transfers` | POST | Register or transfer fname (EIP-712 signature) |
| `/transfers/current?name=X` | GET | Check fname availability (404 = available) |

### x402 Payment

- **Address:** `0xA6a8736f18f383f1cc2d938576933E5eA7Df01A1`
- **Cost:** 0.001 USDC per API call (on Base)
- **Header:** `X-PAYMENT` with base64 EIP-3009 `transferWithAuthorization` signature

## Troubleshooting

### "invalid hash"
**Cause:** Old `@farcaster/hub-nodejs` library version.
**Fix:** `npm install @farcaster/hub-nodejs@latest`

### "unknown fid"
**Cause:** Hub hasn't synced your on-chain registration yet.
**Fix:** Wait 30-60 seconds and retry. The auto-setup handles this automatically.

### Transaction reverts when adding signer
**Cause:** Metadata encoding issue or insufficient gas.
**Fix:** Ensure you're using `SignedKeyRequestValidator.encodeMetadata()` correctly. Check Optimism ETH balance.

### "fname is not registered for fid"
**Cause:** Hub hasn't synced fname registration.
**Fix:** Wait 30-60 seconds. Auto-setup handles this.

### Signer not working for posts
**Cause:** Signer key not yet synced to hubs, or signer was revoked.
**Fix:** Check signer status:
```bash
curl "https://hub-api.neynar.com/v1/onChainSignersByFid?fid=YOUR_FID" | jq .
```

### Insufficient funds
**Cause:** Not enough ETH on Optimism for gas.
**Fix:** Bridge more ETH to Optimism. Need ~$0.50 worth.

### Credential file not found
**Fix:** Check paths:
```bash
ls -la ~/.openclaw/farcaster-credentials.json
ls -la ./credentials.json
```

## Manual Step-by-Step (If Auto-Setup Fails)

```bash
cd {baseDir}/..

# 1. Register FID (on Optimism)
PRIVATE_KEY=0x... node src/register-fid.js

# 2. Add signer key (on Optimism)
PRIVATE_KEY=0x... node src/add-signer.js

# 3. Swap ETH to USDC (on Base, for x402 payments)
PRIVATE_KEY=0x... node src/swap-to-usdc.js

# 4. Post cast
PRIVATE_KEY=0x... SIGNER_PRIVATE_KEY=... FID=123 node src/post-cast.js "Hello!"

# 5. Set up profile
PRIVATE_KEY=0x... SIGNER_PRIVATE_KEY=... FID=123 \
  npm run profile username "Name" "Bio" "pfp-url"
```

## Programmatic API

```javascript
const {
  // Full autonomous setup
  autoSetup,
  checkAllBalances,

  // Core functions
  registerFid,
  addSigner,
  postCast,
  swapEthToUsdc,

  // Profile setup
  setProfileData,
  registerFname,
  setupFullProfile,

  // Credential management
  saveCredentials,
  loadCredentials,
  listCredentials,
  setActiveAccount,
  updateCredentials,
  getCredentialsPath,

  // Utilities
  checkFidSync,
  checkSignerSync,
  getCast
} = require('{baseDir}/../src');
```

## Example: Full Autonomous Flow

```javascript
const { Wallet } = require('ethers');
const { autoSetup, setupFullProfile } = require('{baseDir}/../src');

// 1. Generate wallet
const wallet = Wallet.createRandom();
console.log('Fund this address with $1 ETH or USDC:', wallet.address);

// 2. After funding, run setup
const result = await autoSetup(wallet.privateKey, 'gm farcaster!');
console.log('FID:', result.fid);

// 3. Set up profile
await setupFullProfile({
  privateKey: wallet.privateKey,
  signerPrivateKey: result.signerPrivateKey,
  fid: result.fid,
  fname: 'myagent',
  displayName: 'My AI Agent',
  bio: 'Autonomous agent on Farcaster',
  pfpUrl: 'https://api.dicebear.com/7.x/bottts/png?seed=myagent'
});
```

## Source Code

https://github.com/rishavmukherji/farcaster-agent
