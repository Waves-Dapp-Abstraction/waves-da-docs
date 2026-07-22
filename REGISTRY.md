# Registry — addresses per network

**Mainnet (copy-paste):** `3PR3W9fvDRsDv63enfmhVKK2S3TNDJBQ57j` — `CHAIN_ID=87`

**Testnet (copy-paste):** `3MpHSUmakaCCcQkwATctWuChM6QkX3dBWAr` — `CHAIN_ID=84`

The Registry maps **EOA → active DA wallet**.

**One Registry per network** for the whole ecosystem. Users create their DA once and reuse it across integrated dApps.

---

## Canonical addresses

| Network | Chain ID | Registry address | Status |
|---------|----------|------------------|--------|
| Testnet | `84` | `3MpHSUmakaCCcQkwATctWuChM6QkX3dBWAr` | Deployed |
| Mainnet | `87` | `3PR3W9fvDRsDv63enfmhVKK2S3TNDJBQ57j` | Déployé (canonique mainnet) |

---

## Where to use the same address

| Component | Setting |
|-----------|---------|
| Relayer `.env` | `REGISTRY_ADDRESS` |
| SDK | `registry` in `getActiveDAOrNull(nodeUrl, { registry, eoa })` |
| Your dApp config | Same base58 as the network you target ([REGISTRY.md](REGISTRY.md)) |

**Node URL** must match the network (`nodes-testnet` vs mainnet).

---

## Notes

- Do not deploy a separate Registry unless you run an isolated ecosystem
- A different Registry means separate DA mappings (users must create a new DA)
- Contract upgrades use `SetScript` on the same address (maintainers only)
