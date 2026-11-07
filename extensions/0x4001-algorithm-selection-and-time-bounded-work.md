# Stratum V2 Extension: Algorithm Selection and Time-Bounded Work

`extension_type` = `0x4001`, allocated in Avian's Stratum v2 registry.

This file restates, in the specification's extension format, the normative text
in `doc/sv2.md` of Avian Core (AvianNetwork/Avian). That document is the
registry and is authoritative: where the two differ, it wins and this file is
the one in error. The extension was offered to the upstream registry as
`0x0005` and declined as out of scope (stratum-mining/sv2-spec#250), on the
maintainers' reading that an `extension_type` need only be unique among peers
that can share a connection, and an Avian connection and a Bitcoin one never
do. Section 3.4 of this fork's `03-Protocol-Overview.md` states the registry
that follows from that.

## 0. Abstract

This extension adds two mechanisms for chains whose proof of work is not a
single time-invariant function.

**Algorithm selection** lets a downstream role state which proof-of-work
algorithm a connection carries, on chains that mine several concurrently with
independent per-algorithm difficulty.

**Time-bounded work** lets an upstream role state an upper bound on `ntime`, for
chains whose hash function depends on a time epoch.

It also supplies the reply the Template Distribution Protocol does not define
for `SubmitSolution`, because on such a chain a refused solution has more ways
to be wrong and no cheap way for the client to find out which.

It defines seven new messages and one TLV field, and changes no existing
message's layout. It applies to both the Template Distribution Protocol and the
Mining Protocol, which need the same two statements for the same reasons.

The algorithm identifier is a chain-defined value. This extension specifies how
an algorithm is selected and acknowledged, not which algorithms exist.

Terms like "MUST," "MUST NOT," "REQUIRED," etc., follow RFC2119 standards.

## 1. Overview

### 1.1 Algorithm Selection

Stratum v2 assumes one proof-of-work function for a chain. A chain mining
several concurrently, each with its own difficulty and proof-of-work limit, has
no way for a downstream role to say which one a connection is for. An upstream
role can then only pin one algorithm per listening port, which is a deployment
constraint standing in for a protocol field.

A downstream role sends `AvianSelectAlgorithm` once per connection. The
algorithm is immutable for the life of the connection, because a connection
carries one difficulty and target stream. A Mining Protocol channel MAY select
a different algorithm for itself with the TLV field in section 1.4.

### 1.2 Time-Bounded Work

Stratum v2 bounds `ntime` from below only. `NewMiningJob.ntime_start` is defined
as the `nTime` at which hashing starts, and the specification states that it is
**not** the consensus minimum. No message states a maximum, and the Template
Distribution Protocol says outright that no protocol-level upper bound is
imposed.

For a chain whose hash function is selected by a time epoch, that is a silent
failure. A miner rolling `ntime` across an epoch boundary does not produce a
rejected share: it produces a correct hash of a different function, submitted in
good faith, which is indistinguishable from bad luck at the miner and from an
invalid share at the pool.

An upstream role therefore states both bounds explicitly, as absolute values
rather than as an offset or a mask. Two messages carry the same information
keyed for the two protocols, because `template_id` has no meaning to a worker
and `job_id` has none to a Template Distribution client. Keeping them separate
means an implementation serving both links, such as a pool, cannot confuse a
template id with a job id.

`ntime_min` restates the lower bound rather than deferring to `ntime_start`, and
the restatement is load-bearing: `ntime_start` is where hashing begins and is
explicitly not a rule, whereas `ntime_min` is a bound below which the upstream
role will refuse the submission. The two carry the same number and mean
different things. Carrying both bounds in one message also means an upstream
that computes a window once cannot emit halves that disagree.

### 1.3 Extension Activation (Negotiation Process)

The extension is negotiated through Extensions Negotiation (`0x0001`), after
`SetupConnection.Success` and before any protocol message that depends on it.

1. The downstream role completes `SetupConnection`.
2. It sends `RequestExtensions` listing `0x4001`.
3. On `RequestExtensions.Success` it sends `AvianSelectAlgorithm`, before
   opening a channel on the Mining Protocol, or before
   `CoinbaseOutputConstraints` on Template Distribution.
4. On `RequestExtensions.Error` it has not negotiated the extension and MUST NOT
   send any message defined here.

An upstream role SHOULD list `0x4001` in `required_extensions` and close
connections that do not negotiate it. On a chain where the two failure modes
above are silent, that is the difference between a miner being told and a miner
wasting its work. Avian's node requires it by default.

Nothing from this extension is sent to a client that did not negotiate it. Such
a client is served stock protocol only: no time window, no verdict on a
submission, and the stock `version` rule, under which BIP323's bits 5-28 may be
rolled and a submission that rolls them is refused.

#### Message Exchange Example

```
Downstream -> Upstream:  SetupConnection
Upstream -> Downstream:  SetupConnection.Success
Downstream -> Upstream:  RequestExtensions { request_id, requested_extensions: [0x4001] }
Upstream -> Downstream:  RequestExtensions.Success { request_id, supported_extensions: [0x4001] }
Downstream -> Upstream:  AvianSelectAlgorithm { pow_type }
Upstream -> Downstream:  AvianSelectAlgorithm.Success { pow_type, version_rolling_mask }
```

### 1.4 Per-channel algorithm (Mining Protocol)

`AvianSelectAlgorithm` settles the **connection's** algorithm, which is what a
channel is served on when it asks for nothing. A channel MAY override it for
itself by appending a TLV field (section 3.4.3) to `OpenStandardMiningChannel`
or `OpenExtendedMiningChannel`:

| TLV type | Data Type | Value |
| --- | --- | --- |
| `0x4001`, field `0x01` | `U8` | the `pow_type` this channel carries |

Encoded little-endian, as section 3.1 requires of every multibyte integer, so
an open that asks for `pow_type` 1 ends with `01 40 01 01 00 01`.

A server serves each channel the work, target and `nbits` of the algorithm that
channel asked for, on the same connection as channels asking for something
else. A channel's algorithm is fixed for its life; a worker wanting another
closes the channel and opens a new one. Omitting the field means the
connection's algorithm, which is what every client built before this field
sends, so nothing changed for them.

Two refusals, both sent as `OpenMiningChannel.Error` and refusing that channel
alone, because behind a proxy one rig asking for the wrong thing must not
disconnect the rest:

| `error_code` | Meaning |
| --- | --- |
| `unsupported-pow-type` | not a `pow_type` the chain defines |
| `pow-type-not-enabled` | an algorithm this server was not configured to serve |

A **malformed** TLV region is different and ends the connection.

## 2. Messages Defined by This Extension

All integers are little-endian, as elsewhere in Stratum v2.

### `AvianSelectAlgorithm` (Client -> Server)

| Field Name | Data Type | Description |
| --- | --- | --- |
| `pow_type` | `U8` | The proof-of-work algorithm this connection is for. Values are defined by the chain, not by this extension. |

Sent once per connection. A second `AvianSelectAlgorithm` on one connection
MUST be answered with `AvianSelectAlgorithm.Error` carrying
`algo-already-selected`, and the server MAY then close the connection, since a
client confused about which algorithm it selected cannot be served safely.

### `AvianSelectAlgorithm.Success` (Server -> Client)

| Field Name | Data Type | Description |
| --- | --- | --- |
| `pow_type` | `U8` | The algorithm now bound to this connection. |
| `version_rolling_mask` | `U32` | The bits of the block version the client MAY change. Zero means none. |

`version_rolling_mask` is stated here because a chain that encodes the algorithm
inside the block version cannot leave version rolling to BIP323. BIP323 reserves
bits 5-28 as nonce space, and the Template Distribution Protocol lets a client
set those bits freely in `SubmitSolution`; a chain whose algorithm field lies
inside that range has rolling either fail a consensus check or silently move
the share to another algorithm and another proof-of-work limit.

A client MUST NOT change any bit outside this mask. A server that cannot permit
rolling at all sends zero, and MUST also set `REQUIRES_FIXED_VERSION` in
`SetupConnection.Success` on the Mining Protocol, which is the enforceable half
of the same statement. **In this version of the extension the mask is always
zero**; the field exists so a later version can open bits without a format
change.

### `AvianSelectAlgorithm.Error` (Server -> Client)

| Field Name | Data Type | Description |
| --- | --- | --- |
| `error_code` | `STR0_255` | Why the algorithm was refused. |

Defined values:

| `error_code` | Meaning |
| --- | --- |
| `unsupported-pow-type` | Not an algorithm this chain defines. |
| `pow-type-not-enabled` | An algorithm this chain defines, but not one this server was configured to serve. |
| `algo-already-selected` | A second `AvianSelectAlgorithm` on one connection. |

These values are normative. An extension whose only stated error type is "a
`STR0_255`" cannot be implemented against, and the distinction between the first
two is the difference between "update your software" and "use another port".

### `AvianTimeConstraints` (Server -> Client)

Template Distribution Protocol.

| Field Name | Data Type | Description |
| --- | --- | --- |
| `template_id` | `U64` | The template these bounds apply to. |
| `ntime_min` | `U32` | Lowest acceptable `ntime`, inclusive. |
| `ntime_max` | `U32` | Highest acceptable `ntime`, inclusive. |

Sent after every `NewTemplate`, including an in-place refresh of a template
already issued, and after the `SetNewPrevHash` when one accompanies the
template. Both bounds are inclusive and are absolute values rather than a mask
over `ntime_start`, because an in-place refresh does not re-send
`SetNewPrevHash` and so has nothing to anchor a mask to.

`ntime_min` MUST equal the `ntime_start` the server supplies in `SetNewPrevHash`
for that template, so that a client holding both messages sees one window.

A server MAY re-evaluate any term of `ntime_max` that depends on wall-clock time
when a solution arrives, so that a long-lived template cannot be used to submit
a timestamp the chain would reject by the time it is seen.

Putting this message last on the Template Distribution link is deliberate and
differs from the Mining Protocol ordering below. A Template Distribution client
is a pool and can do nothing with a template until it holds the whole round, so
there is no instant in which it works without knowing its bounds; and a message
that always comes last gives a round an unambiguous terminator, which
`SetNewPrevHash` cannot, since it is sent only when the tip moved.

### `AvianJobTimeConstraints` (Server -> Client)

Mining Protocol. This is a **channel message**: bit 15 of `extension_type` is
set, so it appears on the wire under `0xc001`. Bit 15 is the `channel_msg` flag
and is ignored when an implementation looks up which extension a message belongs
to.

| Field Name | Data Type | Description |
| --- | --- | --- |
| `channel_id` | `U32` | The channel these bounds apply to. |
| `job_id` | `U32` | The job these bounds apply to. |
| `ntime_min` | `U32` | Lowest acceptable `ntime`, inclusive. |
| `ntime_max` | `U32` | Highest acceptable `ntime`, inclusive. |

Sent for every `NewMiningJob` and `NewExtendedMiningJob`. Both bounds are
inclusive. The ordering is normative, because a worker activated by
`SetNewPrevHash` is required to begin hashing immediately and must never hold an
active job whose `ntime_max` it does not know:

```
active job, ntime_start present
    AvianJobTimeConstraints.ntime_min MUST equal NewMiningJob.ntime_start

future job, ntime_start absent
    the constraints MAY be sent with the job, if the bounds are known then
    otherwise they MUST be sent immediately before the SetNewPrevHash that
    activates the job
    either way, the ntime_start that SetNewPrevHash supplies MUST equal the
    ntime_min in force for that job
```

A channel that already has an active job when the tip moves is resolved as the
future-job case: a new future job, then its constraints, then the
`SetNewPrevHash` that activates it.

A client MUST NOT begin hashing a job until it holds both the applicable time
constraints and the activation information.

Where a server holds a job open across a template refresh, it MUST re-send this
message with the new bounds. It applies to extended channels unchanged.

### `AvianSubmitSolution.Success` (Server -> Client)

Template Distribution Protocol. The reply the core protocol does not have: its
message table ends at `SubmitSolution` with no `.Success` and no `.Error`, so a
client that submits a block learns nothing from the protocol, not even that it
was heard.

| Field Name | Data Type | Description |
| --- | --- | --- |
| `template_id` | `U64` | The template the solution claimed. |
| `block_hash` | `U256` | The block hash as the chain defines it. On Avian that is the proof-of-work hash. |

`.Success` means the server constructed the block and its validation accepted
it. It does **not** mean the block became the tip: one that loses a race is
accepted here and then orphaned. A client learns the rest by matching
`block_hash` against the next `SetNewPrevHash`, which is also what `block_hash`
is for on the error-free path, since `template_id` cannot say which of several
nonces in flight against one template is being answered.

### `AvianSubmitSolution.Error` (Server -> Client)

Template Distribution Protocol.

| Field Name | Data Type | Description |
| --- | --- | --- |
| `template_id` | `U64` | The template the solution claimed. |
| `error_code` | `STR0_255` | Why it was refused. |

No `block_hash`, deliberately: most refusals happen before the proof-of-work
hash is computed, because on a chain without a cheap pre-filter that hash costs
more than every check before it put together, and a few attacker bytes must not
be able to buy one. A client with several nonces in flight learns the reason
but not which nonce, which is the right way round: the reason is what it can
act on.

Defined `error_code` values, which are normative:

| `error_code` | Meaning |
| --- | --- |
| `unknown-template` | no such template, or one this connection was never sent |
| `version-mismatch` | `version` differs from the template's |
| `bad-algo-id` | the version's algorithm field names no algorithm this chain defines |
| `ntime-out-of-range` | outside the window `AvianTimeConstraints` gave |
| `ntime-wrong-epoch` | past the end of the template's hash-function epoch |
| `coinbase-too-large` | over the advertised size, weight or the block limit |
| `coinbase-malformed` | does not deserialize as a transaction |
| `coinbase-non-canonical` | deserializes, but re-serializes to different bytes |
| `coinbase-bad-scriptsig` | wrong length, or does not start with `coinbase_prefix` |
| `coinbase-bad-fields` | version, locktime or input sequence differ from the template's |
| `coinbase-bad-witness` | `vin[0]` witness is not the single 32-byte BIP141 reserved value |
| `missing-required-output` | an output from `coinbase_tx_outputs` is absent |
| `bad-commitment` | the witness commitment is wrong or is not the highest-indexed one |
| `value-too-high` | outputs exceed `coinbase_tx_value_remaining` plus the required ones |
| `too-many-sigops` | over the advertised sigop budget |
| `block-too-large` | template plus this coinbase exceeds the block limit |
| `pow-insufficient` | the proof-of-work hash does not meet this algorithm's target |
| `consensus-rejected` | passed every check here and validation refused it anyway |

Two refusals are not answered. A solution that trips a rate limit is not,
because replying to a flood is amplification, and one whose payload does not
decode is not, because nothing in it can be trusted to name a template. Both
MAY drop the connection. A refusal that takes a client past a server's
disconnect threshold MUST be sent before the connection closes, not after.

Neither message can be relied on by its absence. There is no way to ask a peer
which message numbers inside a negotiated extension it implements, so a client
MUST NOT treat silence as a verdict or run a timeout off it. Both are positive
information only.

## 3. Message Types

| Message Type (8-bit) | channel_msg_bit | Message Name | Protocol |
| --- | --- | --- | --- |
| `0x00` | 0 | `AvianSelectAlgorithm` | both |
| `0x01` | 0 | `AvianSelectAlgorithm.Success` | both |
| `0x02` | 0 | `AvianSelectAlgorithm.Error` | both |
| `0x03` | 0 | `AvianTimeConstraints` | template distribution |
| `0x04` | 1 | `AvianJobTimeConstraints` | mining |
| `0x05` | 0 | `AvianSubmitSolution.Success` | template distribution |
| `0x06` | 0 | `AvianSubmitSolution.Error` | template distribution |

## 4. Implementation Notes

### 4.1 One identifier across two protocols

The three `AvianSelectAlgorithm` messages are byte-identical on both protocols,
because they are connection-scoped and carry nothing protocol-specific. Only the
time constraints differ, and only in how they key the work.

One identifier rather than two is deliberate. A pool is a Template Distribution
client facing the node and a Mining Protocol server facing its workers. Under
two identifiers it would negotiate two extensions to obtain one capability, and
the same three message definitions would exist twice.

### 4.2 Narrowing a window downstream

A role that is a client on one link and a server on the next, which is what a
pool is, derives the window it issues from the window it received. It MUST
narrow, never widen: the upstream will refuse a solution outside its own window,
and a worker told it has more room than the upstream allows will produce work
that cannot be submitted.

It MUST also not narrow to nothing. On a chain or network that pins the window
shut, `ntime_min` already equals `ntime_max`, and any narrowing leaves a worker
with no legal `ntime`. An empty window is a protocol error, and the job MUST be
refused rather than issued in a form that cannot be worked.

### 4.3 Refusing a client that demands version rolling

The specification states that a server MUST NOT set `REQUIRES_FIXED_VERSION` if
the client set `REQUIRES_VERSION_ROLLING`, which makes the two mutually
exclusive rather than one overriding the other. A server on a chain that encodes
the algorithm in the block version cannot honour rolling at all, so a client
demanding it MUST be refused with `SetupConnection.Error` rather than answered
and then held to a fixed version it did not agree to. `version_rolling_allowed`
MUST be false on every `NewExtendedMiningJob`.

### 4.4 Error codes a server on such a chain needs

The specification defines no `SubmitShares.Error` values; the familiar ones are
the reference implementation's constants. Three refusals follow from this
extension. The first reuses the reference implementation's string, since it
means the same thing there; the other two have no counterpart:

| `error_code` | Meaning |
| --- | --- |
| `invalid-version` | The submitted version differs from the job's, which `REQUIRES_FIXED_VERSION` forbids and which would move the share to another algorithm's proof-of-work limit. |
| `invalid-ntime` | The submitted `ntime` is outside the window `AvianJobTimeConstraints` gave for that job. The reference implementation's `invalid-ntime-start` names only the lower bound; this covers both. |
| `too-many-shares` | The channel exceeded its submission allowance. Not about validity: a chain with no cheap proof-of-work pre-filter must meter shares, and the remedy is a harder share target, not a retry. |

Two more `SetupConnection.Error` and `OpenMiningChannel.Error` codes follow the
same pattern: `missing-standard-jobs-flag`, for a worker that cleared
`REQUIRES_STANDARD_JOBS` and so may be served jobs it cannot interpret, and
`min-extranonce-size-too-large`, the reference implementation's string, for an
extranonce the 100-byte scriptSig cannot fit beside the height push. The codes
a solo pool uses to refuse a payout address it cannot pay are Avian pool
conventions rather than part of this extension, and are in `doc/sv2.md`.

### 4.5 Forward compatibility within this extension

Message numbers inside an allocated `extension_type` are the extension's own to
assign, so a later message can be added without a new allocation. That holds on
one condition, which implementations MUST observe:

**A receiver MUST ignore a message whose `extension_type` it has negotiated and
whose `msg_type` it does not recognise.** Log it and read the next frame. It
MUST NOT be treated as a protocol violation, and the connection MUST NOT be
closed.

Negotiation agrees an `extension_type`. It says nothing about which message
numbers inside it either side knows, because the two implementations were built
at different times. Without this rule, adding a message is not compatible at
all: it breaks every peer built before it. This is not hypothetical, and the
worked example is in the Block Found Notification extension's section 4.1,
which exists as a separate extension because of it. Messages `0x05` and `0x06`
were added to this extension under the rule, and are replies rather than
capabilities: a client that submits nothing receives nothing.

Three limits on the rule, because it is narrower than it first looks.

**It licenses optional additions only.** A message a peer can do without may be
ignored safely. A message that changes which work is valid may not: a future
constraint that narrowed a window, silently skipped, would leave a client
producing work the server will refuse, which is the failure this extension
exists to prevent. Such a message needs explicit capability negotiation, and
since the `extension_type` is Stratum v2's only capability granularity, that
means another allocation rather than another `msg_type`.

**It binds the endpoint that receives the message.** A proxy or translating
role forwarding Stratum v2 onward MUST forward what it does not recognise
rather than discard it, or it silently strips capability from the roles behind
it.

**It cannot be retrofitted.** Software already deployed does not acquire the
rule by this document existing. Until peers carry it, a new message has to be
suppressed for peers known not to handle it, or placed behind its own
identifier so they never ask for it.

### 4.6 Changing a message that already exists

Changing the layout or meaning of a message number already defined is not
covered by the rule in 4.5 at all. The field layouts of `0x00` through `0x06`
are what a third-party pool or miner compiles against, and altering one
silently is the same class of failure this extension exists to prevent. Treat
any such change as a new extension with its own identifier.

### 4.7 Reference implementation

Avian implements this extension on both links, with two concurrent algorithms,
X16RT and MinotaurX, and a 128-second X16RT epoch that supplies the time bound.
Its mapping is `pow_type` 0 for X16RT and 1 for MinotaurX, which is an example
of a chain-defined mapping rather than part of this format.

- Node (Template Distribution server): Avian Core, `src/sv2/`, documented in
  `doc/sv2.md`, which is the normative text for this extension.
  https://github.com/AvianNetwork/Avian/pull/267
- Pool and miner (Mining Protocol server and client): `avian-sv2`.
