# Delivery modes — client-layer material

**Not normative. Not part of the specification.**

This document holds material extracted from an earlier version of the
specification, where "delivery profiles" were a protocol-level registry of
pre-approved combinations of protection. That was the wrong layer: which parts
of a message are encrypted is a request the protocol fulfils, not a menu it
curates, and the accompanying threat tables and UI obligations are client
concerns a protocol cannot impose.

The protocol now exposes per-field encryption directly. A client composes
*modes* over those primitives — a named preset bundling an encryption
selection, a transport preference, and whether to request optional exterior
fields. What follows describes the superseded protocol-level model.

**It is preserved for its analysis, not its mechanism.** The use cases, the
per-actor threat tables, and the reasoning about what each configuration does
and does not protect remain sound and will inform the client design. The
registry, the authenticated profile code, and the one-message-one-profile rule
are gone.

This material will move to the messenger client's own repository when that
exists. Until then it lives here, outside `docs/`, so the boundary between
specification and client material is visible.

---

*Section references inside the moved text (`§N`) are as they stood in `delivery-profiles-v1.md` and have **not** been renumbered to this document.*

---

*From `delivery-profiles-v1.md` §3.1, verbatim.*

## 1. The profile field  `[PROTOCOL-ENFORCED]`

Every message carries an explicit **`profile`** identifier **inside the authenticated (signed /
sealed-covered) portion** of the envelope. Consequences:

- **Authenticated, always present.** The `profile` code is covered by the signature/AEAD, so a
  plugin cannot relabel a message (e.g. pass a `plaintext-signed` message off as `sealed`) without
  breaking verification. **Every** message carries it explicitly — including the default; there is
  **no "absence means X" convention** (an unwritten default is an invisible, error-prone rule). The
  default `sealed` profile is simply **registry entry `0`**, not a privileged absence.
- **Construction is the backstop.** The label is authenticated for a uniform verify-dispatch path,
  but it is **never load-bearing for security**: profiles are designed to be **mutually
  distinguishable by construction**, so a stripped/corrupted label degrades to "trial the profiles,"
  never to "trust a forgeable claim." The construction is the truth; the label is a convenience.

