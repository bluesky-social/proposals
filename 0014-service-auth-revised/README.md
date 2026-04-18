0014: Service Auth Audience Revised
============================================

*Join the [Github discussion](https://github.com/bluesky-social/atproto/discussions/4708).*

This is a revision of [Proposal 0013: Consistent References to DID Services](../0013-service-auth-refs/), to combine the DID and service name into a single `aud` string in auth JWT tokens. The background and overall goal from the earlier proposal is unchanged.

This update is in response to feedback on the earlier proposal. We are fairly confident in this updated direction and intend to go forward with implementation in the near future. An ecosystem rollout plan is included. This document also proposes updates to how the `iss` field is formatted.


## Combined `aud` field

Service auth tokens will support combined DID and service name syntax in the `aud` field, matching the syntax in service proxy headers. The `com.atproto.server.getServiceAuth` PDS endpoint `aud` field will also support this same syntax (this will be a breaking but low-disruption change to the published lexicon).

In both cases, support for bare DIDs will not be immediately removed, but it will be described as deprecated. Semantically, a bare DID is not a "wildcard" or "any" value, it is a distinct audience. The recommendation is for implementations (eg, SDKs used by services receiving and validating tokens) to support an array of configured `aud` values. That array could include both a full DID-and-fragment, as well as a bare DID, at least during a transition period. Implementations should not treat a bare DID as always valid for any service endpoint registered with service DID.

## Other Changes


The `iss` field in service auth tokens currently allows an optional fragment with a service name. This is not widely used in practice, though it has been used to authenticate labeling services to PDS instances (when both are operated by the same party). Support for the fragment syntax in the `iss` field will be removed: only bare DIDs will be allowed. The `kid` JWT header field will be allowed to identify a signing key ("verification method") from the issuer DID document, with a default value of `atproto`. Note that this is a switch in semantics from identifying the issuing *service* (with a table mapping service types to key identifiers), to identifying the *signing key* itself. Receiving services should *not* 

The `lxm` field in auth JWTs will become required for XRPC endpoint calls. It is currently described as optional in the specs.

## Ecosystem Rollout Plan

The rough plan is to do a two-phase rollout.

The first phase is to prepare receiving services for DID-and-fragment `aud` values, while continuing to use bare DIDs by default in the context of service proxying:

- update the specification text
- change the `getServiceAuth` lexicon, and have the reference PDS allow OAuth clients to request service auth tokens that match their permissions. Note that these tokens might not work with some services yet, but no currently-working client/service flows would be impacted
- update SDKs and cookbook examples as needed, with configurable `aud` validation (eg, an array of acceptable values)
- update popular services to be configurable, and configure them to accept both forms of `aud`
- add a config flag to the reference PDS which enables passing through full DID-with-service for service proxy requests. This defaults to "disabled" in this phase. Most operators would not touch this config.
- fully roll out the `lxm` and `iss` changes in the reference PDS, SDKs, and services. This is not expected to be disruptive to the ecosystem.

The second phase is to gradually shift over to using full DID-and-fragment `aud` with service proxying:

- add `goat` functionality to make direct authenticated XRPC calls, with control of the `aud` value. Use this to test and debug `aud` compatibility on various services
- developers on self-hosted PDS instances can toggle their PDS configuration to test which (if any) real-world services don't accept the new `aud` type, and ask them to update
- once most of the ecosystem has transitioned, large PDS providers can flip the config flag for short periods of time to see if any remaining services break. and then eventually flip on the new format permanently
- eventually services can re-configure to remove support for "bare" DIDs in the `aud` field

We think that this transition plan is realistic, and lets us move ahead with implementation work with only a reasonable amount of disruption.
