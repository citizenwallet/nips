NIP-XX
======

Multi-sig Accounts
--------------------------------------

`draft` `optional`

Allow a group of accounts to define how they will sign messages together.

This NIP defines a method for attributing a Nostr event to a **multi-sig account** (`account_id`) even though the event is **signed by a different signer key**. The virtual account does not have a private key. Instead, a signed policy is published authorizing one or more signer keys to act for it under certain constraints. This policy can be published by anyone as the signed policy includes the signers, threshold and signatures to validate itself.

Relays and clients that implement this NIP perform **additional verification** when the event contains an **Account-Abstraction Proof** (`aa` tag). If verification passes, the event is indexed and rendered as authored by the `account_id`. Non-supporting implementations will still treat it as a valid NIP-01 event signed by the signer.

## Motivation

Allow account abstraction without needing to build private key bunkers in the cloud and without breaking NIP-01 (separation of the account and the credentials).

Enable accounts that require multiple signatures in order to publish (can be a single person or a group of people).

Enable accounts that can be recovered.

Enable accounts that can be easily shared across apps without needing to reveal/export private keys.

This NIP introduces:
- Multi-sig accounts with policy-mediated access.
- Policy-driven accounts with multiple controllers, or M-of-N requirements.
- A standard **multi-sig account descriptor** (policy).
- A proof tag on authored events.
- Relay/client rules for indexing, querying, and state management.

NIP-01 requires that an event's `pubkey` matches the key used to sign it. Limitations:
- Requires that the pubkey in the event matches the key used to sign the event.
- No native support for events authored by a "multi-sig" or policy-controlled account.
- Cannot represent accounts without a corresponding private key.
- No built-in way to delegate control or authorize other keys to publish for an account.
- Does not allow multi-controller or threshold signing without treating them as separate independent accounts.
- All identity is bound directly to possession of a single secp256k1 keypair.

NIP-26 delegation requires the base account to sign a delegation, which is not possible if the account lacks a private key. Limitations:
- Assumes the delegator (authorizing account) can sign the delegation — impossible if the account has no private key.
- Delegations are static: no built-in revocation before expiry unless clients/relays add extra logic.
- Does not define how to discover or represent "multi-sig accounts" — delegates only to existing pubkeys.
- Lacks a standard way to handle multiple simultaneous controllers or threshold requirements.
- No mechanism for deterministic account derivation (e.g., from salt + provider key) to link an account to an identity provider.
- Revocation and renewal are relay-specific; not universally enforced across the network.

## Event kinds

- **10500**: Multi-sig Account Descriptor (Policy) — signed by anyone as long as the policy is valid.

Kinds are provisional until final assignment.

## Multi-sig Account Descriptor (kind: 10500)

A **Multi-sig Account Descriptor** defines a multi-sig account and its authorized signers. It includes all the information to validate itself. The **latest** policy event for an account represents the current state of that account.

**Tags:**
- `["aa-account", <account_id>]` — 32-byte hex identifier for the multi-sig account.
- `["aa-signers", <json>]` — JSON object:
```json
{
  "signers": ["<signer_pk_hex>"],
  "threshold": 1
}
```
- `["aa-signatures", <json>]` — JSON string encoding an array of `[pubkey, signature]` pairs from authorized signers that satisfy the threshold for this policy. Each `signature` is a Schnorr signature over the policy commitment (see below):
```json
[["<signer_pk_hex>", "<schnorr_sig_over_policy_commitment>"], ["<signer2_pk_hex>", "<sig2>"]]
```

Policy commitment to be signed by the `aa-signatures` entries:
- `policy_commitment = sha256("nostr-aa:policy:" + <account_id> + ":" + <exact_aa_signers_json_string>)`
- `<exact_aa_signers_json_string>` is the literal JSON string placed in the `aa-signers` tag value (no re-encoding, no whitespace changes). Implementations MUST verify signatures against this commitment.

