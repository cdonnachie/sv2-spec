# Stratum V2 Extension: Block Found Notification

`extension_type` = `0x4002`, allocated in Avian's Stratum v2 registry.

This file restates, in the specification's extension format, the normative text
in `doc/sv2.md` of Avian Core (AvianNetwork/Avian). That document is the
registry and is authoritative: where the two differ, it wins and this file is
the one in error. The extension was offered to the upstream registry as
`0x0006` and declined as out of scope (stratum-mining/sv2-spec#250), on the
grounds that Bitcoin's non-custodial design already answers it: a Job
Declarator Client publishes its own blocks and knows the outcome. Avian has no
Job Declaration role, so the gap is real there.

## 0. Abstract

This extension lets a Mining Protocol server tell a client what became of a
share that was also a block: that the server forwarded it, and that the server
observed it as its node's active tip.

It defines one message, changes no existing message and adds no TLV fields. It
is chain-agnostic and contains nothing about any particular proof of work.

It is deliberately a **separate, optional** extension rather than a message
inside `0x4001`. See section 4.1, which is the part of this document worth
reading even if the message itself is not interesting.

Terms like "MUST," "MUST NOT," "REQUIRED," etc., follow RFC2119 standards.

## 1. Overview

### 1.1 What is missing today

A client can recognise a block candidate on its own. `SetNewPrevHash` carries
`nbits`, the compact network target for the candidate header, so a client that
decodes it can compare its own hash against it.

What it cannot establish is what happened next:

- whether the server forwarded the solution to a node at all
- whether that node accepted it

`SubmitShares.Success` does not answer either question. It means "accepted as a
share", and a server sends it for shares that are not blocks as well. The
Template Distribution Protocol gives `SubmitSolution` no reply at all, which is
why `0x4001` supplies one; against a node that does not send it, even the
server learns the outcome only by watching the tip.

For a server paying by share this is unremarkable, because what became of the
block is the server's business and the client is paid either way. For a
**non-custodial or solo** arrangement it is backwards: the proposition is that
consensus pays the client directly for the blocks it finds, and the client is
left unable to distinguish a block it won from one that lost a race.

### 1.2 What the server already knows

Nothing here asks a server to acquire information it does not have.

A server knows a share met the network target, because that is the test it
applies in order to decide whether to forward it. And it can observe the
outcome without asking its node anything beyond what it already receives: it
knows the hash it submitted, and the next `SetNewPrevHash` carries the new
prev hash. A match means that node adopted the block.

### 1.3 Extension Activation (Negotiation Process)

Negotiated through Extensions Negotiation (`0x0001`), after
`SetupConnection.Success`.

1. The client completes `SetupConnection`.
2. It sends `RequestExtensions` listing `0x4002`, alongside any other
   extensions it wants.
3. On `RequestExtensions.Success` listing `0x4002`, the server MAY send
   `BlockFound` on that connection.
4. If `0x4002` is not in the accepted list, the server MUST NOT send
   `BlockFound`, and the connection proceeds normally in every other respect.

A server MUST NOT list this extension in `required_extensions`. It is a
notification: a client that does not want it is fully servable without it, and
requiring it would exclude clients for no benefit.

#### Message Exchange Example

```
Client -> Server:  RequestExtensions { request_id, requested_extensions: [0x4001, 0x4002] }
Server -> Client:  RequestExtensions.Success { request_id, supported_extensions: [0x4001, 0x4002] }
...
Client -> Server:  SubmitSharesStandard { ... }
Server -> Client:  SubmitShares.Success { ... }
Server -> Client:  BlockFound { channel_id, sequence_number, block_hash, confirmed: false }
...
Server -> Client:  BlockFound { channel_id, sequence_number, block_hash, confirmed: true }
```

## 2. Messages Defined by This Extension

### `BlockFound` (Server -> Client)

A **channel message**: bit 15 of `extension_type` is set, so it appears on the
wire under `0xc002`. Bit 15 is the `channel_msg` flag and is ignored when an
implementation looks up which extension a message belongs to.

| Field Name | Data Type | Description |
| --- | --- | --- |
| `channel_id` | `U32` | The channel whose share this was. |
| `sequence_number` | `U32` | The `SubmitShares` sequence number it refers to. |
| `block_hash` | `U256` | The hash of the block that share produced. |
| `confirmed` | `BOOL` | False when the server forwarded it. True once the server observed it as its node's active tip. |

A server sends this when it forwards a share to a node as a block, with
`confirmed` false, and MAY send it a second time for the same
`sequence_number` with `confirmed` true.

**`confirmed` is a narrow statement and MUST NOT be read as more.** It means
that one node, the one the server is talking to, had this block as its active
tip at the moment the server looked. It is not a depth, it is not a guarantee
against a later reorganisation, and it says nothing about the rest of the
network. A client wanting more than that has to ask a node of its own.

A block that loses a race receives **no second message**, because a server
cannot distinguish a block that lost from one still propagating. A client MUST
therefore treat the absence of a `confirmed` message as unresolved rather than
as failure.

`block_hash` is the block hash as the chain defines it, in the byte order the
chain's own protocol uses for a block hash in a message field. On Avian that is
the proof-of-work hash, not SHA256d.

## 3. Message Types

| Message Type (8-bit) | channel_msg_bit | Message Name |
| --- | --- | --- |
| `0x00` | 1 | `BlockFound` |

## 4. Implementation Notes

### 4.1 Why this is its own identifier

This began as message `0x05` of `0x4001`, and moving it out was not tidiness.
It fixes a compatibility failure, and the failure is general enough to be worth
stating here.

Extensions Negotiation agrees an `extension_type`. It carries no per-extension
version and no list of supported message types, so **negotiating an extension
does not negotiate every message that extension will ever contain.** Two
implementations built at different times can agree on an identifier and
disagree about what is inside it.

That is exactly what happened. A server was updated to send this notification
inside an extension that every client was required to negotiate. Clients built
before the message existed had negotiated the extension, received a message
they did not recognise, treated it as a protocol violation and ended the
session, then carried on hashing into a connection nobody was serving.

Two fixes exist and only one of them works on deployed software.

The first is a rule that a receiver must ignore an unrecognised `msg_type`
inside an extension it has negotiated. That rule is correct and worth having,
and `0x4001` section 4.5 states it, but it only licenses **optional**
additions, it has to be in place before the addition rather than after, and it
cannot be retrofitted to clients already in the field.

The second is to put the new capability behind its own identifier. Then a
client that does not know about it does not ask for it, and is never sent it.
Nothing has to be ignored, because nothing unexpected arrives. That is this
document. A required extension cannot carry an optional capability, because
negotiating it is mandatory and a client has no way to decline one message
within it.

The general shape, for anyone adding to an extension: a notification a peer can
do without may be added under the ignore rule. Anything a peer must act on, and
anything at all that must work against already-deployed peers, needs its own
identifier. Stratum v2's only capability granularity is the `extension_type`,
so "negotiate it separately" and "allocate another identifier" are the same
sentence.

`0x05` of `0x4001` was later reused for `AvianSubmitSolution.Success`. Nothing
carrying the block notification under that number was ever published, so there
was no peer anywhere that compiled it and nothing to protect by reserving it.

### 4.2 Why this is not upstream

It was offered to the Stratum v2 registry as a chain-agnostic extension and
declined, on the grounds that Bitcoin's non-custodial design already answers
it: a Job Declarator Client publishes its own blocks and knows the outcome.
That is true for Bitcoin and not for a chain without a Job Declaration role,
which is why it lives in Avian's registry.

### 4.3 What a client should do with it

Report it, and not much else. The useful behaviours are telling the operator a
block was found, distinguishing forwarded from adopted, and leaving the
unresolved case unresolved rather than reporting a loss. A client MUST NOT
treat a `confirmed` notification as settled payment.

### 4.4 Reference implementation

Implemented by Avian's Stratum v2 pool, `avian-sv2`, serving its own miner.
Avian's other extension, `0x4001`, is required where this one is optional, and
neither depends on the other.

- Normative text: `doc/sv2.md` in Avian Core.
  https://github.com/AvianNetwork/Avian/pull/267
