# optimism-docker-alt

A docker-compose setup to run an OP Stack L2 (execution: op-geth, rollup: op-node) with the Alt-DA service backed by Celestia. This repo is intended for experimentation and local/test deployments that bridge to an L1 endpoint (e.g., Ethereum Sepolia). It also includes example configs and environment variables to help you get started quickly.

Services brought up by docker-compose:
- op-geth: L2 execution client (Geth with Optimism patches).
- op-node: Rollup node (sequencer enabled) that connects L1<->L2 and to Alt-DA.
- op-batcher: Batches L2 transactions and submits them to L1.
- op-proposer: Posts state roots / proposals to L1 dispute game factory.
- op-alt-da: Alternative Data Availability service (Celestia integration) for blob publishing and verification.

Note: This configuration expects you to provide your own L1 RPC endpoint and private keys for the system actors (sequencer, batcher, proposer). The sample chain parameters (genesis, rollup.json) are included under config/.
If you need help generating those, please take a look here [Create your own Rollup](https://sysrex.com/posts/create-your-won-rollup-opstack/)

## Prerequisites
- Docker and Docker Compose
- An L1 RPC URL (e.g., an Ethereum Sepolia endpoint)
- Private keys for:
  - Sequencer (used by op-node)
  - Batcher (used by op-batcher)
  - Proposer (used by op-proposer)
- Optional: Access to a Celestia bridge node (for Alt-DA)

## Quick start
1) Copy the .env template and fill values

   cp .env.example .env

   Then open .env and provide values:
   - L1_RPC_URL: Your L1 endpoint (e.g., https://sepolia.infura.io/v3/<KEY>)
   - L1_RPC_KIND: Provider type (basic|quicknode|erigon|nethermind|geth). Defaults to basic if not set.
   - L1_CHAIN_ID: L1 chain id (defaults to 11155111 for Sepolia in compose).
   - L2_CHAIN_ID: L2 chain id; must match config/genesis.json and config/rollup.json.
   - GS_SEQUENCER_PRIVATE_KEY: Private key used by the sequencer (hex, 0x-prefixed).
   - GS_BATCHER_PRIVATE_KEY: Private key used by the batcher (hex, 0x-prefixed).
   - GS_PROPOSER_PRIVATE_KEY: Private key used by the proposer (hex, 0x-prefixed).
   - GAME_FACTORY_ADDRESS: L1 dispute game factory address (for op-proposer).
   - Optional Alt-DA settings:
     - CELESTIA_BRIDGE: http://<host>:26658 (Celestia bridge node)
     - CELESTIA_AUTH_TOKEN: Auth token for the Celestia bridge
     - CELESTIA_NAMESPACE: Celestia namespace to publish to

   The .env.example in this repo lists all variables you can set.

2) Start the stack

   docker compose up -d

   What happens:
   - op-geth-init initializes the L2 genesis (only if empty data dir)
   - op-geth exposes JSON-RPC on localhost:8545 and WS on :8546
   - op-node exposes RPC on localhost:9545
   - op-batcher exposes admin RPC on :8548 (optional)
   - op-proposer exposes RPC on :8560
   - op-alt-da exposes HTTP on :3100

3) Check health
- op-geth health: curl -s http://localhost:8545 | head
- op-node health: curl -s http://localhost:9545 | head
- alt-da health: curl -s http://localhost:3100/health

If all are healthy, your L2 should be producing blocks and syncing with L1.

## Ports
- 8545: op-geth JSON-RPC
- 8546: op-geth WebSocket
- 8551: Engine API (JWT)
- 9545: op-node RPC
- 8548: op-batcher admin RPC (optional)
- 8560: op-proposer RPC
- 3100: Alt-DA HTTP

## Configuration files
- config/genesis.json: L2 genesis for op-geth (must align with L2_CHAIN_ID).
- config/rollup.json: Rollup configuration for op-node. Ensure addresses and times match your L1 deployment and expectations.
- config/jwt.txt: Shared JWT secret between op-geth and op-node Engine API.

Important: The chain IDs in both config and .env must match. If you change L2_CHAIN_ID or other chain params, regenerate or adjust both genesis.json and rollup.json coherently.

## Common operations
- Start: docker compose up -d
- Stop: docker compose down
- View logs for a service: docker compose logs -f op-node (or op-geth/op-batcher/op-proposer/op-alt-da)
- Reset L2 state: Stop the stack and remove the op-geth data directory (./op-geth/data), then start again to re-init genesis.

## Funding and keys
- This stack does not generate or fund keys for you. The private keys you provide must be funded appropriately on L1/L2 for their roles (e.g., gas for posting batches/proposals on L1).
- Never commit real/private keys. Use throwaway keys for testing.

## Alt-DA (Celestia) notes
- op-node and op-batcher are configured to use the Alt-DA service at http://op-alt-da:3100.
- Provide CELESTIA_BRIDGE and CELESTIA_AUTH_TOKEN in .env so op-alt-da can connect to your Celestia node.
- You can disable Alt-DA by removing the op-alt-da service and the related flags in op-node and op-batcher in docker-compose.yml.

## Spamoor scenarios (optional)
A sample spamoor configuration is included at spamoor/spamoor-config.yml for generating transaction load patterns (EOA tx, ERC20, Uniswap-like swaps). This repository does not run spamoor by default; use your own spamoor runner and point it to your L2 JSON-RPC (http://localhost:8545).

## Troubleshooting
- op-geth stuck initializing: Ensure ./op-geth/data is writable by Docker and that config/genesis.json is valid.
- op-node cannot connect to L1: Verify L1_RPC_URL and L1_RPC_KIND; some providers require specific kinds.
- Batcher/Proposer failing due to nonces or insufficient funds: Fund the corresponding addresses and verify the keys are correct and 0x-prefixed.
- Alt-DA health check failing: Confirm CELESTIA_BRIDGE and CELESTIA_AUTH_TOKEN; ensure the Celestia bridge is reachable from Docker network.

## Disclaimer
This is an experimental setup for development/testing. Do not use in production. Carefully manage keys and RPC endpoints.