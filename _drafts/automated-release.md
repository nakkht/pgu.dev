---
layout: post
title: "iOS: app upload via GitHub Actions"
author: Paulius Gudonis
---

In previous posts I have covered [how to set up and use access key with App Store API]({% post_url 2021-06-26-app-store-connect-access-key %}) and later [how to automatically generate certificate and profiles using the access key]({% post_url 2021-07-05-auto-cert-and-profile-generation %}). Let's crank it up to 11 and have automated app uploads to App Store using GitHub Actions. For this to work, 2 tasks need to be completed (given that [fastlane](https://fastlane.tools) is available):
* Create fastlane lane with release flow
* Create GitHub workflow based on preferred events

For the first task, the following lane is added:

```ruby
lane :release do
	if is_ci
		setup_ci
	end
	sync_certs
	build_app
	upload_to_testflight
end
```

`build_app` and `upload_to_testflight` lanes most likely look familiar to you. It simply builds/archives app using provisioning profiles and uploads generated `.ipa` file using `upload_to_testflight` lane to the App Store. Refer to [fastlane docs](https://docs.fastlane.tools) for more information about available lanes and possible arguments. 

`sync_certs` function comes from [iOS: automate certificate and profile creation using fastlane]({% post_url 2021-07-05-auto-cert-and-profile-generation %}) post.