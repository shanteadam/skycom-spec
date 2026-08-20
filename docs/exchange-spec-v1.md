# Key Exchange Specification — v1

**Relationship to the messaging protocol.** This is a **separate specification** from the core messaging design (`design-doc-v1.md`), on its own timeline. The two meet at exactly one place: the **keystore contract** (messaging doc §7). Key exchange's only job is to *produce valid keystore entries*; the messaging protocol *reads* those entries and is entirely unaware of how they were established. A bundle could be handed over by QR, by referral, or by emailing it to someone — the messaging layer cannot tell the difference, because by the time it sees anything, the key is already a resolved keystore entry.

**What is normative here.** Only the **contact-bundle format**, **proof-of-possession**, the **exchange-plugin interface**, and the **local-confidence security rule** are normative. Method, ceremony, handshake, and confidence vocabulary are pluggable. This document uses the same **[PROTOCOL-ENFORCED] / [CONFORMANCE-REQUIRED]** discipline as the messaging spec.

**Scope of v1.** v1 exchange is **out-of-band bundle delivery only** (QR / paste / link) — a *symmetric* exchange with **no interactive handshake**. Referral and rendezvous (interactive) are deferred to later reference-plugin versions and later named exchange-protocol specs (§5). Because v1 has no handshake, v1 needs no interactive exchange-protocol spec — the bundle format is the whole wire story.

---

## 1. Purpose & structural tension

Establishing a relationship = each party ends up holding the other's per-relationship **Ed25519 (signing) + X25519 (encryption) public keys**, and giving the other theirs, producing a keystore entry on each side (messaging doc §7 / §5.2).

Every conventional key-exchange scheme assumes a stable, discoverable identifier (phone number, username, DID, directory entry). This design deliberately has none — a directory of one's keys is the exact correlation surface the per-relationship model destroys. So exchange must work *without* any global identifier, and the **man-in-the-middle at first contact** is the one attack the rest of the system does not inherently prevent (an attacker who swaps keys at exchange establishes two relationships and bridges them; everything still verifies). This spec's job is to guarantee **interoperable first contact** and to make honest verification *possible*, without blessing one ceremony.

---

## 2. Contact bundle — [CONFORMANCE-REQUIRED]

A canonical, **versioned** serialization of an offer of per-relationship keys, which any conformant client can produce and consume regardless of which method moved it. This is the interop guarantee for cross-client first contact — the analog of the messaging envelope format: **fixed container, free carrier.**

**The contact bundle is deliberately minimal and contact-facing.** It contains **only**:
- The offered **Ed25519 + X25519 *public* keys** (algorithm-tagged).
- A **proof-of-possession self-signature** (§3).
- Optional **exchange-mechanics facts** about *this exchange*: expiry, a claimed method hint, format version.

