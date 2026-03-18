# SimGame AI

**Agentic AI City Builder — Play to Earn on the Blockchain**

SimGame AI is a blockchain-based city-building simulator inspired by the rules and mechanics of SimCity 3000. AI agents autonomously govern cities, make decisions, and compete for resources — while players earn real value by owning, directing, and deploying those agents.

## What is SimGame AI?

SimGame AI merges classic city simulation with on-chain ownership and autonomous AI agents. Each city is an NFT. Each citizen, mayor, and advisor is an AI agent operating within the game's rules. Players earn tokens by building thriving cities, completing milestones, and outcompeting other cities in the ecosystem.

## Core Concepts

- **AI Agents** — Autonomous agents act as mayors, advisors (transport, power, water, finance), and citizens. They make decisions, respond to events, and evolve over time.
- **Blockchain Ownership** — Cities, land parcels, buildings, and agents are on-chain assets. True ownership, tradable on open markets.
- **Play to Earn** — Players earn the native token ($SIM) through city growth, population milestones, budget surpluses, and inter-city trade.
- **SimCity 3000 Rules** — Zoning (residential, commercial, industrial), utilities (power, water, waste), city services, taxation, demand, and pollution mechanics all derived from SC3K.
- **Emergent Simulation** — Agent decisions create emergent city behavior — traffic jams, industrial booms, housing crises — driven by AI, not scripted events.

## Getting Started

> Documentation and development in progress. See [game-design.md](game-design.md) for the full game design and [tech-design.md](tech-design.md) for the contract architecture.

## Tech Stack

- **Chain:** Base (L2) — gas per zone op ~$0.08
- **Contracts:** Solidity / ERC-721 cities, ERC-20 $SIM
- **AI Agents:** Claude API (Anthropic), Agent SDK
- **No off-chain simulation** — everything ages with blocks, derived on-chain

## License

See [LICENSE](LICENSE).
