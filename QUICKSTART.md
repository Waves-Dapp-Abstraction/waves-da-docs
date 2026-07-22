# Quickstart — integrate DA in your dApp

> **Beta** — testnet and mainnet supported. Not audited. Match `NODE_URL`, `CHAIN_ID`, and `REGISTRY_ADDRESS` to the same network ([REGISTRY.md](REGISTRY.md)).

**Prerequisite:** user already has a DA registered on the Registry for your network.

If not, either:

- **Redirect to [waves-da.com](https://waves-da.com/)** (hosted DA Manager — create, permissions, caps), or
- **Build in-app** — see [INTEGRATION.md](INTEGRATION.md) (SDK deploy flow or your own UI).

---

## 1. Relayer (your backend)

Each project **self-hosts** its own relayer.

```bash
git clone https://github.com/Waves-Dapp-Abstraction/waves-da-relayer.git
cd waves-da-relayer
npm install
cp .env.example .env
```

**Testnet** minimum `.env`:

```env
NODE_URL=https://nodes-testnet.wavesnodes.com
CHAIN_ID=84
REGISTRY_ADDRESS=3MpHSUmakaCCcQkwATctWuChM6QkX3dBWAr
RELAYER_SEED=<seed with WAVES on testnet>
JWT_SECRET=<random string for dev>
```

**Mainnet** — same shape with `CHAIN_ID=87`, mainnet node, and mainnet registry (`3PR3W9fvDRsDv63enfmhVKK2S3TNDJBQ57j`). Use [`.env.mainnet.example`](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer/blob/master/.env.mainnet.example) and set `PRODUCTION=true` for public traffic.

`dappConfig.json` — whitelist your dApp and methods:

```json
{
  "3PYourDappAddress": {
    "myMethod": { "useVerifierMode": false, "sponsorFee": false }
  }
}
```

```bash
npm start
```

| Check | URL |
|-------|-----|
| Health | `GET http://localhost:3000/health` |
| Relayer pubkey (`approveMethods`) | `GET http://localhost:3000/info` → `relayerPubKey` |

Relayer setup details: [waves-da-relayer README](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer/blob/master/README.md)

---

## 2. SDK (your frontend)

```bash
npm i waves-da-sdk @waves/signer @waves/provider-keeper
```

Example (mainnet — use testnet values from [REGISTRY.md](REGISTRY.md) for dev):

```ts
import { getActiveDAOrNull, RelayerAuthClient, RelayerSession } from "waves-da-sdk";
import { Signer } from "@waves/signer";
import { ProviderKeeper } from "@waves/provider-keeper";

const NODE_URL = "https://nodes.wavesnodes.com";
const REGISTRY = "3PR3W9fvDRsDv63enfmhVKK2S3TNDJBQ57j";
const RELAYER_URL = "https://your-relayer.example.com";

const signer = new Signer({ NODE_URL });
signer.setProvider(new ProviderKeeper());

const { address: eoa } = await signer.login();

const da = await getActiveDAOrNull(NODE_URL, { registry: REGISTRY, eoa });
if (!da) throw new Error("No DA wallet");

// Once per user: approve relayer on DA — see INTEGRATION.md

const auth = await new RelayerAuthClient(RELAYER_URL).loginAndAuthenticate(
  signer,
  new RelayerSession()
);

const res = await fetch(`${RELAYER_URL}/invoke`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${auth.token}`,
  },
  body: JSON.stringify({
    eoa: auth.eoa,
    targetDapp: "3PYourDappAddress",
    function: "myMethod",
    args: [],
    payments: [],
  }),
});

const data = await res.json();
if (!res.ok) throw new Error(data.error ?? res.statusText);
console.log(data.txId, data.mode);
```

---

## 3. Checklist

- [ ] `getActiveDAOrNull` — or create DA first (same network as relayer)
- [ ] `approveMethods` on DA with `relayerPubKey` from `/info`
- [ ] Method listed in `dappConfig.json`
- [ ] Correct `useVerifierMode` if you use `originCaller`

---

## Next

| Topic | Doc |
|-------|-----|
| Approve, errors, DA creation | [INTEGRATION.md](INTEGRATION.md) |
| List / binary args | [ARG_TYPES.md](ARG_TYPES.md) |
| HTTP API | [relayer README](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer/blob/master/README.md) |
| Auth details | [relayer AUTH.md](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer/blob/master/AUTH.md) |
| Example code | [sdk authFlow.ts](https://github.com/Waves-Dapp-Abstraction/waves-da-sdk/blob/master/examples/authFlow.ts) |
