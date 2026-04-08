# Integration guide — DApp Abstraction (DA)

This guide is for teams that want users to interact with **your dApp** through a **DA Wallet** and optionally a **relayer**. It complements [`SPEC.md`](SPEC.md) and the component READMEs.

> **Want a shorter path first?** See **[`QUICKSTART.md`](QUICKSTART.md)** — minimal relayer setup + front `fetch` + optional SDK usage.  
> **Registry:** one canonical address per network — [`REGISTRY.md`](REGISTRY.md).

---

## Concepts

### DA Wallet (`sdk/contracts/da/DA.ride` in `waves-da-sdk`)

A smart account that:

- is **bound to an EOA** (the “owner”) via the Registry
- can **`proxy`** invocations to any whitelisted target dApp when a **relayer** has permission
- supports **REGULAR** and **VERIFIER** execution for that proxy call

### Registry (`contracts/registry/Registry.ride`)

Stores:

- `activeDA_<eoa>` → DA address  
- `activeDA_pk_<eoa>` → DA public key  

So off-chain code can resolve “which DA does this user use?”.

### REGULAR vs VERIFIER (when to use which)

| | REGULAR | VERIFIER |
|---|---------|----------|
| **Tx sender** | Relayer | DA (smart account) |
| **`originCaller` on your dApp** | Relayer (not the user’s identity) | DA (matches user’s abstraction identity) |
| **Fee** | Often lower; DA can **reimburse** the relayer (`reimburseFee`) | DA pays; no relayer “sponsorship” of fee in the same way |
| **Choose when** | You only care that the **DA** is the logical actor, or you control checks via `caller` | Your contract logic uses **`originCaller`** (or you need it to match the user/DA story) |

**Relayer policy:** In production, the HTTP relayer must choose mode via **`dappConfig.json`** (`useOrigin`), not via untrusted client input. The SDK’s `buildInvokeViaDA(..., { useOrigin })` is what the relayer uses internally after it looks up the method config.

---

## End-to-end flow

### 1. Deploy Registry and DA (tooling or manual)

- Deploy [`Registry.ride`](../contracts/registry/Registry.ride) and the DA script from [`sdk/contracts/da/DA.ride`](../sdk/contracts/da/DA.ride) (same file ships in the SDK package) per your environment.
- See [`tooling/README.md`](../tooling/README.md) for scripted deploy/init.

### 2. Initialize the DA for the user

1. Before `SetScript`, set **`pendingOwner`** on the DA account to the user’s EOA (base58).
2. Install the DA script (`SetScript`).
3. The **EOA** calls **`initAndRegister(registryAddress)`** on the DA.

The Registry then maps **EOA → DA** for `getActiveDA` / relayer use.

### 3. Allowlist the relayer on the DA (owner)

The relayer exposes its public key via **`GET /info`** (`relayerPubKey`). On-chain permission entrypoints take **`relayerPubKeyBase58`** as the first argument (not the relayer address string). The contract derives the relayer address from that pubkey for storage keys such as `allow_<relayerAddr>_...`.

Typical steps:

1. Store the relayer pubkey on the DA: `relayerPk_<relayerAddr>` must match (handled when you use the provided `approveMethods` / verifier flow in tests).
2. Call **`approveMethods`** (or **`allowAllOnDapp`**) for your **target dApp address** and the **method names** the relayer may call, with optional **`expireHeight`**.

Use **`waves-da-sdk`** owner helpers, e.g. `buildApproveMethodsTx`, from the owner’s wallet.

### 4. Configure the relayer

- Set **`REGISTRY_ADDRESS`**, **`RELAYER_SEED`**, node URL, chain ID, fees (see [`relayer/README.md`](../relayer/README.md)).
- Edit **`dappConfig.json`**: for your dApp address, add each callable you allow, with **`useOrigin`** and **`sponsorFee`** as required.

Example (REGULAR, user must reimburse fee on-chain; refund guard may run):

```json
{
  "3YourTargetDApp...": {
    "deposit": { "useOrigin": false, "sponsorFee": false },
    "withdraw": { "useOrigin": true, "sponsorFee": false }
  }
}
```

`withdraw` here uses VERIFIER (`useOrigin: true`) so your contract sees the expected **`originCaller`**.

### 5. Call via HTTP (`POST /invoke`)

The client sends **`eoa`**, **`targetDapp`**, **`function`**, **`args`**, and optional **`payments`** — not the execution mode, and **not** `reimburseFee` (the relayer sets fee refund on the built tx from `sponsorFee` / `useOrigin` in `dappConfig.json`).

```bash
curl -s -X POST http://localhost:3000/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "eoa": "3N...",
    "targetDapp": "3YourTargetDApp...",
    "function": "deposit",
    "args": [1000000]
  }'
```

If your method is VERIFIER in config, the relayer still only needs the same shape; it builds the VERIFIER tx internally.

### 6. Direct SDK usage (without HTTP)

Integrators can build transactions in the browser or backend using the same SDK the relayer uses.

**REGULAR** — relayer seed signs; empty `relayerPubKeyBase58`:

```ts
import { buildInvokeViaDA } from "waves-da-sdk";

const tx = await buildInvokeViaDA(
  nodeUrl,
  {
    chainId,
    registry: registryAddress,
    eoa: userAddress,
    useOrigin: false,
    feeRegular,
    feeVerifier,
  },
  {
    targetDapp: yourDapp,
    function: "deposit",
    args: [amount],
    reimburseFee: true,
    payments: [{ amount: paymentAmount }],
    relayerPubKeyBase58: "",
  },
  relayerSeed
);
```

**VERIFIER** — `useOrigin: true` and pass relayer pubkey; relayer seed still provides `proofs[0]`:

```ts
import { publicKey } from "@waves/ts-lib-crypto";
import { buildInvokeViaDA } from "waves-da-sdk";

const relayerPubKeyBase58 = publicKey(relayerSeed);

const tx = await buildInvokeViaDA(
  nodeUrl,
  {
    chainId,
    registry: registryAddress,
    eoa: userAddress,
    useOrigin: true,
    feeRegular,
    feeVerifier,
  },
  {
    targetDapp: yourDapp,
    function: "withdraw",
    args: [amount],
    reimburseFee: false,
    payments: [],
    relayerPubKeyBase58,
  },
  relayerSeed
);
```

Full runnable samples: [`sdk/examples/regular.ts`](../sdk/examples/regular.ts), [`sdk/examples/verifier.ts`](../sdk/examples/verifier.ts).

---

## Checklist for your dApp contract

- Document which entrypoints are intended for **DA + relayer** and whether they depend on **`caller`** vs **`originCaller`**.
- If you rely on **`originCaller`**, plan for **VERIFIER** and set **`useOrigin: true`** for those methods in `dappConfig.json`.
- For **REGULAR**, remember the target sees **`originCaller`** as the relayer unless you only check **`caller`** (the DA).

---

## References

| Document | Purpose |
|----------|---------|
| [`REGISTRY.md`](REGISTRY.md) | Canonical shared registry address per network |
| [`SPEC.md`](SPEC.md) | Storage schema, `proxy` arguments, Registry |
| [`../sdk/README.md`](../sdk/README.md) | SDK API and types |
| [`../relayer/README.md`](../relayer/README.md) | Env vars, `dappConfig.json`, HTTP errors |
| [`../contracts/da/README.md`](../contracts/da/README.md) | DA entrypoints and storage keys |
| [`../tooling/README.md`](../tooling/README.md) | On-chain deploy and E2E tests |
