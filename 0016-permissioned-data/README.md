# 0016 Permissioned Data

*This is a draft proposal, not the final specification. Details, terminology, and behaviors are all likely to change.*

## Introduction

The [AT Protocol][ATPROTO] is a protocol for public broadcast data. Users publish records into a repository on their PDS, and applications crawl those repositories to build views. Authority rests in the DID that publishes a record, and records are signed, redistributable, and universally addressable.

This document specifies a **permissioned data protocol** for data that is _not_ public, data with an access perimeter. It runs alongside the public protocol and serves modalities such as:

- **Personal data**: bookmarks, mutes, drafts
- **Gated content**: paid newsletters, subscriber-only posts
- **Socially shared**: private posts, stories
- **Groups**: private forums, communities, group chats

The permissioned protocol shares the abstract shape of public atproto. It retains identity-based authority, per-user repositories, lexicon-typed records, and the general flow of applications crawling PDSes to build views. However it is a **distinct protocol** rather than an extension of the public one. It has its own repository format, sync mechanism, addressing scheme, and resolution path. Public atproto is built for public broadcast (signed, archival, rebroadcastable) while the permissioned protocol is built for party-to-party transmission within an access boundary.
 
This protocol provides **access control, not confidentiality**. It is not end-to-end encrypted. Services (both PDSes and authorized applications) can read the data they handle, which is required for server-side features such as search, indexing, notifications, aggregation, and moderation. E2EE is a separate concern that may be layered on top by an application and is out of scope in this proposal.

### Relationship to public atproto

| | Public atproto | Permissioned protocol |
|---|---|---|
| Unit of data | Record in a repo | Record in a permissioned repo |
| Repo scope | One repo per user | One permissioned repo per (user, space) |
| Record authority | User DID | User DID |
| URI authority | User DID | Space authority DID |
| Commit | Merkle Search Tree root | LtHash set-hash digest |
| Signature | Rebroadcastable, archival | Deniable on rebroadcast |
| Addressing | `at://` URI | `ats://` URI |
| Access | Public | Gated by space credential |

### Terminology

- **Space**: an authorization and sync boundary for a set of permissioned records, identified by an `(authority, type, skey)` triple.
- **Permissioned repo**: one user's records within one space, with a cryptographic commit, hosted on that user's PDS.
- **Repo host**: a service that stores and serves users' permissioned repos.
- **Space host**: a service that answers for a space as a whole, issuing credentials, enumerating writers, and routing notifications.
- **Space authority**: the DID at the root of a space, which resolves to the space host and the key material for issuing credentials.
- **Space credential**: a token issued by the space authority that grants read access to a space.
- **Delegation token**: a token issued by a user's PDS that an application exchanges with a space authority for a space credential.
- **Client attestation**: a token signed by an application's own client authentication key, proving the application's identity to a space authority. Required only when a space gates on app identity.
- **Syncer**: an application that keeps its own copy of a space in sync by pulling from repo hosts.

