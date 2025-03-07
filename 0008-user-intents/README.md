0008: Data Reuse User Intents
=============================

*Some of the proposals we publish are nearly complete, and represent specification drafts. Others are sharing early work for feedback, and communicate a general direction more than the specifics. This proposal is one of the later. We are interested in aligning both technical mechanisms and policy language with other emerging efforts, and are sharing this proposal as a first step.*

This draft proposal describes how atproto accounts (eg, Bluesky users) could declare "intents" (aka, preferences) about certain categories of reuse of their public content. The mechanism and expectations are similar to `robots.txt` files on the web: a machine-readable format, which good actors are expected to abide, and does carry ethical weight, but is not legally enforceable. In particular, there is not a burden on hosting providers (eg, atproto PDS or Relay operators) to "enforce" these preferences by investigating scrapers or blocking requests for user data.

Intents would be limited to specific reuse categories, not free-form declarations or complex parameterized access control lists. The initial categories described here include:

* generative AI
* protocol bridging
* bulk datasets
* public archiving and preservation

These categories are individually defined and discussed below. Public atproto data is still public: this is not a mechanism for users to declare their accounts "private". Virtually all existing infrastructure and applications would continue to operation for all accounts in the network.

Mechanically, user intent declarations would be records in public atproto repositories. Applications (eg, the Bluesky App) would allow configuration of intents in a settings dialog. Intent preferences would be tri-state: explicitly allow, explicitly disallow, or undefined. Design options around mechanisms are described in a later section.

### **Short Example**

Suppose a Bluesky user does not want any of their public data to be used for generative AI training. They would go in to app settings, find the data reuse preferences section, and configure “Generative AI” to “disallow”. The app would create this public record in their repository:

```
{
  "$type": "org.user-intents.declaration",
  "syntheticContentGeneration": {
    "updatedAt": "2025-02-20T21:37:20.362Z",
    "allow": false
  }
}
```