> **Note — this re-freezes the sealed-message vectors.** Adding an authenticated `profile` field to
> the covered header changes the envelope hash (#4) of every sealed message, and therefore every
> frozen vector that pins one (`KV-ENV`, `KV-ENV-HASH`, `KV-WIRE`, `KV-SEAL`, and transitively
> `KV-FRAG-01` and any DAG/dedup/membership vector embedding an envelope hash). This is a
> **format-version change**: an implementation carrying frozen sealed-message vectors from a prior
> format version MUST re-freeze them **once**, against this version. Every regenerated vector must
> be produced **by the implementation** (never hand-edited), and the vector-integrity gate must
> confirm each is still referenced.

---

*From `delivery-profiles-v1.md` §3.2, verbatim.*

## 2. Registry & extensibility  `[CONFORMANCE-REQUIRED]`

`profile` is a **registered unsigned-integer code** from a spec-owned registry (same shape as the
algorithm-agility registry). Rationale for the representation:

- **Not a UUID** — 16 bytes for a value from a small, *centrally specified* set is wrong-shaped;
  UUIDs are for decentralized minting, the opposite of a coordinated registry, and would bloat every
  authenticated header.
- **Not bit-flags** — flags imply *any combination is legal*, which re-opens the runtime-invention
  door (a plugin OR-ing bits to synthesize an unspecified profile). A single enum code means **every
  value is a named, audited construction**; there are no undefined combinations to reason about.

Each registry entry carries **the construction *and* its property/threat table** (§4, §7). The wire
carries only the code. **Extensibility lives in the spec, not the plugin**: a new profile is a
*registered, analyzed construction added to the menu* — never a shape a plugin conjures at runtime.

**Forward-compat `[PROTOCOL-ENFORCED]`.** A receiver seeing an **unrecognized profile code** treats
it as a typed **"unsupported profile"** — surfaced, not crashed, and **never** silently coerced into
a profile it does know. (Same discipline as algorithm-agility's unknown-algorithm rejection and the
fragment parser's malformed→typed-reject.) Old clients reject new profiles **legibly** rather than
misinterpreting them.

---

*From `delivery-profiles-v1.md` §3.3, verbatim.*

## 3. One message = one profile  `[PROTOCOL-ENFORCED]`

A single message has **one** profile — **the same security bound independent of transport**. There
is no mixed-mode message that is `sealed` to some recipients and `plaintext-signed` to others.

A client that wants "send to the whole group, including the aunt who needs plaintext" emits **two
protocol messages** (a `sealed` one and a `plaintext-signed` one) — a **client convenience** producing
two messages, never one message wearing two security bounds. This is honest to the cryptography:
different-profile variants of the same content **are different messages** (different construction →
different envelope hash → different identity, #4), so the protocol treats them as such rather than
pretending one object has two guarantees.


---

*From `delivery-profiles-v1.md` §4, verbatim — §4.1, §4.2 and §4.3 keep their numbers.*

## 4. The profiles

Each profile is a construction with a stated property set. The menu is **extensible** (§3.2); v1
specifies the following. (Per-actor threat tables in §7.)

### 4.1 `sealed` — registry entry `0` (the default)

The `sealed` profile's construction: the media-typed payload is sealed to the recipient
(`design-doc-v1.md` §5.3 single-recipient seal). Properties: **content confidentiality +
authenticity + per-relationship unlinkability**. No change from `design-doc-v1` except that the
`profile=0` code is now explicit in the covered header.

### 4.2 `plaintext-signed` — authenticity without confidentiality

For transports where even an opaque body is a problem (e.g. a ToS that flags ciphertext), but the
recipient must still verify the message legitimately came from the sender. The body is **plaintext**
(readable by the transport, by design); authenticity is a signature.

`[PROTOCOL-ENFORCED]` **The signature binds the body *plus the full envelope context*** — the same
fields a `sealed` envelope already commits to: `(plaintext_body, sender_persona, timestamp,
causal_parents, profile_code, context/recipient binding)`. This makes `plaintext-signed` a
**structural sibling of `sealed`** — same envelope, same content-addressed identity (#4), same causal
binding — with the body's confidentiality swapped from ciphertext to plaintext. Consequences:

- **Replay is absorbed, not merely signed-against.** Because the envelope binds `timestamp` and
  `causal_parents`, a replayed byte-identical copy has the **same envelope hash** → de-duplication by
  hash (`design-doc-v1.md` §11) collapses it and drops it as a duplicate, exactly as for a sealed message. Replay protection is
  **inherited for free** from being an envelope sibling; it is not extra machinery. (This is the
  decisive advantage over signing the body alone, which leaves lift-and-replay open.)
- **Cross-context replay fails** — the persona is the **per-relationship** persona (#14), never a
  stable identity key, and the context is under the signature. A `plaintext-signed` message to the
  aunt cannot be replayed into a different relationship.
- **Downgrade fails** — `profile_code` is under the signature.

`[CONFORMANCE-REQUIRED]` The signing persona is the **per-relationship persona** for the recipient —
**never** a stable identity key (signing with a stable key "so it verifies across channels" is the
isolation footgun that lets two contacts correlate the sender, #14).

**Declared limit (threat table):** `plaintext-signed` gives authenticity + integrity and **zero
content confidentiality** — the transport, the plugin, and any pipe observer read the body in full,
**by design**. This is the profile's *declared property* (the exact trade the aunt case wants), not a
leak. **The UI must surface the reduced guarantee unmistakably** so a user never sends sensitive
content over a `plaintext-signed` transport believing it is protected.

### 4.3 `plaintext-signed + sealed authenticator` — plaintext body, opaque side-channel

Profile 4.2 plus a **small sealed attachment** only the recipient can open (may carry a
freshness nonce, or confidential side-content). For transports with **asymmetric constraints** that
tolerate opaque *attachments* under some size / non-executable bound but not opaque *bodies* — a
distinction **only the plugin knows about its transport** (see §9). Distinct from `decoy-sealed`
(§5.3): here the plaintext body is the **real** (readable) message; the attachment is supplemental.

`[CONFORMANCE-REQUIRED]` The construction binds the plaintext body and the sealed attachment together
(under the same envelope signature) so a body from one message cannot be mixed with an attachment
from another.


---

## 5. Per-profile threat summary

*From `delivery-profiles-v1.md` §7. Only the summary table is moved; the three trust tiers it summarises are protocol reasoning and remain in the specification. Enough of that reasoning to read the table: §7 gives each profile a threat table with a column per **actor** — the transport or network observer, versus a **hostile plugin** — because a plugin sits differently against each property. Some properties hold even against a hostile plugin (they rest on cryptography and key custody, not plugin cooperation); some hold only if the plugin carries the message faithfully (a plugin can deny delivery, not defeat protection); and a plugin necessarily sees whatever routing configuration it was handed. The table reads each profile against those columns.*

*Per-profile summary:*

| Profile | Content confidentiality vs. transport | vs. hostile plugin | Authenticity/integrity | Unlinkability |
|---|---|---|---|---|
| `sealed` | held | **held** (key-blind) | held (re-verified) | held (per-rel. persona) |
| `plaintext-signed` | **dropped by design** | dropped by design | held (envelope-bound sig) | held (per-rel. persona) |
| `plaintext-signed + authenticator` | body dropped; attachment held | attachment held (key-blind) | held | held |
| `sealed` + decoy framing | held (real content sealed) | **held** (key-blind) | held | held; *attachment existence/size/timing observable* |

---

## 6. Profile-specific open items

*From `delivery-profiles-v1.md` §11.1, items 2 and 3, verbatim. Item 1 there (the encoding registry) is a framing-layer matter and stays.*

1. **`plaintext-signed + authenticator` binding detail** — the exact body↔attachment binding (§4.3).
   Pin when the profile is implemented.
2. **Interaction with membership/group send** — a group send is already one signed envelope
   fanned to N recipients; with one-message-one-profile (§3.3), a mixed-audience group is N
