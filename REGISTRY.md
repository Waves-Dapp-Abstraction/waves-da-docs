# Shared Registry

The ecosystem uses **one Registry** contract per network. You deploy it once; **every** integrated dApp and relayer uses the same registry address.

That way a user’s **DA wallet** is registered once and can be reused across **all** projects that participate in this abstraction layer.

## Canonical addresses

Fill in after deployment (keep this file in sync with your deployment):

| Network | Registry address (base58) |
|---------|---------------------------|
| Testnet | `TBD` — replace with your deployed Registry |
| Mainnet | `TBD` |

Use this value everywhere:

| Where | Variable / field |
|-------|------------------|
| Relayer `.env` | `REGISTRY_ADDRESS` |
| Frontend / SDK `getActiveDA()` | `registry` in `{ registry, eoa }` |

Do not use a different registry per project unless you fork the ecosystem and accept that users will have separate DA mappings per registry.

---

## See also

- [`SPEC.md`](SPEC.md) — Registry storage keys and `registerFromDA`
- [`INTEGRATION.md`](INTEGRATION.md) — init flow and Registry invocation
