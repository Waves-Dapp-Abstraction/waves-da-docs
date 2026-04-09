# Integration Guide — DApp Abstraction (DA)

Guide complet pour intégrer DA Wallets dans votre dApp Waves.

---

## Concepts

### DA Wallet
- Smart account **unique par utilisateur**, lié à son EOA
- Proxy les invocations vers votre dApp via un relayer autorisé
- L'utilisateur garde le contrôle (permissions, caps, pause)

### Registry
- Contrat partagé qui mappe `EOA → DA address`
- Même Registry pour tous les projets sur un même network
- Voir [REGISTRY.md](REGISTRY.md) pour les adresses

### Relayer
- Service HTTP qui signe et broadcast les transactions
- Nécessite JWT (challenge-response)
- Configuré par dApp et par méthode via `dappConfig.json`

### REGULAR vs VERIFIER

| | REGULAR | VERIFIER |
|---|---------|----------|
| **Sender** | Relayer | DA |
| **`caller` sur votre dApp** | DA | DA |
| **`originCaller` sur votre dApp** | Relayer | DA |
| **Utiliser quand** | Vous n'utilisez pas `originCaller` | Vous avez besoin de `originCaller` |
| **Fee** | Relayer paie (optionnel remboursement DA) | DA paie |

---

## Flow complet

### 1. L'utilisateur a-t-il un DA wallet ?

```ts
import { getActiveDAOrNull } from "waves-da-sdk";

const da = await getActiveDAOrNull(nodeUrl, {
  registry: REGISTRY_ADDRESS,
  eoa: userAddress
});

if (!da) {
  // Option A : Rediriger vers waves-da.com
  // Option B : Flow de création dans votre app (voir section 2)
}
```

---

### 2. Créer un DA wallet (intégré dans votre app)

Si vous voulez intégrer la création de DA wallet dans votre flow, voici les 4 étapes. Sinon, redirigez vers **waves-da.com**.

```ts
import { randomSeed, address, publicKey } from "@waves/ts-lib-crypto";
import { broadcast, waitForTx } from "@waves/waves-transactions";
import {
  DA_RIDE_SOURCE,
  compileDaScript,
  buildDeployDATx,
  buildSetPendingOwnerDataTx,
} from "waves-da-sdk";

// Étape 1 : Générer le compte DA (côté client uniquement)
const daSeed    = randomSeed(15);
const daAddress = address({ publicKey: publicKey(daSeed) }, chainId);
// ⚠️ Affichez la seed à l'utilisateur et demandez-lui de la sauvegarder

// Étape 2 : Financer le DA (depuis le wallet de l'utilisateur)
await signer.transfer({ amount: 3_000_000, recipient: daAddress }).broadcast();

// Étape 3 : Déployer le script DA
const compiled = await compileDaScript(nodeUrl, DA_RIDE_SOURCE);
const deployTx = buildDeployDATx(
  { chainId, fee: 1_400_000, compiledScript: compiled },
  daSeed
);
await waitForTx((await broadcast(deployTx, nodeUrl)).id, { apiBase: nodeUrl });

// Note: buildSetPendingOwnerDataTx n'est plus nécessaire séparément,
// le pendingOwner est passé via buildDeployDATx depuis la v0.2+

// Étape 4 : Enregistrer dans le Registry (l'utilisateur signe)
await signer.invoke({
  dApp: daAddress,
  call: {
    function: "initAndRegister",
    args: [{ type: "string", value: REGISTRY_ADDRESS }],
  },
  payment: [],
}).broadcast();

console.log("DA wallet créé :", daAddress);
```

---

### 3. Autoriser le relayer sur le DA (une seule fois par utilisateur)

Le propriétaire du DA (l'EOA) doit autoriser le relayer à appeler certaines méthodes.

```ts
import { buildApproveMethodsTx } from "waves-da-sdk";

// Récupérer la pubkey du relayer
const { relayerPubKey } = await fetch("http://your-relayer/info").then(r => r.json());

// Autoriser les méthodes
const approveTx = buildApproveMethodsTx(
  {
    chainId,
    da: daAddress,
    fee: 500_000,
    relayerPubKeyBase58: relayerPubKey,
    targetDapp: YOUR_DAPP_ADDRESS,
    methods: ["deposit", "withdraw"],
    expireHeight: 0,  // 0 = pas d'expiry
  },
  signer
);

await signer.broadcast(approveTx);
```

---

### 4. Configurer le relayer

Editez `dappConfig.json` sur votre relayer :

```json
{
  "3PYourDappAddress...": {
    "deposit":  { "useVerifierMode": false, "sponsorFee": false },
    "withdraw": { "useVerifierMode": true,  "sponsorFee": false }
  }
}
```

- `useVerifierMode: false` → **REGULAR** (utilisez si vous n'avez pas besoin de `originCaller`)
- `useVerifierMode: true`  → **VERIFIER** (DA visible comme `originCaller` sur votre dApp)

---

### 5. Authentifier et appeler via le relayer

```ts
import { RelayerAuthClient, RelayerSession } from "waves-da-sdk";

const authClient = new RelayerAuthClient("http://your-relayer");
const session    = new RelayerSession();

// Une seule étape : login + auth + token JWT
const auth = await authClient.loginAndAuthenticate(signer, session);

// Appeler votre dApp
const res = await fetch("http://your-relayer/invoke", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${auth.token}`,
  },
  body: JSON.stringify({
    eoa: auth.eoa,
    targetDapp: "3PYourDappAddress...",
    function: "deposit",
    args: [1000000],
    payments: [{ amount: 5000000 }],
  }),
});

const { txId } = await res.json();
```

---

## Checklist

- [ ] Vérifier si l'utilisateur a un DA : `getActiveDAOrNull`
- [ ] Gérer le cas "pas de DA" : rediriger ou flow de création
- [ ] Autoriser le relayer sur le DA (une fois par utilisateur)
- [ ] Configurer `dappConfig.json` pour vos méthodes
- [ ] Implémenter l'auth JWT dans votre frontend

---

## Dépannage

**"User has no DA"**
→ L'utilisateur n'a pas encore de DA wallet. Redirigez vers waves-da.com ou lancez le flow de création.

**"401 Unauthorized"**
→ Token expiré. `loginAndAuthenticate` re-authentifie automatiquement.

**"403 METHOD_NOT_ALLOWED"**
→ La méthode n'est pas dans votre `dappConfig.json`. Ajoutez-la.

**"`originCaller` n'est pas l'utilisateur"**
→ Vous êtes en REGULAR mode. Passez à `useVerifierMode: true` pour cette méthode.
