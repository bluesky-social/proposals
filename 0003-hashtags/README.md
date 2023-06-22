# 0003 Hashtags

Social microblogging is kind of an unusual experience. On the one hand, we want to connect with people and so that’s how we build our feeds. On the other hand, we’re looking to engage with our interests, not just with people in a general sense.

When social following is working well is when people and interests intersect. Let’s say I’m really into cooking burgers and Alice posts about that. When I follow Alice, I’m interested in Alice’s recipes specifically because she knows what she’s talking about. It’s not just the interest that matters; it’s who is talking about it.

When social following fails is when people and interests diverge. A common example is when Alice has multiple hobbies that she wants to talk about. She’s more than spicy burger takes; she's also a collector of porcelain cats. It might sound neat to “get the whole person” but the reality is after 15 porcelain cat posts you end up unfollowing Alice.

The inability to speak about multiple topics can feel suffocating. People feel an obligation to their followers to respect the interest that brought them together. Alice wants to be able to share her porcelain cat hobby, but she knows it really annoys the burger lovers and that’s not fun for her.

Another key factor is discovery. Folks are looking to connect over these shared interests. If it’s impossible to discover Alice’s respected burger and/or porcelain cat posts then why is she bothering at all? Alice’s goal is to find people who share her interests.


# The problems with hashtags

It might seem obvious that we should pick up hashtags to solve these problems, but there are some problems with hashtags in previous microblogging networks that need addressing.

The first problem with hashtags is that clicking on them often isn’t worth it. Spam accounts stuff hashtags in their posts knowing they’re an easy way to gain visibility. After 3 or 4 attempts to browse content by a hashtag, most users write them off as useless and never bother again.

The second problem with hashtags is accessibility. Screenreaders often struggle with the words crammed together because the tags don’t allow spaces. They’re not particularly easy for anybody to read, and when they’re highlighted as links they make the entire post difficult to parse.

The third problem is that hashtags have never been used to solve the “plurality of interests” issue. If we want to give better tools for speaking to different audiences, then we need to invest in some tool to solve it, and we suspect hashtags could be it.


# Proposals to do hashtags well


## Visually separate the tags

The first idea – which has been received well among current users – is to separate out the hashtags from the post content as Tumblr does.

Our current thinking is that hashtags can be inline with the post so that they can still serve as a link, but people would be encouraged to put them in a separate field. This separate field can be visually minimized or hidden behind a dropdown, enabling them to do their job without making the post harder to read. The composer UI can also suggest commonly used hashtags to make their addition easier.


## Allow spaces in tags

We can’t keep breaking screen readers. Hashtags need to support spaces.

When tags are placed in a separate field this is fairly easy. To handle them inline, we propose using an underscore (‘_’) to indicate the space. The composer would convert the underscores to actual spaces before uploading.


## Use curated tag search results

Clicking on a hashtag needs to give useful content. This means that the “tag results” algorithm needs to be curating the content it provides.

This can take a variety of forms:

* We can route to custom feeds which have self-declared that they handle the given hashtag. It is likely that custom feeds will often anchor against hashtags to do topical filtering and since custom feeds are designed to curate content, they may be an ideal solution for the task.
* We can route to a user-selected search engine which are designed to give hashtag filtering. Again, the purpose of a search engine is to give useful results, and so they also might be an ideal solution for the task.
* We can lean heavily on user-list communities which are designed to maintain some level of reputation. This may be too coarse-grained to solve the problem, but if it turns out the issue with spam generally gets somehow solved by the reputation network built by lists then it may be sufficient to lean on those lists.

Curation isn’t ever something that’s immediately or permanently solved; by nature it’s a garden that requires tending. We’ll need to experiment with some of these ideas to see which ones hit right. We have some good options though.


## Mute words and hashtags

This isn’t a new idea (at all) but it would help to make this fast and easy to do. The faster you can opt out of a kind of content you don’t want, the faster you’re back to a good vibe.

One way to approach this might be for the UI to collect hashtags that have appeared in the feed recently and put them in a UI that’s easy to one-tap manage. This could include hashtags that are currently muted and thus not shown, giving you a quick way to check in on what you’re missing.

Timed mutes are also useful here. Sometimes a Big Discourse spins up and you just need a break from the topic, but you don’t want to forget about the mute and then never see it again. It’s also interesting to think about subscribable word-muting lists as we’ve done with user mute lists.


## Opt-in hashtags

The idea here is that you could create a post and say “Only people who are interested in ___ hashtag should see it. For everybody else, it should filter out by default.”

This comes with two challenges. The first is indicating that a post should follow these rules. There needs to be a clear and simple UX for this – perhaps an “Audience” field that defaults to “All”? We have to weigh that against some of the other points of configuration so that we’re not overloading people and making the app difficult to use, especially as we’re considering the reply-gating controls.

The second challenge is establishing how users opt into a given tag. If people have to spend a lot of time curating their tags then, again, people simply won’t bother. Since custom feeds are likely to focus on tags, it might be that the “opt in” is going to a feed that aggregates based on a specific tag.

It might be that opting out of hashtags is the simpler option and that we should simply invest in making muting easy, but I don’t want to discard this idea before it’s given a chance.

# Summary

There’s been a lot of discussion about whether hashtags could be replaced by something better, but nothing has really arrived that feels like a better fit. Hashtags are well-understood and simple. It seems like the challenge is just getting real value out of them.

The proposals here aren’t exactly ground-breaking. They boil down to two things: make the hashtags accessible, and then make them consistently useful. If we can do that, then I feel fairly confident they can accomplish a lot for the network.
