# Quickstart — Intégrer DA Wallets dans votre dApp

Ce guide couvre l'essentiel pour connecter votre dApp au relayer et au SDK.

**Prérequis :** Registry déployé, utilisateur avec un DA wallet initialisé.  
Voir [INTEGRATION.md](INTEGRATION.md) pour le flow complet de création.

---

## 1. Setup du relayer (backend)

```bash
git clone https://github.com/Waves-Dapp-Abstraction/waves-da-relayer
cd waves-da-relayer
npm install
cp .env.example .env
```

Dans `.env`, configurez au minimum :

```env
NODE_URL=https://nodes-testnet.wavesnodes.com
CHAIN_ID=T
REGISTRY_ADDRESS=3N...   # adresse du Registry partagé (voir REGISTRY.md)
RELAYER_SEED=your seed phrase here
```

Créez `dappConfig.json` pour whitelist votre dApp :

```json
{
  "3PYourDappAddress...": {
    "deposit": { "useVerifierMode": false, "sponsorFee": false },
    "withdraw": { "useVerifierMode": true,  "sponsorFee": false }
  }
}
```

Démarrez le relayer :

```bash
npm start
```

Vérification : `GET http://localhost:3000/health` → `{"ok":true}`

---

## 2. Frontend — Authentifier et appeler via le relayer

Installez le SDK :

```bash
npm i waves-da-sdk
```

```ts
import { getActiveDAOrNull, RelayerAuthClient, RelayerSession } from "waves-da-sdk";
import { Signer } from "@waves/signer";
import { ProviderKeeper } from "@waves/provider-keeper";

const RELAYER_URL = "http://localhost:3000";
const REGISTRY    = "3N...";  // adresse Registry

const signer = new Signer({ NODE_URL: "https://nodes-testnet.wavesnodes.com" });
signer.setProvider(new ProviderKeeper());

// 1. Vérifier si l'utilisateur a un DA wallet
await signer.login();
const eoa = (await signer.getPublicKey()) // ou signer.address selon le provider
const da = await getActiveDAOrNull(NODE_URL, { registry: REGISTRY, eoa });

if (!da) {
  // Rediriger vers waves-da.com pour créer un DA wallet
  window.location.href = "https://waves-da.com";
  return;
}

// 2. Authentifier auprès du relayer
const authClient = new RelayerAuthClient(RELAYER_URL);
const session    = new RelayerSession();
const auth       = await authClient.loginAndAuthenticate(signer, session);
// Token JWT caché, réutilisé automatiquement lors des prochains appels

// 3. Appeler votre dApp via le relayer
const res = await fetch(`${RELAYER_URL}/invoke`, {
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
console.log("TX ID:", txId);
```

---

## 3. Références

| Besoin | Doc |
|--------|-----|
| Créer un DA wallet pour un utilisateur | [INTEGRATION.md](INTEGRATION.md) |
| Adresses Registry par network | [REGISTRY.md](REGISTRY.md) |
| Endpoints HTTP, codes d'erreur, auth | Relayer README |
| REGULAR vs VERIFIER, permissions | [INTEGRATION.md](INTEGRATION.md) |
| Schéma on-chain et spec | [SPEC.md](SPEC.md) |
