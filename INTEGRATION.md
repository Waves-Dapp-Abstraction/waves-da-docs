# Integration guide — dApp + DA

Builds on [QUICKSTART.md](QUICKSTART.md). Relayer HTTP API: [waves-da-relayer README](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer/blob/master/README.md).

---

## Concepts

| | Role |
|--|------|
| **DA wallet** | Script account that calls your dApp for the user |
| **Registry** | `EOA → DA` (one per network, [REGISTRY.md](REGISTRY.md)) |
| **Relayer** | HTTP: JWT auth + `POST /invoke` |
| **dappConfig.json** | Allowed methods + REGULAR or VERIFIER |

### REGULAR vs VERIFIER

| | REGULAR | VERIFIER |
|---|---------|----------|
| `originCaller` on your dApp | Relayer | DA |
| Network fee | Relayer (optional DA refund) | DA |
| Use when | You don't read `originCaller` | You need `originCaller` |

---

## 1. Does the user have a DA?

```ts
import { getActiveDAOrNull } from "waves-da-sdk";

const da = await getActiveDAOrNull(nodeUrl, {
  registry: "3MpHSUmakaCCcQkwATctWuChM6QkX3dBWAr",
  eoa: userAddress,
});

if (!da) {
  // Send user to DA creation (your UI or hosted manager)
}
```

---

## 2. Create a DA (optional)

- **Recommended:** host or link a DA Manager UI for end users.
- **In-app:** follow the four steps in the [sdk README](https://github.com/Waves-Dapp-Abstraction/waves-da-sdk/blob/master/README.md) (`randomSeed` → fund → deploy script → `initAndRegister`).

---

## 3. Approve the relayer (once per user)

The EOA signs on their DA:

```ts
import { buildApproveMethodsTx } from "waves-da-sdk";

const { relayerPubKey } = await fetch(`${RELAYER_URL}/info`).then((r) => r.json());

const tx = buildApproveMethodsTx(
  {
    chainId: 84,
    da: da.da,
    fee: 500_000,
    relayerPubKeyBase58: relayerPubKey,
    targetDapp: YOUR_DAPP_ADDRESS,
    methods: ["deposit", "withdraw"],
    expireHeight: 0,
  },
  signer
);
await signer.broadcast(tx);
```

---

## 4. Configure your relayer

On your [waves-da-relayer](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer) instance, edit `dappConfig.json`:

```json
{
  "3PYourDapp": {
    "deposit":  { "useVerifierMode": false, "sponsorFee": false },
    "withdraw": { "useVerifierMode": true,  "sponsorFee": false }
  }
}
```

Restart the relayer after changes.

---

## 5. Auth + invoke

```ts
import { RelayerAuthClient, RelayerSession } from "waves-da-sdk";

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
    targetDapp: YOUR_DAPP_ADDRESS,
    function: "deposit",
    args: [1_000_000],
    payments: [],
  }),
});
```

Full example: [examples/authFlow.ts](https://github.com/Waves-Dapp-Abstraction/waves-da-sdk/blob/master/examples/authFlow.ts)

---

## Troubleshooting

| Error | Likely cause |
|--------|----------------|
| `DA_NOT_FOUND` | No DA on Registry for this EOA |
| `401` | Expired JWT — call `loginAndAuthenticate` again |
| `403 METHOD_NOT_ALLOWED` | Method missing from `dappConfig.json` |
| `403 DAPP_NOT_WHITELISTED` | dApp address missing from `dappConfig.json` |
| Wrong `originCaller` | Set `useVerifierMode: true` for that method |
| On-chain reject | Missing `approveMethods` or wrong `relayerPubKey` |
| `REFUND_GUARD_FAILED` | REGULAR: DA lacks WAVES to refund relayer fee |

Advanced args: [ARG_TYPES.md](ARG_TYPES.md)
