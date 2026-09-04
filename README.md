# SatsRouting (Boltz) Backend

[![CI](https://github.com/BoltzExchange/boltz-backend/actions/workflows/ci.yml/badge.svg)](https://github.com/BoltzExchange/boltz-backend/actions/workflows/ci.yml)
[![Version](https://img.shields.io/npm/v/boltz-backend.svg)](https://www.npmjs.com/package/boltz-backend)

This is the source of the fork of the official Boltz Backend. It enables **non-custodial** swaps between different Bitcoin layers. Boltz Backend exposes a RESTful HTTP API that can be used to query information like supported pairs as well as to create and monitor swaps.

## Documentation

See the [Satsrouting API documentation](https://github.com/SatsRouting/boltz-backend).

## Security

**Date:** September 1, 2026
**Component:** `boltz-backend` (swap server)
**Scope of this node:** BTC on-chain, Lightning (LND), and Liquid (L-BTC) swaps only. EVM/Rootstock/Arbitrum, Solana, Tron, ERC20/OFT and bridge/CCTP paths are not enabled on this deployment.

### Overview

Our fork of this project is intentionally scoped to **Bitcoin, Liquid Bitcoin (L-BTC), and Lightning only**. We do not maintain or operate any altcoin, EVM-family (Ethereum/Rootstock/Arbitrum), Solana, Tron, or bridge/token integrations, and we do not plan to. This deliberately narrow focus keeps the codebase we actually run smaller and easier to reason about from a security standpoint, and it shaped how the audit findings below were triaged.

Following a third-party security audit, a series of hardening fixes was reviewed, applied, and verified on the backend. Each finding from the audit was triaged into one of three buckets:

- **Active and relevant** — touches code paths we actually exercise in production (Lightning payment handling, refunds, Liquid asset handling, secrets/logging, API authentication, randomness).
- **Defensive** — correct and merged, but only exercised on a Liquid edge case (sending a Liquid asset other than L-BTC) that is not part of normal usage today.
- **Not applicable** — code paths tied to chains or multi-node configurations we do not run (Core Lightning multi-node setups, EVM-family chains). Left in place for upstream compatibility but inert in our context.

Roughly fourteen fixes fell into the active/relevant category, two were defensive, and a further three were confirmed out of scope and required no action beyond review.

### What was hardened

Without going into implementation-level detail, the backend fixes cluster around the following themes:

- **Payment and refund correctness.** Additional safeguards ensure refunds and settlement decisions are only made once a Lightning payment's outcome is unambiguous, closing edge cases where an in-flight or ambiguous payment state could previously have led to an incorrect refund or a stuck swap. Payment trackers now re-observe on stream errors and non-terminal states rather than assuming a payment is resolved.
- **Liquid asset handling.** Stricter validation was added around confidential transaction data, so malformed or anomalous structures fail explicitly instead of producing silent incorrect values.
- **Secrets and logging hygiene.** Sensitive material — recovery mnemonics, payment preimages, and full configuration dumps — is no longer written to logs or startup error output in the clear. This is a pure exposure-reduction measure and does not change how the underlying secrets are generated or stored.
- **API and process authentication.** Internal RPC/gRPC interfaces now fail closed rather than open when exposed without proper authentication, with tighter authorization checks around a small number of sensitive administrative operations, protection against path traversal on a diagnostic endpoint, and constant-time comparison for authentication tokens to remove a timing side-channel.
- **File and permission hardening.** Access permissions on TLS material, certificate directories, and diagnostic dump files were tightened to reduce local exposure.
- **Randomness and identifiers.** Swap identifiers are now generated with a cryptographically secure random source and use a longer identifier length, reducing predictability.
- **Dependency hygiene.** A small number of third-party libraries were updated to patched versions.

One fix is classed as *defensive*: it hardened a Liquid signing path (multi-asset rescue) that is not triggered under normal operation with L-BTC alone, keeping a sensitive signing nonce strictly in-process rather than in shared cache, and clearing it after use. This path is now covered by automated tests in case it is ever exercised.

A handful of upstream fixes — covering Core Lightning multi-node setups, cross-node double-payment guards, and EVM-specific key handling — were reviewed and confirmed **not applicable** to this deployment, since we run a single LND node and no EVM integrations.

### Verification

- All fixes applicable to our configuration were exercised on the live node: submarine, reverse, and chain swaps across BTC, Lightning, and Liquid; cooperative refunds.
- No regressions were observed on any swap flow.

### Operational notes

- EVM/Rootstock/Arbitrum integrations remain disabled at startup, consistent with our BTC/LN/Liquid-only scope.

### Conclusion

All audit findings relevant to our operational scope (BTC on-chain, Lightning/LND, Liquid L-BTC) have been applied and verified through live testing.

## Resources

- Open a Lightning channel with the platform:
  [LND](https://amboss.space/node/0317235909659a67918dde7786b4986319c68165b72893877e0ff64a973bc62395)
