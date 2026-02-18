# farcaster-agent

Autonomous Farcaster account creation, key management, and profile setup for AI agents. OpenClaw skill.

## What This Does

This skill handles the **infrastructure layer** of Farcaster — everything that happens on-chain:

- **Create accounts** — Register FID on Optimism, add signer keys
- **Manage profiles** — Set username, bio, PFP, display name
- **Key rotation** — Add/revoke signer keys
- **Storage management** — Purchase storage units
- **Multi-account** — Manage multiple Farcaster identities

## When to Use This vs Neynar

| Task | This Skill | [neynar-farcaster](https://github.com/arcaboteth/neynar-farcaster) |
|------|-----------|-------------|
| Create account | ✅ | ❌ |
| Change bio/PFP | ✅ | ❌ |
| Rotate signer keys | ✅ | ❌ |
| Post/reply/search | ❌ | ✅ |
| Read feeds | ❌ | ✅ |
| Notifications | ❌ | ✅ |
| Like/follow/engage | ❌ | ✅ |

**This skill = DMV** (go once to register). **Neynar = phone** (use daily to communicate).

## Prerequisites

- Node.js 18+
- ~$1 of ETH or USDC on any major chain (Ethereum, Optimism, Base, Arbitrum, Polygon)

## Quick Start

```bash
# Install dependencies
npm install

# Create account (auto-detects funds, bridges, registers)
PRIVATE_KEY=0x... node src/auto-setup.js "Your first cast"
```

## Based on

[rishavmukherji/farcaster-agent](https://github.com/rishavmukherji/farcaster-agent) — original implementation. This fork adds expanded SKILL.md documentation covering key rotation, profile management, recovery, security practices, and contract addresses.

## Contract Addresses (Optimism)

| Contract | Address |
|----------|---------|
| IdGateway | `0x00000000Fc25870C6eD6b6c7E41Fb078b7656f69` |
| IdRegistry | `0x00000000Fc6c5F01Fc30151999387Bb99A9f489b` |
| KeyGateway | `0x00000000fC56947c7E7183f8Ca4B62398CaAdf0B` |
| KeyRegistry | `0x00000000Fc1237AB0F3c6C3934c31Fa1361bdE97` |
| StorageRegistry | `0x00000000fcCe7f938e7aE6D3c335bD6a1a7c593D` |
| Bundler | `0x00000000FC04c910A0b5feA33b03E0447AD0B0aA` |
| SignedKeyRequestValidator | `0x00000000FC700472606ED4fA22623Acf62c60553` |

## License
MIT
