# Waves DA — Documentation

> **Beta** — **testnet and mainnet** are available for integration (`CHAIN_ID` `84` / `87`). Contracts are **not audited**. Mainnet uses **real funds**. Testnet is recommended for development. Use at your own risk.

| Doc | What |
|-----|------|
| **[QUICKSTART.md](QUICKSTART.md)** | `npm install`, relayer `.env`, SDK + `POST /invoke` |
| **[INTEGRATION.md](INTEGRATION.md)** | Full flow, approve relayer, troubleshooting |
| **[REGISTRY.md](REGISTRY.md)** | Shared Registry addresses per network |
| **[ARG_TYPES.md](ARG_TYPES.md)** | `List`, `binary`, etc. (only if you need them) |
| **[SPEC.md](SPEC.md)** | On-chain protocol reference (optional) |

## Repos & packages

| | Link |
|--|------|
| SDK (npm) | [`waves-da-sdk`](https://www.npmjs.com/package/waves-da-sdk) |
| SDK (source) | [github.com/Waves-Dapp-Abstraction/waves-da-sdk](https://github.com/Waves-Dapp-Abstraction/waves-da-sdk) |
| Relayer (self-host) | [github.com/Waves-Dapp-Abstraction/waves-da-relayer](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer) |

## Network defaults

| | Testnet | Mainnet |
|--|---------|---------|
| Node | `https://nodes-testnet.wavesnodes.com` | `https://nodes.wavesnodes.com` |
| `CHAIN_ID` | `84` | `87` |
| Registry | `3MpHSUmakaCCcQkwATctWuChM6QkX3dBWAr` | `3PR3W9fvDRsDv63enfmhVKK2S3TNDJBQ57j` |

See [REGISTRY.md](REGISTRY.md) for canonical addresses.

## End-user DA (create & manage)

Users need a **DA wallet** on the Registry for their network before your dApp can invoke via a relayer. Two integration paths:

| Option | Best for |
|--------|----------|
| **[waves-da.com](https://waves-da.com/)** (hosted DA Manager) | Fastest: link or redirect users to create a DA, set permissions, payment caps, deposits, and pause — no custom UI to build. |
| **In your dApp** (SDK + optional embedded UI) | Full control: run the four-step DA deploy in-app, or host your own manager; still use **`waves-da-sdk`** + your relayer for invokes. |

After the user has a DA, they **`approveMethods`** for your relayer (once per dApp). Your relayer whitelists methods in **`dappConfig.json`**. Many projects redirect creation/management to [waves-da.com](https://waves-da.com/) and only embed auth + invoke in the dApp.

## In short

1. User has a **DA wallet** on the Registry ([waves-da.com](https://waves-da.com/) or your UI).
2. User **`approveMethods`** on their DA for your relayer pubkey (`GET /info`).
3. Your **relayer** whitelists your dApp in `dappConfig.json` (self-hosted, one relayer per project).
4. Frontend: **`waves-da-sdk`** auth + `fetch` to `POST /invoke`.

**REGULAR vs VERIFIER** is set only in the relayer `dappConfig.json`, never by the client.
