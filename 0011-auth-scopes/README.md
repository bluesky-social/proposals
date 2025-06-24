
# 0011 Auth Scopes for ATProto

OAuth provides a mechanism for client apps to authenticate with PDS instances to access resources like repository data and API proxying. But the current "transitional" scopes provide only coarse-grained access to resources: apps can either update any repository record, or no repository records.

This proposal describes a granular permission system which allows client apps to request access to only the resources they require. It also describes a system for Lexicon designers to define higher level permission sets, which makes the auth approval flow more legible to end users.

As a high-level summary:

- "Permissions" are a way to name and constrain access to protocol resources.  
- Parameterized permissions can be encoded as scope strings in OAuth requests  
- "Permission Sets" are cohesive groups of permissions, published alongside lexicon schemas  
- Permission set resolution, caching, and updates have security and governance considerations, but improve legibility for users and provide flexibility for developers

Implementation Status: This proposal has been in development for some time and is relatively firm on the overall structure and semantics. Some details (like terminology, permission NSIDs, and Lexicon schema structure) are likely to change in the final specifications. Other tweaks are expected as we implement and experiment with the system.

## What Kind of Resources?

Auth Scopes relate to account resources on PDS instances:

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

Encoding these resources as conventional OAuth strings might look something like the following:

```
# NOTE: these scopes are for comparative purproses and are not the syntax that this document proposes 

# account details
- account:email

# create/manage blobs
- blobs:create
- blobs:update
- blobs:maxSize:10000000

# CRUD public repo records
- repo:app.bsky.feed.post:*
- repo:app.bsky.feed.like:create
- repo:app.bsky.feed.like:delete

# service proxying for appview
- serviceAuth:*@did:web:bsky.app

# service proxying for chat
- serviceAuth:*@did:web:api.bsky.chat

# service proxying for feeds
- serviceAuth:app.bsky.feed.getFeedSkeleton@*
```

While this syntax is terse and understandable, it assumes a global namespace of resource types (`repo`, `serviceAuth`, etc). The names, parameters, and syntax would all need to be defined and agreed upon at the protocol level, whereas atproto’s authorization system is open to evolution and extension.

## Resources and Permissions

For atproto, we know that the set of resources is likely to evolve and be extended over time. For example, we know that PDS instances are likely to support private preferences, restricted group data, and encrypted messaging in the future. Independent projects might want to add additional features and resource types, such as email or calendaring.

To keep the system flexible, permissions are referenced by NSID and can be defined as Lexicon schemas. This includes the names and types of parameters that describe or constrain access to the permission. For example, public repository records could be referred to as `com.atproto.auth.repo` and defined as:

```json
{
  "lexicon": 1,
  "id": "com.atproto.auth.repo",
  "defs": {
    "main": {
      "type": "permission",
      "description": "Grants access to records in the public repo",
      "parameters": {
        "type": "params",
        "required": ["collection"],
        "properties": {
          "collection": {
            "type": "string",
            "description": "The collection being granted access to"
          },
          "create": {
            "type": "boolean",
            "default": false
          },
          "update": {
            "type": "boolean",
            "default": false
          },
          "delete": {
            "type": "boolean",
            "default": false
          }
        }
      }
    }
  }
}
```

Granular permissions have a string syntax which is the permission NSID, followed by any parameters in URL query parameter syntax:

```
com.atproto.auth.repo?collection=app.bsky.feed.post&create=true

com.atproto.auth.blob?create=true&maxSize=1000000
```

The initial set of permission definitions would all be the `com.atproto.*` namespace, and we do not expect PDS implementations to dynamically resolve or support arbitrary permission definitions. The `permission` schema is only meant as a coordination mechanism for evolving these semantics over time or introducing new resources to the PDS.

This proposal document does not include definitions for the initial permissions. They are expected to be published and included in the written specifications when the reference implementation is complete.

It is possible to request wildcard permissions on certain resources. For example, `com.atproto.auth.repo?collection=*&update=true&delete=true&create=true` would grant full permission over all records of any type in the repository. The user should be warned appropriately during the auth flow if such a powerful permission is being requested, and additional levels of authentication and confirmation may be required. Note that the wildcards are all or nothing. It is not possible to specify a namespace prefix, such as `collection=app.bsky.*`.

