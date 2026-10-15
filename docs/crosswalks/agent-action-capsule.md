# Agent Action Capsule Cross-walk

An Agent Action Capsule (AAC) is a signed, hash-chained record of one agent action: what was asked, what was decided, what effect was dispatched or confirmed, under which key, chained to the record before it. This page shows how a Capsule fills TRACE's registered `behavior-trace` reference slot, how the two records cite each other by digest, and how the Capsule's signer binds to the TRACE `subject` when that subject is a DID.

> **Non-normative.** This document is informative only. Nothing here changes TRACE v0.2 schema fields, wire formats, required claims, or conformance requirements, and nothing here changes the Capsule profile. References to "TRACE" mean the TRACE v0.2 specification in this repository; references to "AAC" mean `draft-mih-scitt-agent-action-capsule-05` (format 4).

---

## Purpose

[Spec section 3.1.2](../../spec/trace-v0.2.md#312-references-facts-this-record-points-at) registers `behavior-trace` as "a behavioural record of what the agent did, of which this record is the environment evidence," and the [references registry](../references-registry.md) lists no committed format for it. A Capsule is that record: TRACE says what environment the agent ran in, the Capsule says what the agent did in it. The two are produced by different parties with different keys, and neither needs the other to verify.

This cross-walk follows the same bar as the Acta and CHAP pages: the Capsule travels **next to** a Trust Record, cited by digest, never folded into it. A resolved Capsule is not TRACE evidence (rule 3 of §3.1.2); a Trust Record cited from a Capsule is not AAC evidence. Each is checked under its own specification, and the join is a pair of digests that either party can recompute.

---

## What a Capsule is

A Capsule is a JSON object whose identity is derived from its own bytes:

```
capsule_id = lowercase-hex( SHA-256( UTF-8( JCS( payload ) ) ) )
```

with JCS per RFC 8785, the same canonicalization TRACE §3.2.2 requires, and with `null`, empty containers, floating-point numbers and integers outside ±(2^53 − 1) forbidden in digest-bearing material. The members a relying party will meet in this cross-walk:

| Capsule member | Meaning |
|---|---|
| `operator` | the accountable tenant the action was performed for |
| `developer` | the agent identity and version that performed the action |
| `action_id`, `action_type`, `timestamp`, `epoch_id` | the action, whether a disposition was required, RFC 3339 UTC time, and the operator's configuration epoch |
| `disposition.verdict_class` | the decision class (`executed`, `denied`, …) — every verdict gets a Capsule, not only successes |
| `effect.{type,status,request_digest,response_digest,effect_attestation}` | the side effect, its state (`planned` … `confirmed` … `reverted`), digests of the actual request and observed response, and who observed it |
| `chain.parent_capsule_id` | the producer's own previous Capsule (same custody stream only) |
| `references[]` | citations of records outside that stream, each a typed digest reference `{type, digest_alg, digest}` |
| `assurance.{attestation_mode,effect_mode,ledger_mode}` | self-described tiers a verifier re-derives from the bytes and reports overclaims on |

The Capsule is signed by a **Producer Envelope**: a `COSE_Sign1` (tag 18) whose protected header carries `alg: -8` (EdDSA, Ed25519), `content type: application/agent-action-capsule-id` and `kid` = the raw 32-byte Ed25519 public key, and whose attached payload is the raw 32-byte `capsule_id`. Any number of envelopes may sign one Capsule; none of them changes the id. Optionally the id is registered as an RFC 9943 Signed Statement with a SCITT Transparency Service and carries back an RFC 9942 Receipt; Capsules are also appended to a checkpointed local log whose checkpoints are what a witness registers.

---

## Four questions, two records

A relying party asking about an agent action asks four things, and the two specifications answer them in a fixed division of labour:

| Question | Answered by | Where |
|---|---|---|
| **WHO** acted | TRACE | `subject` (DID or SVID), `cnf`, the `runtime` measurement chain, `delegation.*` |
| **CAN** it act, and under what | TRACE | `policy` (bound policy-set hash, `enforcement_mode`), the `authorized-intent` reference, `data_class` |
| what it **DID** | AAC | the Capsule: disposition, effect, request/response digests, chain |
| **AUDIT** — was that inside the terms | neither record; a third party | AAC Class 1/2 verification and a judge against written terms; cited back from TRACE as `condition-appraisal` |

The composition rule follows: **the Capsule composes TRACE's WHO and CAN into its DID by citing them, and never restates them.** A Capsule does not carry a DID document, a policy, or a measurement; it carries the TRACE record's digest (below), the policy hash the action ran under, and the mandate it ran on, each as a typed digest reference. A Trust Record does not carry the action; it carries the Capsule's digest. Each record stays verifiable under its own specification, and the join is recomputable by anyone holding both.

Concretely, for CAN: the TRACE `policy` hash and the artifact behind `authorized-intent` (a mandate, a permit, an OAuth token's digest) appear in the Capsule as the Effect Record's external-authorization references and/or `constraints[]` entries, `{type, digest_alg, digest}`, so the DID record states *under which CAN* it acted, and a verifier holding the TRACE record checks the two hashes are the same bytes. For WHO: the key binding in the DID-slot section below. For AUDIT: nothing in either record — a judgment is a separate signed artifact that binds to the digest of the composed evidence set and is reported as properties, not folded into either record's `ok`.

---

## Mapping: a Trust Record cites a Capsule

A `references` entry under §3.1.2 is `{rel, id, resolver, retention?, digest?}`. Against a Capsule:

| `references` field | Value | Note |
|---|---|---|
| `rel` | `behavior-trace` | registered in §3.1.2 |
| `id` | the 64-character lowercase-hex `capsule_id` | the Capsule's own identity; it is already a digest |
| `resolver` | the party undertaking to serve the Capsule bytes: the operator's evidence endpoint, or a log that holds the Capsule | TRACE rule 4 applies unchanged: no resolver, no entry |
| `retention` | ISO 8601 duration, optional | an undertaking to keep the bytes with this digest resolvable; it says nothing about the bytes being right, which the digest already says |
| `digest` | `"sha256:" + capsule_id` | TRACE's digest prefix convention over the same value as `id`. A Capsule producer always holds this at issue time, so the entry carries it. |

`id` and `digest` carrying the same hex is not redundancy. `id` is the name within the resolver's system; `digest` is what a relying party recomputes from the bytes it is handed. For a Capsule the two coincide because the name *is* the digest; for other `behavior-trace` formats they may not.

Example (record fields unrelated to the cross-walk elided):

```json
{
  "subject": "did:web:agents.example.com:procurement",
  "eat_profile": "https://agentrust.io/trace/v0.2",
  "iat": 1792012800,
  "references": [
    {
      "rel": "behavior-trace",
      "id": "1f6f40b682c1769fd01d9ad4c2399037ed355920e98d98581a27d1f09447f073",
      "resolver": "https://evidence.example.com/capsules",
      "retention": "P7Y",
      "digest": "sha256:1f6f40b682c1769fd01d9ad4c2399037ed355920e98d98581a27d1f09447f073"
    }
  ]
}
```

The record signature covers the entry (§3.1.2 rule 2), so the Trust Record commits to *pointing at* this Capsule. It commits to nothing about the Capsule's content; that is the Capsule's own signature's job.

---

## Mapping: a Capsule cites a Trust Record

The reverse direction uses the Capsule's `references[]`, whose entries are CPB typed digest references: `{type, digest_alg, digest}`, identity by digest alone, locators optional and never identity.

| Capsule `references[]` field | Value | Note |
|---|---|---|
| `type` | `trace-trust-record` | the provisional CPB artifact type for a TRACE Trust Record |
| `digest_alg` | `sha-256` | |
| `digest` | `hex( SHA-256( JCS( record ) ) )` with the record's `signature` member **present** | this is exactly the §3.1.3 `delegation.parent_record_hash` preimage, so one digest serves TRACE's own chain and the Capsule's citation; no second construction is introduced |

```json
{
  "references": [
    {
      "type": "trace-trust-record",
      "digest_alg": "sha-256",
      "digest": "4e0f…c2a1"
    }
  ]
}
```

A Capsule's `references` participates in its `capsule_id`, so citing the Trust Record is itself a claim the Capsule's envelope signature covers. Mutual citation — each record naming the other's digest — is possible only when one is issued after the other; in practice the Capsule is sealed at the action and the Trust Record cites it, and a later Capsule (for example the epoch-boundary Capsule for the next configuration) cites the Trust Record back.

---

## The DID slot: binding the Capsule signer to TRACE `subject`

TRACE's `subject` is "a SPIFFE SVID or DID URI." A Capsule carries no DID field; it carries `developer` (a string) and the envelope's `kid` (a raw Ed25519 public key). The binding a relying party can actually check runs through the key, not the string:

1. The Trust Record's `subject` is a DID URI, e.g. `did:web:agents.example.com:procurement`.
2. Resolving that DID yields a DID document whose `verificationMethod` lists an Ed25519 key (as `Multikey` or a JWK with `kty: OKP, crv: Ed25519`).
3. The Capsule's Producer Envelope `kid` is a raw 32-byte Ed25519 public key. A relying party compares it byte-for-byte with the key material in step 2.
4. The envelope signature verifies under that key over the Capsule's `capsule_id`.

When all four hold, the relying party has established: *the agent that the Trust Record names as its subject controls a key that signed this Capsule.* That is the whole of the binding. Recommendations that keep it checkable:

- **Put the DID in `developer`.** Nothing in the Capsule profile constrains `developer` beyond "the agent identity and version"; a producer that sets `developer` to the same DID URI as the TRACE `subject` (with the version carried elsewhere, e.g. `epoch_id` or a `#fragment`) lets the two records be joined by eye before they are joined by key. The string is a convenience; **the key comparison in step 3 is the binding, and a verifier MUST NOT treat a matching `developer` string as establishing anything on its own.**
- **`operator` is the controller, not the subject.** The Capsule's `operator` is the accountable tenant. Against a DID it corresponds to the DID's `controller`, or to the organization behind a `did:web` domain, not to the subject itself. Where TRACE carries delegation (`delegation.*`), the Capsule's `operator` is the party at the top of that chain.
- **Two keys, two custodies, do not conflate them.** TRACE's `cnf` key is the record-signing key, TEE-bound where the platform allows. The Capsule's `kid` is the agent's *action* signing key, held wherever the agent runs. They may be the same key only in a profile that says so; by default they differ, and a relying party checks each signature under its own key. Binding between them is by mutual citation (above), never by assuming one key vouches for the other.
- **Rotation is visible on both sides.** A DID document update that rotates the key, and an AAC epoch-boundary Capsule (`chain.relation: "epoch_opens"`) that records the rotation, bracket the same event from two records. A relying party resolving a Capsule signed under a key no longer in the DID document checks the Capsule's `timestamp` against the DID document's history (or a witnessed checkpoint) rather than rejecting outright — the key was live when the action happened or it was not, and the two records together answer which.
- **`did:web` is the floor, not the ceiling.** A `did:web` document is served by the controller over HTTPS; it binds a key to a domain, which is what makes it the obvious join to an organization. A stronger binding — an IdP or key-attestation statement that the key was issued to this client — is a separate artifact the Capsule can cite by typed digest, and is out of scope here.

What this binding does **not** establish: that the DID controller authorized this particular action (that is an `authorized-intent` reference or a cited mandate), that the key was uncompromised (that is the DID method's and the operator's problem), or that the action's effect occurred (that is `effect.response_digest` and `effect_attestation` inside the Capsule).

---

## What a relying party may establish from a resolved `behavior-trace` Capsule

Given the Trust Record, the referenced Capsule bytes and its envelope(s):

- that the Capsule bytes are the bytes the Trust Record cited (`sha256(JCS(payload))` equals `id`/`digest`);
- that a holder of the key in `kid` signed that id, and, through the DID slot above, that the key belongs to the record's `subject`;
- the Capsule's own claims, each independently verifiable under the AAC profile: the disposition and verdict class, the effect's state and request/response digests, who observed the effect (`effect_attestation`), the chain link to the producer's previous Capsule, and the re-derived assurance tiers (`self_attested`/`anchored`, `standalone`/`chained`/`anchored`);
- when the Capsule carries a Receipt, that a named Transparency Service included the Capsule id at a stated tree size.

What it may **not** establish, and should not be led to:

- anything about the TRACE record's assurance (§3.1.2 rule 1: references never move `runtime.platform`);
- that the Capsule's content is true beyond what its own digests bind — a `confirmed` effect with a `response_digest` is an observed result; a `dispatched` effect is a sent request and nothing more;
- completeness: one cited Capsule says nothing about actions that produced no Capsule. Completeness is a property of the producer's checkpointed log and its witness receipts, not of a single reference;
- a score. Both specifications report checkable properties; a relying party combining them reports which properties held and which were not checked.

---

## Verification steps

For a verifier that has a Trust Record carrying a `behavior-trace` entry and has obtained the referenced Capsule:

1. Verify the Trust Record under TRACE §3.3 as usual. The reference does not participate in this step beyond being covered by the signature.
2. Recompute the Capsule's `capsule_id` over its payload (JCS, SHA-256) and compare with the entry's `id` and the hex part of `digest`. A mismatch means the resolver served different bytes; report it as an unresolved reference, not as a TRACE failure (rule 3).
3. Verify each Producer Envelope: tag 18, protected `alg: -8`, content type `application/agent-action-capsule-id`, payload equal to the raw 32-byte id, Ed25519 signature under `kid`.
4. If `subject` is a DID, resolve it and compare the Ed25519 key in `verificationMethod` with the envelope `kid`. Report `subject-key-bound` or `subject-key-unbound` as a property, never as a verdict on the record.
5. Run AAC Class 1 verification on the Capsule: structure, chain linkage, confirmed-effect invariant, assurance re-derivation, registry membership. Report its structured result alongside the TRACE result; do not merge the two `ok` booleans.
6. If the Capsule cites the Trust Record back (`references[].type == "trace-trust-record"`), recompute `sha256(JCS(record-with-signature))` and compare. Report `mutually-cited` as a property.

---

## Digest agreement is checked, not asserted

The join above only works if an AAC verifier and a TRACE verifier compute the same bytes for the same JSON. Both specifications name RFC 8785, but RFC 8785 implementations are documented to diverge (§3.2.2 names the pair). The AAC repository carries a machine-checked fixture — `docs/interop/aac-trace-digest-agreement.md` with a positive Capsule, a mutant with one nested leaf changed, and a boundary-exceeded case — that compares AAC's canonicalizer byte-for-byte against `rfc8785` 0.1.4, the PyPI library this specification names, over non-ASCII strings, non-BMP and high-BMP object keys, and the ±(2^53 − 1) boundary. The pinned digests in that fixture (`1f6f40b6…`, `9f2e3e36…`) were computed against this repository at `62b8993`. A change to §3.2.2 that moved any of those cases would be caught by that test before it reached this page.

---

## Relationship to the other cross-walks

- **Acta** evidences the *decision* before a tool call ran; a Capsule records the decision *and* the dispatched or confirmed effect, and can cite an Acta receipt as its external authorization by typed digest. The two are complementary at different points in the same action.
- **CHAP** review decisions are human approvals; a Capsule with `disposition.approver: "human"` and a chained human-in-the-loop resolution is the behavioural record of what happened after that approval, and cites it as `approval-outcome` from the TRACE side or as a typed reference from the Capsule side.
- **`observed-effect`** (§3.1.2) is an observer's view of a state change; `effect.response_digest` plus `effect_attestation: gate_executed` is the producer's. A relying party holding both compares digests over the same interval.

---

## References

- Agent Action Capsule profile: `draft-mih-scitt-agent-action-capsule-05` (IETF Datatracker); repository `action-state-group/agent-action-capsule`, including `docs/interop/aac-trace-references.md` (the `references` block read against AAC addressing, authored jointly) and `docs/interop/aac-trace-digest-agreement.md` (the fixture above).
- Canonical Payload Binding (typed digest references): `draft-mih-sokolov-scitt-payload-binding`.
- Checkpointed Local Log: `draft-mih-scitt-checkpointed-local-log-02`; the checkpoint discipline behind `trace-registry` PR #51.
- RFC 8785 (JCS), RFC 9052 (COSE), RFC 9942 / RFC 9943 (SCITT receipts and architecture), RFC 8747 (`cnf`).
