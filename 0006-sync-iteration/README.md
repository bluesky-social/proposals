0006: AT Protocol Sync v1.1
============================

*This is a draft proposal, not the final specification. Some of the details, terminology, and behaviors might change, though we are pretty confident in the big picture.*

Authenticated Transfer of public user content is one of the core functionalities supported by atproto. This document summarizes an iteration on this part of the protocol, also called "Sync", which encompasses the repository, firehose, and CAR export mechanisms. The changes should improve and clarify behaviors, while being largely non-disruptive and backwards compatible.

In particular, operating a full-network relay becomes much cheaper!

The direct changes to core sync mechanisms are:

- include additional metadata and MST blocks in firehose `#commit` messages to enable per-commit MST validation ("operation inversion" or "inductive firehose")
- hard size limits on repository diffs (`#commit` messages), and removing the `tooBig` flag
- new `#sync` firehose event type to declare the current repository state, even if discontinuous from the last commit
- new `desynchronized` and `throttled` account hosting statuses, used by consuming services to indicate that an account has lost synchronization, or has exceeded rate-limits (respectively)
- clarification of behaviors at the relay and consuming services when they encounter invalid messages or discontinuities in a repo's sequence of commits

Supporting changes include:

- new `com.atproto.sync.listReposByCollection` API endpoint (on a helper service) to enumerate all accounts which contain a given record type
- variant of `com.atproto.sync.getRepo` API endpoint to return a verifiable subset of full repository (eg, a specific collection), including relevant records and MST nodes, as a CAR file
- define an ordering of MST and record blocks in CAR files, which will enable efficient stream processing of repositories. will not be required for commit "diff" CAR slices
- deprecation and removal of some unused API fields and endpoints

The expected benefits will be:

- services (including relay) can validate full repo MST integrity without storing the full repo (or any of the repo) locally. This reduces disk I/O requirements dramatically
- simplifies the role of relays in the network, moving "mirror" or "archive" roles to a separate service type
- clarifies what a fully-validating downstream service should do to ensure it stays in sync with all relevant record operations for an account, including backfill and error recovery
- more efficient backfill for subsets of the network (eg, only specific record types)


## Rollout Timeline

Some of these changes can be rolled out slowly over time. The changes needed to ensure interoperation in the near term (early 2025) are:

- repo hosts (PDS implementations) must include the additional `#commit` metadata to enable validation; and emit `#sync` events when needed
- repo hosts must avoid generating `tooBig` commit diffs
- fully validating downstream consumers (eg, AppViews, Relays, and other services) should implement the new commit validation process, and add basic support for `#sync` events

Casual, non-validating firehose consumers, which do not attempt `tooBig` handling, should not break. Note that these services could instead be using an un-authenticated mechanism like jetstream.

Our intention is to provide developer resources to help with the transition, such as test vectors, debugging tools, and implementation notes. We will be in communication with service operators as changes are rolled out in the network, can use "ratchet" mechanisms to accommodate legacy services which need more time to upgrade, and expect breakage to be minimal or zero.


## Motivation

This proposal addresses a few issues with the current sync mechanisms:

- validation of repo MST structures requires a local copy of the complete repository tree. As the network grows, this requires more and more fast local disk storage, which is expensive. This was most obvious at Relays, but also impacted AppViews trying to verify synchronization
- most firehose consumer implementations (SDKs and services) do not automatically handle the `tooBig` event flag, or recover from related error conditions
- backfill of the full network is difficult for small applications with new Lexicon schemas

Relays aggregate repository updates from all PDS instances in the network, and combine them into a unified firehose. Downstream consumers can then conveniently connect to just a single stream, instead of needing to manage thousands of separate connections to all in the individual PDS instances. Events are individually *authenticated* in that they are cryptographically signed by the account's public key. Note that the signature only covers a subset of data in the event (records, MST nodes, and the minimal metadata in the commit block itself), not the entire `#commit` message on the wire.

In one sense, the authenticated events can be processed individually: they represent a stream of record creations, updates, and deletes. In this sense, the stream is an "event log" of changes to the user's repository. Many consumers simply process the events as they come. But there is another capability of atproto synchronization, which is to fully replicate the user's repository. A key property of atproto sync is that the full repository tree structure can be reconstructed by tracking events, resulting in an exact replica.

