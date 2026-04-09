# Specification technique — DApp Abstraction (DA) v1

Pour les intégrateurs qui veulent comprendre le protocole en détail.

**Pour débuter rapidement**, voir [QUICKSTART.md](QUICKSTART.md) ou [INTEGRATION.md](INTEGRATION.md).

---

## Architecture

```
EOA (utilisateur)
  └── DA Wallet (smart account)
        └── via proxy() → Target dApp
              (signé par le Relayer)
```

### Composants

**DA Wallet** — Script Ride déployé sur un compte Waves :
- Proxy les invocations vers des dApps whitelistées
- Gère les permissions par relayer, par dApp, par méthode
- Supporte deux modes d'exécution : REGULAR et VERIFIER

**Registry** — Contrat partagé :
- Mappe `EOA → DA address` et `EOA → DA pubkey`
- Utilisé par le SDK et le relayer pour résoudre le DA d'un utilisateur

**Relayer** — Service HTTP off-chain :
- Build et broadcast les transactions `InvokeScript`
- Applique les permissions via `dappConfig.json`
- Gère l'authentification JWT (challenge-response)

---

## Modes d'exécution

### REGULAR
- Sender de la tx = Relayer
- `caller` sur target dApp = DA
- `originCaller` sur target dApp = Relayer
- DA peut rembourser les frais au relayer (`reimburseFee: true`)
- A utiliser quand votre dApp ne dépend pas de `originCaller`

### VERIFIER
- Sender de la tx = DA
- `caller` sur target dApp = DA
- `originCaller` sur target dApp = DA
- Relayer signe via `proofs[0]`, vérifié dans le `@Verifier` du DA
- A utiliser quand votre dApp utilise `originCaller`

---

## Schéma de storage du DA Wallet

### Global
| Clé | Type | Description |
|-----|------|-------------|
| `owner` | String | Adresse EOA du propriétaire |
| `pendingOwner` | String | Propriétaire en attente (avant init) |
| `initialized` | Boolean | DA initialisé et enregistré |
| `paused` | Boolean | DA en pause |

### Relayer
| Clé | Type | Description |
|-----|------|-------------|
| `relayerPk_<relayerAddr>` | String | Pubkey du relayer (pour VERIFIER) |

### Permissions
| Clé | Type | Description |
|-----|------|-------------|
| `allow_<relayerAddr>_<dApp>_<method>` | Boolean | Méthode autorisée |
| `expire_<relayerAddr>_<dApp>_<method>` | Int | Hauteur d'expiry (0 = jamais) |
| `allowAll_<relayerAddr>_<dApp>` | Boolean | Toutes les méthodes autorisées |
| `allowAllExp_<relayerAddr>_<dApp>` | Int | Expiry pour allowAll |

### Payment caps
| Clé | Type | Description |
|-----|------|-------------|
| `maxPayUnit_<relayerAddr>_<dApp>_<method>` | Int | Max WAVES (0 = illimité) |
| `maxPayAsset_<relayerAddr>_<dApp>_<method>_<assetId>` | Int | Max asset (0 = illimité) |

---

## Entrypoints du DA Wallet

### `initAndRegister(registryDapp)`
Initialise le DA et l'enregistre dans le Registry.
- Appelé par l'EOA (propriétaire)
- Prérequis : `pendingOwner` défini, `initialized == false`

### `proxy(targetDapp, fn, argRefs, intArgs, strArgs, boolArgs, binArgs, reimburseFee, paymentAssetIds, paymentAmounts, relayerPubKeyBase58)`
Exécute une invocation via le DA.
- `relayerPubKeyBase58 == ""` → REGULAR
- `relayerPubKeyBase58 != ""` → VERIFIER

### Gestion des permissions
- `approveMethods(relayerPubKey, targetDapp, methods, expireHeight)`
- `revokeMethods(relayerPubKey, targetDapp, methods)`
- `allowAllOnDapp(relayerPubKey, targetDapp, expireHeight)`
- `revokeAllOnDapp(relayerPubKey, targetDapp)`
- `pause()` / `unpause()`

### Retraits
- `withdrawWaves(amount)`
- `withdrawAsset(assetId, amount)`

---

## Entrypoints du Registry

### `registerFromDA()`
Enregistre un DA dans le Registry.
- Appelé par le DA via `initAndRegister`
- `originCaller` = EOA, `caller` = DA

### `deleteMyDA()`
Supprime le mapping EOA → DA.
- Appelé par l'EOA directement

---

## Configuration relayer (`dappConfig.json`)

```json
{
  "<dappAddress>": {
    "<methodName>": {
      "useVerifierMode": false,
      "sponsorFee": false
    }
  }
}
```

| Champ | Description |
|-------|-------------|
| `useVerifierMode` | `false` = REGULAR, `true` = VERIFIER |
| `sponsorFee` | `true` = relayer paie sans remboursement. `false` = DA rembourse le relayer |

---

## Endpoints HTTP du relayer

### `POST /invoke`
```json
{
  "eoa": "3N...",
  "targetDapp": "3P...",
  "function": "myMethod",
  "args": [1, "hello", true],
  "payments": [{ "amount": 1000000 }]
}
```

Headers : `Authorization: Bearer <jwt-token>`

Réponse succès :
```json
{ "ok": true, "mode": "regular", "txId": "..." }
```

### `GET /health`
```json
{ "ok": true }
```

### `GET /info`
```json
{ "relayerAddress": "3N...", "relayerPubKey": "..." }
```

### Codes d'erreur
| Code | Signification |
|------|--------------|
| `DAPP_NOT_WHITELISTED` | dApp non configurée dans dappConfig.json |
| `METHOD_NOT_ALLOWED` | Méthode non configurée |
| `NO_DA_FOUND` | L'EOA n'a pas de DA wallet |
| `401` | Token JWT manquant ou invalide |
