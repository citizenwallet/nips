NIP-XX
======

Account-Abstraction Verification (AAV)
--------------------------------------

`draft` `optional`

This NIP defines a method for attributing a Nostr event to a **virtual account** (`account_id`) even though the event is **signed by a different controller key**. The virtual account does not have a private key. Instead, an **owner key** publishes a signed policy authorizing one or more controller keys to act for it under certain constraints.

Relays and clients that implement this NIP perform **additional verification** when the event contains an **Account-Abstraction Proof** (`aa` tag). If verification passes, the event is indexed and rendered as authored by the `account_id`. Non-supporting implementations will still treat it as a valid NIP-01 event signed by the controller.

## Motivation

NIP-01 requires that an event’s `pubkey` matches the key used to sign it. This prevents:
- Accounts without signing keys (derived identities).
- Managed accounts with provider-mediated access.
- Policy-driven accounts with time bounds, kind scoping, multiple controllers, or M-of-N requirements.

NIP-26 delegation requires the base account to sign a delegation, which is not possible if the account lacks a private key. This NIP introduces:
- A standard **owner-signed account descriptor** (policy).
- A proof tag on authored events.
- Relay/client rules for indexing, querying, and revoking controllers.

## Event kinds

- **10500**: Account Descriptor (Policy) — signed by the owner key.
- **10501**: Account Revocation — signed by the owner key.

Kinds are provisional until final assignment.

## Account Descriptor (kind: 10500)

An **Account Descriptor** defines a virtual account and its authorized controllers.

**Tags:**
- `["aa-account", <account_id>]` — 32-byte hex identifier for the virtual account.
- `["aa-controllers", <json>]` — JSON object:
```json
{
  "controllers": [
    {
      "pk": "<controller_pk_hex>",
      "kinds": [1, 6],
      "not_before": 1736600000,
      "not_after": 1737500000
    }
  ],
  "threshold": 1
}
```
- Optional: `["aa-derivation", <json>]` — helps deterministic derivation without revealing the salt:
```json
{"owner": "<owner_pk_hex>", "h": "<sha256(salt || owner_pk)>"}
```

**Example:**
```json
{
  "kind": 10500,
  "pubkey": "<owner_pk_hex>",
  "created_at": 1736610000,
  "tags": [
    ["aa-account", "2d6f9c0cfe1a...a9b2"], 
    ["aa-controllers", "{\"controllers\":[{\"pk\":\"<Y1_hex>\",\"kinds\":[1],\"not_before\":1736600000,\"not_after\":1737500000}],\"threshold\":1}"],
    ["aa-derivation", "{\"owner\":\"<owner_pk_hex>\",\"h\":\"9c8c2b...\"}"]
  ],
  "content": "AAV policy for Alice",
  "sig": "<sig_by_owner_pk>"
}
```

## Account Revocation (kind: 10501)

Revokes controller(s) or the entire account.

**Tags:**
- `["aa-account", <account_id>]`
- `["aa-revoked-at", <unix_timestamp>]`
- Optional: `["aa-revoke-controller", <controller_pk_hex>]` — if absent, revokes all controllers.

**Example:**
```json
{
  "kind": 10501,
  "pubkey": "<owner_pk_hex>",
  "created_at": 1736620000,
  "tags": [
    ["aa-account", "2d6f9c0cfe1a...a9b2"],
    ["aa-revoke-controller", "<Y1_hex>"],
    ["aa-revoked-at", "1736620000"]
  ],
  "sig": "<sig_by_owner_pk>"
}
```

## Account-Abstraction Proof tag on authored events

Events signed by controllers include an `aa` tag proving authority to act for the account.

Format:
```
["aa", <account_id>, <owner_pk_hex>, <policy_event_id>, <owner_sig_over_policy_event_id>]
```

- `policy_event_id`: ID of the latest valid policy event (`10500`).
- `owner_sig_over_policy_event_id`: Schnorr signature by `owner_pk` over the `policy_event_id`.

**Example:**
```json
{
  "kind": 1,
  "pubkey": "<Y1_hex>",
  "created_at": 1736610500,
  "tags": [
    ["aa", "2d6f9c0cfe1a...a9b2", "<owner_pk_hex>", "<policy_event_id>", "<sig_by_owner_over_policy_id>"]
  ],
  "content": "Hello from Account X (via controller Y1)",
  "sig": "<sig_by_Y1_over_event_id>"
}
```

## Verification

1. **Base NIP-01 check**: `verify_schnorr(sig, id, pubkey)` must pass.
2. If no `aa` tag → treat as normal NIP-01 event.
3. If `aa` tag present:
   - Verify `owner_sig` over `policy_event_id`.
   - Retrieve `10500` policy:
     - Signed by `owner_pk`.
     - `aa-account` matches `account_id`.
     - Controller list contains `pubkey` with allowed kind/time.
     - If `threshold > 1`, check co-signatures (see **Threshold**).
   - Ensure no `10501` revocation invalidates the controller at `created_at`.
4. If all pass, index and render author = `account_id`.

## Threshold (optional)

If `threshold > 1` in policy, authored event MUST include `aa-m` tag:
```
["aa-m", "<threshold_int>", "<pk2>", "<sig2>", "<pk3>", "<sig3>", ...]
```
Verify all co-signatures against the event `id`, ensure they are allowed controllers, and that the total unique valid controllers ≥ threshold.

## Queries and Indexing

- REQ with `authors: [<account_id>]` SHOULD return events with valid AAV proofs for that account.
- Relays SHOULD dual-index by `account_id` and controller `pubkey`.

## Derivation of account_id

Optionally:
```
account_id = sha256("nostr-aa:account" || salt || owner_pk)
```
The salt SHOULD be secret or hashed to avoid enumeration.

## Security Considerations

- No account private key reduces custodial risk.
- Compromise of `owner_pk` compromises all accounts under it.
- Use time bounds to limit replay.
- Deterministic derivations can create linkability; use KDF/pepper if privacy needed.

## Backward Compatibility

AAV is additive. Non-supporting implementations ignore `aa` tags and display the controller as author. Events remain valid NIP-01 events.

## Reference Implementation

A TypeScript verifier and indexing helper is provided in the appendix or project repository.
