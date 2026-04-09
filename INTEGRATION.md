# Integration Guide — DApp Abstraction (DA)

Guide pour intégrer **DA Wallets** dans votre dApp Waves.

**TLDR :** Utilisateurs → DA Wallet → Relayer → Votre dApp.

---

## Concepts clés

### DA Wallet
- Smart account **unique par utilisateur**, lié à son adresse EOA
- Peut **appeler vos fonctions** via un relayer de confiance
- L'utilisateur signe une fois pour authentifier le relayer

### Registry
- Contrat qui mappe : `EOA → DA Wallet address`
- Permet au relayer de trouver le DA de l'utilisateur

### Relayer
- Service HTTP qui signe les transactions au nom de l'utilisateur
- Nécessite une authentification JWT (challenge-response)
- Gère les frais et l'économie des transactions

---

## Workflow complet

### 1. L'utilisateur a-t-il déjà un DA wallet ?

```ts
import { getActiveDAOrNull } from "waves-da-sdk";

const da = await getActiveDAOrNull(nodeUrl, {
  registry: "3N...",
  eoa: userAddress
});

if (!da) {
  // Créer un nouveau DA (voir étape 2)
} else {
  // DA existe, continuer à l'étape 3
}
```

### 2. Créer un DA wallet pour l'utilisateur

Si l'utilisateur n'a pas de DA, il faut le créer une seule fois.

```ts
import { randomSeed, address, publicKey } from "@waves/ts-lib-crypto";
import { broadcast, waitForTx } from "@waves/waves-transactions";
import {
  DA_RIDE_SOURCE,
  compileDaScript,
  buildDeployDATx,
  buildSetPendingOwnerDataTx,
  buildInitAndRegisterTx,
} from "waves-da-sdk";

// 1. Générer une adresse DA (nouvelle clé)
const daSeed = randomSeed(15);
const daAddress = address({ publicKey: publicKey(daSeed) }, chainId);

// 2. Financer le DA (transfert simple depuis l'EOA)
await signer.transfer({
  amount: 3000000,
  recipient: daAddress
}).broadcast();

// 3. Compiler et déployer le script DA
const compiled = await compileDaScript(nodeUrl, DA_RIDE_SOURCE);
const deployTx = buildDeployDATx(
  { chainId, fee: 1400000, compiledScript: compiled },
  daSeed
);
await waitForTx((await broadcast(deployTx, nodeUrl)).id, { apiBase: nodeUrl });

// 4. Définir l'EOA comme propriétaire
const dataTx = buildSetPendingOwnerDataTx(
  { chainId, fee: 500000, eoaAddress: userAddress },
  daSeed
);
await waitForTx((await broadcast(dataTx, nodeUrl)).id, { apiBase: nodeUrl });

// 5. Enregistrer le DA dans le Registry (depuis l'EOA)
await signer.invoke({
  dApp: daAddress,
  call: {
    function: "initAndRegister",
    args: [{ type: "string", value: registryAddress }],
  },
  payment: [],
}).broadcast();

console.log("DA wallet créé :", daAddress);
```

> **Important :** Montrez à l'utilisateur la **graine du DA** et demandez-lui de la sauvegarder (pour la récupération).

### 3. Authentifier l'utilisateur avec le relayer

```ts
import { RelayerAuthClient, RelayerSession } from "waves-da-sdk";
import { Signer } from "@waves/signer";
import { ProviderKeeper } from "@waves/provider-keeper";

const signer = new Signer({ NODE_URL: "https://nodes-testnet.wavesnodes.com" });
signer.setProvider(new ProviderKeeper());

const authClient = new RelayerAuthClient("http://localhost:3000");
const session = new RelayerSession();

// Une seule étape : wallet login + authentification relayer
const auth = await authClient.loginAndAuthenticate(signer, session);

// Token automatiquement caché et réutilisé
console.log("Authentifié :", auth.eoa);
```

### 4. Configurer la permission du relayer (une seule fois)

Le propriétaire du DA doit autoriser le relayer à appeler certaines fonctions.

```ts
import { buildApproveMethodsTx } from "waves-da-sdk";

// L'EOA (propriétaire du DA) signe :
const approveTx = buildApproveMethodsTx(
  {
    chainId,
    da: daAddress,          // adresse du DA
    fee: 500000,
    relayerPubKeyBase58: relayerPublicKey,  // depuis GET /info du relayer
    targetDapp: yourDappAddress,
    methods: ["deposit", "withdraw"],
    expireHeight: 0,        // 0 = pas d'expiry
  },
  signer
);

await signer.broadcast(approveTx);
```

### 5. Appeler votre dApp via le relayer

Une fois authentifié (étape 3) :

```ts
const response = await fetch("http://localhost:3000/invoke", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${auth.token}`,
  },
  body: JSON.stringify({
    eoa: auth.eoa,
    targetDapp: "3YourDAppAddress...",
    function: "deposit",
    args: [1000000],
    payments: [{ amount: 5000000 }],
  }),
});

const result = await response.json();
console.log("Tx ID:", result.txId);
```

---

## Checklist pour votre dApp

- [ ] Utilisateurs peuvent-ils checker s'ils ont un DA ? (via `getActiveDAOrNull`)
- [ ] Avez-vous un bouton "Créer un DA" pour les nouveaux utilisateurs ?
- [ ] L'authentification relayer est-elle seamless (une seule connexion wallet) ?
- [ ] Les fonctions que vous appelez via relayer acceptent-elles le DA comme appelant ?

---

## Dépannage

**Q: "User doesn't have a DA"**  
A: Lancez l'étape 2 (création du DA). C'est un one-time setup par utilisateur.

**Q: "401 Unauthorized"**  
A: Token expiré ou invalide. Rappelez au signer de se reconnecter via `loginAndAuthenticate`.

**Q: "originCaller n'est pas l'utilisateur"**  
A: Vous voyez l'adresse du relayer. C'est normal en **REGULAR mode**. Demandez au relayer de basculer en **VERIFIER mode** pour votre méthode.

---

## Références

- [`sdk/README.md`](../sdk/README.md) — API complète du SDK
- [`relayer/README.md`](../relayer/README.md) — Setup et configuration du relayer
- [`REGISTRY.md`](REGISTRY.md) — Adresses Registry per network
