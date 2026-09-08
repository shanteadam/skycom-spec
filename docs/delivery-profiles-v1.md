# Delivery Profiles & Transport Framing — v1

*Companion to `design-doc-v1.md`. This document introduces a security axis the base spec does not
contemplate: `design-doc-v1` assumes **sealed-everything** (its §3–§8 have no plaintext-content
mode; its §3.2's "as private as its transport allows" concerns **metadata** leakage of sealed
traffic, not **content** plaintext). Graduated content-confidentiality is new protocol surface,
specified here.*

*Status: normative. Uses `[PROTOCOL-ENFORCED]` / `[CONFORMANCE-REQUIRED]` tags. Invariant
references (`#N`) are to `design-doc-v1.md` §15. A bare `§N` section reference is to **this
document**; a reference to a section of another document names that document explicitly.*

---

## 1. Purpose & the boundary this note draws

A skycom client may be someone's **only** communication client — carrying skycom-encrypted traffic
alongside ordinary email/SMS/chat, over transports with wildly different constraints (a ToS that
flags opaque bodies; an aunt who only uses iMessage; a photo board that accepts only images). To
serve that, the protocol must let a **transport plugin choose how a message is delivered** without
letting the plugin **define what security a message has**.

That is the boundary, and it is the spine of this note:

> **[PROTOCOL-ENFORCED] The protocol specifies *constructions* and their *properties*; plugins
> *compose* from that menu and decide *execution, transport, and UI*. A plugin never *invents* a
> construction, and never modifies a construction the core produced.**

The line is drawn at **who touches the bytes** — and it is enforced cryptographically, not trusted:

- While the core **produces** an artifact and the plugin **carries it unmodified**, the artifact's
  stated properties hold.
- If a plugin (buggy, hostile, or compromised) **modifies** the bytes, the modification is **caught
  by the receiver's verification** — a tampered artifact does not verify as a weaker-but-valid
  message; it is **rejected**. Modification denies delivery; it cannot forge or silently downgrade.
- Some properties hold **even against a modifying plugin** (see §7): a plugin without keys cannot
  read a sealed body no matter what it does to the ciphertext. Confidentiality rests on **key
  custody**, not plugin trust.

`design-doc-v1`'s own principles already point here: its §8.4 ("send policy is client config; the
core stays policy-free") and its §3.2 ("out-of-envelope fields are adapter-conditional — the
adapter, which knows its transport's leakage, decides"). This note generalizes them into a
security model.

---

## 2. The two-layer model

Every delivery decomposes into **two orthogonal layers**, and keeping them orthogonal is what makes
each guarantee analyzable in isolation:

| Layer | What it decides | When | Authenticated? | Who owns it |
|---|---|---|---|---|
| **Profile** | the message's **security bound** (confidentiality / authenticity / unlinkability) | at **construction**, before sealing | **yes** — inside the covered envelope | the **core** produces it; the client *requests* which one |
| **Transport framing** | how the sealed bytes are **rendered onto a pipe** (fragmentation, encoding, decoy-wrapping) | **after** sealing | **no** — exterior, security-neutral | the **plugin / UI** |

