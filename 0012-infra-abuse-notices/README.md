0012: Infrastructure Abuse Notices
==================================

*Join the [Github](https://github.com/bluesky-social/atproto/discussions) discussion here.*

*This is a early proposal for discussion and feedback, particularly related to analogous systems in other ecosystems. Nothing in this document has been implemented or finalized.*

This proposal describes a simple mechanism for moderation services to send abuse notices across organizational boundaries to other infrastructure operators. For example, a large moderation service could send abuse notices to small independent PDS operators. It is as much a *protocol for inter-organizational cooperation* as it is a *technical mechanism*.


## Summary

"Abuse Notices" are voluntary messages sent from participating AT moderation services to independent AT infrastructure operators, via authenticated XRPC Procedure (HTTP POST) requests. Notices could also be sent to other services (such as AppView and Relays), or from small mod services to large operators. But for simplicity this proposal will describe the "large moderation service, small PDS operator" use case.

PDS operators would configure which moderation services ("authorities") they accept notices from, and what actions to take when receiving a valid notice: forward to admin email, forward via webhook to another system, and/or take automated moderation interventions.

Moderation services would set their own policies around which notices to send to which operators based on their own policy scopes and resources. Notices might be sent on a best-effort basis, or organizations might negotiate formal agreements out-of-band. It should be expected that some notices will be rejected or ignored by operators, who hold ultimate control. This proposed system should be flexible to multiple arrangements and evolution over time. See further discussion below.

This proposal is intentionally minimal about configuration of automated actions. There is not a complete "rules engine", just some basic built-in filters and behaviors. Operators wanting more flexibility can use the webhook mechanism to do more granular policy customization. Consistent interfaces make it possible for operators to collaborate on modular tooling.

## Background

One of the top concerns from smaller service operators getting started in the AT network is rapid response times for infrastructure moderation. By "infrastructure moderation" we mean baseline interventions that any internet infrastructure service must engage with when hosting uploaded content: abuse media, illegal content, and bulk network attacks. This is distinct from more opinionated "content" and "behavioral" moderation which takes place at the application layer. An example of infrastructure moderation would be a cloud host or transit provider cutting off service to a hosted website.

In more specific terms, folks who are considering running PDS instances, AppViews, and CDNs are nervous about child abuse images and legal liability. They do not want to host this content, but do not have resources to do proactive scanning or 24/7/365 rapid response. This issue becomes more accute as the AT network grows and starts to include more applications. The moderation service operated by Bluesky PBC currently focuses on the Bluesky App experience, but does include a limited degree of automated moderation of all content types. But the channels of communication between large moderation services (not limited to Bluesky PBC) and independent operators is limited.

As a summary of the status quo:

- PDS instances can list a contact email in their service metadata and receive free-form email reports from any party. A human PDS admin would need to manually receive, investigate, and respond to abuse notices. For a small operator with no on-call rotation, this could take days.
- PDS instances can grant admin API access to a moderation authority (eg, Ozone instance), but this grants full visibility and control over the PDS instance. It is currently designed for use within a single organization, not collaboration between organizations.
- The Bluesky appview implementation can receive moderation events from an Ozone instance via a private event API, distinct from the public label stream. This design pattern is not documented or described publicly, so other AppView developers don’t know how to replicate the pattern. And it is not designed for cross-organizational use.
- It is important that AT services have the freedom to develop service agreements and technical mechanisms appropriate to their own communities and jurisdictions. But it is unrealistic to expect every project to invent novel solutions from scratch. More organizational structures and patterns are helpful.

## Abuse Notice Pseudo-Schemas

These are not yet full lexicon schemas, just short-hand sketches.

The main interface is the endpoint on the PDS (or other services) for receiving notices:

```json
com.atproto.moderation.sendAbuseNotice (POST)
  noticeId (integer, required)
  authority (string, required, DID)
  subjects[] (array of objects, required, minLen=1, some sane maximum)
    accountDid (string, required, DID): one acconut per notice
    records[] (object array, optional): relevant records, all from same account
      uri (string, required)
      cid (string, optional): this isn't a strongRef
    blobCids[] (string array, optional): specific blobs, all from same account (but not necessarily indicated records)
  abuseType (string, required, known values): media, legal, spam, extremist, other
  comment (string, optional)
```

Inter-service auth would be required: notices can not be submitted anonymously. Response error codes could indicate if the notice had been rejected.

The PDS might have some additional admin endpoints for managing received notices:

```json
com.atproto.moderation.listAbuseNotices
  Simple approach is to just scroll notices, with "received" and "acknowledged" timestamps.
  More complex would be a "queryAbuseNotices" with many filter params.
  
com.atproto.moderation.updateAbuseNotice
  Used to update "acknowledged" timestamp on individual notices.

com.atproto.moderation.deleteAbuseNotice
```

## Service Configuration

This is a sketch of how a PDS (or other service) might be configured to process abuse notices. To re-iterate, the built-in filtering logic is very minimal; see below for discussion.

- `ABUSE_NOTICE_AUTHORITY_DIDS`: comma-separated array of DIDs for known moderation services. Only notices from these services will be accepted. Wildcard (`*`) is allowed, but not in combination with auto-action.
- `ABUSE_NOTICE_AUTO_ACTION`: boolean, whether abuse notices from any of the configured authorities will be auto-actioned (eg, takedown).
- `ABUSE_NOTICE_AUTO_TYPES`: comma-separated array of `abuseType` strings; only matching types will be auto-actioned (others will only be forwarded)
- `ABUSE_NOTICE_AUTO_NOTIFY_USER`: boolean, whether impacted accounts are automatically notified of auto-actions (eg, whether a takendown account is immediately notified)
- `ABUSE_NOTICE_FORWARD_EMAILS`: comma-separated array of admin/moderator email addresses which receive automated emails when any abuse notices are received.
- `ABUSE_NOTICE_FORWARD_WEBHOOK`: URL (or service DID+ref?) for web service which will receive an HTTP POST request containing the webhook as a body. Probably with service auth coming from the PDS service DID?

Webhooks could be forwarded to an Ozone instance (this would require a bit of new Ozone functionality).

## Discussion

Should moderation services send notices to unknown services by default? This might help keep abusive content out of the network, which would be great, but it could also have unintended negative consequences. It could create centralized dependency on a small number of moderation services. It could create unsustainable expectations and obligations for moderation services. It may be better to start by having explicit agreements between moderation services and PDS operators.

Should PDS software have a default configuration to accept and auto-action notices from known moderation services? Again, this could be very beneficial in reducing abuse, but could also have negative long-term consequences. It would remove agency and awareness from operators, and put a large degree of network power in the hands of the configured authorities. A safer default configuration would be to accept notices from some known services, but only notify admins via email.

A common concern when defending against organized abuse is that rapid feedback loops help attackers understand how they are being detected, and iterate on their methods. Services could inject small delays in sending notices to mask techniques. If a PDS is confirmed to be operated by a bad actor, a moderation service might chose not to send them notices at all.

The PDS reference implementation could support a more flexible filtering and rules system. For examples, logic like "if report comes from X and is type Y with confidence ≥=0.7, then send email to admin; If ≥0.9 then suspend entire account; but limit actions to 5 accounts a day". This can rapidly spiral in complexity and become difficult to understand and debug. And at the same time would not meet the desires of many operators. It is possible that a slightly more granular system would make sense, but the general approach here is to promote the development of a "sidecar" service which can be more flexible and accessible.

One potential use-case for abuse notifications would be broad automated image matching, resulting in automated notice. It might be beneficial to have notices indicate if they are automated or have been confirmed. It might also be helpful to have a mechanism to retract reports.

Does the PDS need to store notices locally, or only forward? Delivery failure is likely to occur, and it is more reliable to retain a copy than simply drop them.

Any time an API supports a comment field, it becomes a potential general purpose messaging channel. This raises the question of whether the system should support replies, or in other words full-featured bi-directional messaging between operators. The design requirements of such a system very quickly become "email". The PDS service declaration API already supports declaring an administrative email, and the recommendation would be to use that channel when needed.
