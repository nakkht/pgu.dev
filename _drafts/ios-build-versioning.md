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

To avoid manually increasing build number, let's automatically generate it!

On iOS, the build number is a static value within `Info.plist` file. Every time it is set, `Info.plist` changes and git counts it as a change. Not ideal, as automatically generating and setting it, will cause unstaged changes in git repository. We don't want it and we definitely can do better.

Apparently, xcode can dynamically generate final `Info.plist` file using preprocessor by setting `Preprocess Info.plist File` to `Yes`. After that we only need to set the `Info.plist Preprocessor Prefix File` for preprocessor to prefix `Info.plist`. That's where we can set our generated build number and Xcode will automatically generate it at build time.

This is how it looks in [onout](https://onout.com) project:
![prefix-preprocessor](/assets/post-image/xcode-prefix-preprocessor.png){:.width-50}

Remember to add prefix file in `.gitignore` file to prevent git tracking the changes.

Now that we have build settings configured, let's add the scripts. 
For this we need two scripts:
 - Script which generates build number
 - Script which parses build number back to git commit

According to the `CFBundleVersion` documentation can only contain decimal numbers separated by up to two periods. Furthermore, the build number must use unique with increasing numbers for each `CFBundleShortVersionString`. 
For that we can use minutes from an epoch, which is unique and increases over time, as its major build number. Then, as its minor build number, we decimalize the actual git commit.

```sh
#!/bin/sh -euo pipefail
SECONDS_FROM_EPOCH_TO_NOW=$( date "+%s" )
SECONDS_FROM_EPOCH_TO_DATE=$( date -j -f "%b %d %Y %T %Z" "January 1 $(date +"%G") 00:00:00 GMT" "+%s" )

MINUTES_SINCE_DATE=$(( $(( ${SECONDS_FROM_EPOCH_TO_NOW}-${SECONDS_FROM_EPOCH_TO_DATE} ))/60 ))
GIT_HASH_DECIMAL=$(printf "%d" 0x1"$(git rev-parse --short HEAD)")

BUNDLE_VERSION="${MINUTES_SINCE_DATE}"."${GIT_HASH_DECIMAL}"

mkdir -p "${SRCROOT}"/Plist
touch "${SRCROOT}"/Plist/Prefix

cat <<EOF > "${SRCROOT}"/Plist/Prefix
#define BUNDLE_VERSION ${BUNDLE_VERSION}
EOF
```

```sh
#!/bin/sh -euo pipefail
if [ ${#} -eq 0 ]; then
echo "No git hash supplied"
exit 0
else
BUNDLE_VERSION="${1}"
fi

POSSIBLY_DECIMAL_GIT_HASH=$( echo "${BUNDLE_VERSION}" | sed 's/[0-9][0-9]*\.\([0-9][0-9]*\)/\1/' )

ALLOWED_CHARACTERS="0123456789"
DECIMALIZED_GREP_REGEX='^['"${ALLOWED_CHARACTERS}"']['"${ALLOWED_CHARACTERS}"']*$'
DECIMAL_GIT_HASH=$( echo "${POSSIBLY_DECIMAL_GIT_HASH}" | grep "${DECIMALIZED_GREP_REGEX}" ) || {
echo "\"${BUNDLE_VERSION}\" doesnt look like a CFBundleVersion we expect. It should contain two dot-separated numbers." >&2
exit 1
}

POSSIBLY_PREFIXED_GIT_HASH=$( printf "%x" ${DECIMAL_GIT_HASH} )

PREFIXED_GIT_HASH=$( echo "${POSSIBLY_PREFIXED_GIT_HASH}" | grep '^1..*$' ) || {
echo "\"${BUNDLE_VERSION}\"'s second number, \"${POSSIBLY_DECIMAL_GIT_HASH}\", is \"${POSSIBLY_PREFIXED_GIT_HASH}\" in hex, which didnt start with a \"1\"." >&2
exit 2
}

GIT_BASH="${PREFIXED_GIT_HASH:1}"

echo "${GIT_BASH}"
```

The build number can be only 18 characters long, and we must use 1 character for our period to separate our major and minor versions, so we have 17 characters remaining for data. git-rev-parse defaults to seven characters for a short revision, so if we decimalize the largest possible seven-character hexadecimal with a leading “1” (0x1fffffff) we get 536,870,911 (9 characters long), so we have 8 characters remaining for our number of minutes.

It would be convenient to use the Unix epoch as our build number epoch, but 24,560,967 minutes (eight characters worth) have passed since then, so while we don’t NEED to define a new epoch, it makes me feel a little safer knowing there’s a buffer. It’s safe to define a new epoch for every version number since build numbers only need to be unique within a given version. However, 99,999,999 minutes (seven characters worth) is slightly less than 190 years, so we don’t need to update our epoch too often.


Finally, we write our version number and build number into $SRC_ROOT/Versions/versions.h, which we prepend to Info.plist with “Info.plist Preprocessor Prefix File” (INFOPLIST_PREFIX_HEADER build setting) as mentioned above.

Our final version number looks like


2861.410204888

Where “2861” is the number of minutes from our epoch and “410204888” is our decimalized git commit short revision. We have a script for converting this build number into a real git commit short revision.

git_hash_from_cfbundleversion.bash

Build Phase, Dependent Target

We use versions.bash in an Xcode Run Script Build Phase to update $SRCROOT/Versions/versions.h (which we ignore in git) with every build. Each build number is unique as long as they are triggered at least one minute apart. Unfortunately, Xcode preprocesses Info.plist as the first step of a target (even before your target’s first Run Script Build Phase). To work around this problem, we use an Aggregate Target with only a Run Script Build Phase that runs versions.bash.


Then we make our app’s target dependent on the Versions target.


Final Thoughts

Using Xcode’s Info.plist preprocessing and a bunch of bash, you too can have unique, incrementing, and meaningful build numbers for your apps without polluting git with a new commit just to change a version number.

Caveats:
 - Requires new version on new year
 - ensure commit is immutable (hash does not change)
 - Part of build value is not the same if re-generated