**Example (Initial Policy):**
```json
{
  "kind": 10500,
  "pubkey": "<any_publisher_pk_hex>",
  "created_at": 1736610000,
  "tags": [
    ["aa-account", "2d6f9c0cfe1a...a9b2"], 
    ["aa-signers", "{\"signers\":[\"<Y1_hex>\"],\"threshold\":1}"],
    ["aa-signatures", "[[\"<Y1_hex>\",\"<sig_by_Y1_over_policy_commitment>\"]]"]
  ],
  "content": "Initial AAV policy for Alice",
  "sig": "<sig_by_any_publisher_pk>"
}
```

## Account-Abstraction Proof tag on authored events

Events signed by authorized signers include an `aa` tag proving authority to act for the account.

Format (single and multi-signature are unified):
```
["aa", <account_id>, <policy_event_id>, "<pk2>", "<sig2>", "<pk3>", "<sig3>", ...]
```

- `policy_event_id`: ID of the latest valid policy event (`10500`).
- Repeating pairs `"<pkN>", "<sigN>"` provide additional co-signatures when the policy `threshold > 1`.

Co-signature commitment to sign (to avoid circular dependencies with tags):
- Let `event_commitment_base` be the standard NIP-01 serialization of the event with the `aa` tag truncated to its first three elements only: `["aa", <account_id>, <policy_event_id>]`.
- Each additional co-signer included in the `aa` tag MUST Schnorr-sign `sha256(event_commitment_base)`.

**Example (1-of-N):**
```json
{
  "kind": 1,
  "pubkey": "<any_publisher_pk_hex>",
  "created_at": 1736610500,
  "tags": [
    ["aa", "2d6f9c0cfe1a...a9b2", "<policy_event_id>", "<Y1_hex>", "<sig_by_Y1_over_event_commitment>"]
  ],
  "content": "Hello from Account X",
  "sig": "<sig_by_any_publisher_over_event_id>"
}
```

**Example (2-of-3):**
```json
{
  "kind": 1,
  "pubkey": "<any_publisher_pk_hex>",
  "created_at": 1736610600,
  "tags": [
    ["aa", "2d6f9c0cfe1a...a9b2", "<policy_event_id>", "<Y1_hex>", "<sig_by_Y1_over_event_commitment>", "<Y2_hex>", "<sig_by_Y2_over_event_commitment>"]
  ],
  "content": "2-of-3 example",
  "sig": "<sig_by_any_publisher_over_event_id>"
}
```

## Verification

1. **Base NIP-01 check**: `verify_schnorr(sig, id, pubkey)` must pass.
2. If no `aa` tag → treat as normal NIP-01 event.
3. If `aa` tag present:
   - Retrieve the **latest valid** `10500` policy for the account where `aa-account` matches `account_id` and `created_at` is the most recent among valid policies.
   - Validate the policy event itself (see Policy Validation Rules below).
   - Ensure the policy's `created_at` is not after the authored event's `created_at`.
   - Let `threshold` be the policy threshold. Compute the co-signature commitment as the NIP-01 serialization with the `aa` tag truncated to `["aa", <account_id>, <policy_event_id>]`.
   - From the `aa` tag, collect one or more `(pkN, sigN)` pairs. Verify each `sigN` over this commitment against `pkN` and ensure each `pkN` is in the policy's `signers`.
   - Let `unique_signers = { pkN of valid pairs }`. Require `|unique_signers| ≥ threshold`.
4. If all pass, index and render author = `account_id`.

## State Management

- **Latest State**: The most recent `10500` event for an account represents the current state.
- **State Updates**: To modify account permissions, publish a new `10500` event with updated signers.
- **Removal**: To remove a signer, publish a new `10500` event without that signer in the list.
- **Account Deletion**: To delete an account, publish a new `10500` event with an empty signers list.
- **No Revocations**: No separate revocation events are needed; the latest policy always represents the current state.

## Policy Update Verification

When publishing a new `10500` event to update account state, the update MUST be authorized according to the current account's policy. The authorization is embedded directly in the policy event via `aa-signatures`:

