# 0009: Moderation Report Granularity

> [!IMPORTANT]
> Update Oct 7, 2025: this document has been updated to reflect the final set of
report reasons that will be implemented. Please note the changes and use the
latest version of this document as the reference. See the [lexicon
PR](https://github.com/bluesky-social/atproto/pull/4262) for
additional information.

Feedback and discussion on this proposal on Github Discussions:
https://github.com/bluesky-social/atproto/discussions/3880

Building on our [previous improvements to moderation report
routing](https://github.com/bluesky-social/atproto/discussions/3581), this
proposal expands the set of reporting reasons available to users and labelers.
The goal is to provide greater granularity that benefits both users reporting
content and moderators processing those reports.

## Motivation

Since launching stackable moderation in March 2024, we've received feedback that
the existing report reason categories were too broad. Users want options that
better represent their specific concerns, and moderators need more context to
make informed decisions quickly. This expansion addresses both needs.

## User Experience

We anticipate presenting users with a multi-step flow where they choose a
category first (such as "Violence" or "Harassment"), followed by a specific
report reason within that category (such as "Threats or incitement" or "Targeted
harassment"). This will streamline the reporting process while providing
necessary detail.

## New Lexicon Namespace

As part of this effort, these new reporting reasons will be defined on the
`tools.ozone.*` namespace. This will mark a small step towards removing
tool/client specific definitions out of the lower-level `com.atproto.*` namespace.
However, due to backwards compatibility constraints, the new reasons will be
included as valid values in existing `com.atproto.*` definitions, such as the
`knownValues` definition of `com.atproto.moderation.defs#reasonType`. While this
somewhat muddies the approach, `knownValues` is purely a hint, not a type
constraint, and we feel this decision does not block further progress towards
separation of concerns down the road.

## New Reason Categories

The expanded set includes several major categories, each with specific
subcategories:

- **Violence:** Including threats, graphic content, self-harm, and animal welfare violations
- **Sexual Content:** Including unlabeled adult content and non-consensual imagery
- **Child Safety:** Various categories related to protection of minors
- **Harassment:** Including targeted harassment, hate speech, trolling, and doxxing
- **Misleading Content:** Including spam, scams, impersonation, and misinformation
- **Self-Harm or Dangerous Behaviors:** Harmful or high-risk activities
- **Other Site Rules:** Including security violations and ban evasion

## Handling Sensitive Reports

Not all report reasons will be forwarded to community labelers. Reports
concerning illegal content or those that could pose legal risks if mishandled
(such as CSAM or extremist content) will be routed exclusively to the
application’s Moderation Authority. These exceptions are clearly marked in the
lexicon with notes like "These reports will be sent to Moderation Authorities
only." At the moment, the following report reasons are marked as such:

- `tools.ozone.report.defs#reasonChildSafetyCSAM`
- `tools.ozone.report.defs#reasonChildSafetyGroom`
- `tools.ozone.report.defs#reasonChildSafetyOther`
- `tools.ozone.report.defs#reasonViolenceExtremistContent`

## Backwards Compatibility

We are not replacing or deprecating our existing report reasons with this
expanded vocabulary. Instead, we’re asking developers to adopt the new report
reasons where possible or otherwise applicable. Existing report reasons will
continue to function. Notes will be added to the lexicons providing preferred
usage as an optional, but recommended, migration path.

- `com.atproto.moderation.defs#reasonSpam` → `tools.ozone.report.defs#reasonMisleadingSpam`
- `com.atproto.moderation.defs#reasonViolation` → `tools.ozone.report.defs#reasonRuleOther`
- `com.atproto.moderation.defs#reasonMisleading` → `tools.ozone.report.defs#reasonMisleadingOther`
- `com.atproto.moderation.defs#reasonSexual` → `tools.ozone.report.defs#reasonSexualUnlabeled`
- `com.atproto.moderation.defs#reasonRude` → `tools.ozone.report.defs#reasonHarassmentOther`
- `com.atproto.moderation.defs#reasonOther` → `tools.ozone.report.defs#reasonOther`
- `com.atproto.moderation.defs#reasonAppeal` → `tools.ozone.report.defs#reasonAppeal`

Apps, including Bluesky, should send reports using the reasons supported by a
given labeler, declared via the reasonTypes property on the labeler’s
declaration record.

## Next Steps

We welcome feedback from the developer community on these changes. We encourage
labelers to update their declarations to include the new reasonTypes they're
willing to handle, using the guidance provided in the previous [Report Routing
RFC](https://github.com/bluesky-social/atproto/discussions/3581).

Published in tandem with this RFC is a Pull Request containing all changes
described here. Following any public discussion, Bluesky will merge this PR and
begin integration with Ozone. Once Ozone is ready, and community labelers have
had adequate time to update Ozone and migrate to new report reasons, we will
build and ship a new reporting flow in the Bluesky app.

Community developers can expect this process to take some time, and we
appreciate your patience and consideration.

## Full List

| Category                  | Description                                                 | Report Reason                                     | Report Reason Lexicon             | Who will receive reports? |
| ------------------------- | ----------------------------------------------------------- | ------------------------------------------------- | --------------------------------- | ------------------------- |
| Child Safety              | Content harming or endangering minors' safety and wellbeing | Child Sexual Abuse Material (CSAM)                | #reasonChildSafetyCSAM            | Bluesky only              |
|                           |                                                             | Grooming or predatory behavior                    | #reasonChildSafetyGroom           | Bluesky only              |
|                           |                                                             | Minor privacy violation                           | #reasonChildSafetyPrivacy         | Any                       |
|                           |                                                             | Minor harassment or bullying                      | #reasonChildSafetyHarassment      | Any                       |
|                           |                                                             | Other child safety                                | #reasonChildSafetyOther           | Bluesky only              |
| Violence or Physical Harm | Threats, calls for violence, or graphic content             | Animal welfare                                    | #reasonViolenceAnimal             | Any                       |
|                           |                                                             | Threats or incitement                             | #reasonViolenceThreats            | Any                       |
|                           |                                                             | Graphic violent content                           | #reasonViolenceGraphicContent     | Any                       |
|                           |                                                             | Glorification of violence                         | #reasonViolenceGlorification      | Any                       |
|                           |                                                             | Extremist content                                 | #reasonViolenceExtremistContent   | Bluesky only              |
|                           |                                                             | Human trafficking                                 | #reasonViolenceTrafficking        | Any                       |
|                           |                                                             | Other violent content                             | #reasonViolenceOther              | Any                       |
| Sexual and Adult Content  | Adult, child or animal sexual abuse                         | Adult sexual abuse content                        | #reasonSexualAbuseContent         | Any                       |
|                           |                                                             | Non-Consensual Intimate Imagery                   | #reasonSexualNCII                 | Any                       |
|                           |                                                             | Deepfake adult content                            | #reasonSexualDeepfake             | Any                       |
|                           |                                                             | Animal sexual abuse                               | #reasonSexualAnimal               | Any                       |
|                           |                                                             | Unlabelled adult content                          | #reasonSexualUnlabeled            | Any                       |
|                           |                                                             | Other sexual violence content                     | #reasonSexualOther                | Any                       |
| Harassment or Hate        | Targeted attacks, hate speech, or organized harassment      | Trolling                                          | #reasonHarassmentTroll            | Any                       |
|                           |                                                             | Targeted harassment                               | #reasonHarassmentTargeted         | Any                       |
|                           |                                                             | Hate speech                                       | #reasonHarassmentHateSpeech       | Any                       |
|                           |                                                             | Doxxing                                           | #reasonHarassmentDoxxing          | Any                       |
|                           |                                                             | Other harassing or hateful content                | #reasonHarassmentOther            | Any                       |
| Misleading                | Spam, scams, false info or impersonation                    | Fake account or bot                               | #reasonMisleadingBot              | Any                       |
|                           |                                                             | Impersonation                                     | #reasonMisleadingImpersonation    | Any                       |
|                           |                                                             | Spam                                              | #reasonMisleadingSpam             | Any                       |
|                           |                                                             | Scam                                              | #reasonMisleadingScam             | Any                       |
|                           |                                                             | False information about elections                 | #reasonMisleadingElections        | Any                       |
|                           |                                                             | Other misleading content                          | #reasonMisleadingOther            | Any                       |
| Breaking Network Rules    | Hacking, stolen content or prohibited sales                 | Hacking or system attacks                         | #reasonRuleSiteSecurity           | Any                       |
|                           |                                                             | Promoting or selling prohibited items or services | #reasonRuleProhibitedSales        | Any                       |
|                           |                                                             | Banned user returning                             | #reasonRuleBanEvasion             | Any                       |
|                           |                                                             | Other                                             | #reasonRuleOther                  | Any                       |
| Self-harm                 | Harmful or high-risk activities                             | Content promoting or depicting self-harm          | #reasonSelfHarmContent            | Any                       |
|                           |                                                             | Eating disorders                                  | #reasonSelfHarmED                 | Any                       |
|                           |                                                             | Dangerous challenges or activities                | #reasonSelfHarmStunts             | Any                       |
|                           |                                                             | Dangerous substances or drug abuse                | #reasonSelfHarmSubstances         | Any                       |
|                           |                                                             | Other dangerous content                           | #reasonSelfHarmOther              | Any                       |
| Other                     | An issue not included in these options                      | [none]                                            | #reasonOther                      | Any                       |
| Appeal                    | Issued for appeals                                          | [none]                                            | #reasonAppeal                     | Any                       |
