# Waves DA — Documentation

**Testnet is ready to integrate.** Mainnet is not available yet.

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

## Testnet defaults

| | Value |
|--|--------|
| Node | `https://nodes-testnet.wavesnodes.com` |
| `CHAIN_ID` | `84` |
| Registry | `3MpHSUmakaCCcQkwATctWuChM6QkX3dBWAr` |

## In short

1. User has a **DA wallet** registered on the Registry.
2. User **`approveMethods`** on their DA for your relayer pubkey (`GET /info`).
3. Your **relayer** whitelists your dApp in `dappConfig.json`.
4. Frontend: **`waves-da-sdk`** auth + `fetch` to `POST /invoke`.

**REGULAR vs VERIFIER** is set only in the relayer `dappConfig.json`, never by the client.