Service Auth, which allows sending authenticated API requests to remote servers, is a good case study for wildcard behaviors. A common application extension point is to allow a specific request type to arbitrary service providers configured by the user. For example, users might configure new moderation services to send reports to, or subscribe to new feed generators. In other situations, such as private chat, a client could be restricted to a single provider (defined by the client), but use many API endpoints. If a permission requests a specific service provider, the auth server must resolve the service reference (DID) and display the service hostname which would be proxied to.

Parameterized permission strings referencing atproto resources are the minimal core of the Auth Scopes proposal. Client apps can request them in their OAuth scopes, Auth Servers (PDS or entryway) can render them to end users in the auth flow, and Resource Servers (PDS) can use them to allow or deny requests.

## Permission Sets

Full featured client apps will require a large number of granular permissions to function: dozens or even hundreds of individual permissions. This presents a user experience issue (long lists of permissions are pretty ugly), a developer experience issue (defining and maintaining these lists is toilsome). It is also a security concern: users who are routinely presented with long lists of scopes will become less discerning as to which scopes they approve and it becomes easier for an unscrupulous client developer to sneak in any scopes they like.

To simplify permission management, Lexicon designers will be allowed to define "sets" of permissions as part of the schemas they publish. These permission sets are themselves Lexicon schemas and are referred to by NSID. Auth Servers resolve and process them dynamically, and include human-meaningful names and descriptions which can be displayed to end users as part of the auth approval flow. Permission sets are published publicly and can be used by any client developer.

The example set below describes the basic permissions needed by a Bluesky client app. Remember that the specific terminology (eg, "permission-set" versus "auth-bundle"), parameters, and schema structure ("$type" versus "ref", etc) are likely to change.

```json
{
  "lexicon": 1,
  "id": "app.bsky.authBasic",
  "defs": {
    "main": {
      "type": "permission-set",
      // Note: will also need to add a mechanism for internationalization
      "descriptions": [
        { "lang": "en", "text": "Creation of bluesky posts & likes and authenticate the user  towards Bluesky AppViews." }
      ],
      "permissions": [
        {
          "$type": "com.atproto.auth.repo",
          "collection": "app.bsky.feed.post",
          "create": true,
          "update": true,
          "delete": true
        },
        {
          "$type": "com.atproto.auth.repo",
          "collection": "app.bsky.feed.like",
          "create": true,
          "update": true,
          "delete": true
        },
        {
          "$type": "com.atproto.auth.repo",
	    "collection": "app.bsky.feed.like",
	    "create": true,
	    "delete": true
        },
        {
          "$type": "com.atproto.auth.endpoint",
          // "aud" deferred to permission set parameter
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
          "$type": "com.atproto.auth.endpoint",
          "aud": "*",
          "lxm": "app.bsky.feed.getFeedSkeleton"
        }
      ]
    }
  }
}
```

Note that permissions are expressed as objects in a permission set definition document, not rendered as scope strings. Permissions can be transformed back and forth between string representation and object representation.

