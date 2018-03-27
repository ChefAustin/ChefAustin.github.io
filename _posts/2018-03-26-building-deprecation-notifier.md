---
layout: post
title:  "Building Deprecation Notifier"
date:   2018-03-26 19:10:00 -0700
categories: tech
tags: macos
---

Let this post serve as a brief overview on how to build a customized, signed build of Google's Deprecation Notifier utility. I say brief as it assumes you've got a mechanism to deploy, and already have a means by which to trigger the application. Onward...

#### Pre-Requisites:

1. GitHub account
1. Apple Developer Account

#### Requirements:

2. Xcode IDE
2. Local clone of Google's macops repository

#### Steps:

3. First, let's clone the repository and open it in Xcode:
 >`$ git clone https://github.com/google/macops.git`
 >
 >`$ open macops/deprecation_notifier/DeprecationNotifier.xcodeproj`

3. Next, we need to modify `Localizable.strings` to declare the macOS version that is desired (`expectedVersion`) and a variety of other settings which determine Deprecation Notifier's behavior. As you'll see, all of these settings are commented with descriptions, so you can piece that together.

 ![Alt Text]({{ "/assets/depnotifier_localizable_strings.gif" | absolute_url }})

3. We've now done all that is _absolutely necessary_ for us to build and deploy Deprecation Notifier, but I want to go a step further and put that Apple Developer certificate to good use, so let's configure the project to sign our `.app` after building.

 Navigate back to the projects root in Xcode's left-hand pane, and then head to the 'General' view from the top navigation bar. Here we're going to modify the [Bundle Identifier](https://cocoacasts.com/what-are-app-ids-and-bundle-identifiers/) to match our company's [reverse domain name](https://en.wikipedia.org/wiki/Reverse_domain_name_notation), and allow Xcode to sign our build of `DeprecationNotifier.app` with the Apple Developer certificate in our Keychain:

 ![Alt Text]({{ "/assets/depnotifier_general_settings.gif" | absolute_url }})

3. Ready to build? Not so fast; let's test this sucker, first.

 Xcode allows us to build and execute the application within the IDE before we actually churn out a `.app` file by going to 'Product' > 'Run' from the toolbar (or `Command+R`). Assuming your local macOS version does not meet the `expectedVersion` set in `Localizable.strings`, you should be hit with the infamous blackout.

 ![Alt Text]({{ "/assets/depnotifier_test_run.gif" | absolute_url }})

3. Finally, we're ready to let Xcode throw together `DeprecationNotifier.app` and sign it, then we will validate the code signature and you'll have a deployable application!

 To build our production application, we navigate to 'Product' > 'Build For' > 'Running' from the toolbar (or `Command+Shift+R`). After thinking for a moment, you'll see that glorious alert...

 > "Build Succeeded"

 After you see your build was successful, we can then verify the code signature using:

 > `$ /usr/bin/codesign -dv /path/to/your/DeprecationNotifier.app`

 ![Alt Text]({{ "/assets/depnotifier_build_and_verify_signature.gif" | absolute_url }})

 #### That's it!

 Now you just need to deploy it, and determine your preferred means of invoking it; I prefer using a custom LaunchAgent, but there's no reason that you couldn't have it triggered by a run of Chef, as a postflight to a Munki run, or any whatever ace you've got up your sleeve.
