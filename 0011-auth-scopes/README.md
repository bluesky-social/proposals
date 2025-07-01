
# 0011 Auth Scopes for ATProto

*Leave feedback on this proposal [on Github Discussions](https://github.com/bluesky-social/atproto/discussions/4013)*

OAuth enables applications to log into “Personal Data Servers” (PDSes) and access a user’s account. It is the preferred method for creating sessions. The OAuth flow requires permission “scopes” which describe the level of access granted with the session. At the time of writing, ATProto uses a set of temporary coarse-grained scopes called the “transitional scopes.”

This proposal introduces a complete system of permission scopes to be used with OAuth in the AT Protocol. These scopes are granular, meaning they allow applications to request access to only the resources they require. The goal for these scopes is to comprehensively describe all kinds of needed access in the AT Protocol.

This proposal also introduces updates to Lexicon to facilitate the permission flows:

- Lexicon-defined descriptions of resources (such as records) that will be presented to users during the login flow.
- Lexicon-defined permission bundles that make the authorization flow more legible to users.
- Localization mechanisms for user-facing descriptions.

Implementation Status: This proposal has been in development for some time and is relatively firm on the overall structure and semantics. Some details (like naming, scope string syntax, and Lexicon schema structure) are likely to change in the final specifications. Other tweaks are expected as we implement and experiment with the system.

## What Kinds of Resources?

Permissions relate to account resources on PDS instances:

- Public Repository (records and collections)
- Service Authentication (API calls to external services)
- Blobs (uploaded media files)
- Identity (DID and handle)
- Account (hosting status, email address)

As an example, the Bluesky Social client app needs access to:

- Create, update, and delete public repo records of particular collections:
  - `app.bsky.actor.profile` (create, update)
  - `app.bsky.feed.post` (create, delete)
  - `app.bsky.feed.like` (create, delete)
- Upload blobs
- Use service proxying to authenticate users on the Bluesky appview
  - Audience: `did:web:bsky.app#bsky_appview`
  - Methods: `app.bsky.feed.getTimeline`
- Use service proxying to authenticate users on the Bluesky chat service
  - Audience: `did:web:api.bsky.chat#bsky_chat`
  - Methods: `<any>`
- Submit moderation reports to any configured moderation service:
  - Audience: `<any>`
  - Methods: `com.atproto.moderation.createReport`
- Update identity data (DID)
  - Handle updates
- Access and update account hosting details
  - Deactivate account
  - Update account email

The set of resources is likely to evolve and be extended over time. For example, we know that in the future PDS instances are likely to support private preferences, restricted group data, and encrypted messaging.

The initial named protocol resources are: `repo`, `rpc`, `blob`, `identity`, and `account`.

## Permissions

Permissions are a new protocol primitive for describing access to a given resource. Each resource type has a defined set of parameters that can attenuate the permission.

Permissions can be represented as a JSON object, as part of the Lexicon language. An example permission looks like:

```json
{
  "type": "permission",
  "resource": "repo",
  "collection": ["app.bsky.feed.post"],
  "action": "*",
}
```

The `type` indicates that this is a permission, and `resource` indicates that it covers records in the user's public repository. The `collection` field is specific to the `repo` resource type and is one or more NSIDs indicating the type of records. Likewise, `action` describes which actions are granted. Each permission resource type has defined parameters, default values, and semantics around overlapping or conflicting permissions.

Some other examples (note that parameter names and semantics are not finalized):

```
{
  "type": "permission",
  "resource": "rpc",
  // "lxm" is the existing field name used to indicate the Lexicon endpoint being authorized
  "lxm": ["app.bsky.feed.getFeedSkeleton"],
  // "aud" is short for "audience", indicating the remote service being authenticated with
  "aud": ["*"]
}

{
  "type": "permission",
  "resource": "blob",
  "accept": ["image/*"],
  "maxSize": 1000000
}
```

When used as an OAuth scope, a permission needs to be reduced to a string representation with no spaces. For this case, we use the following structure: `resource[:positional][?additional=params]`

For example:

```
repo:app.bsky.feed.post

rpc:app.bsky.feed.getFeedSkeleton?aud=*

blob?maxSize=1000000
```

The string syntax operates with the following rules:

* The parameters of a permission are supplied as URL-encoded query parameters.
* Resource types can specify one field as a positional argument.
  * This positional argument depends on the resource type. (E.g. it differs in meaning between `repo` and `rpc`.)
  * Positional arguments are optional. The same field could also be provided as query parameters. This is required if multiple values (an array) are provided for the same parameter in a single string representation.
  * The positional argument should not include the prefix separator character (`:`), or characters which would require any form of escaping (such as "?")

Permissions can be transformed back and forth between string representation and JSON object representation. A motivating use case for the object representation will be described below.

This proposal document does not include full definitions for the initial permissions. They are expected to be published and included in the written specifications when the reference implementation is complete.

It is possible to request wildcard permissions on certain resources. For example, `repo:*` would grant full permission over all records of any type in the repository. The user should be warned appropriately during the auth flow if such a powerful permission is being requested, and additional levels of authentication and confirmation may be required. As a general design pattern, prefix matching of NSIDs (such as `repo:app.bsky.*`) is not supported for any of the initial resource types. See design notes at the end of this document for details, though note that each resource defines syntax and semantics for parameters, and this syntax might be supported by other resources in the future.

RPC calls allow sending authenticated API requests to remote servers using Service Auth. They are a good case study for wildcard behaviors. A common application extension point is to allow a specific request type to arbitrary service providers configured by the user. For example, users might configure new moderation services to send reports to, or subscribe to new feed generators. In other situations, such as private chat, a client could be restricted to a single provider (defined by the client) but use many API endpoints. If a permission requests a specific service provider, the auth server must resolve the service reference (DID) and display the service hostname that would be proxied to.

To summarize the core permissions system described in this section:

* Client apps can request Permissions in their OAuth scopes
* Auth Servers (PDS or entryway) can render them to end users in the auth flow
* Resource Servers (PDS) can interpret them to allow or deny requests

For very narrowly scoped client apps, this may be all the functionality required, and the additional functionality described below could be skipped.

## Permission Sets

Full-featured client apps will require a large number of granular permissions to function: dozens or even hundreds of individual permissions. This presents a user experience challenge, as long lists of permissions are difficult to parse, and a developer experience issue, as defining and maintaining these lists is toilsome. It is also a security concern: users who are routinely presented with long lists of scopes will become less discerning as to which scopes they approve, and it becomes easier for an unscrupulous client developer to sneak in any scopes they like.

To simplify permission management, Lexicon designers will be allowed to define "sets" of permissions as part of the schemas they publish. These permission sets are themselves Lexicon schemas and are referred to by NSID. For example:

```
app.bsky.authFull
```

This NSID could be defined as “permissions needed to create a fully-featured Bluesky Social client.”

Auth Servers resolve, authenticate, and process permission-sets dynamically. Sets include human-meaningful names and descriptions that are displayed to end users as part of the auth approval flow. Permission sets are published publicly and can be used by any client developer. (Caching and fallback behaviors for dynamic resolution are discussed below.)

Expanding on our example, the permission-set below describes the basic permissions needed by a Bluesky client app. The final "full" permission-set will, of course, include many more permissions.

*Note: the specific terminology ( "permission-set"), parameters, and schema structure ("type" versus "ref", etc) may change as this proposal evolves.*

```
{
  "lexicon": 1,
  "id": "app.bsky.authFull",
  "defs": {
    "main": {
      "type": "permission-set",
      // Note: will also need to add a mechanism for internationalization
      "descriptions": [
        { "lang": "en", "text": "Creation of Bluesky posts & likes and authenticate the user  towards Bluesky AppViews." }
      ],
      "permissions": [
        {
          "type": "permission",
          "resource": "repo",
          "collection": ["app.bsky.feed.post"],
          "action": "*"
        },
        {
          "type": "permission",
          "resource": "repo",
          "collection": ["app.bsky.feed.like"],
          "action": "*"
        },
        {
          "type": "permission",
          "resource": "rpc",
          // set of parameters to inherit from permission-set; syntax might change
          "inherit": ["aud"],
          // supplying multiple values in one permission, as a possible syntax
          "lxm": [
            "app.bsky.actor.getPreferences",
            "app.bsky.actor.putPreferences",
            "app.bsky.actor.getProfile",
            "app.bsky.actor.putProfiles",
            "app.bsky.feed.getActorLikes",
            "app.bsky.feed.getAuthorFeed",
            "app.bsky.feed.getFeed"
            ...
          ]
        },
        {
          "type": "permission",
          "resource": "rpc",
          "aud": ["*"],
          "lxm": ["app.bsky.feed.getFeedSkeleton"]
        }
      ]
    }
  }
}
```

Permission sets can be referenced as scope strings and requested in OAuth scopes by using the `include` resource and the NSID of the published set as the positional parameter. They can also be parameterized with URL query parameter syntax. For example:

```
include:app.bsky.authFull?aud=did:web:api.bsky.app%23bsky_appview
```

Sets themselves do not define parameters. Instead, permissions define semantics around merging permission-level parameters and set-level parameters, sometimes configured using attributes like `inherit`. In the example above, the `rpc` resource type (for [service auth](https://atproto.com/specs/xrpc#inter-service-authentication-jwt)) would declare that the `aud` parameter (specifying the audience service) may default to a set-level param if the more granular permission-level param is undefined. In the above example, the permission to call `app.bsky.feed.getFeedSkeleton` on any audience (wildcard) would be retained even with the `aud` defined on the set scope string.

Clients are free to mix permission sets and granular permissions in their requested auth scopes. Auth Servers will attempt to proactively simplify permission requests presented to users, so that individual permissions covered by a permission-set do not need to be rendered to the user. The full set of permissions could be displayed in "fine print". This behavior helps helps keep the auth screen simple for users.

For example, a client might request:

```
// basic bluesky auth (with bluesky appview)
include:app.bsky.authFull?aud=did:web:api.bsky.app%23bsky_appview

// basic bluesky DMS auth (with bluesky chat service)
include:chat.bsky.authFull?aud=did:web:api.bsky.chat%23bsky_chat

// blob uploads; Auth Server might decide not to show this permission request in the context of already requesting app.bsky.authFull
blob?maxSize=1000000

// service auth to all feed generators; this is already included in app.bsky.authFull, so might not render in request flow
rpc:app.bsky.feed.getFeedSkeleton?aud=*

// create statussphere statuses (a permission, not a permission set)
repo:xyz.statusphere.status
```

Permission sets are Lexicon schemas and are published and fetched using the Lexicon resolution system, which includes cryptographic authentication. Permission sets are expected to be updated over time as new schemas are added to a namespace (eg, new record types or API endpoints). Auth Servers are expected to maintain a cache of resolved sets, but to re-resolve them periodically. The permissions associated with an Access Token should remain fixed, but when a client refreshes their tokens (obtaining a new access token), the computed permissions for the session may be updated to reflect changes to sets requested by the client.

This adds an intentional degree of temporal dynamism to auth sessions involving Permission sets. Lexicon designers can define new resources (eg, record types), update published sets to include permissions to those resources. Client software can then be updated to take advantage of those new lexicons, without requiring users to re-authenticate their sessions.

Permission sets are a powerful feature, but rely heavily on lexicon resolution and maintenance. The following sub-sections describe limitations and mitigations to ensure the system stays safe, reliable, and interoperable.

### Namespace Authority

Permission sets are limited to expressing permissions that reference resources under the same NSID namespace as the set itself. To start, this means they can only include repo record and service auth permissions. For example, a set can not include a permission to manage account identity; that needs to be requested as a separate scope.

Authority is based on the relative structure of NSIDs, without "siblings" or special namespaces. Broadly, this ensures that sets can not request permissions across namespaces. Specifically:

* Permission sets can address resources in the same NSID “group”; or “children” (sub-domains), recursively deep
* can not address “sibling groups" or “parents” in the NSID hierarchy

For example, the set `app.bsky.feed.authOnlyPost` could include permissions to `app.bsky.feed.post` records and making `app.bsky.feed.getPostThread` API endpoint requests to remote services. But it could not grant permissions to `app.bsky.actor.profile`. A set `app.bsky.authFull`, which is a level up in the hierarchy, could include permissions to all these resources, or even further down the hierarchy.

### Permission Set Resolution, Caching, and Aggregators

Permission set Lexicons need to be resolved by auth servers (eg, PDS instances). The existing [Lexicon publication and resolution system](https://atproto.com/specs/lexicon#lexicon-publication-and-resolution) describes how schemas are published and verified.

To reduce network traffic and increase resiliency to outages, auth servers are expected to heavily cache all aspects of set resolution. This is analogous to re-fetching OAuth client metadata documents and caching those fetches. Caches may be shared across all accounts and sessions on an Auth Server.

Permission set schemas should be cached with long "expiration" times but shorter "stale" times. Stale Lexicons should get updated (aka, attempt resolution refresh), but if resolution fails, the previously existing stale values can be used. The recommended “stale” lifetime is 24 hours, and this is intended as a firm upper bound on cache lifetime. The firm lower bound on cache lifetime is that of access token lifetimes, meaning 15-30 minutes. The recommended "expiration" lifetime is 90 days, but this is not a firm bound.

At the start of a session, if a permission set can not be resolved (and is not already in a local cache), the auth request will fail.

As an optional mechanism to trigger a cache purge, client instances can set an updated `software_version` field in their client metadata document. If the auth server sees a client version change, it could ensure that any permission sets referenced by the client would be refreshed. The motivation is to ensure that roll-out of new client software that requires updated sets can rely on auth servers giving the updated permissions within a timeframe shorter than the 24 hour “stale” cache lifetime.

Lexicon Aggregators are proposed services that aggregate all published Lexicons by monitoring the firehose. They are an *optional* mechanism to offload the work of resolution by making API calls to an aggregator service instead of resolving DNS and connecting to a PDS directly. The degree to which an Auth Server offloads resolution and validation work to an aggregator is ultimately left to the Auth Server operator: for example, whether the Auth Server re-verifies NSID DNS delegation, or verifies repo proof signatures.

## Other Details and Mechanisms

This section addresses some specific design decisions and updates to adjacent specifications.

### Lexicon Resolution Overrides

Permission set dynamism is mostly intended to allow expansion of permissions over existing sessions, but permissions might also be removed. Lexicon designers should remove or attenuate permissions very sparingly, because it could break other software and sessions in the network.

To mitigate the risk of lexicon namespace hijacking, or intentional disruption of the network by lexicon maintainers, lexicon aggregators and auth server operators might intervene in the lexicon resolution process to prevent breakage. To formalize this type of intervention, we are introducing the concept of "Lexicon Override Repositories".

Services which rely on resolution would be configured with an ordered list of override repository DIDs. When resolving an NSID, the service would first check each override repository, in order, for schema definitions. Only if override definitions are not found would the service attempt resolution against the live network.

The expectation is that a small handful of security teams would maintain override repositories. If a disruptive change to a popular lexicon takes place, the team would publish an old version of the schema in their override repository. This process would be transparent and verifiable using the usual repository authentication process. Override repositories could be used to address:

- security incidents related to over-permissive auth scopes
- lexicon authority hijacking (eg, hacking DNS registrations)
- “self-destruction” of popular Lexicons
- abandoned lexicons (eg, domain registration lapses)

Lexicon aggregators could help offload this behavior by allowing override DIDs to be specified as part of Lexicon resolution API requests.

### No Partial NSID Wildcards

This proposal prohibits partial wildcards (eg, prefix match), and only permits full wildcards (eg, any value is allowed). For example, `repo:app.bsky.*` is not allowed, but `repo:*` is allowed.

This was discussed further in an earlier draft of this proposal: [https://github.com/bluesky-social/atproto/discussions/3655](https://github.com/bluesky-social/atproto/discussions/3655)

### Auth Server / Resource Server Scope Synchronization

One deployment pattern supported by atproto is to have multiple PDS instances with a single "entryway" service that manages accounts. In OAuth terminology, the PDS instances are Resource Servers (RS) and the entryway is the Auth Server (AS). In this architecture, the AS would resolve permission sets to a concrete set of permissions. There would be a need to synchronize the set of permissions allowed for a given access token when requests are made to the RS.

We think the synchronization mechanism should be left to implementations, not included in the protocol specification. However, we will describe some possible implementation patterns in this proposal.

A simple approach would be to include the scope strings in the access token directly. The access token format is implementation-specific, but common formats like JWTs allow additional claim fields. The downside with this is that large sets of scopes make the token large, and the token is included as an HTTP header on every single API request.

An alternative would be to have the auth server compute the full set of permissions in a normalized format (eg, a sorted file of scope strings, or CBOR encoded permission objects), and compute the CID (hash). The permission set would be stored in a database, and the CID would be included in the access token. When the RS (PDS) receives the access token, it would make an API call to the AS to fetch the permissions, and then cache the results locally. Caching could have an indefinite lifetime (the CID always identifies the same set of permissions), and could be shared across accounts, sessions, and client implementations. This means the cache hit rate would be very high.

### Client Metadata Document Caching

The auth scopes requested by an OAuth client when starting a session (PAR) must be a subset of those declared in their client metadata document. Auth Servers are already expected to periodically re-fetch client metadata documents, for example, to check for confidential client attestation key removals (which might invalidate associated sessions). That need predates this proposal, but the expectations around cache lifetimes were not specified earlier.

The client metadata document cache lifetime should be about the same as access token lifetime, which means 15-30 minutes.
