---
layout: post
title:  "caffeinate > Caffeine"
date:   2018-01-26 17:10:00 -0700
categories: macos
---


To put it quite simply: anymore, there is no need to be installing an all-to-common tool called [Caffeine](http://lightheadsw.com/caffeine). Apple has taken the liberty of saving us the legwork, and went ahead and included this handy little binary in macOS. As far as the man page is concerned, `caffeinate` allows one to...

> prevent the system from sleeping on behalf of a utility

...which succinctly explains why `caffeinate` is a bit more useful than Caffeine insofar that its much more customizable. What if we don't care about customizability, and we just want to mimic the same `Activate For: <LengthOfTime>` functionality that Caffeine has?


#### _You got it!_

`caffeinate -t <Seconds> # Prevent idling for a specified time`

`caffeinate # Prevent idling indefinitely`

***

#### But if you're like me... you like to tinker.

For instance, say you wanted to run a test which will take 6+ minutes to complete. During this time, you plan on getting into a lengthy thumb-war with your co-worker. The only issue is that your company's IT department has ~~booby-trapped~~ secured your computer with a [Chef cookbook](https://github.com/facebook/IT-CPE/tree/master/chef/cookbooks/cpe_screensaver) to ensure that the system locks after 5 minutes of inactivity.


#### Enter: `caffeinate`
I'll employ the 'Show, don't tell' method here, and give you a glimpse what this looks like in real life:

![Alt Text]({{ "/assets/caffeinate_chef-client.gif" | absolute_url }})

#### What's going on here?
On the right terminal we call `caffeinate` and pass it another command, `chef-client`, `caffeinate` then forks a process and executes `chef-client` within it. But we want proof, right? So, then we run `ps` to grab the PID for `caffeinate`, then throw the PID into `top` to monitor its status. As you can see, when the forked `chef-client` process ends, as does  `caffeinate`.

We now have achieved a means by which we can ensure the system will not idle so long as its forked process is still running.
