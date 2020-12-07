---
layout: post
title: "iOS versioning: build number"
author: Paulius Gudonis
---

> **Note**: This post is heavily inspired by ["iOS versioning"](https://blog.twitch.tv/en/2016/09/20/ios-versioning-89e02f0a5146/) article written by Heath Borders at Twitch. 

While the post focuses on automated way to generate build versions for iOS explicitly, described strategy could potentially be applied to other projects which have similar semantic versioning for builds and use git. 

### The case

While developing [onout](https://onout.com), I frequently upload new builds to TestFlight to enable test groups to try out new features, confirm fixes or just provide feedback. In practice, dozens of new builds are generated before app is released publicly to the App Store.

In general, it is common to tag the version which is being uploaded to the App Store. In iOS case, it would look similar to this: `v0.1.0(1)`, `v0.1.0(2)`, etc. However, in cases, when there are frequent build uploads, tags will litter git history. Furthermore, we should be interested in tagging the release that reaches the public anyways: `v0.1.0`, `v0.2.0`

How many times have you seen git history with the following scenario repeating over and over again:
![git-build-tag](/assets/post-image/git-build-tag.png){:.width-50}
Release is tagged with specific build version, but the actual commit does not contain claimed change set. Subsequent commit is created to resolve mistake. This does not bring confidence in release management.

### Version overview

Quick recap on iOS versioning. Let's say we have the following version `0.1.0(42)`, here **0.1.0** represents `CFBundleShortVersionString` and **42** - `CFBundleVersion`
 - [CFBundleShortVersionString](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-111349) - specifies the release version number of the bundle, which identifies a released iteration of the app. E.g.: `1.0.0`, `1.4.2` - version which is normally seen by the end-user. Must be in ascending sequential order for each release.
 - [CFBundleVersion](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-102364) - specifies the build version number of the bundle, which identifies an iteration (released or unreleased) of the bundle. It must be in ascending sequential order and unique **only** per each `CFBundleShortVersionString` version. The build number not necessarily has to be a single digit. In fact, it can follow semantic versioning the same as `CFBundleShortVersionString`, thus values as such are valid: `1.0.0`, `1.0.1`, etc.

### Generating build number




The build number is a key in Info.plist. If the Info.plist is a static file, there must be a new git commit every time it changes. Thus, we can’t embed the current git commit into a static Info.plist directly because we’ll have a chicken-and-egg problem (which comes first: the Info.plist with the current git commit inside, or the current git commit with the Info.plist inside?). Xcode can dynamically generate the final Info.plist file using the C preprocessor* by setting “Preprocess Info.plist File” (INFOPLIST_PREPROCESS build setting) to “YES”. We can also use the “Info.plist Preprocessor Prefix File” (INFOPLIST_PREFIX_HEADER build setting) to prepend our git commit variable (TWBundleShortVersionString) into our Info.plist. These two settings turn our Info.plist file into a template that Xcode will use to generate the final Twitch.app/Contents/Info.plist file at build time.

According to the rules above, the build number must only contain decimal numbers separated by up to two periods, but a git commit is a hexadecimal number, so we can’t use git’s $Id$ keyword expansion within Info.plist. Also, the build number must use unique with increasing numbers for each new binary we submit to iTunes Connect. Thus, every build includes the number of minutes from an epoch — which is unique and increasing — as its major build number. Then, as its minor build number, we decimalize the git commit, and prepend a “1” onto it in case it starts with a leading “0”:

decimalize_git_hash.bash

The build number can be only 18 characters long, and we must use 1 character for our period to separate our major and minor versions, so we have 17 characters remaining for data. git-rev-parse defaults to seven characters for a short revision, so if we decimalize the largest possible seven-character hexadecimal with a leading “1” (0x1fffffff) we get 536,870,911 (9 characters long), so we have 8 characters remaining for our number of minutes.

It would be convenient to use the Unix epoch as our build number epoch, but 24,560,967 minutes (eight characters worth) have passed since then, so while we don’t NEED to define a new epoch, it makes me feel a little safer knowing there’s a buffer. It’s safe to define a new epoch for every version number since build numbers only need to be unique within a given version. However, 99,999,999 minutes (seven characters worth) is slightly less than 190 years, so we don’t need to update our epoch too often.

minutes_since_date.bash

Finally, we write our version number and build number into $SRC_ROOT/Versions/versions.h, which we prepend to Info.plist with “Info.plist Preprocessor Prefix File” (INFOPLIST_PREFIX_HEADER build setting) as mentioned above.

versions.bash

Our final version number looks like


2861.410204888

Where “2861” is the number of minutes from our epoch and “410204888” is our decimalized git commit short revision. We have a script for converting this build number into a real git commit short revision.

git_hash_from_cfbundleversion.bash

Build Phase, Dependent Target

We use versions.bash in an Xcode Run Script Build Phase to update $SRCROOT/Versions/versions.h (which we ignore in git) with every build. Each build number is unique as long as they are triggered at least one minute apart. Unfortunately, Xcode preprocesses Info.plist as the first step of a target (even before your target’s first Run Script Build Phase). To work around this problem, we use an Aggregate Target with only a Run Script Build Phase that runs versions.bash.



Then we make our app’s target dependent on the Versions target.



Final Thoughts

Using Xcode’s Info.plist preprocessing and a bunch of bash, you too can have unique, incrementing, and meaningful build numbers for your apps without polluting git with a new commit just to change a version number.

Warning: while “Preprocess Info.plist File” (INFOPLIST_PREPROCESS build setting) gives you the power of the C Preprocessor, you should still keep your files syntactically valid plist xml or Xcode won’t be able to render them in the Info.plist viewer or in the project general settings pane. Using the C Preprocessor only for variable substitution as described above is safe and syntactically valid plist xml.
In response to auibrian’s comment, I clarified how we use a dependent target to run versions.bash in the Build Phase, Dependent Target section.






Caveats:
 - Requires new version on new year
 - ensure commit is immutable (hash does not change)