Permission sets can be referenced as scope strings similar to permissions, and requested in OAuth scopes. They can also be parameterized with URL query parameter syntax. Note that URL escaping is required to encode the hash symbol (\#) in query parameters. For example:

```
app.bsky.authBasic?aud=did:web:api.bsky.app%23bsky_appview
```

Sets themselves do not define parameters. Instead, permissions define semantics around merging permission-level params and set-level params. In the example above, the `com.atproto.auth.endpoint` permission type (for [service auth](https://atproto.com/specs/xrpc#inter-service-authentication-jwt)) would declare that the `aud` parameter (specifying the audience service) may default to a set-level param if the more granular permission-level param is undefined. In the above example, the permission to call `app.bsky.feed.getFeedSkeleton` on any audience (wildcard) would be retained even with the `aud` defined on the set scope string.

Clients are free to mix permission sets and granular permissions in their requested auth scopes. Auth Servers may attempt to proactively simplify permission requests presented to users. The motivation for this would be to reduce "permission review fatigue" for end users. The full set of permissions could be displayed in "fine print". For example, if a client requests a permission which is already covered by a set, the Auth Server could present just the set to the user. Or, more aggressively, an Auth Server might be configured to not prompt the common low-stakes permissions if a broad set of permissions are already being requested.

For example, a client might request:

```
// basic bluesky auth (with bluesky appview)
app.bsky.authBasic?aud=did:web:api.bsky.app%23bsky_appview

// basic bluesky DMS auth (with bluesky chat service)
chat.bsky.authBasic?aud=did:web:api.bsky.chat%23bsky_chat

// blob uploads; Auth Server might decide not to show this permission request in the context of already requesting app.bsky.authBasic
com.atproto.auth.blob?create=true&maxSize=1000000

// service auth to all feed generators; this is already included in app.bsky.authBasic, so might not render in request flow
com.atproto.auth.endpoint?aud=*&lxm=app.bsky.feed.getFeedSkeleton

// create statussphere statuses (a permission, not a permission set)
com.atproto.auth.repo?collection=xyz.statusphere.status&create=true
```

Permission sets are Lexicon schemas, and are published and fetched using the Lexicon resolution functionality. In particular, sets are expected to be updated over time as new schemas are added to a namespace (eg, new record types or API endpoints). Auth Servers are expected to maintain a cache of resolved sets, but to re-resolve them periodically. The permissions associated with an Access Token should remain fixed, but when a client refreshes their tokens (obtaining a new access token), the computed permissions for the session may be updated to reflect changes to sets requested by the client.

This adds an intentional degree of temporal dynamism to auth sessions involving Permission sets. Lexicon designers can define new resources (eg, record types), update published sets to include permissions to those resources. Client software can then be updated to take advantage of those new lexicons, without requiring users to re-authenticate their sessions.

Permission sets are a powerful feature, but rely heavily on lexicon resolution and maintenance. The following sub-sections describe limitations and mitigations to ensure the system stays safe, reliable, and interoperable.

### Namespace Authority

Permission sets are limited to expressing permissions that reference resources under the same NSID namespace as the set itself. To start, this means they can only include repo record and service auth permissions. For example, a set can not include a permission to manage account identity; that needs to be requested as a separate scope.

Authority is based on the relative structure of NSIDs, without "siblings" or special namespaces. Broadly, this ensures that sets can not request permissions across namespaces. Specifically:

* Permission sets can address resources in the same NSID “group”; or “children” (sub-domains), recursively deep  
* can not address “sibling groups" or “parents” in the NSID hierarchy

For example, the set `app.bsky.feed.authOnlyPost` could include permissions to `app.bsky.feed.post` records and making `app.bsky.feed.getPostThread` API endpoint requests to remote services. But it could not grant permissions to `app.bsky.actor.profile`. A set `app.bsky.authBasics`, which is a level up in the hierarchy, could include permissions to all these resources, or even further down the hierarchy.

### Permission Set Resolution, Caching, and Aggregators

Permission set Lexicons need to be resolved by auth servers (eg, PDS instances). The existing [Lexicon publication and resolution system](https://atproto.com/specs/lexicon#lexicon-publication-and-resolution) describes how schemas are published and verified.

To reduce network traffic and increase resiliency to outages, auth servers are expected to heavily cache all aspects of set resolution. This is analogous to re-fetching OAuth client metadata documents, and caching those fetches. Caches may be shared across all accounts and sessions on an Auth Server.

Permission set schemas should be cached with long "expiration" times but shorter "stale" times. Stale Lexicons should get updated (aka, attempt resolution refresh), but if resolution fails the previously existing stale values can be used. The recommended “stale” lifetimes is 24 hours, and this is intended as a firm upper bound on cache lifetime. The firm lower bound on cache lifetime is that of access token lifetimes, meaning 15-30 minutes. The recommended "expiration" lifetime is 90 days, but this is not a firm bound.

At the start of a session, if a permission set can not be resolved (and is not already in a local cache), the auth request will fail.

As an optional mechanism to trigger a cache purge, client instances can set an updated `software_version` field in their client metadata document. If the auth server sees a client version change, it could ensure that any permission sets referenced by the client would be refreshed. The motivation for this would be to ensure that roll-out of new client software which requires updated sets could rely on auth servers giving the updated permissions within a timeframe shorter than the 24 hour “stale” cache lifetime.

Lexicon Aggregators are proposed services which aggregate all published Lexicons, by monitoring the firehose. They are an *optional* mechanism to offload the work of resolution, by making API calls to an aggregator service instead of resolving DNS and connecting to a PDS directly. The degree to which an Auth Server offloads resolution and validation work to an aggregator is ultimately left to the Auth Server operator: for example whether the Auth Server re-verifies NSID DNS delegation, or verifies repo proof signatures.

### Lexicon Resolution Overrides

Permission set dynamism is mostly intended to allow expansion of permissions over existing sessions, but permissions might also be removed. Lexicon designers should remove or attenuate permissions very sparingly, because it could break other software and sessions in the network.

To mitigate the risk of lexicon namespace hijacking, or intentional disruption of the network by lexicon maintainers, lexicon aggregators or auth server operators might intervene in the lexicon resolution process to prevent breakage. To formalize this type of intervention, we are introducing the concept of "Lexicon Override Repositories".

Services which rely on resolution would be configured with an ordered list of override repo DIDs. When resolving an NSID, the service would first check the repos, in order, for schema definitions. Only if override definitions are not found would the service attempt resolution against the live network.

The expectation is that a small handful of security teams would maintain override repos. If a disruptive change to a popular lexicon takes place, the team would publish an old version of the schema in their override repo. This process would be transparent and verifiable using the usual repo authentication process. Override repos can be used to address:

- security incidents related to over-permissive auth scopes  
- lexicon authority hijacking (eg, hacking DNS registrations)  
- “self-destruction” of popular Lexicons  
- abandoned lexicons (eg, domain registration lapses)

Lexicon aggregators could help offload this behavior, by allowing override DIDs to be specified as part of Lexicon resolution API requests..

## Other Details and Mechanisms

This section addresses some specific design decisions and updates to adjacent specifications.

### No Partial NSID Wildcards

For example, requesting This proposal prohibits partial wildcards (eg, prefix match), and only permits full wildcards (eg, any value is allowed). For example, `com.atproto.auth.repo?collection=app.bsky.*` is not allowed, but `com.atproto.auth.repo?collection=*` is allowed.

This was discussed further in an earlier draft of this proposal: [https://github.com/bluesky-social/atproto/discussions/3655](https://github.com/bluesky-social/atproto/discussions/3655) 

### Auth Server / Resource Server Scope Synchronization

One deployment pattern supported by atproto is to have multiple PDS instances with a single "entryway" service which manages accounts. In OAuth terminology, the PDS instances are Resource Servers (RS) and the entryway is the Auth Server (AS). In this architecture, the AS would resolve permission sets to a concrete set of permissions. There would be a need to synchronize the set of permissions allowed for a given access token when requests are made to the RS.

We think the synchronization mechanism should be left to implementations, not included in the protocol specification. However, we will describe some possible implementation patterns in this proposal.

A simple approach would be to include the scope strings in the access token directly. The access token format is implementation-specific, but common formats like JWTs allow additional claim fields. The downside with this is that large sets of scopes make the token large, and the token is included as an HTTP header on every single API request.

An alternative would be to have the auth server compute the full set of permissions in a normalized format (eg, a sorted file of scope strings, or CBOR encoded permission objects), and compute the CID (hash). The permission set would be stored in a database, and the CID would be included in the access token. When the RS (PDS) receives the access token, it would make an API call to the AS to fetch the permissions, and then cache the results locally. Caching could have an indefinite lifetime (the CID always identifies the same set of permissions), and could be shared across accounts, sessions, and client implementations. This means the cache hit rate would be very high.

### Client Metadata Document Caching

The auth scopes requested by an OAuth client when starting a session (PAR) must be a subset of those declared in their client metadata document. Auth Servers are already expected to periodically re-fetch client metadata documents, for example to check for confidential client attestation key removals (which might invalidate associated sessions). That need predates this proposal, but the expectations around cache lifetimes were not specified earlier.

The client metadata document cache lifetime should be about the same as access token lifetime, which means 15-30 minutes.