The creation of this record is broadcast over the firehose, and would be included in any bulk repository exports (CAR files). The public web site (https://bsky.app) would include relevant HTML metadata (and/or HTTP headers) in responses for the user's data. Companies and research teams building AI training sets are expected to respect this intent when they see it, either when scraping websites, or doing bulk transfers using the protocol itself.


# Intent Definitions

## **Generative AI (`syntheticContentGeneration`)**

**Short Definition:** Use of account data as input to machine learning models which could be used to generate synthetic content or interactions.

This intent would apply regardless of the intended use case for resulting models, and regardless of individual or organizational status. For example, there would *not* be exceptions for scientific research, academic or educational environments, non-profits, volunteer open source efforts, etc.

Use cases which this intent would apply to:

* training of foundational large language models
* use with generative AI, at any stage
* human language translation

And which it would not apply to:

* automated classification
* vector embeddings
* recommendation systems
* automated moderation

This intent in particular would be signaled as a "beta" or "experimental". There are multiple efforts underway to standardize permission scopes and language around machine learning, and the long-term goal would be to align with whatever consensus definitions emerge. Instead of changing the definition of this intent over time, new intents with new definitions could be introduced when they are ready, and this original intent eventually being deprecated. See references at bottom to related work.

## **Archiving / Preservation (`publicAccessArchive`)**

**Short Definition:** Public access to or replay of account data as part of archiving and preservation efforts.


Public archives of network content would respect user-initiated content deletion or account deletion by default, even if this intent is to to allow. Archives which redistribute content publicly should also have policies and processes for impacted individuals to request exclusion of content from public view, even if they are not the original account holder.

Public-interest archives generally have their own policies about notable figures, government announcements, etc. This intent mechanism could be a factor in such policies, but could, under policy, be ignored in some situations.

Use cases which this intent would apply to:

* web archiving, such as Internet Archive's [Wayback Machine](https://web.archive.org) or the [New Zealand Web Archive](https://natlib.govt.nz/collections/a-z/new-zealand-web-archive/)
* digital collections, such as the [Library of Congress](https://loc.gov)

And which it would not apply to:

* protocol-native live network infrastructure (relays, mirrors, "archive" backfill services)
* private, personal web archives (such as [ArchiveBox](https://archivebox.io/))
* individual screenshots of individual posts
* legally mandated preservation efforts (eg, in specific jurisdictions)

## **Bulk Datasets (`bulkDataset`)**

**Short Definition:** Inclusion of account data in bulk "snapshot" datasets which are publicly redistributed, even if only for a fixed time period.

This intent applies to off-protocol mechanisms for archiving and re-distributing public content. It does not apply to the firehose mechanism, or to repository exports (as CAR file) in general. It is specifically about aggregating exports in to a corpus or snapshot, and then publicly redistributing that snapshot.

Regardless of bulk dataset type or purpose, any and all user intent declarations should be included in the dataset. This allows downstream users of the dataset to respect more specific expectations around data reuse. For example, any bulk dataset for any purpose should include user intents about generative AI use cases, in case a user permits inclusion in bulk datasets generally, but does not permit downstream reuse for that specific purpose.

Use cases which this intent would apply to:

* bulk collection of repository CAR exports
* captures of the relay firehose
* bulk social graph metadata
* data tables of posts
* data distributed via bittorrent
* web crawl datasets (eg, [Common Crawl](https://commoncrawl.org/))
* periodic snapshots uploaded to [archive.org](http://archive.org/) or [Hugging Face](https://huggingface.co/)

And which it would not apply to:

* restricted access research datasets
* live services which respect deletion and other protocol events from the firehose
* protocol-native live network infrastructure (relays, mirrors, "archive" backfill services)

## **Protocol Bridging (`protocolBridging`)**

**Short Definition:** Bridging account data or interactions into distinct social web protocol ecosystems.

Project-specific or protocol-specific preferences might override this overall intent.

Use cases which this intent would apply to:

* Fediverse / ActivityPub
* Matrix
* nostr

And which it would not apply to:

* alternative clients or views of network data (eg, web interfaces, comment sections on websites, VR)
* embedding posts in websites, or social card generation

# Implementation Details

This proposal makes a few opinionated design decisions:

* intentions are expressed with three states: "allow", "disallow" or "undefined". It is possible to express only a subset of the intents, or none at all
* these intents are account-wide for simplicity, not application-specific or configurable on individual pieces of content. Applications may have their own intent mechanisms.
* the available intents are a small currated set. They are not free-form, and there is not a direct extension mechanism.
* governance of these intents and their definitions is separate from AT Protocol (`com.atproto.*`) and from the Bluesky app (`app.bsky.*`)

Having intents expressed as three states makes it clearer when a user has made an explicit decision or not. Realistically, a large majority of users may stick with the default "undeclared" state. In that situation, downstream projects will need to make their own policy decisions around whether content re-use is acceptable or not. We think that user agents (eg, client apps) should not set intents automatically for users.

Some intents and preferences will make more sense at the application level, or attached to individual pieces of content. For example the Bluesky app has a "threadgate" mechanism to declare rules for interacting in a thread.

Intents live in public atproto records. Assume an independent Lexicon domain like `user-intents.org`, at least as a placeholder. Existing community efforts like `lexicons.community` have governance structures, and could be another possible home for the Lexicon.

Intent metadata might be hydrated in protocol-layer or application-specific API responses ("views"). They could be included in HTML metadata or HTTP headers, as appropriate. Protocol SDKs and tooling could include helpers to automatically enforce user intents. Eg, tools for generating snapshots of the network could be configurable to respect or ignore specific intents.

## Schema

To keep things simple, the record format would have the intent name as a field (key), and a shared simple object schema for declaring opt-in/opt-out. This would intentionally *not* be an array of intents with open-union intent types. That would be a best-practice for extensibility, but in this case we want to constrain extensibility to maximize compliance and interoperation. Putting all declarations in a single record makes it easier to fetch/lookup (eg, can use `getRecord`).

The per-intent declaration is simply an optional (or nullable) boolean field, named `allow`. It could be `true`, `false`, or undeclared.

A more complete declaration record, with comments:

```
{
  "$type": "org.user-intents.declaration",
  "updatedAt": "2025-02-20T21:37:20.362Z",
  "publicAccessArchive": {
    "allow": true, // true means "allow"
    "updatedAt": "2025-02-20T21:37:20.362Z"
  },
  "syntheticContentGeneration": {
    "allow": false, // false means "disallow"
    "updatedAt": "2025-02-20T21:37:20.362Z"
  },
  "protocolBridging": {
    // NOTE: missing 'allow' field means "undefined" preference (do not set to 'null')
    "updatedAt": "2019-02-20T22:44:20.000Z",
  }
}
```

# Discussion

Intents explicitly and intentionally do not reference or rely on legal rights, such as copyright or amoral rights. Other mechanisms built on those rights could complement user intents. For example, apps could support licensing and attribution metadata on individual pieces of media (images, video, etc).

The declarations themselves would always be considered public data, similar to identity metadata (DID document). This means intent data could be aggregated and distributed in bulk form.

One example of an intent which probably works better at the "per-post" or "per-application" level is algorithmic recommendation. Many users are willing to have their content publicly visible, but do not want it to be amplified to strangers by automated recommendation systems. Another example would be opting out of inclusion in a specific feature, like Bluesky starter packs.

Individual projects could always build their own more focused opt-in / opt-out mechanisms. Projects should probably respect the intents described here, unless there is an explicit more granular preference expressed. For example, Bridgy Fed has an explicit opt-in / opt-out mechanism, and if used could override a general "bridging" intent.

Having the generic intent mechanism be configurable to specific apps (eg, collections or NSID prefixes) might make sense. That level of granularity would increase implementation complexity, and apps could always design their own intent mechanisms. For example, simple boolean flags at the account layer are simple to hydrate in to other API "views", and efficient to store as columns in account status database roles. Lists of strings (collection names) certainly can be stored and transferred as needed, but might require separate storage tables and API requests.

# References and Related Work

Recent IETF proposals (2025):

* [Short Usage Preference Strings for Automated Processing](https://www.ietf.org/archive/id/draft-thomson-aipref-sup-00.html)
* [Vocabulary for Expressing Content Preferences for AI Training](https://www.ietf.org/archive/id/draft-vaughan-aipref-vocab-00.html)