A PDS fulfills both the roles of a **repo host** and a **space host**. However, these roles are discussed separately because they do not necessarily need to be filled by a PDS. A permissioned repo or a space may be hosted by any service that implements the [required APIs](#xrpc-api).

## Spaces

A **space** is an authorization and sync boundary representing a shared social context. A space may include many different types of records from many users. The space does not colocate records. Instead, each user stores their own records for a given space in a [permissioned repo](#permissioned-repos) on their own repo host. A space is the union of these per-user repos across the network: an application presenting a space pulls each member's repo from its host, assembles the view, and applies access control to requesting users.

Each space is identified by three values:

1. **authority**: a DID, the root of authority for the space
2. **type**: an NSID describing the modality of the space
3. **space key** (`skey`): a string distinguishing spaces of the same type under the same authority

Reading or syncing a space requires a **space credential** signed by the declared signing key of the space authority. The space authority decides whether to issue one based on the requesting user and client application. The protocol does not define how that decision is made and carries no member list (see [Access Control](#access-control)). Spaces scale from a single user's personal data (e.g. bookmarks) to communities of millions of users.

### Addressing

A permissioned record is addressed by an `ats://` URI of six components:

```
ats://{spaceDid}/{spaceType}/{skey}/{authorDid}/{collection}/{rkey}
```

| Component | Type | Description |
|---|---|---|
| `spaceDid` | DID | Space authority DID |
| `spaceType` | NSID | Space type |
| `skey` | string | Space key |
| `authorDid` | DID | DID of the record's author |
| `collection` | NSID | Record collection |
| `rkey` | string | Record key |

All six URI segments are necessary to identify a **permissioned record**. The first three components may be used to reference a **space**:

```
Space:  ats://{spaceDid}/{spaceType}/{skey}
Record: ats://{spaceDid}/{spaceType}/{skey}/{authorDid}/{collection}/{rkey}
```

### Space authority

A space's **authority** is the DID at the root of the space and the issuer of its [credentials](#access-control). It may be a user's own DID as for personal data such as bookmarks or mutes. Or it may be a dedicated DID which lets a shared space transfer between users independently of any individual account.

The authority MUST expose two entries in its DID document:

- a **verification method** with id `#atproto_space`: the public key used to verify the space's credentials
- a **service** entry with id `#atproto_space_host`: the endpoint of the space host

Both values MAY resolve to the same values as `#atproto` and `#atproto_pds` from the public data protocol.

### Space type

A space's **type** is an NSID that names its modality and resolves to a [space declaration](#space-type-declarations). It identifies the kind of data a space holds before any network resolution, much as a collection NSID does in public atproto. Because a type names a concrete modality, every space is some specific kind of space rather than a generic container.

The type is also the **OAuth consent boundary**. Access is granted to a user by type, e.g. "access to your AtmoBoards forums" (see [OAuth scopes](#oauth-scopes)).

### Space type declarations

A space type NSID resolves to a **space type declaration**: a Lexicon definition with `"type": "space"`.

```json
{
  "lexicon": 1,
  "id": "com.atmoboards.forum",
  "defs": {
    "main": {
      "type": "space",
      "description": "A discussion forum",
      "key": "any",
      "name": "AtmoBoards Forum",
      "name:lang": { "es": "Foro AtmoBoards", "ja": "AtmoBoards 掲示板" },
      "collections": ["com.atmoboards.thread", "com.atmoboards.reply"]
    }
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `"space"` | yes | Marks this as a space declaration. Must be the `main` definition. |
| `description` | string | no | Description of the space type for developers. Not shown to users. |
| `key` | string | yes | Specifies the recommended space key type (similar to [record key types](https://atproto.com/specs/record-key#record-key-type-tid)) |
| `name` | string (1–64) | yes | Human-readable name for the space type, shown to users on consent screens. |
| `name:lang` | map<lang, string> | no | Localized `name` values by language code. |
| `collections` | array of NSID | yes | Collections clients should expect in a space of this type. |

The `collections` field is the default `collection` set for a [`space:` scope](#oauth-scopes) of this type. However, ultimately any collection may be written to any space and is not constrained at the protocol level by the collections in a space type declaration.

### Space key (skey)

The **space key** (`skey`) is a string distinguishing spaces of the same type under the same authority, analogous to a record key (`rkey`): a slug, a TID, a cryptographic identifier, or a reserved string such as `self`. Maximum length 512 bytes with the same syntax requirements as an `rkey`.

## Access control

Reading a space is gated by a **space credential** issued by the space authority. The authority issues one based on two axes:

- **which user** is being acted for: established by a **delegation token** minted by the user's PDS
- **which application** is acting: established by a **client attestation** signed by the application itself

The delegation token is always required. The client attestation is required only when a space gates on app identity. An application obtains a credential by getting a delegation token from a user's PDS, then presenting that token (together with its client attestation, if needed) to the space authority in exchange for a credential. The authority decides whether to issue the credential. The protocol does not define the decision procedure (the policies of the PDS's space-management implementation are described under [`simplespace`](#required-pds-space-management-simplespace)).

Some spaces do not require a client attestation. This can be detected by querying the space configuration, or by simply making a request for a credential without an attestation and seeing whether an error is returned.

### Delegation token

A **delegation token** proves that an application is acting on a user's behalf when it asks an authority for a space credential. The user's PDS mints it, requested through [`com.atproto.space.getDelegationToken`](#xrpc-api). It asserts only the delegation from user to application. Whether the user is a member of the space is the authority's determination, and the delegation token says nothing about it. The token is single-use, short-lived (default 60 seconds), and addressed (`aud`) to the space authority.

A delegation token is structurally similar to an atproto [service auth token](https://atproto.com/specs/xrpc), but it differs in a few ways that make it its own credential class rather than an interchangeable one: 
- The `typ` field in the header is set to `atproto-space-delegation+jwt`.
- It does not include an `lxm` claim.
- It is bound to a target space through the `sub` claim.

Example JWT header and payload (before base64url encoding and signing):

```json
{
  "typ": "atproto-space-delegation+jwt",
  "alg": "ES256K", // or ES256
  "kid": "#atproto" // Key Identifier - MUST be "#atproto"
}
{
  "iss": "did:example:user_did", // User DID
  "sub": "ats://did:example:space_did/com.example.space_type/space_key", // Space being requested
  "aud": "did:example:space_did#atproto_space_host", // Space host (service fragment of the authority DID)
  "iat": 1738368000, // Issued-at (unix seconds)
  "exp": 1738368060, // iat + 60 (60 seconds)
  "jti": "f47ac10b58cc4372a5670e02b2c3d479" // random nonce
}
```

The delegation token asserts only the user-to-app delegation; it says nothing about which application is acting. App identity is established independently by the [client attestation](#client-attestation), which the authority verifies itself. The two are presented together but signed by different parties and evaluated independently.

The signature for the delegation token is computed using the regular JWT process, using the account's signing key. For more details, see the [Inter-Service Authentication](https://atproto.com/specs/xrpc#inter-service-authentication-jwt) section of the AT Protocol spec.

### Client attestation

A **client attestation** is a short-lived JWT that the application presents to the space authority alongside the delegation token to identify itself, when the space requires it. It is structurally a `private_key_jwt` [client assertion](https://atproto.com/specs/oauth), the same shape an atproto confidential client already presents to its authorization server, but addressed to the space authority rather than to the PDS.

```json
{
  "typ": "atproto-client-attestation+jwt",
  "alg": "ES256",
  "kid": "key-1" // Key id from the client's published JWKS
}
{
  "iss": "https://app.example.com/client-metadata.json", // The client_id
  "sub": "https://app.example.com/client-metadata.json", // The client_id (== iss)
  "aud": "did:example:space_did#atproto_space_host", // Space host being asked for a credential
  "iat": 1738368000, // Issued-at (unix seconds)
  "exp": 1738368060, // short-lived
  "jti": "b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8" // random nonce, replay protection
}
```

The authority verifies the attestation by resolving `iss` (the `client_id`) to the client's `client-metadata.json`, fetching its published JWKS (`jwks` or `jwks_uri`), and verifying the signature against the key in that JWKS identified by the attestation's `kid`. 

### Space credential

A **space credential** is the token an application presents to a repo host to read a permissioned repo within a space. The authority mints it in exchange for a delegation token, requested through [`com.atproto.space.getSpaceCredential`](#xrpc-api). It is short-lived (default 2 hours) and signed by the space authority's signing key, so any repo host can verify it against the authority's key without contacting the authority.

A space credential resembles a [space delegation token](#delegation-token), differing in these ways:

- The `typ` field in the header is set to `atproto-space-credential+jwt`.
- It is signed by the space authority rather than the user.
- It has no `aud`: it is presented to any repo host serving a repo in the space, not to a single recipient.

Example JWT header and payload (before base64url encoding and signing):

```json
{
  "typ": "atproto-space-credential+jwt",
  "alg": "ES256K", // or ES256
  "kid": "#atproto_space" // Key Identifier - MUST be "#atproto_space"
}
{
  "iss": "did:example:space_did", // Space authority DID
  "sub": "ats://did:example:space_did/com.example.space_type/space_key", // Space the credential reads
  "iat": 1738368000, // Issued-at (unix seconds)
  "exp": 1738375200, // iat + 7200 (2 hours)
  "jti": "9f8e7d6c5b4a3210fedcba9876543210" // random nonce
}
```


The signature for the space credential is computed using the regular JWT process, using the space authority's signing key.

### Credential flow

```
┌──────┐          ┌────────────┐          ┌─────────────┐          ┌─────────────────┐
│ User │          │ User's PDS │          │ Application │          │ Space Authority │
└───┬──┘          └──────┬─────┘          └──────┬──────┘          └────────┬────────┘
    │                    │                       │                          │
    ├── OAuth consent ───►                       │                          │
    │                    ├───── OAuth token ─────►                          │
    │                    │                       │                          │
    │                    ◄─ getDelegationToken ──┤                          │
    │                    ├── delegation token ───►                          │
    │                    │                       │   getSpaceCredential     │
    │                    │                       ├─(token [+ attestation])─►│
    │                    │                       ◄──── space credential ────┤
```

1. The user authorizes the application via OAuth.
2. The application calls `com.atproto.space.getDelegationToken` on the user's PDS, receiving a delegation token.
3. The application presents the delegation token to the space authority via `com.atproto.space.getSpaceCredential`, adding its own client attestation if the space gates on app identity. The authority verifies what it received and, on authorization, returns a space credential.
4. The application reads the repo from each member's repo host with the credential.

An application serving several users of a space does not necessarily need to maintain a space credential for each user. It may obtain its credential using any one user's session. When it loses all OAuth sessions for a space, it can no longer renew the credential and loses access.

## Permissioned repos

A **permissioned repo** is one accounts's set of records within one space, generally stored on that user's own PDS. A user has one permissioned repo per space it participates in. Abstractly, a permissioned repo offers a similar interface to a public [atproto repository](https://atproto.com/specs/repository): a key/value mapping where keys are path names (a collection NSID and an rkey) and values are CBOR-encoded records. PDSes expose a CRUD interface for interacting with a user's permissioned repos.

Each permissioned repo is summarized by a **commit**. A commit is a short, signed digest that allows a syncer to check whether its copy of a repo matches the source without resyncing it in its entirety.

### Commit digest

The digest in a commit is computed from a **set hash** over the records a repo currently contains. It is independent of write/delete order, so two repos with the same records always produce the same digest. Adding or removing a record is a single cheap operation, so a repo host maintains the set hash incrementally on each write.

The construction is [LtHash][LTHASH], a homomorphic hash built on a lattice problem (and thus quantum-secure). A record is added to or removed from the digest with a single addition or subtraction rather than a recomputation over the whole repo.

The state is a fixed 2048-byte buffer, interpreted as 1024 little-endian unsigned 16-bit lanes.

Each record in the repo maps to one element, the UTF-8 bytes of `{collection}/{rkey}/{record_cid}`. As with the public repository, the components are currently limited to ASCII, so the encoding (though specified as UTF-8) is a no-op.

To **add** an element:

1. Expand the element to 2048 bytes with BLAKE3 in XOF (extendable-output) mode.
2. Read those bytes as 1024 little-endian `uint16` lanes.
3. Add each lane into the corresponding state lane, modulo 65536 (2^16, i.e. with wraparound)

To **remove** an element, perform the same process, but subtract its lanes instead (modulo 65536). 

Both operations are commutative, so the state depends only on the current set of records, not the order of writes. The empty repo's state is all zeroes.

The commit's `hash` is `sha256(state)`, a 32-byte digest of the 2048-byte buffer. The repo host maintains the full state. Only the `hash` is carried in a commit.

### Commit signature

A user does not sign the digest directly since a signature over the content digest would be a rebroadcastable proof of what the user wrote in a private space. Instead the signature covers only random per-commit bytes, and the digest is bound to those bytes by a symmetric MAC. A reader in the sync flow gets full authenticity and integrity, but a leaked commit is deniable and proves nothing about its contents to a third party.

Both the signature and the MAC are domain-separated by a single context string, `ctx`, built once and reused for both. `ctx` uses the variable-length-vector encoding from [TLS 1.3][RFC8446] (§3.4). It is composed of a fixed protocol tag followed by each variable field length-prefixed with a big-endian `uint16`.

Note: these length prefixes are big-endian, following the TLS convention for wire encodings. This is the opposite byte order from the little-endian lanes of the [commit digest](#commit-digest), which follows the LtHash reference construction. The two come from different specs and each keeps its native byte order.

```
ctx = "atproto-space-v1"            // fixed protocol tag
   || uint16be(len(space)) || space  // space URI (ats://authority/type/skey)
   || uint16be(len(rev))   || rev    // commit revision (TID)
   || uint16be(len(ikm))   || ikm    // per-commit nonce (below)
```

A commit is then produced as follows:

1. Generate `ikm`, 32 fresh random bytes. A new `ikm` is generated for each reader the commit is served to.
2. Compute `sig = sign(ctx)` with the user's signing key. The signed message only contains the space, revision and ikm, not the current repository hash. 
3. Compute `mac = HMAC-SHA256(HKDF-SHA256(ikm, ctx), hash)`, binding the repository hash to this commit's context.

A reader verifies `sig` against the user's signing key (authenticity), then recomputes `mac` and compares (integrity). Because the digest is bound by a *symmetric* MAC keyed from the public `ikm`, anyone holding the commit can compute a valid `mac` for any `hash`, so a rebroadcast commit cannot prove what the user wrote, only that they signed a `(space, rev, ikm)` context. 

The signed commit (`com.atproto.space.defs#signedCommit`):

| Field | Type | Description |
|---|---|---|
| `hash` | bytes | `sha256` of the LtHash state (32 bytes) |
| `ikm` | bytes | per-commit nonce (32 random bytes) |
| `sig` | bytes | `sign(ctx)` by the user's signing key |
| `mac` | bytes | `HMAC-SHA256(HKDF-SHA256(ikm, ctx), hash)` |
| `rev` | string | commit revision (TID), also bound into `ctx` |

## Sync

Permissioned data sync is functionally similar to public atproto. Applications build views by pulling repos from their hosts. The major difference is that there is no relay to provide a collated firehose of data for the network as permissioned repositories are by their nature non-rebroadcastable. An application pulls directly from each repo host and is responsible for keeping its own copy in sync.

### Incremental sync

A syncer keeps its own copy of a repo and, alongside it, its own running [set hash](#commit-digest) over that copy. This is the same digest that the repo host maintains. When the two digests agree, the syncer knows that their copy is exactly up-to-date with the hosted repo. If there is a disagreement, then the syncer knows their copy is behind or corrupted.

To advance, a syncer calls `com.atproto.space.listRepoOps` with a `since` revision. A repo host keeps an **operation log** of recent writes to each repo and returns the operations after `since`. Each entry is `{ rev (TID), collection, rkey, cid, prev }`. The field `cid` is null for a delete and `prev` is null for a create. Multiple records may be mutated atomically, and this is captured by entries sharing a `rev`. The syncer applies each operation to its copy and updates its running set hash accordingly.

If a given oplog response includes the last available operation that occurred to a repo, then the response must also include the member's current signed [commit](#commit-signature). The syncer compares the commit's `hash` against its own running set hash. If they match, the syncer is fully in sync and the commit's signature authenticates that state. If they do not, the syncer has diverged and must fall back to a recovery process.

Sync is self-healing because its correctness rests on the set hash comparison rather than on receiving every operation. A missed operation is detected on a subsequent sync. A total disjunction in repository history is also recognizable, and is recovered from by syncing the repository contents in their entirety.

The oplog is a transport optimization rather than a committed data structure. Its contents and history are not guaranteed, and a repo host may compact or drop it, retaining only a backfill window.

### Full-state recovery

In cases in which a syncer cannot proceed incrementally, it must recover by syncing the full state of the repository.

To do so, a syncer enumerates the repo's structure (paths → CIDs) with `com.atproto.space.listRecords`, diffs that against its local copy, and fetches the record values it is missing with `com.atproto.space.getRecord`. It then rebuilds its set hash from the recovered structure and compares it against the signed commit. 

### Blob sync

Blobs referenced by permissioned records are stored on the authoring repo's host and fetched via `com.atproto.space.getBlob` with the space credential. 

### Write notifications

Write notifications inform syncers that a repo has advanced, so they can pull promptly instead of continuously polling for updates. A notification contains no record data, and states only that a given repo in a given space has reached a new revision.

A syncer subscribes to notifications by calling `com.atproto.space.registerNotify`. When called on a space host, this method subscribes to writes for all repos in a space. Generally, syncers should subscribe to the space host for all write notifications from the space. However, it can also be called on particular repo hosts to receive notifications for specific repos. `registerNotify` is authenticated with a space credential. The service that was registered against should return the expiration time for the registration which may be longer than the expiration window of the space credential. 

When a member writes, their PDS sends a `com.atproto.space.notifyWrite` to each endpoint registered for that repo. A PDS may not otherwise know which services are syncing the space, which is why the **space authority** registers itself as a subscriber on each repo host. Members notify the authority, and the authority forwards each notification to the endpoints registered with it for the space. Each notified syncer then pulls the updated repo directly from the relevant repo host. The authority only routes notifications and does not carry record data.

Notifications are **best-effort** and are not required for correctness. If a notification is dropped, the affected repo is caught up by a later write's notification, or by a periodic sweep by the syncer.

### The sync boundary (writer set)

Syncing a single repo requires the syncer to know that the repo exists. To sync a space in full, or to begin syncing a space for the first time, an application requires the set of accounts whose repos hold data in the space. This **writer set** is retrieved from the space authority via `com.atproto.space.listRepos`.

Because it subscribes to updates from every repository in the space, the authority can easily maintain a complete and current record of which repos hold data in the space as well as their current rev.

The writer set is a simple fetch and carries no commit or history. It enumerates only accounts that have **written** to the space. Accounts that may read the space are never enumerated.

## Space deletion

A space may be deleted by its authority. The authority stops issuing credentials and no longer answers for the space. The authority also deletes its own repo in the space. 

The authority then notifies registered syncers and known repo hosts of participating members with `com.atproto.space.notifySpaceDeleted`, over the same best-effort path as write notifications. 

A **syncer** should delete every copy of the space's data it holds, both the repos it pulled and any derived state, as it is no longer authorized to retain them. A **repo host** should instead flag the member's repo as belonging to a deleted space rather than erase it. The data is the user's own, so the repo host decides how and whether to surface it to the user (e.g. an export, a grace period) before garbage-collecting.

## OAuth scopes

A user grants an application access to spaces through a `space:` OAuth scope. The scope identifies a set of spaces by their `(did, spaceType, skey)` identifier and states what the grant permits within them.

```
space:<spaceType>[?did=<did>][&skey=<skey>][&collection=<nsid>...][&action=<action>...][&manage=<op>...]
```

| Parameter | Position | Multiple | Default | Values |
|---|---|---|---|---|
| `spaceType` | positional (required) | no | N/A | a space-type NSID, or `*` for any type |
| `did` | query | no | `*` | a space authority DID, or `*` for any authority |
| `skey` | query | no | `*` | a space key (1–512 chars), or `*` for any key |
| `collection` | query | yes | the space type lexicons declared `collections` | a record collection NSID, or `*` for any |
| `action` | query | yes | `read`, `create`, `update`, `delete` | `read_self`, `read`, `create`, `update`, `delete` |
| `manage` | query | yes | _(none)_ | `create`, `update`, `delete` |

`did`, `spaceType`, and `skey` select **which spaces** the grant covers, matching the first three segments of a space URI. `action` (and `collection`) govern operations on the **records** in those spaces. `manage` governs operations on the **spaces themselves**.

### Read access

`read` is all-or-nothing at the space boundary. There is no partial, per-record, or per-author whole-space read grant.

A `read` grant confers two things:

- access to the read and sync methods (`com.atproto.space.getRecord`, `listRecords`, `getBlob`, `getRepoState`, `listRepoOps`) on the holder's own PDS, sufficient to read the holder's own repo in the space
- access to `com.atproto.space.getDelegationToken` for that space, which an application exchanges for a space credential to read  any repo in the space

`read_self` is the narrower grant. It confers access to the same read and sync methods, but only for the holder's **own** repo in the space, and it does **not** grant `getDelegationToken`. An application holding only `read_self` can read its user's own records but cannot reach the rest of the space. `read` implies `read_self`.

A **space credential** grants whole-space read/sync access directly, so the read and sync methods accept **either** a covering OAuth scope or a space credential. Write methods accept only an OAuth credential, since a write is attributed to the authoring user.

### Matching

A request is authorized by a grant when its target space matches the grant's `(did, spaceType, skey)` (each component equal to the grant's value or covered by its `*`) and the grant permits the requested operation, per the rules below.

**Record operations** are governed by `action`:

- `read` covers every repo in the space and ignores `collection` since read access is all-or-nothing.
- `read_self` covers only the holder's own repo and is constrained by `collection`. A `read` grant also satisfies a `read_self` request.
- `create`, `update`, and `delete` act on a specific record, so they are additionally constrained by `collection`

Omitting `action` grants `read`, `create`, `update`, and `delete` (`read` is inclusive of `read_self`).

Omitting `collection` defaults it to the `collections` declared by the space type's [declaration](#space-type-declarations). A bare `space:com.atmoboards.forum` grant therefore permits writing the forum's own record types, the same way a bare `repo:` scope permits writing the collections it names. An application may narrow this by listing a subset of collections, or widen it with `collection=*`. When `spaceType` is `*` there is no declaration to draw from, so the default is empty and the grant confers no write targets unless provided.

**Space management operations** are governed by `manage`, which takes the same `create`/`update`/`delete` verbs applied to the space itself rather than to its records. `manage` ignores `collection`. It is omitted by default, so an ordinary record-access grant confers no administrative capability.

The protocol does not enumerate what each `manage` verb permits, because space management is implementation-defined (see [`simplespace`](#required-pds-space-management-simplespace)). Each space-management implementation maps the verbs onto its own administrative surface. For example, in `com.atproto.simplespace`, `manage=update` authorizes `com.atproto.simplespace.updateSpace` as well as `addMember` and `removeMember`.

`manage=create` authorizes creating a space of the given `spaceType` under the given authority. Unlike every other operation, it concerns a space that does not yet exist, so scoping it to a concrete `skey` is unusual. It is typically granted with `skey=*` ("this app may create spaces of this type").

### Examples

- `space:com.atmoboards.forum`: every `com.atmoboards.forum` space the user is in, with read access for the entire space and write access to the collections the forum's declaration lists (`com.atmoboards.thread`, `com.atmoboards.reply`). This is the typical grant for a forum client.
- `space:com.atmoboards.forum?action=read`: the same spaces, read-only. No `collection` is needed because `read` is not constrained by collection.
- `space:com.atmoboards.forum?action=read_self&collection=*`: read-only, and only the user's own repo in those forums. Suitable for a personal export or backup tool that should not see other members' posts. Note that `collection=*` is required to read all records (not just those declared in the Lexicon).
- `space:com.atmoboards.forum?collection=*`: read access plus write access to every collection, not only the declared ones.
- `space:com.atmoboards.forum?skey=default&collection=com.atmoboards.thread&action=create&action=update`: create and update `com.atmoboards.thread` records in the forum keyed `default`, under any authority.
- `space:com.atmoboards.forum?action=read_self&manage=update&manage=delete`: administer the user's forums (update and delete the spaces), with read access, but no record-write access. 
- `space:com.atmoboards.forum?manage=update&manage=delete`: administer the user's forums (update and delete the spaces), with full read/write access to records in the space. 
- `space:*?did=did:plc:abc123`: read every space under authority `did:plc:abc123`, any type.

### Consent

A `space:` scope is presented to the user on the OAuth consent screen and requires user-legible text. Each space `type` resolves to a [space declaration](#space-type-declarations). The consent screen then displays the declaration's `name` (e.g. "AtmoBoards Forum") in place of the raw NSID. 

If a particular `did` is specified in a scope, that DID should be presented to the user as the bidirectionally linked handle associated with the DID. If no handle bidirectionally validates, then the DID itself should be shown.

A scope may request wild card access on both `did` and `type`. This is a very broad grant, and as such the consent screen should present such a scope with a promnient warning. 

### Permission sets

Space permissions can also be bundled, usually with more user-friendly verbiage, into a [permission set](https://atproto.com/specs/permission).

```json
{
  "type": "permission-set",
  "title": "AtmoBoards",
  "detail": "Read and post in your AtmoBoards forums",
  "permissions": [
    { 
      "type": "permission",
      "resource": "space",
      "spaceType": "com.atmoboards.forum",
      "collection": ["com.atmoboards.thread", "com.atmoboards.reply"],
      "action": ["read", "create"]
    },
  ]
}
```

Within a permission, `"type": "permission"` is the entry discriminator, `"resource": "space"` selects the space resource, and the remaining fields carry the same parameters as the `space:` scope string: `spaceType`, `did`, `skey`, `collection`, and `action`.

A set permission must name a concrete space type. In other words, the `spaceType` parameter may **not** be a wildcard inside a permission set. The other parameters may still be wildcards, including `did` and `collection`. A cross-type grant (`spaceType=*`) is expressible only as a standalone `space:` scope requested directly.

When expressing space resources in a permission set, the `spaceType` must follow the [Namespace Authority](https://atproto.com/specs/permission#namespace-authority) requirements associated with permission sets. However, the collection parameter may be a wildcard or list collections under a different namespace authority than the space and the permission set.

## XRPC API

All protocol XRPC methods for permissioned data are currently defined under the `com.atproto.space` namespace.

These methods fall into a few loose groups:
- **Repo** methods concern an account's [permissioned repo](#permissioned-repos) within a space, and are implemented by a repo host.
- **Host** methods concern a space as a whole, and are implemented by a space host.
- **PDS** methods are required set of baseline PDS methods that applications can build against.
- **Syncer** methods concern a syncer getting real-time notifications of updates to a space or repos in a space.

This grouping describes kinds of methods, not separate services. A single service (e.g. a PDS) is usually both a repo host for its accounts and a space host for the spaces anchored on it.

| Method | Role | Type | Auth | Description |
|---|---|---|---|---|
| `getSpace` | host | query | space credential | Describe a space, including an open-union `config` carrying host-specific configuration. |
| `getSpaceCredential` | host | procedure | delegation token (+ client attestation) | Exchange a delegation token for a space credential. A client attestation is also required when the space gates on app identity. |
| `listRepos` | host | query | space credential | List the known repos that hold data in a space. |
| `getRecord` | repo | query | OAuth / space credential | Fetch a single record's value. |
| `listRecords` | repo | query | OAuth / space credential | List the record keys (`collection`, `rkey`, `cid`) in a repo. |
| `getBlob` | repo | query | OAuth / space credential | Fetch a blob by CID. |
| `getRepoState` | repo | query | OAuth / space credential | The current signed [commit](#commit-signature) for a repo. |
| `listRepoOps` | repo | query | OAuth / space credential | Primary sync mechanism. A repo's [operation log](#incremental-sync) since a given revision. |
| `getDelegationToken` | pds | query | OAuth | Mint a [delegation token](#delegation-token) for a space. Served by the requesting user's PDS. |
| `createRecord` | pds | procedure | OAuth | Create a record in the caller's permissioned repo for a space. |
| `putRecord` | pds | procedure | OAuth | Create or update a record. |
| `deleteRecord` | pds | procedure | OAuth | Delete a record. |
| `applyWrites` | pds | procedure | OAuth | Apply a batch of creates, updates, and deletes to one repo atomically. |
| `listSpaces` | pds | query | OAuth | The spaces the caller holds a repo in. |
| `registerNotify` | repo/host | procedure | space credential | Register an endpoint to be notified of writes. On the space host, subscribes to the whole space. On a repo host with a `repo`, subscribes to that repo. |
| `notifyWrite` | syncer/host | procedure | service auth | Notify that a repo advanced. Sent by a repo host to the space host, and forwarded by the space host to registered syncers. |
| `notifySpaceDeleted` | syncer/repo | procedure | service auth | Notify that a space was deleted. Sent by the authority to members and registered syncers. |

## Required PDS space management: `simplespace`

The protocol does not specify how spaces are created or how an authority decides who may read them. Those are the concern of each space-management implementation, which sits above the protocol and is identified by its own lexicon namespace.

`com.atproto.simplespace` is the space-management implementation that every PDS MUST support. It gives applications a baseline that is available on every account's PDS to build against. `simplespace` spaces are anchored on a user's own DID and governed by an explicit member list (or the `public` and `managing-app` describe policies below).

`simplespace` is neither the only permitted implementation nor a privileged one. It is simply the one that PDSs are required to support. Other space types may define their own management implementations and are full protocol participants, but they are hosted on bespoke space services rather than on the PDS.

All simplespace methods are called with an OAuth credential with the relevant `manage` scope.

| Method | Type | Description |
|---|---|---|
| `createSpace` | procedure | Create a space (caller becomes the authority) |
| `updateSpace` | procedure | Update config (more details below) |
| `deleteSpace` | procedure | Delete the space (see [Space deletion](#space-deletion)). |
| `addMember` | procedure | Add a member (by DID) to view the space. |
| `removeMember` | procedure | Remove a member (by DID). |
| `listMembers` | query | List the current members of a space. |

### Configuration

`simplespace` spaces can be further configured along a few dimensions. This configuration is updated through `updateSpace` and is surfaced through `com.atproto.space.getSpace`.

| Field | Values | Description |
|---|---|---|
| `mintPolicy` | `public` \| `member-list` \| `managing-app` | How the authority decides whether to authorize a _user_. |
| `appAccess` | open union (`#open` \| `#allowList`) | How the authority decides whether to authorize an _app_. |
| `managingApp` | service identifier (DID + fragment) | Used to route application requests, and as the access check traget in `managing-app` mode. |

A user must be authorized by `mintPolicy` **and** their app by `appAccess` for a credential to be minted. A syncing app needs a valid delegation token regardless.

**Mint policy** decides per-user authorization:
- `member-list` (default): authorize requesters present on the member list.
- `public`: authorize any requester.
- `managing-app`: at mint time, ask `managingApp` whether to authorize the request, via [`checkUserAccess`](#the-managing-app) below. Enables dynamic policies (e.g. follower-gating) without an app maintaining a list.

**App access** decides per-app authorization. It is an open union with two current variants:
- `#open` (default): any application may access the space. No [client attestation](#client-attestation) is required, so public clients work.
- `#allowList`: only the apps named in `allowed` may access the space. The authority evaluates the list against the **attested** `client_id` (the `iss` of the verified client attestation), so it is enforceable rather than advisory.

### The managing app

Spaces may list a `managingApp`. Any account with access to the space can read the selected managing app in the space config through `com.atproto.space.getSpace`. This endpoint may be used to route certain application-level requests that cannot be handled by a generic PDS implementation, such as join requests. Calls to managing apps are generally authenticated using service auth tokens.

When a space's `mintPolicy` is `managing-app`, the space authority defers to the space's `managingApp` at mint time by calling `com.atproto.simplespace.checkUserAccess`. 

Unlike the other `simplespace` methods, `checkUserAccess` is served by the `managingApp`, not the PDS. The authority calls it with itself as `iss` and the `managingApp`'s service identifier as `aud`, so the app can verify the call genuinely originates from the space's authority. It passes the space, the requesting user, and the requesting client (the **attested** `client_id`, if any), and the managing app returns whether to authorize.

The app evaluates the request against whatever application-layer state it maintains (e.g. follower graphs, paid-subscription status, join approvals) and returns its decision. The authority mints the credential only if the app authorizes and applies `appAccess` as usual.

<!-- references -->

[ATPROTO]: https://atproto.com
[PERMISSION-SET]: https://atproto.com/specs/permission#permission-sets
[LTHASH]: https://eprint.iacr.org/2019/227
[RFC8446]: https://www.rfc-editor.org/rfc/rfc8446
[OAUTH-ATTEST]: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth/
