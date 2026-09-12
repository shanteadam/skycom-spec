# Group Membership & Addressing — v1

**Status.** Normative. Membership events are ordinary causal-DAG messages, so they compose with the ordering layer (`design-doc-v1.md` §9/§10) and require **no change to the receive path**. Membership is defined **above** the messaging core, and the wire carries **no group API**. *Reference convention: a bare `§N` is a section of **this document**; a reference to a section of another document names that document explicitly.*

---

## 1. The model in one paragraph

A **group is a keystore context record** — a sibling to an individual contact entry — holding an **append-only** set of member **signing** public keys (how you recognize their messages), their group **encryption** keys (whom you fan out to), and an optional **local** group name. The record is **derived by folding** `create`/`add` **membership events** carried as DAG messages; it is not a mutable shared object and there is **no group key**. A group has a **content-addressed identity**: its ID *is* the founder's `create` event's envelope hash. Membership grows by **introduction** — *anyone* already in the group may introduce a newcomer (an `add` event, fanned to all members, carrying the newcomer's group-scoped keys); whether a receiving client accepts, mutes, or ignores that introduction is **client policy, not protocol**. Each key carries an **origin** — an opaque string the client/plugin/user sets, which the protocol **stores and surfaces but never interprets or authenticates**. And "is this a group message?" is **not a property of the message**: the wire stays group-blind; attribution is a **read** that surfaces one DAG node in *every* context record whose key matches the message's signing key (0, 1, or many).

---

## 2. Group = a keystore context record (sibling to a contact entry)

The individual-contact keystore entry (`design-doc-v1.md` §7) holds *their* per-relationship signing public key (recognize their messages), your keys for them, and **local** metadata. A **group context record** is the same structure at **N-member cardinality**:

- an **append-only** map: member **signing** public key → { that member's group **encryption** key, the key's **origin** } — supporting *multiple keys per member* (like a contact may have several), each with its **own** origin record;
- the group **ID** (§4);
- an optional **local** group name (self-facing, never on the wire — like a contact label).

A contact entry is "a context with one member's key(s)"; a group record is "a context with N members' keys." They feed **one** attribution mechanism (§7): a message's signing key is looked up across **all** context records — contact and group alike — and surfaces wherever found. This is why the group is a *sibling* to a contact, not a new kind of thing: same lookup, different cardinality.

The record is a **derived fold** (§5) over the membership events you have seen — local, monotonic, convergent. It is not signed, not shared, not authoritative; it is *your view*, assembled from events.

---

## 3. Membership events are DAG messages

Two typed events, carried as ordinary signed envelopes (`design-doc-v1.md` §6) with a membership content-type:

- **`create`** — the founder opens the group. Carries the founder's group-scoped persona keys (signing + encryption), signed by the founder. **The group ID is this event's envelope hash** (§4).
- **`add`** — a member introduces a newcomer. Carries the newcomer's group-scoped keys (provably owned — proof-of-possession, as contact bundles do), an origin for those keys, and the group ID; signed by the introducing member.

Because they are DAG messages, membership events are **causally ordered, parked when a causal parent is not yet held, gap-checked, and applied to the DAG** exactly like any other message (`design-doc-v1.md` §9, §10) — the fold reads them in causal order. No new transport, no new ordering machinery.

---

## 4. Content-addressed group identity

The group **ID is the founder's `create` event's envelope hash** (`design-doc-v1.md` §6.4) — content-addressed, like every other identifier in the protocol (message IDs, DAG nodes, dedup keys). Consequences:

- The founder is **cryptographically bound** to the group (the genesis is signed by their group persona; its hash is the group's name).
- Every `add` **references** that ID; the keyring record is **keyed** by it.
- A **fork** (§8) records the **parent** group's ID as lineage — an edge to the genesis it split from.
- No new identifier type, no uniqueness/collision machinery, no external registry — the ID inherits all the content-addressing guarantees the protocol already relies on.

The ID is not human-memorable — nothing here is; the client labels it locally (like contact labels).

---

## 5. The fold — deriving the keyring from events

The keyring is an **append-only fold** over the `create` + `add` events in the causal DAG:

- **`create`** seeds the record with the founder as sole member (their keys, their origin).
- each **`add`** appends the newcomer's key(s) + origin, attributed to the **introducer** (recorded: *who* vouched — provenance, §6).
- **Permissive validation:** an `add` folds if it is well-formed and **signed by some group member** (so you know *who* introduced whom); the protocol does **not** enforce a privilege hierarchy or a chain back to the founder. *Anyone* in the group may introduce.
- **Monotonic ⇒ convergent:** events only add, never contradict (there is no `remove` — §8). Two members who have folded the same events have the same keyring; one who has not yet seen an `add` simply lacks that member until the event arrives, then converges. No concurrency policy is needed (the remove-conflict problem is dissolved, not solved).

---

## 6. Key origin — recorded, surfaced, never interpreted

Each key carries an **origin**: an **opaque, client-set string** describing how the key arrived (e.g. `secure-channel-exchange`, `introduction:<introducer>`, `key-file-import`, `unauthenticated`, `unknown` — *examples*, not an enum). The protocol has **no authority to authenticate how a key arrived** — there is no programmatic oracle for "this really came from a secure exchange" — so it **cannot and does not validate, interpret, or trust** the origin. It:

- **stores** the origin verbatim, per key, and **surfaces** it to clients on every attribution;
- **authenticates only what it can** — the *signature* (this message was signed by this key) — and hands the origin alongside as **unverified metadata**.

The **client** (or exchange plugin, or the user importing a key by hand) **authors** the origin string and **decides what it means** — what to surface, whether to warn, how to render trust. The honest primitive is **objective provenance the client interprets**, not a protocol-assigned judgment: the protocol records how a key was obtained and assigns it no meaning. This is the **same model the whole specification uses** — contact keys and group keys alike carry an opaque, client-set origin (`design-doc-v1.md` §7.1).

---

## 7. Context attribution — a "group message" is a read, not a wire fact

The wire is **group-blind** (a message carries no group field). **Read that as a claim about protocol fields:** the envelope defines no group identifier for the core to set or read. A membership event's *sealed payload* does carry the group ID (§3), but that is typed content the core never inspects — and an ordinary group message carries no group ID at all, which is why the question below is answered by a read rather than a lookup. "Which context(s) does this message belong to?" is a **receive-side read** over the keyring records:

- Given a DAG node, look up its **signing key** across **all** context records (contact and group).
- Surface the node in **every** matching context (**0, 1, or many**). One message can appear in a group *and* a 1:1 if the same key is in both records.
- **One node, always** — the DAG holds a single node per message (the causal-DAG ordering layer is unchanged, `design-doc-v1.md` §9/§10); context is a **read-side lens**, not a partition. There are no per-context DAGs.

A key normally lives in **exactly one** context (a fresh persona per relationship — per-relationship isolation, `design-doc-v1.md` §4.3). So a message normally fans to exactly one view. A key matching **more than one** context is the **anomaly** — key reuse across contexts — and the fan-to-both is precisely how the client *sees* it ("why is this in both my group and my Alice thread?"). The **one-to-many attribution IS the compromise indicator**, surfaced as-is; the protocol **detects and reports** the collision, it does **not** resolve, dedup-to-one, or reject it. What the client does with it (flag "possibly compromised", warn, ignore) is client policy.

---

## 8. Membership change without removal

- **No removal primitive.** Membership is append-only and forward-only: once added, a member is in the group for as long as it exists.
- **Fork** for exclusion: to continue without X, a member issues a **new `create`** (a new genesis, new group ID, new context, new group-scoped personas) that **records the parent group's ID as lineage**. Consistent with the emergent model (a group is not an object to edit); maps to real behaviour ("the chat without X"). G′ mints its own personas, so it is not linkable to G except by its own members.
- **Mute** is a **local** client filter: decline to *surface* a member's messages. **[Hard boundary — mute ≠ removal]** X retains access, others still fan out to X, X still sees the group. Never fuse mute into removal (same discipline as store-vs-display `design-doc-v1.md` §9.6, origin-is-client §6). If someone genuinely needs X gone, that is a **fork**, not a mute.

---

## 9. Honest limits (stated plainly)

1. **A member can reshare keys.** Nothing stops a member from forwarding the group's keys (or messages) to an outsider to let them listen. Unavoidable in a local-first design — you cannot stop someone from sharing what they can already read. The introduction gate authenticates *who* introduced whom; it does **not** prevent resharing.
2. **No forced removal** (§8): append-only membership; recourse is fork or mute. Weaker than server-mediated groups; the honest local-first answer (you cannot compel another member's client).
3. **No-arbiter at the addressing layer** (`design-doc-v1.md` §9.3): keyring views may differ; two members can address different sets; convergence only over the shared event history.
4. **In-group correlation is unavoidable:** members necessarily learn they share a group (one persona per group message, and fan-out copies converge to a single node; per-recipient isolation *within* a group is not available, only *across* contexts).
5. **Key reuse is detectable, not preventable** (§7): the protocol surfaces a key in multiple contexts; it cannot stop a peer from reusing one.

---

## 10. What this reuses (composes, does not re-open)

- The **signed envelope** (`design-doc-v1.md` §6 — the membership event is a typed payload), the **envelope hash** (`design-doc-v1.md` §6.4 — the group ID), the **keystore** (`design-doc-v1.md` §7 — the context record is a sibling entry; proof-of-possession for added keys as bundles do), **fan-out** (`design-doc-v1.md` §8.2 — an `add` and a group message are fan-outs to the members' encryption keys), the **causal-DAG ordering** (`design-doc-v1.md` §9/§10 — events are DAG messages, ordered/parked/applied), and the **receive/resolution** path (`design-doc-v1.md` §10 — attribution is a read over the same resolution machinery, unchanged).
- The **addressing participant** (whom to fan out to) reads the keyring's **encryption** keys; the **attribution** side reads its **signing** keys. One record, both directions.

---

## 11. Open items (deferred — documented, not resolved here)

- **Group-persona key rotation / supersession** under append-only (how a member rotates a group key without a `remove`). Genuinely hard; not needed for a first working layer.
- **Membership-event schema specifics** beyond the model above — deferred.
- **Retrieval-privacy / transport interaction** (a group over a shared board) — transport is itself a deferred epoch (`design-doc-v1.md` §10.1, §8.x).

---

## 12. Dependencies & guard

- **Depends on the causal-DAG ordering layer** (`design-doc-v1.md` §9/§10): membership events are causal-DAG messages — ordered, parked when a causal parent is not yet held, gap-checked, and applied to the DAG.
- **No change to the receive path**: the core still sees only fan-out units + independent per-member receive. This track **consciously extends** the "no group API on the wire" guarantee — the keyring is **derived, self-facing state above** the core, and a "group message" is a **read**, never a group object *inside* the core or a field *on* the wire.

---

*Membership docks above the messaging core, composing with the ordering layer.*
