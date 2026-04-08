# Quickstart — relayer (back) + app (front)

Minimal steps to connect an app to the **HTTP relayer** and, on the frontend, use **`waves-da-sdk`** when it helps.  
For full detail (REGULAR/VERIFIER modes, on-chain schema, errors, caps), see [`INTEGRATION.md`](INTEGRATION.md) and [`SPEC.md`](SPEC.md).

**Assumptions:** Registry and DA are deployed, the user has an initialized DA, and the relayer is **allowed** on the DA for your dApp and callable (see tooling or [`INTEGRATION.md`](INTEGRATION.md)). Use the **same canonical registry address** everywhere (`REGISTRY_ADDRESS` in the relayer and in `getActiveDA`) — see [`REGISTRY.md`](REGISTRY.md).

---

## 1. Relayer (backend)

```bash
cd relayer
npm install
cp .env.example .env
```

In **`.env`**, set at least `REGISTRY_ADDRESS`, `RELAYER_SEED`, and `NODE_URL` / `CHAIN_ID` if needed.

Create or edit **`dappConfig.json`** in the relayer root (or set `DAPP_CONFIG_PATH`):

```json
{
  "3PYourDappAddressHere": {
    "myMethod": { "useVerifierMode": false, "sponsorFee": false }
  }
}
```

- Replace `3PYourDappAddressHere` and `myMethod` with your dApp address and callable name.
- `useVerifierMode: false` is **REGULAR** mode (common default).
- Fee refund on the built tx (`reimburseFee` passed into `DA.proxy`) is decided **only by the relayer** from `useVerifierMode` + `sponsorFee` — the HTTP client does **not** send `reimburseFee`.

Run:

```bash
npm start
```

Check: `GET http://localhost:3000/health` → `{"ok":true}`.

---

## 2. Frontend — call the relayer (with authentication)

The relayer requires **JWT authentication** for all `/invoke` calls. Use the SDK's `RelayerAuthClient` for seamless login + authenticate in one flow:

```ts
import { RelayerAuthClient, RelayerSession } from "waves-da-sdk";
import { Signer } from "@waves/signer";
import { ProviderKeeper } from "@waves/provider-keeper";

// Setup
const signer = new Signer({ NODE_URL: "https://nodes-testnet.wavesnodes.com" });
signer.setProvider(new ProviderKeeper());

const authClient = new RelayerAuthClient("http://localhost:3000");
const session = new RelayerSession();

// One-liner: login → authenticate (no intermediate steps!)
const auth = await authClient.loginAndAuthenticate(signer, session);

// Now send invoke with the token
const res = await fetch("http://localhost:3000/invoke", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${auth.token}`, // Token from auth
  },
  body: JSON.stringify({
    eoa: auth.eoa,
    targetDapp: "3PYourDappAddressHere",
    function: "myMethod",
    args: [1, "hello", true],
  }),
});
const data = await res.json();
// typical success: { ok: true, mode: "regular", txId: "..." }
```

**Next calls:** Token is cached in localStorage; `loginAndAuthenticate()` reuses it if valid (zero signing!)

---

## 3. Frontend — SDK (`waves-da-sdk`) for advanced usage

If you need to build transactions yourself (without the HTTP relayer):

Install: `npm i waves-da-sdk`

**Common case:** read or display the user’s **active DA** (Registry lookup via the node).

```ts
import { getActiveDA } from "waves-da-sdk";

const { da, daPubKey } = await getActiveDA(NODE_URL, {
  registry: REGISTRY_ADDRESS,
  eoa: userAddressBase58,
});
```

**Advanced:** you build and broadcast a `proxy` tx yourself (without the HTTP relayer) — see `buildInvokeViaDA` in [`../sdk/README.md`](../sdk/README.md) and [`../sdk/examples/`](../sdk/examples/).

---

## 4. Where to go next

| Need | Doc |
|------|-----|
| Canonical **registry** address (one per network) | [`REGISTRY.md`](REGISTRY.md) |
| Full endpoints / error codes / auth details | [`../relayer/README.md`](../relayer/README.md) and [`../relayer/AUTH.md`](../relayer/AUTH.md) |
| SDK API (signatures, owner txs, RelayerAuthClient) | [`../sdk/README.md`](../sdk/README.md) |
| End-to-end flow, permissions, REGULAR vs VERIFIER | [`INTEGRATION.md`](INTEGRATION.md) |
| On-chain schema, `proxy`, Registry | [`SPEC.md`](SPEC.md) |