The dividing test is **temporal and cryptographic**: anything decided **before** sealing and that
**changes what is guaranteed** is a *profile* (authenticated, in the envelope). Anything applied
**after** sealing that is **lossless, reversible, and security-neutral** is *framing* (exterior,
unauthenticated). Framing **cannot** be a field inside the sealed envelope — the envelope was
already sealed before the framing was chosen; putting it inside would be circular. This is the same
reasoning that makes fragmentation "framing below the envelope" (#16).

**Convergence (why framing is security-neutral).** Framing is lossless and reversible, so **any**
framing of a message decodes/reassembles to **byte-identical envelope bytes → identical envelope
hash (#4) → the same message**. Whole-vs-fragmented, raw-vs-QR, one fragmentation vs another —
all converge to one node. A wrong framing label cannot downgrade you: it simply fails to decode →
no valid envelope → rejected. Framing changes delivery, never identity or security.

---

## 3. The profile axis

### 3.1 The profile field  `[PROTOCOL-ENFORCED]`

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
> **one-time re-freeze of our own conformance vectors, done pre-ship** — the correct time to make a
> breaking wire change (there are no deployed messages; the "corpus" is our own scaffolding). Every
> regenerated vector must be produced **by the implementation** (never hand-edited), and the
> vector-integrity gate must confirm each is still referenced.

### 3.2 Registry & extensibility  `[CONFORMANCE-REQUIRED]`

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

### 3.3 One message = one profile  `[PROTOCOL-ENFORCED]`

A single message has **one** profile — **the same security bound independent of transport**. There
is no mixed-mode message that is `sealed` to some recipients and `plaintext-signed` to others.

A client that wants "send to the whole group, including the aunt who needs plaintext" emits **two
protocol messages** (a `sealed` one and a `plaintext-signed` one) — a **client convenience** producing
two messages, never one message wearing two security bounds. This is honest to the cryptography:
different-profile variants of the same content **are different messages** (different construction →
different envelope hash → different identity, #4), so the protocol treats them as such rather than
pretending one object has two guarantees.

---

## 4. The profiles

Each profile is a construction with a stated property set. The menu is **extensible** (§3.2); v1
specifies the following. (Per-actor threat tables in §7.)

### 4.1 `sealed` — registry entry `0` (the default)

Today's construction: the media-typed payload is sealed to the recipient (`design-doc-v1.md` §5.3 single-recipient
seal). Properties: **content confidentiality + authenticity + per-relationship unlinkability**. No
change from `design-doc-v1` except that the `profile=0` code is now explicit in the covered header.

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

## 5. The transport-framing layer

Framing is **exterior, unauthenticated, security-neutral, lossless, reversible, and convergent**
(§2). It is applied by the plugin/UI **after** the core produces the sealed artifact, and reversed on
receipt back to the identical envelope. The core is **framing-blind** except where a construction
requires binding to the ciphertext (§9).

### 5.1 Fragmentation

Specified in `design-doc-v1.md` §8.6. Splits the sealed wire-unit bytes into transport-sized fragments;
crypto-blind; index-only headers (#15); reassembles to byte-identical envelope bytes. Framing par
excellence — the archetype this layer generalizes.

### 5.2 Encoding

A **registered set** (for interop, so plugins signal consistently): e.g. `raw`, `base64`, `qr`, ….
An encoding is a lossless, reversible transform of the **already-sealed bytes** for a specific pipe
(base64 for a text field; a QR image for an image-only channel). Because it is applied post-seal, the
encoding identifier is **exterior and unauthenticated** — it lives at the framing layer alongside the
fragment header (#15), **not** in the sealed envelope. A wrong encoding label just fails to decode →
rejected; it cannot downgrade security. (The registry is for consistent signaling; it is *not*
authenticated the way `profile` is.)

*Note:* `qr` is an **encoding of** a profile (a rendering), not a profile — QR-of-`sealed` and
raw-`sealed` are the same message. Separating "what is guaranteed" (`profile`) from "how it is
rendered" (`encoding`) prevents combinatorial explosion and keeps each guarantee analyzable
independent of its wire rendering.

### 5.3 Decoy-wrapping — UI/transport-side, core-blind

A `sealed` message may be delivered wrapped with a **plaintext decoy body** (an innocuous cover
story) plus the sealed artifact as an attachment, so a content-inspecting transport sees something
normal. Key decisions:

- **Decoy is NOT a profile.** The recipient's guarantee is identical to plain `sealed` (they discard
  the decoy, open the sealed attachment). The decoy buys only **sender-side transport-invisibility**
  — a *framing* property, not a recipient guarantee. So decoy-wrapping is **framing**, not a profile.
- **The decoy is pure untrusted cover.** `[PROTOCOL-ENFORCED]` The receiver **discards** the decoy
  entirely — never displays it, never trusts it, never acts on it. It carries no meaning, so it needs
  **no authentication** (nothing to protect) and cannot cause confusion. (A meaningful decoy would be
  re-inventing `plaintext-signed` inside decoy-wrapping — profiles never borrow each other's jobs.)
- **The core never generates or sees the decoy.** No cryptographic correlation is needed between the
  decoy and the sealed content (the sealed attachment is self-contained and self-authenticating;
  nothing binds). So the decoy is **UI/transport-side entirely** — the plugin, which knows what
  "looks normal" on *its* transport, supplies the cover text. The core just produces the sealed
  pieces. *(The correlation test — §9 — is what places decoy on the framing side of the line.)*

**Declared limit (threat table):** decoy-wrapping defeats **content inspection** of the body, not
**metadata correlation** — the **existence, size, and timing** of the attachment remain observable
(the traffic-analysis floor, `design-doc-v1.md` §3.1). A transport that scans bodies is fooled; one that notices "this
innocuous message always carries a ~2 KB attachment" is not. The UI must not over-trust the cover.

---

## 6. The key-blind adapter boundary  `[PROTOCOL-ENFORCED]`

Confidentiality rests on **key custody**, not plugin trust (§7). Therefore the transport-adapter
interface passes **sealed artifacts + routing config across the seam — never keys**. Keys stay
core-side; the plugin is **key-blind**. A compromised transport plugin can then leak its config and
deny delivery (§7 tier 2–3) but can **never** read a sealed body or forge the sender — *by
construction*, not by trust. This is the structural expression of "confidentiality does not depend on
plugin behavior," enforced the same way every other no-keys boundary in the build is (the reassembler
owns no keys; the pool does no crypto; attribution imports only records).

---

## 7. Threat model — three trust tiers, per-actor tables

The properties do **not** stand or fall together. For each profile, the threat table has a column per
**actor** (the transport/network vs. a **hostile plugin**), because a plugin sits differently against
each property. The three tiers:

1. **Guaranteed even against a hostile plugin** (depends on cryptography + key custody, not plugin
   cooperation): **content confidentiality of a sealed body** (the plugin lacks keys) and
   **unforgeability** (the receiver re-verifies; tampering is rejected, never a valid downgrade).
2. **Guaranteed only if the plugin carries faithfully** (plugin can *deny*, not *defeat*):
   **delivery** of an intact message. A hostile plugin can drop/corrupt/stall — but corruption fails
   verification, so the failure mode is **non-delivery, never a forged or silently-downgraded
   message**. (Availability was never guaranteed anyway — `design-doc-v1.md` §8.5.)
3. **Exposed to the plugin by *what it is given*, independent of the bytes**: the plugin necessarily
   sees its **configuration and routing surface** — recipient addresses on its transport, which
   contacts use it, send timing/frequency, message sizes, the `(profile, encoding)` framing. A plugin
   leaks **exactly the union of {what any transport observer sees} ∪ {the config it was handed}** —
   and, for sealed profiles, **nothing more** (specifically **not** content).

**Mitigation for tier 3 `[CONFORMANCE-REQUIRED]`:** scope-minimize plugin config — hand each plugin
the **least** it needs to route (this transport's contacts + addresses, not the whole address book;
per-relationship personas), so a compromised iMessage plugin learns "*some* persona talks to this
Apple ID," not "this is the same human my Signal plugin also carries" (#14 at the plugin boundary).

*Per-profile summary:*

| Profile | Content confidentiality vs. transport | vs. hostile plugin | Authenticity/integrity | Unlinkability |
|---|---|---|---|---|
| `sealed` | held | **held** (key-blind) | held (re-verified) | held (per-rel. persona) |
| `plaintext-signed` | **dropped by design** | dropped by design | held (envelope-bound sig) | held (per-rel. persona) |
| `plaintext-signed + authenticator` | body dropped; attachment held | attachment held (key-blind) | held | held |
| `sealed` + decoy framing | held (real content sealed) | **held** (key-blind) | held | held; *attachment existence/size/timing observable* |

---

## 8. Authenticating channels skycom does not carry — signatures, not secrets

A recurring pattern falls out of §§4–6, and it is worth stating as a principle because it governs
both an existing profile and a future capability: **at a channel boundary, skycom's job is to
*attest*, not to hand over secrets.** When a sensitive channel is carried by a mechanism skycom does
not own, skycom's contribution is a **signature over the thing that authenticates that channel** —
never the sharing of a key into it.

Two instances:

- **`plaintext-signed` (§4.2)** — the sensitive channel is a plaintext body on an untrusted transport;
  skycom's contribution is a **signature over the body + context**, not encryption of it.
- **Real-time media (calls, below)** — the sensitive channel is an SRTP media stream in an untrusted
  media stack; skycom's contribution is a **signature over the media-key fingerprint**, not derivation
  of the media keys.

Same move both times: skycom attests; it does not surrender secrets across the boundary.

### 8.1 Real-time voice/video calls — a plugin on existing core surface  `[CONFORMANCE-REQUIRED]`

A WebRTC (or equivalent) calling **plugin** provides the entire media plane — SDP, ICE/STUN/TURN,
DTLS-SRTP, jitter buffering, codecs. This is **transport-side**, exactly like any other adapter, and
it is **not** a core track. The question this note answers is only: *does core provide the pieces the
plugin needs to make the call end-to-end-authenticated by contact keys?* **It does, with no new core
surface**, via the following construction:

1. **Signaling is `sealed` messages.** The plugin sends its SDP offer/answer — **including the DTLS
   fingerprint** and SRTP parameters — to the contact as an ordinary `sealed`, signed message (a
   dedicated call-signaling content-type). Confidential, authenticated, and signed with the
   **per-relationship persona** (#14) — all for free from the existing send path (E/F/H).
2. **The fingerprint binding is automatic.** Because the fingerprint travels **inside the signed
   body**, it is bound by skycom's normal envelope signature — no separate "sign this fingerprint"
   primitive is needed. Putting the fingerprint in a sealed message **is** the binding.
3. **Verification is the normal receive path.** The peer's offer/answer is verified (its signature is checked) and attributed
   to the contact before its fingerprint is trusted — existing surface.
4. **`[PROTOCOL-ENFORCED]` The plugin MUST enforce fingerprint-match before media flows.** The
   DTLS-SRTP handshake's peer certificate fingerprint **must** equal the one that arrived in the
   skycom-authenticated signaling message; if it does not, an intermediary (signaling server, TURN
   relay, SFU) has substituted keys → **abort**. This enforcement is the plugin's single hard
   correctness obligation. ("Encrypted call works even when you forget to verify" is the classic
   footgun; the match check is what closes it.)

This upgrades WebRTC from *encrypted against the network* to *end-to-end authenticated against your
contact*: a MITM would have to forge a skycom signature over the fingerprint, which is precisely what
skycom makes infeasible. The media keys stay **ephemeral, per-call, and inside the media stack**;
skycom holds only the identity keys that **sign the fingerprint that authenticates** them (§6
key-custody: a compromised media stack can degrade the call but cannot impersonate the contact or
touch a skycom secret).

### 8.2 Rejected alternative — deriving media keys from the relationship secret

An alternative was considered and **rejected**: keying SRTP from a skycom-derived shared secret (an
ECDH over the per-relationship keys → KDF → media keys), so media keys are contact-*derived* rather
than contact-*attested*. It removes the "forgot to verify" footgun by fusing authentication into
keying — but the cost is wrong:

- **It routes the relationship's root secret through the riskiest component in the system.** The
  per-relationship secret protects *every message* with that contact; the media stack (libwebrtc,
  codecs, DTLS state machines) is a large, exposed attack surface. Fingerprint-binding never exposes
  the relationship secret to the media plane at all (DTLS uses its own throwaway keys; skycom signs a
  hash). `[CONFORMANCE-REQUIRED]` **The relationship secret is never handed to the media plane.**
- **It forfeits per-call forward secrecy.** DTLS-SRTP generates fresh ephemeral keys per call;
  deriving from the *static* relationship secret couples all calls (and calls-to-messages) to one
  long-lived root. Recovering ephemerality would require mixing in fresh per-call DH — reinventing
  what DTLS-SRTP already does.

The footgun it closes is closable without it, by the mandatory fingerprint-match (§8.1 step 4). So a
generic `deriveSharedKey(contact, contextLabel, length)` capability is **not** added for calls; if a
future channel genuinely needs an exported shared secret (not mere attestation), it is evaluated then,
on its own merits, under this same "a signature when a signature would do" scrutiny.

---

## 9. The correlation test & the deferred steganographic construction

**The test that places a cover technique on the right side of the line:**

> **Does the cover need to *cryptographically bind* to the ciphertext?**
> - **No** → it is **framing** (UI/transport-side, core-blind). The cover and the sealed artifact are
>   **separable** pieces sitting side-by-side; the core produces the sealed piece and never sees the
>   cover. *(This is `decoy-wrapping`, §5.3.)*
> - **Yes** → it is a **construction** the **core participates in**, because binding is a
>   cryptographic operation the core owns. The core takes **both** the real content and the carrier as
>   input and produces a single woven artifact.

`[DEFERRED]` **Steganographic / format-coherent embedding** is the "yes" case: hiding the sealed bytes
*inside* a carrier that must stay valid — LSB steganography in an image that must still render as a
normal photo; format-preserving encoding where the ciphertext must *be* syntactically-valid cover.
There the ciphertext is **woven into** the cover (not side-by-side), so producing it is a
core-involved construction taking a carrier as input. **Deferred to a future track.** When built, it
is a **registered construction** with its own threat table — never a plugin bolting bytes onto a
carrier however it likes.

**Principle (the spine of this note, restated):** *the core produces **security**; the transport
layer produces **plausibility** — and they re-couple **only** when plausibility requires binding to
the ciphertext (steganography), the sole cover technique that crosses back into the core.*

**Meta-principle `[PROTOCOL-ENFORCED]`:** the spec specifies **constructions and their properties**;
it **never rules a construction out on transport-viability grounds** — whether a given profile/framing
is usable on a given transport is the **plugin's** judgment about its transport, by design. (E.g. the
spec does not decide whether an opaque attachment is "too big" or "too suspicious" for iMessage; the
iMessage plugin does.)

---

## 10. Consequences & invariants touched

- **#4 (content-addressing).** `profile` is in the covered bytes, so it participates in the envelope
  hash — different profiles of the same content are different messages with different IDs (§3.3).
  Framing is post-seal and convergent, so it does **not** affect identity (§2).
- **#14 (per-relationship isolation).** `plaintext-signed` signs with the per-relationship persona;
  tier-3 config leakage is bounded by per-relationship personas at the plugin boundary (§7).
- **#16 (seal-before-everything).** Framing (fragmentation + encoding + decoy) is below the envelope;
  `profile` is decided **before** sealing and lives **inside** the envelope. The two-layer split is a
  direct generalization of #16.
- **#3 / `design-doc-v1.md` §3.2–3.3 (exterior fields).** The `encoding` identifier and decoy are exterior,
  unauthenticated, adapter-conditional fields — each must pass the exterior three-tests
  (readable-by-transport OK, non-authoritative, safe-before-verify), which they do (security-neutral).
- **One-time vector re-freeze** of all sealed-message vectors (this document's §3.1, the profile field) — pre-ship, mechanical but
  load-bearing (regenerate by implementation; keep the vector-integrity gate green).

## 11. Open items & deferred tracks

### 11.1 Small open items (pin when implemented)

1. **Encoding registry contents** — the concrete `raw`/`base64`/`qr`/… codes and their transforms
   (§5.2). Pin when the first non-raw encoding is needed by a transport.
2. **`plaintext-signed + authenticator` binding detail** — the exact body↔attachment binding (§4.3).
   Pin when the profile is implemented.
3. **Interaction with membership/group send** — a group send is already one signed envelope
   fanned to N recipients; with one-message-one-profile (§3.3), a mixed-audience group is N
   separate sends per distinct profile. Confirm no additional construction is needed. (Believed clean;
   flagged for the implementing track.)

### 11.2 `[DEFERRED TRACK]` Streaming seal / large payloads — the "B" track

The base spec assumes a **single AEAD over the whole envelope** (`design-doc-v1.md` §5.3.1) and that **payloads fit in RAM**
(the send path buffers the whole message; `fragment` takes the whole byte array; reassembly buffers
before the envelope is applied). That is fine for chat and small clips and **false for any large payload** — a multi-GB
encrypted video cannot be single-AEAD-sealed or verified without buffering the whole thing. This is a
**new sealing construction**, deferred to its own design track:

- **Scope:** large encrypted **files** *and* **recorded (non-live) voice/video messages** — any
  finite payload too large to hold in memory. (Live calls are *not* here — see §11.3.)
- **The construction:** a **chunked / streaming AEAD** — chunk-wise sealing with per-chunk tags plus a
  **chain or Merkle-tree binding** so chunks cannot be reordered, dropped, or substituted. This is a
  **profile-layer construction** (it changes what is guaranteed and how verification works — core-owned,
  authenticated, registered per §3.2), e.g. a `sealed-chunked` entry in the profile menu — **not**
  framing.
- **Consequences to resolve in that track:**
  - **#4 identity** — is a large payload's message-id the hash of the whole thing (a full pass) or a
    **root hash of the chunk tree**? (Chunk-tree root enables streaming verification + partial
    integrity; changes how #4 is computed for these messages.)
  - **`content_length` reopens** — reassembly deferred the `content_length` **streaming** optimization as
    "presumes a chunked seal we don't have." With a chunked seal we *do* have it, so streaming
    reassembly (decrypt chunk 0 → learn extent → know how many chunks) becomes real. Revisit that
    reassembly deferral.
  - **Receive pipeline** — the receive path assumes reassemble→whole-envelope→apply; a streamed payload may apply
    progressively and must never require full buffering. The RAM assumption gets fixed here.
  - **Framing composes unchanged** — a chunked-sealed payload is still fragmented/encoded by the
    transport layer (§5); framing stays security-neutral and below the (now chunked) seal.

### 11.3 `[DEFERRED — CORE-INVOLVED]` Steganographic / format-coherent embedding

The one cover technique that **binds** to the ciphertext and so re-enters the core (§9): hiding sealed
bytes *inside* a carrier that must stay valid (LSB stego in a real image; format-preserving encoding).
A **registered construction** with its own threat table when built — never a plugin bolting bytes onto
a carrier. Distinct from decoy-wrapping (§5.3), which does not bind and stays UI-side.

### 11.4 Live calls are **not** a deferred core track

Per §8.1, a WebRTC calling plugin is buildable on **existing core surface** (sealed signaling +
per-relationship signing + receive verification), with the plugin's mandatory fingerprint-match as its
only hard obligation. There is **no new core primitive** for calls, and secret-derivation (§8.2) is
**rejected**. Live calls are recorded here only to fix their status: **a plugin, built when wanted, not
a core track** — and explicitly *not* fused with the streaming-seal track (§11.2), which they share a
word with and almost no architecture.

### 11.5 Not in scope here (the transport-adapter layer)

The transport **adapter contract** itself (sizes, availability, rate/cadence, `send`/`inbound`), the
**send-path assembly** (the `design-doc-v1.md` §8.1 resolution chain), and the **key-ID prune hint**
(`design-doc-v1.md` §6.3) — those belong to the **transport-adapter layer**, which composes this note's
profile and framing definitions.
