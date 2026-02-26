# Waves DA — SPEC v1 (Draft)

## 0. Goal

Provide **DApp Abstraction (DA)** on Waves:
- A **DA Wallet** (smart account / account dApp script) can execute `invoke()` calls on behalf of users.
- Users delegate permissions to relayers.
- Relayers submit transactions for better UX.
- Two execution modes are supported:
  - **REGULAR** (fee refund possible)
  - **VERIFIER** (originCaller-safe)

This spec is **Waves/Ride oriented**: it defines **entrypoints**, **parameter formats**, and **storage schema**.

---

## 1. Components

### 1.1 DA Wallet (account dApp script)
Ride script deployed on a Waves account. Provides:
- `@Callable` methods for init, permissions, proxy calls, withdrawals.
- `@Verifier` logic for VERIFIER mode.

### 1.2 Registry (account dApp script)
Ride script maintaining:
- `activeDA_<EOA> -> <DA address>`
- `activeDA_pk_<EOA> -> <DA publicKey>`

Registry is used by SDK/frontend to resolve the active DA for an EOA.

### 1.3 Relayer backend (off-chain)
Open-source HTTP service:
- session/auth
- builds & broadcasts transactions
- auto-selects REGULAR/VERIFIER via config
- optional `refundGuard` (kept in v1)

---

## 2. Execution modes

### 2.1 REGULAR mode
Goal: best UX and optional fee sponsoring.

- Transaction sender = relayer
- In DA: `i.caller` = relayer
- DA invokes target dApp and can optionally refund fee to relayer:
  - `ScriptTransfer(relayer, i.fee, i.feeAssetId)` if `reimburseFee=true`
- Limitation: NOT compatible with dApps relying on `originCaller`.

### 2.2 VERIFIER mode
Goal: originCaller compatibility.

- Transaction sender = DA
- Relayer signs tx; DA pays the fee
- In DA: `i.caller == this`
- Target dApp sees correct `originCaller` (DA)
- No fee refund (by design)
- Allowlisted relayers only:
  - relayer pubkey must be stored in DA state and verified in `@Verifier`

---

## 3. DA Wallet v1 — Interface & Data Schema

### 3.1 Storage keys (schema)

Global:
- `pendingOwner` : String (EOA address base58) — set before init/register
- `owner` : String (EOA address base58)
- `initialized` : Boolean
- `paused` : Boolean

Relayer PK registry:
- `relayerPk_<relayerAddr>` : String (relayer pubkey base58)

Permissions:
- `allow_<relayerAddr>_<dApp>_<method>` : Boolean
- `expire_<relayerAddr>_<dApp>_<method>` : Int (height; 0 = no expiry)
- `allowAll_<relayerAddr>_<dApp>` : Boolean
- `allowAllExp_<relayerAddr>_<dApp>` : Int

Payment caps:
- `maxPayUnit_<relayerAddr>_<dApp>_<method>` : Int (0 = unlimited)
- `maxPayAsset_<relayerAddr>_<dApp>_<method>_<assetId>` : Int (0 = unlimited)

Debug (non-contractual):
- `lastTargetDapp`, `lastTargetFn`, `lastRelayer`

### 3.2 Entrypoints (high level)

#### `initAndRegister(registryDapp: String)`
Preconditions:
- `initialized == false`
- `pendingOwner` exists
- `i.caller == pendingOwner`

Actions:
- invoke Registry `registerFromDA()`
- set `owner`, `initialized=true`, delete `pendingOwner`

#### `proxy(...)`
Purpose:
- execute `invoke(targetDapp, fn, args, payments)` with checks
- supports REGULAR and VERIFIER lanes

Parameters (v1 baseline aligned with POC, may be refined):
- `targetDapp: String` (address base58)
- `fn: String`
- `argRefs: List[Int]`
- `intArgs: List[Int]`
- `strArgs: List[String]`
- `boolArgs: List[Boolean]`
- `binArgs: List[ByteVector]`
- `reimburseFee: Boolean`
- `paymentAssetIds: List[String]` ("WAVES" or assetId base58)
- `paymentAmounts: List[Int]`
- `relayerPubKeyBase58: String` ("" => REGULAR, non-empty => VERIFIER)

Rules:
- reject if not initialized or paused
- determine lane:
  - REGULAR if `relayerPubKeyBase58 == ""`
  - VERIFIER otherwise (and enforced by `@Verifier`)
- resolve relayer identity:
  - REGULAR: `relayerAddr = i.caller`
  - VERIFIER: `relayerAddr = addressFromPublicKey(relayerPubKeyBase58)`
- permission checks:
  - allowAll OR allow(method) and not expired
- payments caps checks
- perform invoke
- REGULAR only: if `reimburseFee==true` then refund fee to relayer

Owner controls:
- approve/revoke methods
- allowAll/revokeAll per dApp
- set max payment caps
- pause/unpause
- withdraw waves/assets

### 3.3 Verifier (VERIFIER mode enforcement)
When tx calls `proxy` with non-empty `relayerPubKeyBase58`:
- reject if not initialized or paused
- check allowlist: `relayerPk_<relayerAddr> == relayerPubKeyBase58`
- verify proof signature: `sigVerify(tx.bodyBytes, tx.proofs[0], relayerPubKeyBytes)`

---

## 4. Registry v1 — Interface & Data Schema

### 4.1 Storage
- `activeDA_<eoaAddr>` : String (DA address base58)
- `activeDA_pk_<eoaAddr>` : String (DA pubkey base58)

### 4.2 Entrypoints

#### `registerFromDA()`
- `eoaStr = originCaller`
- `daStr = caller`
Checks:
- no existing active DA for `eoaStr`
- DA state contains `pendingOwner == eoaStr`
Writes:
- activeDA and activeDA_pk

#### `deleteMyDA()`
- called by EOA
- removes `activeDA_*` entries

---

## 5. Relayer backend — Contract (v1)

- Session auth via server challenge + TTL + anti-replay
- Project config drives mode selection:
  - `useOrigin=false` => REGULAR
  - `useOrigin=true` => VERIFIER
- Builds and broadcasts InvokeScript tx
- Optional `refundGuard` kept in v1 (feature flag)

---

## 6. Upgrade policy (Waves)
Waves supports upgrading an account script via **SetScript** while keeping the same address.
- v1: define an upgrade procedure (documented)
- future: optional multisig/timelock/recovery patterns