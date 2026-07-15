# FHISH Relayer V2

> **Off-chain relayer for the FHISH encrypted-voting stack** — watches on-chain events, decrypts ciphertext handles through a self-hosted gateway, signs the results, and fulfills them back on-chain.

## Overview

FHISH Relayer V2 is a small, single-purpose Node.js service that bridges an FHE (fully homomorphic encryption) voting contract with an off-chain decryption gateway. Encrypted votes are cast on-chain as opaque ciphertext *handles*; this relayer polls for those events, asks the gateway/KMS to turn each handle into plaintext, and pushes the decrypted tally (or a signed fulfillment) back to the contract.

It is meant to run as an authorized backend for the FHISH system — it talks only to your own RPC endpoint and gateway, never to any third-party FHE infrastructure. It ships with a Docker setup and a lightweight HTTP health endpoint for deployment.

## Features

- **Event polling** — robustly scans new blocks on a fixed interval (no dependence on flaky WebSocket subscriptions), tracking the last processed block.
- **Two relay flows**:
  - `src/index.ts` — self-contained flow that listens for `VoteCast` events, decrypts each vote's handles, tallies YES/NO, and (when the relayer is the contract admin) calls `setDecryptedResult` on-chain.
  - `src/listener.ts` + engine/responder — generic flow that listens for `PublicDecryptionRequest`, decrypts every `ctHandle`, and calls `fulfillPublicDecryption` with a signed result.
- **Gateway KMS client** (`src/kms.ts`) — fetches ciphertext by handle (from the gateway contract or its HTTP service) and POSTs to `/decrypt`, guarded by a shared `x-fhish-relayer-secret` header.
- **Retry with backoff** — the `RelayerEngine` retries each decryption up to 3 times with increasing delay before failing.
- **Signed fulfillment** — results are ABI-encoded and signed (keccak256 + EIP-191 `signMessage`) before being submitted on-chain.
- **Health endpoint** — an Express server exposes `GET /health` reporting relayer address and uptime.
- **Container-ready** — a `Dockerfile` and a `docker-compose.yml` that wires the relayer to a `gateway` service.

## Tech Stack

- **Runtime:** Node.js 20+, ES modules
- **Language:** TypeScript 5 (run with `tsx`, built with `tsc`)
- **Blockchain:** [ethers](https://docs.ethers.org/) v6
- **HTTP:** axios (gateway/KMS calls) · express (health server)
- **Config:** dotenv
- **Ops:** Docker + Docker Compose

## Getting Started

```bash
# 1. Clone and install
git clone https://github.com/nickthelegend/fhish-relayer-v2.git
cd fhish-relayer-v2
npm install

# 2. Configure environment
cp .env.example .env
# then edit .env — set PRIVATE_KEY, RPC_URL, VOTING_ADDRESS (or GATEWAY_ADDRESS),
# GATEWAY_URL and FHISH_RELAYER_SECRET

# 3. Run in dev (auto-reload)
npm run dev

# 3b. or run once
npm run start

# 4. Build to dist/
npm run build
```

Run with Docker instead:

```bash
docker compose up --build
```

### Environment variables

| Variable               | Description                                              |
| ---------------------- | -------------------------------------------------------- |
| `PRIVATE_KEY`          | Relayer wallet private key (signs & submits txs)         |
| `RPC_URL`              | EVM RPC endpoint (e.g. Sepolia)                          |
| `VOTING_ADDRESS`       | On-chain voting contract (used by `src/index.ts`)        |
| `GATEWAY_ADDRESS`      | On-chain FHISH gateway contract (optional handle source) |
| `GATEWAY_URL`          | Gateway HTTP endpoint, default `http://localhost:8080`   |
| `FHISH_RELAYER_SECRET` | Shared secret sent as `x-fhish-relayer-secret`           |
| `HEALTH_PORT`          | Port for the health server, default `3001`               |

## Project Structure

```
fhish-relayer-v2/
├── src/
│   ├── index.ts        # Main entry: VoteCast listener → decrypt → tally → setDecryptedResult
│   ├── listener.ts     # Generic PublicDecryptionRequest event poller
│   ├── compute.ts      # RelayerEngine: orchestrates decryption with retry/backoff
│   ├── kms.ts          # KMSClient: fetch ciphertext + POST /decrypt to the gateway
│   └── responder.ts    # Signs & broadcasts fulfillPublicDecryption transactions
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── package.json
└── tsconfig.json
```

---

Built by [nickthelegend](https://github.com/nickthelegend) · [nickthelegend.tech](https://nickthelegend.tech)
