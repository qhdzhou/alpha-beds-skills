# Alpha Beds Agent Skills

AI agent skills for hotel booking via Alpha Beds.

## Available Skills

| Skill | Description | Install |
|-------|------------|---------|
| `alpha-beds-hotel-fluxa` | Hotel booking with FluxA USDC payment | `npx skills add -s alpha-beds-hotel-fluxa -y -g qhdzhou/alpha-beds-skills` |

## About Alpha Beds

Alpha Beds provides AI agents access to 700,000+ hotels worldwide with real-time pricing, natural language search (English & Chinese), and crypto payment support (USDC on Base via FluxA).

### Booking Flow

```
Search (natural language) → Browse Rooms → Verify Price → Create Order → Pay with USDC
```

## Quick Start

```bash
# Install the skill
npx skills add -s alpha-beds-hotel-fluxa -y -g qhdzhou/alpha-beds-skills

# Prerequisite: FluxA Agent Wallet
npx skills add -s fluxa-agent-wallet -y -g FluxA-Agent-Payment/FluxA-AI-Wallet-MCP
```

## License

MIT
