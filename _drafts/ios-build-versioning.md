---
layout: post
title: "iOS versioning: build number"
author: Paulius Gudonis
---

> **Note**: This post is heavily inspired by ["iOS versioning"](https://blog.twitch.tv/en/2016/09/20/ios-versioning-89e02f0a5146/) article written by Heath Borders at Twitch. 

While the post focuses on automated way to generate build versions for iOS explicitly, described strategy can potentially be applied to other projects which have similar semantic versioning for builds and use git. 

### The case

While developing [onout](https://onout.com), I frequently upload new builds to TestFlight to allow test groups to try out new features, confirm fixes or just provide feedback. In practice, dozens of new builds are generated before app is released publicly to the App Store.

In general, it is common to tag the version/build which is being uploaded to the App Store. In iOS case, it would look similar to this: `v0.1.0(1)`, `v0.1.0(2)`, etc. However, in cases, when there are frequent build uploads, tags will litter git history and harder to follow. Furthermore, we are interested in tagging the actual release that reaches the public, like	 `v0.1.0`, `v0.2.0`, etc.

How often do you see git history with the following scenario repeating over and over again:
![git-build-tag](/assets/post-image/git-build-tag.png){:.width-50}
Release is tagged with specific build version, but the actual commit does not contain claimed change set - it probably missed correct build number. Subsequent commit is created to resolve mistake. This does not bring confidence in release management.

### Version overview

Before we look deeper into an automated solution, let us quickly recap how iOS versioning works. Let's say we have the following version `0.1.0(42)`, here **0.1.0** represents `CFBundleShortVersionString` and **42** - `CFBundleVersion`
 - [CFBundleShortVersionString](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-111349) - specifies the release version number of the bundle, which identifies a released iteration of the app. E.g.: `1.0.0`, `1.4.2` - version which is normally seen by the end-user. Must be in ascending sequential order for each release. Referred as `MARKETING_VERION` within Xcode build settings:
![marketing-version-build-setting](/assets/post-image/build-settings-marketing-version.jpeg){:.width-80}

 - [CFBundleVersion](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-102364) - specifies the build version number of the bundle, which identifies an iteration (released or unreleased) of the bundle. It must be in ascending sequential order and unique **only** per each `CFBundleShortVersionString` value. The build number not necessarily has to be a single digit. In fact, it can follow semantic versioning the same as `CFBundleShortVersionString`, thus values as such are valid: `1.0.0`, `1.0.1`, etc.

### Generating build number

To avoid manually increasing build number, let's automate it!

On iOS, the build number is a static value within `Info.plist` file. Every time it is set, `Info.plist` changes and git counts it as a change. Not ideal, as automatically generating and setting it, will cause unstaged changes in git repository. We don't want that - we can do better.

Apparently, Xcode can dynamically generate final `Info.plist` file using preprocessor by setting `Preprocess Info.plist File` to `Yes`. After that we only need to set the `Info.plist Preprocessor Prefix File` for preprocessor to prefix `Info.plist`. That's where we can set our generated build number and Xcode will automatically generate it at build time.

This is how it looks in [onout](https://onout.com) project:
![prefix-preprocessor](/assets/post-image/xcode-prefix-preprocessor.png){:.width-70}

Remember to add prefix file (in my case `Plist/` folder) in `.gitignore` file to prevent git tracking the file.

Now that we have build settings configured, let's add the scripts. For this we really need two scripts:
 - Script which generates build number - triggered by Xcode build system
 - Script which parses build number back to git commit - so we can find it in git history when needed

Let's start with shell script.

According to the documentation, `CFBundleVersion` can only contain decimal numbers separated by up to two periods. Furthermore, the build number must be unique with increasing numbers per each `CFBundleShortVersionString` value. 
To counter that we can use minutes/hours/days from an epoch for major version, which is unique and increases over time. Then, as for minor build number, we decimalize the actual git commit. 
> Careful with the length of `CFBundleVersion` as [it has not to exceed 18 characters](https://stackoverflow.com/questions/38330781/the-value-for-key-cfbundleversion-in-the-info-plist-file-must-be-no-longer-than) (including dot separators). 

We will use `git rev-parse --short HEAD` command to get seven characters of the commit we are going to decimalize. Important note: the commit hash can start with 0 and that will cause loss of information when converting to decimal value. To avoid this issue, we'll prefix commit with 1. The largest possible value when decimalizing hex value with a leading “1” (0x1fffffff) in our case is 536,870,911 (9 characters long). That results in having 18 - (9 + 1) = 8 characters left for major version - time value.

For time value we can use multiple strategies based on how frequently the builds are uploaded. Using seconds based value would be overkill and would require some adjustments as more than 1607898099 seconds passed since Unix epoch and that's more characters than we are allowed to have. More realistic time count would be minutes/hours or even days. While using minutes, we have unique values until `18 February 2160` - equivalent to 99,999,999 minutes since Unix epoch.

Shell script to generate `CFBundleVersion` value from elapsed time in second since the beginning of the year:
```sh
#!/bin/sh -euo pipefail
MINUTES_SINCE_EPOCH=$(( $(date "+%s")/60 ))
GIT_HASH_DECIMAL=$(printf "%d" 0x1"$(git rev-parse --short HEAD)")

BUNDLE_VERSION="${MINUTES_SINCE_EPOCH}"."${GIT_HASH_DECIMAL}"

mkdir -p "${SRCROOT}"/Plist
touch "${SRCROOT}"/Plist/Prefix

cat <<EOF > "${SRCROOT}"/Plist/Prefix
#define BUNDLE_VERSION ${BUNDLE_VERSION}
EOF
```

The output of the latter script will be written to `"${SRCROOT}"/Plist/Prefix`. Remember to adjust paths to the ones you are using.

If you feel to using smaller values for time, just change the division of Unix epoch time to the one that suits you:
- For hour based: `HOURS_SINCE_EPOCH=$(( $(date "+%s")/3600 ))`
- For day based: `DAYS_SINCE_EPOCH=$(( $(date "+%s")/86400 ))`

Shell script to convert our decimal back to hex value (it is able to accept full build number which was outputted by the previous shell script or just single decimalized commit values):
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
echo "\"${BUNDLE_VERSION}\"'s second number, \"${POSSIBLY_DECIMAL_GIT_HASH}\", is \"${POSSIBLY_PREFIXED_GIT_HASH}\" in hex, which did not start with a \"1\"." >&2
exit 2
}

GIT_BASH="${PREFIXED_GIT_HASH:1}"

echo "${GIT_BASH}"
```

### Integrating into Xcode build process

To fully integrate script for generating build numbers, we need to add few adjustments to the project. Since Xcode pre-processes `Info.plist` as the first step of a target, we need to use an `Aggregate Target` with added `Run Script Build Phase` that runs our shell script. This is necessary as Xcode preprocesses `Info.plist` way before we can even run our script.

To solve the issue, simply:
- Add `Aggregate` target into the project
- Add `Run Script` under `Build Phases` pointing to the script
- And finally, in your product target, under `Build Phases` -> `Dependencies`, include `Aggregate` target which you created previously. This will allow to execute dependency target's scripts way before our product target.

If you try to build the project and build succeeds, `Prefix` file should hold values similar to this:

```sh
#define BUNDLE_VERSION 26798317.466604321
``` 

**Important**: `BUNDLE_VERSION` (or the name you have chosen) should be added into `Info.plist` and associated to `CFBundleVersion` key.

### Trade-offs

`git switch -d $(.scripts/parse_version.sh 18609.466604321)` can be replaces with git alias.

- Requires new version on new year based on configuration
- reduces clutter and does not require to tag every build upload
- works as a global identifier - no need for marketing version 
- marketing version can be whatever is better for the company
- reduces error - (wrong number etc.)
- ensure commit is immutable (hash does not change)
- Part of build value is not the same if re-generated - parts differ.