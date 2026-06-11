0015: JSON Event Stream Encoding
===============================

*This is a draft proposal, not the final specification. Some of the details, terminology, and behaviors might change.*

Atproto event streams (subscriptions) are currently encoded as binary [DRISL-CBOR](https://dasl.ing/drisl.html) over WebSockets, with each frame holding two concatenated CBOR objects (a header and a payload). This works well for many usecases, but it can be awkward for browsers and casual consumers who need to pull in a CBOR decoder, and then deal with the two-object framing before they can touch the data.

This proposal introduces a JSON encoding for atproto subscriptions, so that consumers can read a stream with `JSON.parse` over standard text WebSocket frames. The immediate motivation is Jetstream v2, but the encoding is generic and may apply to any `subscription` Lexicon. 

Along the way, we formalize a versioning scheme for the wire transport and define a cleaner frame structure in which every message is a single, self-describing object. The JSON and CBOR encodings of this new structure are defined symmetrically, but only the JSON encoding is being shipped in the near term.

## Versioned Transports

We give the wire transport an explicit version, negotiated at connection time. There are two versions:

- **`xrpc.v0.cbor`** — the current protocol: each WebSocket frame contains two concatenated DRISL-CBOR objects, a header (`op`, `t`) followed by a payload. This is the **legacy default**: when no version is negotiated, this is what a connection gets. It is unchanged by this proposal.
- **`xrpc.v1.json`** and **`xrpc.v1.cbor`** — the new framing described below: one message per frame, each a single self-describing object. The `v1` framing is defined identically for both encodings; they differ only in how that single object is serialized.

The `v1` framing is specified here in both encodings so that the version scheme stays coherent and the two encodings never drift. However, only **`xrpc.v1.json`** is being shipped in the near term (for Jetstream v2). `xrpc.v1.cbor` is available to implementations that want it, but it is not the default for any existing stream, and migrating the existing firehose to it is explicitly out of scope (see below).

Keeping `xrpc.v0.cbor` as the unchanged default means the existing `com.atproto.sync.subscribeRepos` firehose is untouched by this work. That stream is heading to the IETF, and we do not want to perturb its wire format while that is in progress.


## Framing (`v1`)

In `v1`, each WebSocket frame contains exactly one object. The object is discriminated by its `$type` field. There are two specified frame types: `event` and `error`.

An **event** frame wraps a Lexicon message in an envelope:

```json
{"$type":"event","payload":{"$type":"com.atproto.sync.subscribeRepos#commit","seq":42,"repo":"did:plc:abc","rev":"3kx2"}}
```

An **error** frame carries an error type and optional description:

```json
{"$type":"error","error":"FutureCursor","message":"requested cursor is in the future"}
```

Error frame fields match `v0`: `error` is a required string (an error type name, no namespace or `#` prefix), and `message` is an optional human-readable description. As in the existing spec, a stream should close immediately after an error frame is sent.

The CBOR encoding (`xrpc.v1.cbor`) uses the identical structure, just serialized as a single DRISL-CBOR object per frame rather than JSON text.

A key properties here are that messages are a **single object** and **lex parser compatible**. Unlike `v0`, there is no separate header object carrying `op` and `t`. The `payload` object is exactly the Lexicon message, with the full `<nsid>#<fragment>` value in its `$type` field, so it can be handed directly to an existing Lexicon decoder with no unwrapping or transformation. The `event` and `error` discriminators are not NSIDs. They behave like other compound types such as `$type: "blob"` or `$type: "bytes"` in the data model. Existing tooling, such as lex SDKs and data parsers, work on these frames much more nicely.


## Negotiation

The encoding is negotiated at connection time using the standard `Sec-WebSocket-Protocol` header. 

- Client offers `xrpc.v1.json`, `xrpc.v1.cbor`, in preference order.
- The server selects one it supports and echoes it in its `Sec-WebSocket-Protocol` response header.
- If the client offers nothing recognized (or no `Sec-WebSocket-Protocol` header at all), the connection falls back to the `subprotocol` listed in the subscriptions lexicon document. If no subprotocol is lsted, subprotocol is assumed to be the legacy **`xrpc.v0.cbor`** transport. This preserves the behavior of all existing clients.

### `subprotocol` field

`subscription` Lexicons gain an optional `subprotocol` field, which declares the default transport for a stream when a client does not negotiate one. The value is a single subprotocol token, such as `xrpc.v1.json` or `xrpc.v0.cbor`.

This lets a stream that is defined from the outset to be JSON-native (such as Jetstream v2) declare `xrpc.v1.json` as its default, so that an unnegotiated connection receives JSON rather than legacy CBOR. When the field is absent, the default is `xrpc.v0.cbor`, preserving the behavior of every existing subscription.

The field only sets the default. Clients can still negotiate any encoding the server supports via `Sec-WebSocket-Protocol`, regardless of what the Lexicon declares.


## Practical Notes

**Compression.** The JSON encoding, with its repeated `$type` strings and textual representation, is larger on the wire than the equivalent CBOR. We recommend (but do not mandate) that servers support the standard `permessage-deflate` WebSocket compression extension to close most of this gap. Compression is negotiated separately from the subprotocol, via the standard WebSocket extension mechanism. 


## Rollout Plan

This change is purely additive and non-breaking. Because `xrpc.v0.cbor` remains the default, nothing in the existing network is affected by specifying these encodings.

The near-term goal is to specify the `v1` framing and negotiation mechanism and implement `xrpc.v1.json` in Jetstream v2. Other services interested in offering JSON streams may do the same.

Future updates will include broad SDK support, upgrading existing subscription endpoints, and subprotocol negoiation.
