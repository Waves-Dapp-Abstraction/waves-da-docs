# Supported Argument Types

> Only needed for `List`, `ByteVector`, or mixed args. Basic `Int` / `String` / `Boolean` → see [QUICKSTART.md](QUICKSTART.md).

The Waves DA system supports **all callable argument types** from the Waves blockchain:

- **Scalars**: Int, String, Boolean, ByteVector
- **Collections**: List[...] (with scalar elements)

This document shows how to encode each type when calling dApps through the relayer or SDK.

---

## Int

Arbitrary-precision 64-bit signed integer.

### JSON encoding
```json
42
```

### SDK (TypeScript)
```ts
args: [42]
```

### Relayer HTTP example
```json
{
  "eoa": "3N...",
  "targetDapp": "3N...",
  "function": "deposit",
  "args": [1000000]
}
```

---

## String

UTF-8 text, max ~7KB (depends on blockchain limits).

### JSON encoding
```json
"hello world"
```

### SDK (TypeScript)
```ts
args: ["hello world"]
```

### Relayer HTTP example
```json
{
  "args": ["my-memo"]
}
```

---

## Boolean

`true` or `false`.

### JSON encoding
```json
true
```

### SDK (TypeScript)
```ts
args: [true, false]
```

### Relayer HTTP example
```json
{
  "args": [true]
}
```

---

## ByteVector (Binary)

Raw binary data, encoded as **Base64 string** in JSON.

### JSON encoding
```json
{ "binary": "AQIDBA==" }
```

The value inside `"binary"` must be valid Base64 (no prefix required; `"base64:"` prefix is optional and stripped).

### SDK (TypeScript)

Use `Uint8Array`:

```ts
const data = new Uint8Array([1, 2, 3, 4]);
args: [data]
```

Internally, the SDK converts it to Base64 for JSON serialization.

### Relayer HTTP example

```json
{
  "args": [
    { "binary": "SGVsbG8gV29ybGQ=" }
  ]
}
```

Example: `SGVsbG8gV29ybGQ=` is Base64 for "Hello World".

### Generating Base64 in JavaScript

```js
const text = "Hello World";
const binary = new Uint8Array(text.split('').map(c => c.charCodeAt(0)));
const base64 = Buffer.from(binary).toString('base64');
console.log(base64); // "SGVsbG8gV29ybGQ="
```

---

## List[T]

Array of scalar elements. **Nested lists are not allowed.**

Supported element types:
- `List[Int]`
- `List[String]`
- `List[Boolean]`
- `List[ByteVector]`

### JSON encoding

```json
{ "list": [element1, element2, ...] }
```

### SDK (TypeScript)

```ts
// List[String]
args: [["tag1", "tag2", "tag3"]]

// List[Int]
args: [[10, 20, 30]]

// List[Boolean]
args: [[true, false, true]]

// List[ByteVector]
const data1 = new Uint8Array([1, 2, 3]);
const data2 = new Uint8Array([4, 5, 6]);
args: [[data1, data2]]
```

### Relayer HTTP example

#### List[String]
```json
{
  "args": [
    { "list": ["nft", "art", "rare"] }
  ]
}
```

#### List[Int]
```json
{
  "args": [
    { "list": [100, 200, 300] }
  ]
}
```

#### List[ByteVector]
```json
{
  "args": [
    { "list": [
      { "binary": "AQID" },
      { "binary": "BAUG" }
    ] }
  ]
}
```

#### List[Boolean]
```json
{
  "args": [
    { "list": [true, false, true] }
  ]
}
```

---

## Mixed Arguments Example

A single function call can mix multiple types:

### Ride function signature
```ride
@Callable(i)
func complexCall(
  count: Int,
  label: String,
  active: Boolean,
  data: ByteVector,
  tags: List[String],
  amounts: List[Int]
) = [
  // ... implementation
]
```

### SDK (TypeScript)
```ts
const tags = ["premium", "verified"];
const amounts = [1000000, 2000000, 3000000];
const binaryData = new Uint8Array([0xFF, 0x00, 0xAA]);

const response = await fetch("http://localhost:3000/invoke", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${token}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    eoa: auth.eoa,
    targetDapp: "3N...",
    function: "complexCall",
    args: [
      42,                      // Int
      "transaction-001",       // String
      true,                    // Boolean
      binaryData,              // ByteVector (will be Base64'd by SDK)
      tags,                    // List[String]
      amounts                  // List[Int]
    ],
  }),
});
```

### Relayer HTTP (JavaScript)
```js
const base64Binary = Buffer.from([0xFF, 0x00, 0xAA]).toString('base64');

const response = await fetch("http://localhost:3000/invoke", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${token}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    eoa: eoa,
    targetDapp: "3N...",
    function: "complexCall",
    args: [
      42,                                    // Int
      "transaction-001",                     // String
      true,                                  // Boolean
      { "binary": base64Binary },            // ByteVector
      { "list": ["premium", "verified"] },   // List[String]
      { "list": [1000000, 2000000, 3000000] } // List[Int]
    ],
  }),
});

const result = await response.json();
console.log("txId:", result.txId);
```

---

## Payments

In addition to `args`, you can attach **payments** (WAVES or other assets):

### SDK
```ts
const response = await fetch("http://localhost:3000/invoke", {
  method: "POST",
  headers: { "Authorization": `Bearer ${token}` },
  body: JSON.stringify({
    eoa: auth.eoa,
    targetDapp: "3N...",
    function: "deposit",
    args: [1000000],
    payments: [
      { "amount": 500000 },                    // WAVES
      { "amount": 100, "assetId": "DUC..." }   // Other asset
    ],
  }),
});
```

Max **2 payments** per transaction.

---

## Error Handling

### Invalid Base64
If binary data is not valid Base64:
```json
{
  "ok": false,
  "code": "VALIDATION_ERROR",
  "error": "args[0].binary must be a valid base64 string"
}
```

### Wrong arg count
```json
{
  "ok": false,
  "code": "VALIDATION_ERROR",
  "error": "Expected 4 args, got 3"
}
```

### Nested lists
```json
{
  "ok": false,
  "code": "VALIDATION_ERROR",
  "error": "Nested lists are not supported"
}
```

---

## Summary Table

| Type | JSON | SDK |
|------|------|-----|
| `Int` | `42` | `42` |
| `String` | `"hello"` | `"hello"` |
| `Boolean` | `true` | `true` |
| `ByteVector` | `{ "binary": "AQID" }` | `new Uint8Array([1,2,3,4])` |
| `List[Int]` | `{ "list": [1, 2, 3] }` | `[1, 2, 3]` |
| `List[String]` | `{ "list": ["a", "b"] }` | `["a", "b"]` |
| `List[Boolean]` | `{ "list": [true, false] }` | `[true, false]` |
| `List[ByteVector]` | `{ "list": [{ "binary": "AQ==" }] }` | `[new Uint8Array(...)]` |

---

## References

- **SDK**: [waves-da-sdk on npm](https://www.npmjs.com/package/waves-da-sdk)
- **Relayer HTTP API**: [waves-da-relayer README](https://github.com/Waves-Dapp-Abstraction/waves-da-relayer/blob/master/README.md) (HTTP API section)
- **Integration Guide**: See `docs/INTEGRATION.md`
- **Full Protocol Spec**: See `docs/SPEC.md`
