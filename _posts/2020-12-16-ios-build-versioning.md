---
layout: post
title: "iOS versioning: build number"
author: Paulius Gudonis
---

> **Note**: This post is heavily inspired by ["iOS versioning"](https://blog.twitch.tv/en/2016/09/20/ios-versioning-89e02f0a5146/) article written by Heath Borders at Twitch. 

While the post focuses on automated way to generate build versions for iOS explicitly, described strategy can potentially be applied to other projects which have similar semantic versioning for builds and use git. 

### The case

While developing [onout](https://onout.com), I frequently upload new builds to TestFlight to allow test groups to try out new features, confirm fixes or just provide feedback. In practice, dozens of new builds are generated before app is released publicly to the App Store.

In general, it is common to tag the version/build which is being uploaded to the App Store. In iOS case, the version would look similar to this: `v0.1.0(1)`, `v0.1.0(2)`, etc. However, in situations, when there are frequent build uploads, tags will litter git history and make it harder to follow. Furthermore, we are interested in tagging the actual release that reaches the public, like `v0.1.0`, `v0.2.0`, etc.

How often do you see git history with the following scenario repeating over and over again:
![git-build-tag](/assets/post/git-build-tag.png){:.width-50}
Release is tagged with specific build version, but the actual commit does not contain claimed change set. Subsequent commit is created to resolve mistake. This does not bring confidence in release management.

### Version overview

Before we look deeper into an automated solution, let us quickly recap how iOS versioning works. Let's say we have the following version `0.1.0(42)`, here **0.1.0** represents `CFBundleShortVersionString` and **42** - `CFBundleVersion`
 - [CFBundleShortVersionString](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-111349) - specifies the release version number of the bundle, which identifies a released iteration of the app. E.g.: `1.0.0`, `1.4.2` - version which is normally seen by the end-user. Must be in ascending sequential order for each release. Referred as `MARKETING_VERION` within Xcode build settings:
![marketing-version-build-setting](/assets/post/build-settings-marketing-version.jpeg){:.width-80}

 - [CFBundleVersion](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-102364) - specifies the build version number of the bundle, which identifies an iteration (released or unreleased) of the bundle. It must be in ascending sequential order and unique **only** per each `CFBundleShortVersionString` value. The build number not necessarily has to be a single digit. In fact, it can follow semantic versioning the same as `CFBundleShortVersionString`, thus values as such are valid: `1.0.0`, `1.0.1`, etc.

### Generating build number

To avoid manually increasing build number, let's auto-generate it!

On iOS, the build number is a static value within `Info.plist` file. Every time it is set, `Info.plist` changes and git counts it as a change. Not ideal, as automatically generating and setting it will yield unstaged changes in git repository. We don't want that - we can do better.

Apparently, Xcode can dynamically generate final `Info.plist` file using preprocessor by setting `Preprocess Info.plist File` to `Yes`. After that we only need to set the `Info.plist Preprocessor Prefix File` for preprocessor to prefix `Info.plist` with. That's where we can set our generated build number and Xcode will include it in `Info.plist` file.

This is how it looks in [onout](https://onout.com) project:
![prefix-preprocessor](/assets/post/xcode-prefix-preprocessor.png){:.width-50}

Remember to add prefix file (in my case `Plist/` folder) in `.gitignore` file to prevent git tracking the file.

Now that we have build settings configured, let's add the scripts. For which we need 2:
 - Script which generates build number - triggered by Xcode build system
 - Script which parses build number back to git commit - so we can find it in git history when needed

Let's start with `version.sh` script.

According to the documentation, `CFBundleVersion` can only contain decimal numbers separated by up to two periods. Furthermore, the build number must be unique with increasing numbers per each `CFBundleShortVersionString` value. 
For major version, we will use minutes from Unix epoch, which is unique and increases over time. Then, as for minor version, we decimalize the actual git commit.

> Careful with the length of `CFBundleVersion` as [it has not to exceed 18 characters](https://stackoverflow.com/questions/38330781/the-value-for-key-cfbundleversion-in-the-info-plist-file-must-be-no-longer-than) (including dot separators). 

We will use `git rev-parse --short HEAD` command to get seven characters of the commit hash and decimalize it. 

**Important note**: the commit hash can start with 0 and that will cause loss of information - leading zeros are truncated from each `CFBundleVersion` integer and will be ignored. To avoid this issue, we will prefix each commit with 1. The largest possible value when decimalizing hex value with a leading “1” `0x1fffffff` in our case is 536,870,911 (9 characters long). That results in having 18 - (9 + 1) = 8 (18 characters max, 9 for commit decimalized value and 1 for dot) characters left for major version - the time value.

For time value we can use multiple strategies based on how frequently the builds are uploaded. Using seconds based value would be overkill and would require some adjustments as more than `1607898099` seconds have already passed since Unix epoch and that's more characters than we are allowed to have. More realistic time count would be minutes/hours or even days. While using minutes, we have unique values until `18 February 2160` - equivalent to 99,999,999 minutes since Unix epoch.

`version.sh` script to generate `CFBundleVersion` value from elapsed minutes since the beginning of Unix epoch will look something like this:
```sh
#!/bin/sh -euo pipefail

# Convert elapsed seconds from Unix epoch until now to minutes
MINUTES_SINCE_EPOCH=$(( $(date "+%s")/60 ))

# Convert current git HEAD commit (7 characters) to decimal value 
GIT_HASH_DECIMAL=$(printf "%d" 0x1"$(git rev-parse --short HEAD)")

# Merge both values to a single string
BUNDLE_VERSION="${MINUTES_SINCE_EPOCH}"."${GIT_HASH_DECIMAL}"

# Prepare directory / file where the generated value will be written.
mkdir -p "${SRCROOT}"/Plist
touch "${SRCROOT}"/Plist/Prefix

# Write content to a file
cat <<EOF > "${SRCROOT}"/Plist/Prefix
#define BUNDLE_VERSION ${BUNDLE_VERSION}
EOF
```
**Beware**: `${SRCROOT}` is only accessible via Xcode project, thus if you are planning to use script outside Xcode, consider adjusting the path.

If you'd like using smaller values for time, change the division of Unix epoch time to the one that suits you:
- For hour based (major build number will change on hourly rate): `HOURS_SINCE_EPOCH=$(( $(date "+%s")/3600 ))`
- For day based (major build number will change on daily rate): `DAYS_SINCE_EPOCH=$(( $(date "+%s")/86400 ))`

And here's `pasrse_version.sh` script to convert our decimal back to hex value (it is able to accept full build number that was outputted by the previous shell script or just single decimalized commit value):
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
echo "\"${BUNDLE_VERSION}\" does not look like a CFBundleVersion we expect. It should contain two dot-separated numbers." >&2
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

To fully integrate `version.sh` script into Xcode for generating build version, we need to add few adjustments to the project. Since Xcode pre-processes `Info.plist` as one of the earliest steps of a target build, even before we are allowed to run script in build phases, that's why we have to use an `Aggregate Target` as dependency with added `Run Script Build Phase` that will execute our shell script at the right time.

To solve the issue, simply:
- Add `Aggregate` target into the project (in my case, it is named `Version`)

![aggregate-target](/assets/post/aggregate-target.png){:.width-20}

- In `Aggregate`, target under `Build Phases`, add `New Run Script phase` pointing to the script:

![aggregate-target-run-script](/assets/post/aggregate-target-run-script.png){:.width-80}

- Finally, in your product target, under `Build Phases` -> `Dependencies`, include `Aggregate` target which you created previously. This will allow to execute dependency target's scripts way before product target.

![aggregate-dependency](/assets/post/aggregate-dependency.png){:.width-80}

If you try to build the project and build succeeds, `Prefix` file should be created and hold values similar to this:

```sh
#define BUNDLE_VERSION 26798317.466604321
``` 

**Important**: `BUNDLE_VERSION` should be added into `Info.plist` and associated to `CFBundleVersion` key as such:
![bundle-version-key](/assets/post/bundle-version-key.png){:.width-80}

To parse build number back to commit, simply run:
```
parse_version.sh 26798317.466604321
```

To quickly switch directly to commit as detached head, you can use the following:
```
git switch -d $(parse_version.sh 26798317.466604321)
```

The following will parse build number and immediately switch git `HEAD` to commit represented by the number.

### Trade-offs

It is important to note that as many techniques, the following has some advantages/disadvantages:

Cons:
- Based on your configuration, major build version value might increase per each build as it is based on time. Depending on your setup, this may or may **not** be an issue for you.
- The process requires that commit is immutable - hash should never be lost/changed. This means it requires to disable ability of re-write/delete git history.

Pros:
- Removes the whole need of tags for each build, making git history less polluted.
- Since build number can essentially become unique across lifetime of the app, we don't have to rely on marketing version (`CFBundleShortVersionString`) at all.
- Less error prone: no requirement of setting build number manually and adding extra commit - less clumsy commits/tags.
- Less cognitive load on 'To-Do list' before uploading builds.
- Most importantly - consistency: whether it is a single developer or dozens, the process of creating new build will stay the same.	