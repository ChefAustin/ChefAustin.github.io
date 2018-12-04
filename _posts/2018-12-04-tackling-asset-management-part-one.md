---
layout: post
title:  "Tackling Asset Management: Part One"
date:   2018-12-04 16:30:00 -0700
categories: tech
tags: asset management tracking oomnitza
---
### What problem are we trying to solve?
At the organization I work for, we tend to look at the implementation of a new system from a very specific perspective: "What is the problem we are trying to solve?"

Using this as a starting point, we can begin to easily craft answers to the timeless questions of "who, what, when, where, why, and how". So, what better place to start with assessing the needs of an asset management/tracking system than with this question.

#### Isolating Current Issues and Limitations
If I've seen it once, I've seen it a hundred times: an IT department whose best means of discerning a device's current state is by affixing to it a bright, neon sticky note scrawled with "No OS, but works" (What does "works" mean?!?). This is the stuff [MacDevOpsYVR talks](https://youtu.be/7Dk3aD6JMVk) are made of.

While this simplistic approach works for smaller orgs, it _very_ quickly breaks down at-scale. It's one thing to have a dozen machines sitting around an IT storage closet, but it is a whole new ballgame when you start to think about that solution at a company which deploys 50+ machines over the course of a week. Then when you start to consider not only new machines for incoming employees, but also outgoing, deprecated machines; the need for a strong system of organization increases exponentially.

#### What do we need?
When we kicked-off the 'requirements gathering' phase of our asset management system implementation we came up with a list of requirements for such a system, as well as a list of "should-have" and "nice-to-have" features. After all was said and done, the requirements list looked something like this:
- Full-CRUD, robust RESTful API
- Supports association of an asset to a physical, barcode asset tag
- Customizable reporting
- Location awareness
- License, warranty, and service contract tracking
- Jira, Slack, and SMTP integrations
- Child/parent asset relationships

This turned out to be quite difficult to find a solution which could fit every single one of our requirements. But, lo and behold, [Oomnitza](https://oomnitza.com) met _nearly_ every single of them (the sole exception being Jira, as they don't currently have an on-prem Jira integration; currently only cloud-hosted Jira instances are supported). In fact, one of the biggest selling points for Oomnitza was its highly-capable API. Because of the solution's API, we have been able to bolt-on robustness to the system quite easily.

When it comes to asset tracking/management, the market space for paid-solutions is not very saturated, and those which _do exist_ didn't seem to be the droids we were looking for. And to be fair, Oomnitza has some functionality we simply didn't need, and has other shortcomings (specifically, the "necessity" to spin up a box for their Oomnitza Connector which acts as, essentially, a facilitator for systems synchronization). In the end, we decided on synchronization through the API without using their abstraction layer as it allowed us to 1.) more granularly control syncs and 2.) likely respond to any bug-fixes in an expedient manner.

#### Until next time...
This series is less about Oomnitza itself, and more about what we've done with it to create a cross-functional tool which pleases the finance folks, while also nourishing the IT nerds. I want to briefly touch on the financial aspects of asset management, its importance, and how we can provide hard metrics which validates IT spend. I'm hoping to outline the surrounding automations in place, and how it impacts our IT team, as well as provide any insights we gleaned over the course of this roll-out.
