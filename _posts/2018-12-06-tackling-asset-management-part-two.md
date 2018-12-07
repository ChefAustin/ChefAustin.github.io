---
layout: post
title:  "Tackling Asset Management: Part Two"
date:   2018-12-06 18:15:00 -0700
categories: tech
tags: asset management tracking oomnitza
---
### No REST for the wicked
In order to "automate the boring stuff" a programmatic means by which to hit the API was surely needed. Fifteen minutes of dancing around GitHub proved that this was an adventure that no one had embarked upon.

Having never built a from-scratch client library, it seemed like a good opportunity to also get into the nitty-gritty of HTTP calls with native Ruby: `Net::HTTP`. While I could have just as easily employed the use of [`HTTParty`](https://github.com/jnunemaker/httparty) (because _"When you HTTParty, you must party hard!"_) or other similar gems, I had a strong desire to learn more about Ruby _without_ the convenient abstractions, and really dig into what's going on under the hood.

### In the morning, sow your seed...
Writing this was a fantastic exercise in keeping things DRY, as the first whack at the code opened my eyes quite a bit to where things could be abstracted and compacted. Of course, such a realization didn't come until _after_ pouring over the [API reference documentation](https://docs.google.com/document/d/1CYw-RP62Arqgh55cLGjrHWkN-GO_KBwUmuNkmt9edRk/view#) and writing a hundred functions for individual endpoint actions. Something about seeing the repetition of blocks across many-a-file made it hit home that I was working hard, but assuredly not smart.

### ...and in evening withhold not your hand
Taking a step back from it, it was clear a big refactor was needed: all of the code existed in one "god class", it was hard (and generally laborious) to read, and many aspects of the code made it error-prone when dealing with one endpoint but not with another. Thanks to _much_ help from a colleague, most all of this was able to be fixed before it was shipped.

That said, I didn't want to find myself in "refactor paralysis" whereby every subsequent revision of the code resulted in more new things to "fix" before publishing the gem. In fact, I was anticipating where the greater community would take the codebase that I, or any reviewer close to me, might not of even considered. While that is no excuse to ship shit-code, it was something that I wanted to allow room for.

### "Fuck it! We'll do it live!"
And with that, I give you the [Oomnitza client library](https://github.com/chefaustin/oomnitza).
