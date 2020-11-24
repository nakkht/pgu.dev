---
layout: post
title: "iOS build versioning"
author: Paulius Gudonis
---

> **Note** The following post is heavily inspired by the ["iOS versioning"](https://blog.twitch.tv/en/2016/09/20/ios-versioning-89e02f0a5146/) post made by Heath Borders at Twitch.

This post focuses on generating build versions for iOS explicitly, however, described strategy could be applied to other projects which have similar semantic versioning for builds and use git. 

While developing [onout](https://onout.com), I frequently upload new builds to TestFlight to allow test groups to test new features, confirm hot fixes or just provide feedback. Potentially, there could be dozens of new builds and iterations before app is released publicly to the App Store.