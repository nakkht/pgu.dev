---
layout: post
title: "iOS build versioning"
author: Paulius Gudonis
---

> **Note** The following post is heavily inspired by the ["iOS versioning"](https://blog.twitch.tv/en/2016/09/20/ios-versioning-89e02f0a5146/) post made by Heath Borders at Twitch.

This post focuses on generating build versions for iOS explicitly, however, described strategy could be applied to other projects which have similar semantic versioning for builds and use git. 

While developing [onout](https://onout.com) app, I frequently upload new builds to TestFlight. This allows test users to try out new features, confirm bug fixes or just provide feedback of overall state of the application. This results in dozens of new builds and iterations before app is released publicly to the App Store. As a consequence, creating tag for new TestFlight build becomes redundant and clutters git history. This where 