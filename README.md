# skycom

**A local-first protocol for cryptographic identity and messaging.**

skycom is a specification for messaging that keeps conversation state with the participants rather than with a server, carries it over any transport, and holds its security properties without trusting the carrier.

This repository holds the **normative specification**. It defines what any conformant implementation must do, and describes no particular implementation.

> **Status: a proposal, preparing for submission as an Internet-Draft.**
> Version 1 is specified and has a reference implementation used for testing, but it has not been through independent review or a standards process. Each document carries its own named open items and deferred tracks. It should be treated as a design under evaluation, not a settled standard.
>
> **No license is currently granted — see [`NOTICE.md`](NOTICE.md) before copying, modifying, or implementing.**

---

## What skycom is

**Local-first.** A conversation's structure lives in a causal hash-DAG carried inside the messages themselves. No server holds the shape of a conversation, so there is no party to compel for it.

**Transport-agnostic.** The protocol produces sealed bytes and hands them to a transport. The same bytes ride an SMS gateway or a relay, and the framing layer is built to reach channels as constrained as a board that accepts only images — though the concrete encodings that would do so are illustrative, not yet pinned. How much metadata a message exposes is a property of the transport a deployment chose, not a fixed property of the protocol — and the protocol adds no linkage of its own.

**Key-blind at the transport seam.** A carrier receives opaque sealed bytes and routing information. It cannot read a message or forge one, and this does not depend on the carrier behaving well.

**Per-relationship isolation.** Every relationship uses independently-generated keys. Nothing in the protocol links two contacts, and no facility exists for asking it to.

skycom is **not** an application, a transport, a network, a key-exchange mechanism, a server, or a delivery guarantee. It is the layer beneath all of those.

---

## The documents

| | |
|---|---|
| [**`design-doc-v1.md`**](docs/design-doc-v1.md) | The core specification: identity, the envelope format, sealing and opening, causal ordering, fragmentation, storage boundaries, and the twenty-one invariants an implementation must uphold. **Start here.** |
| [**`exchange-spec-v1.md`**](docs/exchange-spec-v1.md) | Key exchange: contact-bundle format, proof of possession, and the plugin boundary. Deliberately separate — messaging has no awareness of how keys were established. |
| [**`delivery-profiles-v1.md`**](docs/delivery-profiles-v1.md) | Delivery profiles, transport framing, and the key-blind adapter boundary. The adapter contract itself — sizes, availability, `send`/`inbound` — belongs to the transport-adapter layer and is out of scope there. |
| [**`membership-design-v1.md`**](docs/membership-design-v1.md) | Group membership and addressing: membership events as ordinary DAG messages. |

Section 15 of the core specification lists the twenty-one invariants, each tagged with its enforcement class — whether a violation is caught by a counterparty's cryptography, or merely forbidden. That distinction is the most useful thing to understand before reading anything else.

---

## Where to start

**Evaluating the design, or reviewing it for weaknesses.** Read `design-doc-v1.md` §3 (what rides outside the seal, and the enforcement classification) and §15 (the invariants). The reference implementation's repository carries an architecture and security review guide written for exactly this purpose, including a threat model and a statement of where the evidence is weakest.

**Building an independent implementation.** The specification is the contract; the reference implementation is one reading of it. Where they disagree, the specification governs. The conformance vectors — byte-level fixtures an implementation must reproduce — live with the reference implementation and are the practical test of whether your reading matches. **Note the licensing position in [`NOTICE.md`](NOTICE.md) before starting.**

**Using skycom.** There is nothing to use yet. The reference implementation is a library for testing the specification, not a product; there is no client and no production transport adapter.

---

## The reference implementation

[`skycom-core`](https://github.com/shanteadam/skycom-core) implements this specification for testing purposes: to demonstrate the specification is complete enough to build from, to pin the constructions with byte-level conformance vectors, and to give reviewers something concrete to attack.

It is not a product and does not aim to be one. It performs no key exchange, no retrieval, and no user interaction — those are separate layers, some of which do not exist yet.

---

## Versioning

Specification versions are tagged, and section numbers are stable **within** a tagged version. Once a version is tagged, renumbering requires a version bump — so a citation that names its version, of the form *design-doc §5.3.1 (v1.0)*, stays valid against that tag. That is the form to cite in once a tag exists.

**No version has been tagged yet.** Section numbers in the current text are therefore not yet stable and may move before the first tag; one section has already been renumbered during preparation, before anything external could depend on it. Until a tag exists there is no version to cite against, so cite the current text only provisionally.

The reference implementation states which specification version it conforms to.

---

## Contributing

Not yet open to contributions. The licensing position ([`NOTICE.md`](NOTICE.md)) has to be settled first, and accepting contributions before then would foreclose options that are currently open.

Review, questions, and reports of specification defects are welcome via issues.
