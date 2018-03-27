---
layout: post
title:  "Deprecation Notifier (Part Two): Expanding functionality"
date:   2018-03-26 19:50:00 -0700
categories: tech
tags: macos macops deprecation_notifier
---


A few months ago I found myself hacking away at Google's [Deprecation Notifier source code](https://github.com/google/macops/tree/master/deprecation_notifier) in hopes of finding a way to close an [open issue](https://github.com/google/macops/issues/67) relating to version comparison<sup>[1](#Footnotes)</sup>. More specifically, Deprecation Notifier would only allow an admin to utilize the tool for major updates (e.g. 10.12 to 10.13), but didn't have the logic inherent to perform comparisons between minor versions (e.g. 10.13.2 to 10.13.3). Seemed straight-forward enough, so I brought out the shovel and began to dig...

Before we get into the weeds, let's understand what Deprecation Notifier is, what it does, and what it doesn't do. From the [horse's mouth...](https://github.com/google/macops/blob/master/deprecation_notifier/README.md)


#### What is Deprecation Notifier?
> DeprecationNotifier is a small utility to nag users into upgrading their machine. It's intended as a penultimate 'stick' after the user has ignored the 'carrot'.

Quickly, it is worth noting that this utility is designed to nag the user if their current version of macOS is 'deprecated' (whereby the admin defines what constitutes as 'deprecated').

#### What Deprecation Notifier does...
> ...it will pop-up a full-screen overlay every hour (configurable) with a countdown before the user can close the window and return to work. Each time the window appears the countdown gets a little longer until it reaches a configurable threshold.

To reiterate, Deprecation Notifier is _not_ fitted (yet... working on it) to nag users about deprecated applications, and is solely checking current version of macOS.

#### Where to begin...
Let it be known, that when I first embarked on this journey I had never touched Objective-C (I didn't even have a full installation of Xcode on my box) before and that this would not have been possible without significant help from [a really great guy](https://github.com/mplewis).
cd Anywho...

With Xcode open and a gang of Yerba Mate cans at the hip, I cloned the repository and began to sift through the source looking for something that was familiar. Having started with `Localizable.strings` (because, as the documentation states, that is where "all customization should be possible") was a good choice as this led me to the `expectedVersion` string.

#### Working backwards from a single string
`expectedVersion` is the crux of the utility; in fact, it is the only variable needing to modified in order to (re-)build the application for newer macOS releases (but I recommend [a touch more](https://chefaustin.github.io/tech/2018/03/27/building-deprecation-notifier.html) personalization). By understanding that this string is the version the admin is desiring, we can then start to unpack the source and begin to understand:
- The comparison logic for current-to-desired-version
- From where the current version is queried
- How to _best_ extend the utility for minor versions

#### sdrawkcaB
After taking one-look at `Localizable.strings`, one can assume a reasonable amount about its role in the application (read: come to see it is where your nerd knobs and dork dials exist). From there, I began traversing the source for instances of the variable name.

 ![Alt Text]({{ "/assets/expectedVersion_instances.png" | absolute_url }})

Pretty easy to discern where to start poking next...

#### Whatchutalkin' bout, `DNAppDelegate.m`?
Clearly, there was a lot going on in this file, but I had not the slightest clue as to what to make of it. With much help from Matt ("really great guy" above), he was able to help me separate the wheat from the chaff in terms of what I was trying to accomplish:
```objective-c
NSString *expectedVersion = NSLocalizedString(@"expectedVersion", @"");
NSDictionary *systemVersionDictionary = [NSDictionary dictionaryWithContentsOfFile:
                                       @"/System/Library/CoreServices/SystemVersion.plist"];
NSString *systemVersion = systemVersionDictionary[@"ProductVersion"];
```
:point_up: Here is where we begin to understand a couple of the above items we needed to unpack...

(If you're even _remotely familiar_ with ObjC, I'll save you some cringeworthy greenhorn's explanation; skip to the next section...)

#### Block-by-Block, Line-by-Line
##### Line 41
```objective-c
NSString *expectedVersion = NSLocalizedString(@"expectedVersion", @"");
```
No surprises here, this simply links the value of `Localizable.strings`'s `expectedVersion` to a local variable which will be used in the (version) string comparison.

##### Line 42-44
```objective-c
NSDictionary *systemVersionDictionary = [NSDictionary dictionaryWithContentsOfFile:
                                       @"/System/Library/CoreServices/SystemVersion.plist"];
NSString *systemVersion = systemVersionDictionary[@"ProductVersion"];
```
These lines were pretty interesting to stumble upon. Again, being my first time touching Objective-C, a lot of the syntax/keywords were a bit tough to swallow. What interested me here, though, was the presence of a `/path/to/some.plist`. Any admin who works with Macs will know that plists are treasure troves for configuration; furthermore, I'd wager that when most MacAdmins come across a `*.plist`, the next thing entered into the terminal is `defaults read`:

```bash
sh-3.2# /usr/bin/defaults read /System/Library/CoreServices/SystemVersion.plist
{
    ProductBuildVersion = 17D47;
    ProductCopyright = "1983-2018 Apple Inc.";
    ProductName = "Mac OS X";
    ProductUserVisibleVersion = "10.13.3";
    ProductVersion = "10.13.3";
}
```
Ok, now I was getting somewhere... I could now reasonably surmise that Deprecation Notifier was reading a value (`ProductVersion`) from this plist (`/System/Library/CoreServices/SystemVersion.plist`) to determine the current, running version of macOS.

Now, knowing where the `expectedVersion` originated (`Localizable.strings`), and where `systemVersion` came from (`SystemVersion.plist`), the next thing to conquer was the comparison logic.


#### Boiling it down
To be quite frank, I don't really consider myself a coder/programmer/binary-caresser. So when I read code in a "foreign" language, I rely heavily on the underlying English:
```objective-c
  NSArray *systemVersionArray = [systemVersion componentsSeparatedByString:@"."];
  NSArray *expectedVersionArray = [expectedVersion componentsSeparatedByString:@"."];

  if (systemVersionArray.count < 2 || expectedVersionArray.count < 2) {
    NSLog(@"Exiting: Error, unable to properly determine system version or expected version");
    [NSApp terminate:nil];
  } else if (([expectedVersionArray[0] intValue] <= [systemVersionArray[0] intValue]) &&
             ([expectedVersionArray[1] intValue] <= [systemVersionArray[1] intValue])) {
    NSLog(@"Exiting: OS is already %@ or greater", expectedVersion);
    [NSApp terminate:nil];
}
```
Armed with some quick Google-fu, I was able to understand _enough_ about the above block to know _what_ it was doing (but not the intricacies of _how_):

- For both the `expectedVersion` and the `systemVersion` strings, convert their respective string values into an array using `.` as a delimiter for each element in the array.
- Control flow was implemented to ensure that both `expectedVersionArray` and `systemVersionArray` contained more than 1 element (if not, terminate).
- The two arrays were being compared against each other, element by element.

**This was good!** I had now gotten to where the root of the issue was: the comparison logic did not take anything after the 2nd element into consideration, and was merely comparing `systemVersionArray[0]` against `expectedVersionArray[0]` and `systemVersionArray[1]` against `expectedVersionArray[1]`. After a quick ObjC crash course (and a [couple rejected PR's](https://github.com/google/macops/pull/68) with now-glaringly-obvious issues), it was starting to make sense what needed to be done:
1. Any array containing a version number _needed_ to have 3 elements. If it wasn't like so intrinsically (as is the case with the value of `SystemVersion.plist ProductVersion` on macOS 10.13.0, which returns `10.13`) then a 3rd element (`0`) is to be injected.<sup>[2](#Footnotes)</sup>
1. The `if` statement needed to be amended to terminate unless both arrays contain 3 elements.
1. The 3rd elements (`expectedVersionArray[2]`, `systemVersionArray[2]`) needed to be appended to the `else if` block where the core comparison logic resides.

#### Wrapping it up
```objective-c
NSArray *systemVersionArray = [systemVersion componentsSeparatedByString:@"."];
  if (systemVersionArray.count == 2) {
      systemVersionArray = [systemVersionArray arrayByAddingObject:@"0"];
    }

  NSArray *expectedVersionArray = [expectedVersion componentsSeparatedByString:@"."];
  if (expectedVersionArray.count == 2) {
      expectedVersionArray = [expectedVersionArray arrayByAddingObject:@"0"];
  }

  if (systemVersionArray.count < 3 || expectedVersionArray.count < 3) {
    NSLog(@"Exiting: Error, unable to properly determine system version or expected version");
    [NSApp terminate:nil];
  } else if (([expectedVersionArray[0] intValue] <= [systemVersionArray[0] intValue]) &&
             ([expectedVersionArray[1] intValue] <= [systemVersionArray[1] intValue]) &&
             ([expectedVersionArray[2] intValue] <= [systemVersionArray[2] intValue])) {
    NSLog(@"Exiting: OS is already %@ or greater", expectedVersion);
    [NSApp terminate:nil];
  }
```
Above, we can see that the declaration of each `*VersionArray` is directly followed by an `if` statement just aching to inject a `0`, if need be. Also, an additional comparison has been dropped into the `else if` statement to check the patch versions.

Then, after running a baker's dozen tests (of course), I was able to make the most meaningful change yet:

![Alt Text]({{ "/assets/github_localizable_strings_delta.png" | absolute_url }})

#### Where to now...
This project was really great to work on for a variety of reasons: touching a new language, pairing on something with Matt, and finally getting a chance to contribute to community that has helped me in so many ways.

**So what's next?**
Well, upon coming to understand how Deprecation Notifier determines the current `systemVersion` (via a plist lookup)... and knowing that Munki stores data about its most recent run in `/Library/Preferences/ManagedInstalls.plist`... Well, you do the math.




<br/><br/><br/>
##### Footnotes
1. Full disclosure: The original GitHub issue about this was raised by [one](https://github.com/mikedodge04) of the very talented engineers on [Facebook's CPE team](https://github.com/facebook/it-cpe). I did not notice this when I started working on this project, and I'm glad I didn't. Had I of known, I likely would've chalked this up to being "out of my wheelhouse" because someone much smarter than I didn't know the answer. Lesson learned: Don't let things like this discourage you.
2. While this was not explicitly necessary for the `expectedVersionArray` (as I could've just commented the code for the admin to configure the `expectedVersion` string variable with proper values), it was the most resilient way to go about this.