1. **Retrieve the latest policy** for the account being updated (if any).
2. **Compute the new policy commitment** from the new event's `aa-account` and the exact `aa-signers` JSON string:
   - `policy_commitment = sha256("nostr-aa:policy:" + <account_id> + ":" + <exact_aa_signers_json_string>)`
3. **Verify authorization** using signatures included in `aa-signatures`:
   - If this is the first policy for the account: signatures in `aa-signatures` MUST be from signers listed in the new `aa-signers`, and the number of unique valid signatures MUST satisfy the new `threshold`.
   - If updating an existing policy: signatures in `aa-signatures` MUST be from signers authorized by the current (previous) policy, and the number of unique valid signatures MUST satisfy the current policy's `threshold`.
4. The `pubkey`/`sig` of the policy event itself are irrelevant to authorization; any publisher may submit the event as long as `aa-signatures` validate.

## Policy Validation Rules

When verifying a `10500` policy event, the following constraints MUST be enforced:

1. **Threshold Minimum**: `threshold` MUST be ≥ 1
2. **Signer Count**: The number of signers MUST be ≥ `threshold`
3. **Valid Signers**: All signers in the policy MUST have valid public keys (32-byte hex strings)
4. **Policy Signatures**: The `aa-signatures` tag MUST include `[pubkey, signature]` pairs that verify over the policy commitment. Duplicate signers count once.
5. **Update Authorization**: For updates (non-initial policy), the public keys in `aa-signatures` MUST be authorized by the current policy and MUST meet its `threshold`.

**Invalid Policy Examples:**
- `threshold: 0` (below minimum)
- `threshold: 3, signers: [Y1, Y2]` (insufficient signers)
- `threshold: 2, signers: [Y1]` (insufficient signers)
- Empty signers list

**Valid Policy Examples:**
- `threshold: 1, signers: [Y1]` (minimum valid)
- `threshold: 1, signers: [Y1, Y2]` (1-of-2) // account with recovery key
- `threshold: 2, signers: [Y1, Y2, Y3]` (2-of-3) // 3 board members acting together

**Example Policy Update:**
```json
{
  "kind": 10500,
  "pubkey": "<any_publisher_pk_hex>",
  "created_at": 1736620000,
  "tags": [
    ["aa-account", "2d6f9c0cfe1a...a9b2"], 
    ["aa-signers", "{\"signers\":[\"<Y1_hex>\",\"<Y2_hex>\"],\"threshold\":2}"],
    ["aa-signatures", "[[\"<Y1_hex>\",\"<sig_by_Y1_over_new_policy_commitment>\"],[\"<Y2_hex>\",\"<sig_by_Y2_over_new_policy_commitment>\"]]"]
  ],
  "content": "Updated AAV policy for Alice (now requires 2-of-2)",
  "sig": "<sig_by_any_publisher_over_event_id>"
}
```

## `aa` tag: single and multi-signature

There is no separate `aa-m` tag. The `aa` tag handles both cases:
- If `threshold = 1`: `tags` MAY contain `["aa", <account_id>, <policy_event_id>]` with no co-signature pairs.
- If `threshold > 1`: include enough `(pk, sig)` pairs in the `aa` tag to meet the threshold when combined with the event's own `pubkey`.
Implementations MUST verify co-signatures over the commitment defined in the Account-Abstraction Proof section.

## Queries and Indexing

- REQ with `authors: [<account_id>]` SHOULD return events with valid AAV proofs for that account.
- Relays SHOULD dual-index by `account_id` and controller `pubkey`.
- Relays SHOULD maintain the latest policy event for each account for efficient verification.

## account_id

Any 32-byte hex identifier which does not already have a policy.

## Security Considerations

- No account private key reduces custodial risk.
- Latest-state approach simplifies verification but requires reliable access to the most recent policy.

## Backward Compatibility

AAV is additive. Non-supporting implementations ignore `aa` tags and display the controller as author. Events remain valid NIP-01 events.

## Reference Implementation

A reference implementation is available at...