As an example of this distinction, a PDS might emit a stream of commit events for an account which each contain a single record, as if the repo tree contained a single record at any point in time, without tracking deletions or updates. Each commit event could have a valid signature, but they would not combine into a repository containing multiple records. Non-validating services would never see "deletions", so they would index more and more content, but the indexed application state would not correspond to the current state of the repository.

To validate that individual commits are consistent with the previous state of the repository, the Relay has needed to "apply" the diff blocks (MST nodes) to a local copy of the tree. This meant that processing events required reading a number of blocks from storage. As the network has grown, both the total storage required, and the rate of requests (disk I/O) has grown. While growth can be accommodated by horizontal scaling (sharding by DID), it makes Relay operation less accessible.

What this proposal describes is a trick to verify that each individual commit is consistent with the previous state of the repository tree. It also formalizes the mechanism for recovering if validation fails, or the sequence of commits is otherwise broken. This makes it easier and cheaper for all synchronizing services (AppViews as well as Relays) to fully validate.


## Commit Validation: MST Operation Inversion

The `#commit` message type on the `com.atproto.sync.subscribeRepos` event stream (aka, firehose) contains two important fields:

- `blocks` is a small CAR file containing a "slice" of the repository MST data structure, with only the new data (blocks) included
- `ops` is an array of record operations, describing the changes reflected in the `blocks`

The MST in the `blocks` is signed, while the `ops` are not. Today, the two fields can be partially cross-verified, in that all the `ops` can be verified against the `blocks`. However, the `blocks` can not be fully verified against the `ops`. For example, it is possible that many records were deleted in the same commit, and this is represented in the `blocks`, but that not all the deletion ops are enumerated in the `ops`. This could be due to an implementation mistake, a hostile intermediary, or a PDS intentionally generating malformed messages.

