0013: Consistent References to DID Services
============================================

*Join the [Github discussion](https://github.com/bluesky-social/atproto/discussions).*

This is a request for feedback on a proposed change to how inter-service auth tokens, PDS request proxying, and OAuth permissions all combine in atproto.

If feedback is positive, we will rapidly finalize details of the proposed plan, update the atproto specifications, and update reference implementations in the coming weeks. We would also publish focused developer guidance with specific deadlines and changes needed by service type.

## The Problem: Inconsistent DID Service References

AT Protocol has a concept of DID service references. These are the combination of a DID and an id fragment that references a service entry in the DID document. For example, the DID `did:plc:newitj5jo3uel7o4mnf3vj2o` (corresponding to the handle `@xblock.aendra.dev`) has a service entry with id `#atproto_labeler`. Sometimes the service entry is combined in a single string, like `did:plc:newitj5jo3uel7o4mnf3vj2o#atproto_labeler`.

Service references appear in a few different places:

- PDS service proxy header (`atproto-proxy`, set by client apps)
- service auth JWTs (in the `aud` field, and in the `iss` field)
- the `com.atproto.server.getServiceAuth` PDS endpoint (which generates service auth JWTs) has an `aud` field
- `rpc` permissions (`aud` field)

Including a service reference as part of a service auth `iss` field is not used very frequently, but it has been used between moderation services (labelers) and PDS instances, to fetch private account metadata. In this situation the service id fragment is mapped to a verification method (signing key) in the DID document.

Unfortunately, there has been inconsistency between whether the service type name (eg, the fragment) is required or optional in these different contexts:

- It is unambiguously required in proxy headers, because the PDS needs to resolve the specific endpoint to send the request to.
- According to the specs, it has been optional in service auth JWTs (in the `aud` field). But in practice the PDS reference implementation has only included the bare DID.
- The `getServiceAuth` has an `aud` query param with `did` format type, which does not allow a service fragment.
- The `rpc` permission field requires `aud` to include both DID and service ref (bare DID is not allowed), according to the specs.

One problem with this situation is that it is effectively impossible to use `getServiceAuth` with an OAuth client session unless `aud=*` is specified. This is because the `aud` param in the API request and the `rpc` permission must match, but they have incompatible required syntax (the permission requires the service name, while the endpoint forbids it).

Another problem is that service auth JWTs with bare DIDs in the `aud` field can be used for any service endpoint under that DID, even though they may have been generated with a specific endpoint in mind. This allows for privilege escalation.

## The Goal: Consistent References

The goal is to get the entire ecosystem to a place where:

- service auth JWTs always include the full `aud` information (both the DID and service name specified)
- client requests and permissions always include the full `aud` information (both the DID and service name specified)

Many implementations and deployed services will need to be updated:

- services need to support full `aud` information in JWTs; and eventually require that info to always be present
- PDS implementations need to require and pass-through the full `aud` information when generating service auth tokens (including with the `getServiceAuth` endpoint)
- client apps need to always include the full `aud` information (including when calling the `getServiceAuth` endpoint

The details of how to specifying and implement these changes are flexible. What is needed is a plan for how to sequence rolling out these changes in a way that won't cause broken user experiences in the live network.

For example, we don’t want to roll out PDS changes which would break old appview implementations; or appview changes that will break JWTs issued by old PDS implementations. And we do want OAuth clients to be able to use `getServiceAuth` in a way consistent with password session clients, as quickly as possible.

## Rough Proposal: Split JWT Audience Fields

Instead of packing both DID and service name in the `aud` field of service auth JWTs, we could put the service name in a separate field.

- Service auth JWTs get a new service type field, and support for service name in `aud` field is dropped (`aud` should only contain a DID). The new field could be called something like `aud_svc` or `svc`.
- An analogous new field could be added for service names with the `iss` field, called something like `iss_svc` (retaining the service mapping), or `iss_key` (identifying a verification method). Support for combined strings in `iss` would be deprecated.
- New param on the `getServiceAuth` endpoint to allow specifying the service name. This is implicitly required for OAuth client sessions (used for permission validation); and would soon become required for all session types
- PDS can immediately start using the above two fields: emitting JWTs with the additional field when known
- Client apps should start specifying service name when using the `getServiceAuth` endpoint. For session auth clients, this is a change; for oauth clients it is not. there would not be breakage if a new client app requests this param with an old PDS implementation
- In the near future (eg, by May 2026), services should start requiring this field when verifying service auth

An advantage of this proposal is that sequencing the roll-out is relatively simple. A disadvantage is that there ends up being two different ways to represent service refs in the protocol: as a single string (eg, in proxy headers) or as two separate strings (in JWTs).

## Alternatives

An obvious alternative would be to include the DID and service name in the `aud` field more consistently, and to allow that syntax in `getServiceAuth` endpoint requests. The disadvantage of this option is that the ecosystem sequencing of updates and deprecations is ambiguous, and could drag out indefinitely.

## Possible Related Changes

While making changes to service auth JWTs, we could address some other issues:

- Specify a JWT `typ` field: currently the generic value `JWT` is used. An atproto-specific field could be specified.
- Switch JWT `iss` field semantics from “service” to “verification method”. That is, remove the mapping between service ids to public key ids.
- Verify that id names in DID documents are unique across all arrays. For example, it is invalid (entire DID document rejected) to have a verification method and service with the same id name.
- Clarify that the `lxm` JWT field is required for XRPC endpoint calls
- Specify details of the “DID plus ID/name” string syntax for the purpose of atproto. This would ensure consistent parsing of PDS proxy headers (for example)
- Clarify which HTTP headers are (and are not) expected to be passed through for PDS proxy requests and responses
