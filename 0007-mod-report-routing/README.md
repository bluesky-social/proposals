0007: Moderation Report Routing
===============================

*Feedback and Discussion on this proposal on Github Discussions: https://github.com/bluesky-social/atproto/discussions/3581*

Stackable Moderation allows independent moderation services to receive reports and take moderation actions, such as applying labels. The overall system was [originally released](https://docs.bsky.app/blog/blueskys-moderation-architecture) in March 2024\. Since then, one of the most consistent challenges for independent moderators has been a flood of graphic and out-of-scope reports. This document proposes changes to the reporting aspect of the system, to improve "routing" of reports.

Moderation services (also called "labelers") will be able to declare which subject types and report reasons they are willing to receive. Client apps (including the Bluesky Social app) will selectively display reporting options based on services' declared reason types.

This proposal is just one update to the overall labeling and reporting architecture, not a comprehensive overhaul. It tries to be backwards compatible with existing moderation service declarations and operating instances, and minimize disruption to active services.

## Moderation Service Metadata

Moderation service declarations (`app.bsky.labeler.service` records) will include the following new fields. Client apps should only send reports to a specific service if the report subject and reason match all of the declared criteria. See bottom about un-declared fields and backwards compatibility.

**Reason Types** (`reasonTypes`, array of strings, Lexicon token references): matching the existing field on reports, indicating the reason the subject is being reported.

**Subject Types** (`subjectTypes`, array of strings, with known values): specifies which kinds of subject can be reported to the service. Services can infer this information from existing reports, based on the subject. Specified values::

- `account`: overall account (report on a DID, not a piece of content)  
- `record`: report on a specific record ("strongRef", meaning an AT-URI and a content hash)  
- `chat`: reports about DMs.

**Subject Collections** (`subjectCollections`, array of strings, as NSIDs): which record types ("collection") a service accepts reports for. Reports themselves already contain this context in the AT-URI. Values look like `app.bsky.feed.post` (they should not include a fragment like `#main`). This field both allows mod services clarify which types of records they moderation within a specific app (eg, that only posts and profiles are in-scope, not feeds or lists), and allows services declare which overall applications they support (eg, Bluesky app content vs whtwnd.com blog posts, or both).

To help with backwards compatibility, missing fields in service declaration records are interpreted as defaults similar to report routing as it currently works: all `reasonTypes`, in-scope;  `subjectCollections` as just `app.bsky` posts and profiles; and the `account` and `record` subject types.

## Report Reasons

In this update, the set of report reasons declared in `com.atproto.moderation.defs` will not change. This is a small set of reasons, and does not give services much granularity in their declarations. We expect to expand the set to around 20-30 reason codes in the near future. This iteration on reason codes will likely align with the Bluesky Community Guidelines.

In the long run, the set of report reasons should evolve beyond what the Bluesky Moderation Service makes use of. This might involve a governance process, or it might be allowing services to declare their own reasons in some way; see discussion below.

## Example Declarations

Here is an example declaration record for a service which does relatively complete moderation for the Bluesky application (except for `chat`):

```json
{
  "$type": "app.bsky.labeler.service",
  "createdAt": "2025-02-20T23:51:29.593Z",
  "policies": {
    "labelValueDefinitions": [...],
    "labelValues": [...]
  },
  "subjectTypes": ["record", "account"],
  "subjectCollections": [
    "app.bsky.feed.post",
    "app.bsky.actor.profile",
    "app.bsky.graph.list",
    "app.bsky.feed.generator",
    "app.bsky.labeler.service",
  ],
  "reasonTypes": [
    "com.atproto.moderation.defs#reasonSpam",
    "com.atproto.moderation.defs#reasonViolation",
    "com.atproto.moderation.defs#reasonMisleading",
    "com.atproto.moderation.defs#reasonSexual",
    "com.atproto.moderation.defs#reasonRude",
    "com.atproto.moderation.defs#reasonOther"
  ]
}
```

Here is a declaration for a simpler service, which only attempts to identify inauthentic accounts (bots). It only accepts reports on accounts (not records), and only for the "misleading" reason:

```json
{
  "$type": "app.bsky.labeler.service",
  "createdAt": "2025-02-20T23:51:29.593Z",
  "policies": {
    "labelValueDefinitions": [...],
    "labelValues": [...]
  },
  "subjectTypes": ["account"],
  "subjectCollections": [],
  "reasonTypes": [
    "com.atproto.moderation.defs#reasonMisleading"
  ]
}
```

## Discussion and Alternatives

One alternative architecture would be to directly re-use label definitions as report reasons. This would let independent services receive reports for customized labeling options, and could be conceptually simpler for users and moderation teams: reports would be tied to a concrete action. The current reporting APIs already use a different syntax for "reasons", so a change like this would require more coordination. Direct mapping would also not cover the other action types, such as  takedowns/suspensions, "generic filtering" actions (like a `!hide` label), or inclusion on specific moderation lists. Finding ways to align or merge the "report reason" and "label value" concepts would be an improvement in the long run.

Another alternative would be to allow custom report reasons, defined by the moderation services that a user subscribes to. For example, a service could define an "Arachnophobia" report type, and have that automatically appear in the reporting flow. We would like to support this kind of flexibility in the long run, as a complement to baseline report options, but for now the implementation complexity would draw out the process.

"Appeals" are a special type of report, with a dedicated reason type, and a separate UI flow. This proposal doesn't change how appeals work. For now, services can not opt-out of receiving appeals from client apps, even if the service does not support the relevant workflow or API endpoint. This might change in a future iteration.

Chat/DMs are a special subject type, both in that chat messages are private, and that the current chat system (`chat.bsky.*`) is centralized. To start, the `chat` `subjectType` may be limited to the Bluesky Moderation Service in the Bluesky Social client app.

The Lexicons for reporting and labeling are currently spread between the `com.atproto.moderation.*` namespace (associated with the AT Protocol) and the `app.bsky.labeler.*` namespace (associated with the Bluesky app). In the long run, we would like to migrate some of these Lexicons to the `tools.ozone.*` namespace (associated with the Ozone moderation service). While moderation is an important part of AT Protocol, and the current reporting mechanism is application-neutral, the design is opinionated and not the only way that moderation reporting could work in the protocol. For example, some moderation systems make it possible to reference multiple pieces of content in a single report, to provide context about a pattern of behavior. The current reporting flow would not support that functionality, but alternative reporting APIs could. In terms of implementing a migration like this, moving API endpoints is simpler than record schemas, but still requires planning and coordination. We are not starting that process in this iteration.

An open user interface question is whether to make it easy to submit reports to multiple moderation services at once, as is currently possible in the Bluesky Social app. The reduced friction has pros and cons: it is an easier workflow for active reports, but it also results in redundant labor by moderators. A middle option would be allowing a single configured moderation service, plus any "mandatory" services encoded in the app (eg, the Bluesky Moderation Service, for the Bluesky Social client app).