The new way to fully cross-verify verify the `ops` with the `blocks` is to *invert* all of the operations against the (partial) MST in the `blocks`. If the list of operations is complete, then after inverting all of the operations, the tree will be back in the prior state just before the commit. This can be verified by keeping track of the previous value of the MST root CID (the commit block's `data` value).

That is, each "create" operation will be inverted as a "delete" operation and applied to the tree. Every "delete" will become a "create" of the record. Every "update" will be updated back to the previous value.

To make it possible to validate individual `#commit` messages in isolation, the previous `data` CID is included in the new `prevData` field on the message. This field is not authenticated (signed), and still needs to be verified against the actual previously seen `data` value.

The inversion process allows individual commits to be verified, without retaining any MST nodes or record data locally. Assuming an initial repository state can be verified, and each commit can be verified, the final repository state is verified, "by induction". This method is sometimes called "inductive firehose".

A few schema changes are needed to enable validation via inversion:

- the new `prevData` field on `#commit` messages, mentioned above
- the `repoOp` objects (in `ops` on `#commit` messages) need to include the previous value (CID) for "update" and "delete" actions, so they can be inverted. Note that the record values are not needed, only the CID.
- in some cases, additional unchanged MST nodes need to be included in `blocks`, so that the tree data structure can be updated

Which additional nodes need to be included, and when? The tautological answer is "whatever blocks are needed for operation inversion". One attempt at a definition is to include blocks which contain the MST keys (paths) directly adjacent to keys added or removed to the tree by any operations. We will provide test vectors covering these situations, and will work towards a more formal definition for the final specifications.


## Staying Synchronized: `#sync` event, auto-repair, and account status

The "inductive firehose" process works when each `#commit` is fully verified, in a sequence. What happens if things get off-track? As much as possible, the network should automatically recover from error conditions.

There are two broad categories of errors that can disrupt synchronization:

- **Invalid Commit Message:** the message has invalid CBOR encoding, invalid MST structure, missing blocks, mismatching operation list, etc. Usually caused by *upstream implementation bugs*.
- **Discontinuous Commits:** re-transmitted or missing commits, `prevData` does not match the actual previous commit, duplicate `rev`, etc. Often caused by *operational issues*, though might also be due to implementation bugs.

A general assumption is that implementation bugs will persist until there is developer intervention, while operational issues are usually transient and can be recovered from automatically. This means that invalid commits are usually dropped by consumers. Some types of discontinuous commits (like retransmission) are also ignored, while others (forks, skipped messages) result in re-synchronization.

A new message type (`#sync`) is added to help with some error conditions. This message is similar to a `#commit` message, in that it contains a single authenticated commit, encoded in a CAR file. It only contains the commit block, not any MST nodes or records (even if the commit did mutate the directory). The semantics of the `#sync` message are: "this is the current state of the repository". When receiving a `#sync`, consumers should check if the `rev` is old (in which case it is ignored) or matches the current state of the repository (a no-op, also ignored). Otherwise, if there is a gap, the consumer needs to resynchronize the repository by fetching a full CAR file using `com.atproto.sync.getRepo`. They should make this request to the same host they are consuming the firehose from (eg, a relay); see the "Sync Boundary" section below for more details.

`#sync` can be emitted by a PDS or Relay to force a repair of the account if needed, for example after fixing an implementation bug. They should also be emitted (after `#account` events) at certain account lifecycle moments, such as when the account is first created, after account migration, or after an account recovery using a CAR file import.

However, in most operational error conditions, it is the expectation that consumers will re-synchronize automatically, without waiting for a `#sync` message. It is not the responsibility of a PDS, or a relay, to proactively detect if downstream consumers have gotten out of sync.

The re-synchronize process could start as soon as a problem is detected, or it might be enqueued as a background job. The resync might fail or timeout, and require retries. To keep track of things during this period, and make it transparent to operators, a new account state is added: `desynchronized`. An account in this state has lost sync, and repair has not been completed yet. An account in this state is not necessarily `active=false`: data should remain indexed, and can be redistributed (though services may set their own policies and behaviors on this point).

Re-synchronization can require fetching and processing a full repo CAR file, which is a relatively expensive operation. See the "Sync Boundary" section for details of this process, and "Streaming CAR Processing" for ways to make it cheaper for consumers. To prevent network abuse and avoid resource consumption, services should keep counters and rate-limit re-synchronization attempts. Accounts which are in a rate-limited state (for exceeding this or other limits) can use the new `throttled` account state.

Relays do not perform the re-synchronization process. They should drop invalid commit messages (not pass-through). They should keep track of discontinuous commits, and may set the `throttled` state if limits are exceeded. Otherwise they should pass through discontinuous commits as-is (and update their "last commit" state).


## Sync Boundary and Record Operation Stream

Not all services working with atproto data need to implement "Authenticated Transfer" and verify synchronization. They might skip some aspects of verification, or use an un-authenticated event stream (like Jetstream).

The intent of Authenticated Transfer is to enable interoperation (possibly adversarial interoperation) between service providers who have no relationship, all over the open internet. This authenticated synchronization often occurs indirectly, across multiple hops and providers. There is a "sphere of authenticated transfer" which starts at the account's repository host (eg, PDS), which signs commits. It terminates at the last service in the transfer chain which implements full validation. Intermediate services, like a relay, might do validation, but that is not integral or required for their role, and if they are operated by an untrusted party, they are not the "trust boundary".

The "trust boundary" also corresponds with a "sync boundary": the terminating service is also responsible for re-synchronizing accounts via a `getRepo` call.

This service also often unpacks commits into individual record operations. Commits can contain multiple record operations, but many applications process and index records individually. An example of this pattern is Jetstream, which terminates the sync protocol by consuming commit events, and emits record operation events. This is also the step where many applications filter out irrelevant records.

Application indices (eg, AppView) often want to ensure they have processed all relevant record operations for a repository. There is a relatively efficient process to keep track, including resynchronization events:

- the terminating service keeps a table of all relevant records that have been tracked, including at least the account DID, collection, record key, and record CID. the record value itself does not need to be stored. Only records with relevant collections need to be tracked
- when new commits are processed, after they are fully verified, the table is updated with all the record operations in the commit, and the record operations are indexed (or passed along to other services for processing)
- in the event of a re-synchronization, the service fetches the repo CAR file, and then walks through all records in the tree, in key-sorted order, comparing against the table of record state. Any mismatches (created, updated, deleted records) are updated in the table, and a record op is emitted for processing
- if an account needs to be fully deleted, the table of records can be iterated through, with "delete" record operations processed for every current record

This entire commit-to-record-op process can be implemented as a generic SDK or microservice. Multiple downstream services can share a single sync termination service; for example multiple services operated by the same organization, or a trusted group of collaborators, or anybody with a formal agreement or relationship (eg, contract or service agreement) with a service provider.


## Commit Size Limits

The `tooBig` flag on `#commit` messages played a role similar to `#sync` events in the past: it indicated that the commit message was incomplete, and that data would need to be fetched out-of-band (eg, via `getRepo`). The `#sync` can fill some of these use cases (such as bulk-updating an account via CAR import), but is an expensive operation and use should be minimized (and rate-limited).

In normal operation, commits should never result in `tooBig` commits being emitted. In Sync 1.1, this is being formalized by deprecating the flag, and having a limit on `#commit` message sizes.

The size limits maybe be tweaked, and then evolve over time, but as some rough numbers:

- all event stream messages (WebSocket frames) limited to 5 MBytes
- the `blocks` field in `#commit` messages limited to 2 MBytes; note this includes record blocks, MST node blocks, the commit block, block CIDs, and the CAR header
- individual records limited to 1 MByte
- maximum of 200 record operations per commit

Note that limits can not be maximized together: you can't have a commit with 10 records, each at the maximum size. Users and application designers should steer well clear of all these limits. For example, records should be less than 10 KByte in almost all situations; blobs should be used for larger data.


## Partial Synchronization

As the atproto network has grown, it has become more costly and inefficient to process every single account or every single record in the network. This is particularly true when working with applications using novel Lexicon schemas: in that situation, most of the 30 million repositories in the network contain no relevant records; and for the repositories which do, only a small fraction of the records in the repository are relevant. As part of the overall Sync 1.1 project, we are adding features to help with both of these challenges.

To help with identifying the subset of repositories (DIDs) which are relevant to a given application, there is a new `com.atproto.sync.listReposByCollection` API endpoint. This takes a collection (NSID), and enumerates a list of all DIDs in the network which contain records of that type (using a cursor for pagination). This is particularly helpful when bootstrapping services from scratch. This endpoint may be implemented by relays, though it should be considered an optional additional service, not a required part of the relay role in the network.

To help with fetching a subset of records from a repository, a variant of the `com.atproto.sync.getRepo` endpoint will be specified which allows fetching a "subtree" which includes records matching a provided collection prefix (which might include multiple collections/NSIDs). The commit block and all records in the range would be included, as well as MST nodes "proving" the records, as well as "proving" the keys (if any) directly adjacent to the key range, which provides a "proof" that there are no other matching keys.


## Streaming CAR Processing

Implementing automatic re-sychronization will involve more `com.atproto.sync.getRepo` fetches by consuming services. It is important to make these efficient to process.

One mechanism is to request only the record range needed, as described above in "Partial Synchronization".

Another is to make it possible to read repo CAR files in a "streaming" fashion, instead of loading the full tree into memory. The common case is to "walk" the keys in the MST data structure, starting from the commit block and then descending in depth-first order starting with the lowest key. If the blocks in the CAR file are in just the right order, then the tree can be walked by reading a single block at a time, and retaining only a few "parent" parsed tree nodes in memory.

The exact order requirement will need to be formalized more, but would be roughly:

- commit CID as the first entry in the `root` list in the CAR header (already the case)
- commit block first, then the root MST node of the tree
- for every node in the tree, include blocks corresponding to the entries in the node. if the entry slot is a child node, include that node block, and recurse depth-first. if the entry slot is a record, include the record block


## Deprecations

A few fields and endpoints will be deprecated, or fully removed if deprecated earlier. No Lexicon-breaking deprecations will be finalized, meaning no "required" fields will be removed.

For example:

- the "optional and nullable" `prev` CID field on `#commit` events will be fully removed
- the deprecated `#handle`, `#tombstone` and `#migrate` firehose events will be fully removed
- the `blobs` array of CIDs in `#commit` will be deprecated and set to an empty array, though not fully removed (the field is required)
- the deprecated `com.atproto.sync.getHead` endpoint will be removed
- the deprecated `com.atproto.sync.getCheckout` endpoint will be removed
- the deprecated `commit` parameter to `com.atproto.sync.getRecord` will be removed

The `com.atproto.sync.getBlocks` endpoint will be considered optional, not an expected part of the "sync interface" for PDS and relay implementations.


## Out of Scope

There are a few related changes we are interested in making, but will not be included in this Sync 1.1 project:

- the firehose sequence and cursor mechanism
- the XRPC event stream wire format, specifically the two-CBOR-object WebSocket framing
- renaming the `#commit` message `repo` field to `did`, for consistency with other message schemas
