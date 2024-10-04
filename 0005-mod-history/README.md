# 0005 Ozone Moderation History

**Proposal summary**:

- allow users to view the history of reports they have submitted to a given moderation service, and what the current status of the reported subject is (review pending, takendown, etc)
- allow users to inspect the history of actions taken on their content or overall account by a given moderation service, including appeals of those actions
- these views are private (require user authentication), and get served by the relevant moderation service (Ozone instance)
- moderation services can ultimately decide whether and how to support these APIs, though are encouraged to do so in most cases


## Overview

The current Ozone moderation system has very limited mechanisms for communication between the service (moderators) and end users (reporters and subjects of moderation). The Bluesky moderation service can send emails to accounts on Bluesky PDS instances, but:

- not all mod actions result in an email (nor should they: eg, nudity labels don't need an email)
- there is no record or history of reports or appeals (eg, visible to the reporter), and how they were resolved
- Bluesky can not send emails to accounts on independent PDS hosts
- third-party mod services have no mechanism to send emails in any situation/context
- many accounts don't have verified email, or don't check email

What is needed near-term:

- no additional abuse surface (aka, does not enable spam and harassment)
- only relevant mod actions and history are exposed to end users (aka, moderation history does not become public)
- reporters and appeal-ers can see a log of their reports, and what the current status of the reported subject is
- the whole system should be optional for moderation services, at least until there is more experience and confidence in the design and policy/regulatory structure
- something that works for independent moderation services, and for user accounts on all PDS instances

Stretch goals:

- inactive accounts (takendown, suspended, etc) can view mod history in-app and appeal those account-wide actions. In particular, for users on independent PDS instances who use the Bluesky AppView.
- ability for "subject" accounts to view mod actions/history from labelers that they do not actually subscribe to
- ability to suppress access to mod history for an account, in extreme/antagonistic situations. For example, do not reveal early actions against sophisticated bot networks, or early actions/review against accounts distributing illegal abuse content, both of which could leak information about anti-abuse mechanisms
- keep track of content deletion and content updates in ozone, for some retention window, so the original content can be displayed as context in the history (both for reporters and subject accounts)
- moderation history for chat (DMs) in addition to accounts and records

This proposal is to add a set of authenticated API endpoints on Ozone instances, which allow users to query moderation history relevant to their account (mod actions on their account or content; and status of reported content). This would be exposed in-app as a "history" list of actions when looking at a labeler account. All of this history/state would be served by individual labelers (Ozone instances), not facilitated or mediated by Bluesky infrastructure.


## Sketch of Bluesky App User Experience

Logged-in users can browse to individual moderation service accounts (labelers), and there is a new "History" tab.

Within that tab, there are two sub-tabs, one for "reports" (for content the user has reported) and one for "actions" (for actions taken on the account's content).

Both of these views show a time-sorted list of "subjects", each with a brief summary of the content ("Post on 2024-01-02 at 9pm", handle and avatar of an account, etc); timestamp of when the report or action took place (this is the sort order); and what the current status of the subject is (active, needs-review, suspended, takendown, deleted, etc).

Clicking through on a subject is an additional view which shows:

- click-through link to the content or account (if available)
- summary of the content, but likely not the full content itself
- current status of the content: takedown, content deleted, account deleted, account deactivated, etc
- list of any labels applied by the current moderation service (labeler), with the full definition of the label (from declaration record)
- limited history list of mod actions on the subject, each with a summary and timestamp
- for account actions view, a button to appeal moderation action (eg, appeal label, appeal takedown)

History would be relatively truncated/redacted. All reports and appeals by the current account would display. Any major public actions would be displayed (takedown, un-takedown, suspend, label). Ozone-private content would *definitely* not display (internal moderator comments, internal moderator tags), and the individual moderator account involved would *definitely* not be revealed. All moderaiton actions are attributed to the overall service; it should not be possible (by default) for users to correlate which individual moderators are making which decisions. "Escalation" action would probably not display; "needs review" and "escalated" would probably look the same? In the future, it might be beneficial to indicate if an action taken was automated (eg, automod, auto-labeling).

Email actions could be included for the recipient account ("Email sent at this time"). In the future, would be good to have the email body text display in-line, and/or create an additional event type which allows moderators to send arbitrary text messages/comments to users. However, it is not the intent for this interface/feature to become full bi-directional "chat" or "messaging".

Metadata about subjects and reports should not be leaked. For example, an account should not be able to see if their content (or account) has been reported or not; that is private. Users should not be able to see if others have also reported a piece of content; that is private.

In a future iteration, it might be helpful to retain previous versions of content to display in the moderation history. For example, profile records are easily updated. If profile text or avatar image was updated in response to moderation action, it would be helpful for users to see the original record (which was actioned), and then any history of updates and the state of the record at that time, as part of the record. Similarly, self-deletion of content would be helpful to view as an event in history. Note that these events are *not* already stored in Ozone, and are *not* available via any existing service API.


## Design Notes

This proposal doesn't address *infrastructure moderation* in the general case. For example, a takedown at an independent Relay or AppView might not be discoverable. One pattern could be for infrastructure operators to indicate a moderation service (eg, Ozone instance) that they have delegated decision making to. This mod service could be operated by the same infra operator, or a third party, or multiple parties. Users would then view history and make appeals to that indicated Ozone instance (or multiple).

The specific case of a takedown at the PDS is not directly addressed by this proposal. Fetching moderation history would not directly work in that case, because the PDS would no longer be proxying API requests to other providers. In the future, we could special-case these API endpoints in the PDS, or change the semantics of what a "takendown" or "deactivated" account mean, at the protocol level, to allow certain kinds of proxied API calls. This would enable a user experience where an account attempts to log in, authenticates successfully, but no actions are allowed other than viewing moderation history and submitting an appeal in-app. This would move infrastructure takedown appeals in-app and out of email flows.

This proposal is not protocol-level (atproto), but also isn't specific to the Bluesky application. Record types from other Lexicons would be supported by the API/schemas. This would be an Ozone-system feature, though other moderation software could implement the same API. Note that this is distinct from labeling itself, which is a protocol-level (atproto) concept and API. Reporting and appeals APIs are currently under `com.atproto.*` but are ambiguous and arguably Ozone-specific.

There is a trade-off with this functionality between transparency and accountability to on the one hand, and enabling bad-faith actors on the other hand. Showing more details about the timeline of moderation actions provides attackers (bot farm operators, brigading teams, harassers, etc) more information which could help them evade detection or harass the moderation service directly. Making moderation actions more transparent could also make more people aware of actions they find objectionable, especially from services they don't even subscribe to, which could stir up disputes which would otherwise have gone ignored. Running a moderation service is already a difficult and stressful undertaking. We intend to collect feedback on this proposal both internally and externally, and provide services with flexibililty about whether and how to expose moderation history.

"Will prior moderation history be displayed by the Bluesky Moderation Service" is an open question. In other words, will moderation history from 2023 become visible, or only history starting from when this feature is implemented.

"What should Ozone do out of the box" is an open question. Should Ozone instances default to enabling moderation history (opt-out), or should this be an opt-in feature? Leaning towards opt-in, especially for existing services.


## Sketch of API

To avoid accidental leaking of private metadata, the plan is to add new Lexicon types with explicit metadata transformation, instead of using the existing `modEvent` types under `tools.ozone.moderation.*`.

This is an informal summary of what the Lexicons might look like. Final versions will be designed and iterated on as actual Lexicon schemas (JSON documents) in the atproto repository.

```
tools.ozone.history.defs#eventView
    subject: string uri (DID or AT-URI)
    createdAt: string datetime: server-reported timestamp (NOT createdAt from report, if different)
    event: union

tools.ozone.history.defs#eventTakedown
tools.ozone.history.defs#eventLabel
tools.ozone.history.defs#eventReport
tools.ozone.history.defs#eventEmail
tools.ozone.history.defs#eventResolve: rename of "acknowledge" and "resolve appeal". could also be "Closed" or "Reviewed"

tools.ozone.history.defs#subjectBasicView
    subject: string uri (DID or AT-URI)
    status: string: current status of the subject in the network (active, takedown, etc)
    reviewState: string: whether the subject has been reviewed and acted upon, in the context of the current account's report or appeal
    subjectProfile: app.bsky.actor.defs#profileViewBasic (or equivalent: handle, displayname, labels, avatar URL)
    labels[]: array of com.atproto.label.defs#label

tools.ozone.history.accountActions (GET): fetches all subjects from a single account with mod subjects
    params
        account: string DID (optional, normally the requester's did)
        limit: optional int
        cursor: optional string
    response:
        subjects[]: array of tools.ozone.history.defs#subjectBasicView
        cursor: optional string

tools.ozone.history.reportedSubjects (GET): fetches all subjects reported by the account
    params
        account: string DID (optional, normally the requester's did)
        limit: optional int
        cursor: optional string
    response:
        subjects[]: array of tools.ozone.history.defs#subjectBasicView
        cursor: optional string

tools.ozone.history.subjectSummary (GET)
    params
        subject: string uri (DID or AT-URI)
    response:
        subject: tools.ozone.history.defs#subjectBasicView (NOTE: could add a "detailed" view if needed? but seems like "basic" is sufficient?)

tools.ozone.history.subjectHistory (GET)
    params
        subject: string uri (DID or AT-URI)
        limit: optional int
        cursor: optional string
    response:
        events[]: array of tools.ozone.history.defs#eventView
        cursor: optional string
```