**It MUST NOT carry identity metadata, other or nested identities, private keys, or a display name.** No user-chosen metadata travels to the contact by default (strict phone-number-exchange model — the contact labels you locally, `design-doc-v1.md` §4.4). Leaking nested-identity data to a contact would recreate the cross-identity correlation the per-relationship model destroys (design-doc invariants #2, #12).

**This is NOT the *identity bundle*** (`design-doc-v1.md` §4.10) — the self-facing, always-wrapped artifact that *does* hold metadata + nested identities + private keys and never reaches a contact. The two artifacts must never be conflated.

The bundle format is required; the *method* that transports it and the *labels* interpreting it are not.

*Open item:* the concrete **wire encoding / version tagging** of the bundle and its PoP signature — pin in the key-exchange implementation story. (Shape decided per §2–§3; only the byte encoding remains. Reusing the messaging spec's deterministic-CBOR/COSE choice is the natural default so implementations share one serializer.)

---

## 3. Proof-of-possession — [CONFORMANCE-REQUIRED]

The bundle MUST carry **one Ed25519 self-signature covering the entire bundle** (all offered keys *and* the self-asserted facts). This proves the sender holds the Ed25519 private half, binds the X25519 key and the facts under that signature (so expiry / method-hint cannot be tampered without breaking PoP), and closes offering-keys-you-don't-hold and a class of key-substitution. A bundle failing PoP verification is **rejected**. [PROTOCOL-ENFORCED once received: the signature check is cryptographic.]

(The X25519 key cannot itself sign; its possession is asserted by inclusion under the Ed25519 signature, not by a separate proof.)

---

## 4. Exchange-plugin interface — [CONFORMANCE-REQUIRED]

The *interface* is required; the plugins behind it are not. **The interface belongs to the exchange layer, not the messaging core** — the messaging protocol has no exchange hooks and never initiates an exchange. A plugin can:

1. **Request new local keys** for an exchange — mint a fresh per-relationship keypair set (Ed25519 + X25519) to be offered.
2. **Ingest a remote bundle** — verify version + PoP — and **write the resulting contact key set** as a keystore entry (messaging doc §7).
3. **Provide optional exchange metadata** — a locally-computed confidence/security level and info (§6).
4. Surface, at the plugin's discretion, whether the counterpart's exchanger is compatible or lacking. Detectable compatibility is limited to what's in the bundle (version always; self-asserted method / expiry as *claims* only). Bundle-level compatibility is guaranteed by §2; method / confidence adequacy is the plugin's judgment, not the spec's.

---

## 5. Method, handshake & confidence vocabulary — [NOT REQUIRED / pluggable]

The exchange **method** (QR, paste, link, NFC, rendezvous, …) and the **confidence/security vocabulary** are outside the normative core. A **canonical reference set** of exchange plugins and a **reference confidence/security enumeration** ship as defaults; other clients MAY ignore this metadata or substitute plugins that define security criteria / labels differently.

**Tiered structure for interactive exchanges (forward-looking).** Symmetric / out-of-band methods (QR-in-person, paste, NFC) have **no conversation** — two parties each produce a bundle and hand it over; the bundle format is sufficient and nothing further need be specified. *Interactive* methods (rendezvous, invite-then-response, referral) involve a back-and-forth that two *different* clients must agree on to interoperate. Such a handshake is specified as a **named, versioned exchange-protocol spec** (e.g. "Referral v1", "Rendezvous v1") — separate from both the messaging core and any single plugin. A plugin *conforms to* a named exchange protocol rather than inventing one; two clients interoperate on an interactive exchange iff they implement the same named protocol. **v1 defines no interactive exchange protocol** (out-of-band only).

**Safety-number / fingerprint.** A derivation (a hash over both parties' keys, rendered as comparable words/digits, verified over an independent second channel to defeat first-contact MITM) is provided as a **reference utility** for plugins, **not** mandated. If used, the *derivation* must be shared for the number to be portable across clients; *enforcing* verification is a client policy.

---

## 6. Confidence is local, never trusted from the peer — [SECURITY RULE, all plugins]

Confidence/security **labels are computed and stored by the *receiving* client** from how *it* obtained and verified the bundle — **never** a value transmitted and trusted from the other party. Self-asserted *facts* (PoP, expiry, claimed method) MAY travel in the bundle and be *verified*; *judgments* (confidence level) MAY NOT be taken from the wire. Otherwise a malicious sender marks a MITM'd key "verified / high confidence" and a trusting client renders an attacker's key as a green checkmark. This rule binds every plugin, reference or third-party. [CONFORMANCE-REQUIRED]

---

## 7. Referral (deferred to a later reference-plugin version)

A **referral** unifies two operations into one object: an authenticated statement, signed by a key the recipient already trusts, vouching for another key.

- **Structure:** a **§2 contact bundle counter-signed by an established contact** — i.e. "a bundle for C, endorsed by B." Ingested through the existing plugin interface (§4) as "a bundle that arrived vouched-for rather than out-of-band."
- **Transitive introduction (A meets C through B):** A verified *B*, not C, and is now trusting B's judgment about C. A referral is an authenticated channel to establish C's key, **not** a guarantee the key is really C's — a malicious or lazy B can introduce A to an attacker-controlled "C" and MITM from birth. Per §6, A computes its **own** local confidence, **no higher than** min(A's confidence in B, B's claimed confidence in C), and marks the result a distinct, lower tier ("introduced, not directly verified").
- **Self-rotation is the degenerate case** (referrer == referred == self): "my new key is X, signed by my old key." This is how key migration — including **PQ / algorithm migration** — reaches established contacts in-band, without redoing the out-of-band ceremony. It is a **continuity mechanism, not a recovery one**: it works only while the old key is **uncompromised and available to sign**. A compromised key lets an attacker sign a referral to their own key (hijack); a lost key cannot sign at all. Either case falls back to fresh out-of-band exchange. This limit MUST be documented wherever self-rotation is offered.
- **Interactive:** referral is mildly interactive, so shipping it means writing **"Referral v1"** as the first named exchange-protocol spec (§5). Deferred.

---

## 8. Forward compatibility — algorithm / PQ agility

Exchange must not foreclose post-quantum migration (the near-certain future reason to change primitives). Cheap rules to honor now:
- **Algorithm-tag every key and signature** in the bundle (lean on COSE algorithm IDs; never positional "this field is always X25519"). This is the highest-value, lowest-cost agility move.
- **Treat key / signature sizes as variable-length** — PQ keys/signatures are 10–50× larger (ML-KEM ~1KB, ML-DSA ~3KB+); no fixed-size assumptions.
- **The bundle carries a *set* of algorithm-tagged keys**, so a PQ KEM key is simply another entry alongside X25519, enabling **hybrid** (classical + PQ combined) offers later.
- **Distribution of a new (e.g. PQ) key to established contacts** is a **self-rotation referral** (§7) — no messaging-core change; the messaging protocol only ever sees the resulting keystore entry.

*(All deferred: no PQ, hybrid, referral, or rotation in v1. These rules only keep the door open.)*

---

## 9. v1 invariants (exchange)
1. Only the contact-bundle format and the exchange-plugin interface are normative; method, handshake, and confidence vocabulary are pluggable.
2. Every bundle carries one Ed25519 self-signature (proof-of-possession) covering the whole bundle; a bundle failing PoP is rejected.
3. Exchange confidence/security is computed locally by the receiver from how it obtained the bundle — never taken from a peer-supplied label. Self-asserted facts may be verified; confidence judgments may not be trusted from the wire.
4. Exchange produces keystore entries and nothing else; the messaging protocol has no awareness of exchange, and there are no exchange hooks in the messaging core.
5. Interactive handshakes are named, versioned exchange-protocol specs a plugin conforms to — never invented per-plugin. v1 defines none (out-of-band only).
6. Keys and signatures are algorithm-tagged and variable-length, so PQ/hybrid migration needs no format-breaking change.
