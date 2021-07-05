---
layout: post
title: "iOS: automate certificate and profile creation using fastlane"
author: Paulius Gudonis
---

In the [previous post]({% post_url 2021-06-26-app-store-connect-access-key %}) I've covered how to setup App Store Connect API key and utilize it while uploading builds to the App Store using fastlane. This sets a good base for automating more tasks, such as generating certificates and profiles.

This is useful even if you are sole developer working on multiple products and especially useful if you have multiple people working on the project, simply have to renew certificates (happens every year) or just want to automate process using CI tools such as GitHub Actions, TravisCI, CircleCI, etc.

If you have tried setup in my [previous post]({% post_url 2021-06-26-app-store-connect-access-key %}), you should be able to upload iOS build straight to App Store by simply running `fastlane upload`. Given we already have App Store Connect API keys generated and working, we need to do 2 more things to automate certificate and profile creation:

* Setup private repository which will store encrypted certificates and profiles
* Setup fastlane to sync certificates and profiles between repository and development machine

First, create a private repository and create access token to repository which will enable to read/write certificates to the repository without user credentials.
> **Note**: Access token is only required if you plan to download certificates using other systems/machines, such as CI environments, where using your keychain login is not possible/is unsafe. 

Right after that, run `fastlane match init` in the project folder (the one which is going to use certificates) and follow setup questions. After finished, you should be able to see `Matchfile` in `fastlane` directory with similar content

```ruby
git_url("repository URL")
storage_mode("git")
type("appstore") # type line can be removed if custom lane will be used
```

Now we can generate certificates using command `fastlane match appstore`/`fastlane match development` or if you want to have a bit more control and predictable profile naming, introduce new fastlane lane in `Fastfile`. For example, this is how it looks in [onout](https://onout.com) project. The following lane is able to generate (if necessary) and sync certificates/profiles with your machine for both, development and distribution app variants.

```ruby
lane :sync_certs do

	match(
		git_branch: "main",
		type: "appstore",
		profile_name: "Onout AppStore",
		app_identifier: "com.neqsoft.onout"
  	)

  	match(
		git_branch: "main",
		type: "development",
		profile_name: "Onout Development",
		app_identifier: "com.neqsoft.onout"
  	)
end
```

> **Note**: If your storage mode set to `git`, you will need to create a passphrase which will be used to encrypt/decrypt files. It will be installed in your private `keychain`. It is important to remember the password and keep it somewhere safe. If for some reason you lost the passphrase, you will need to revoke old certificates/profiles and generate new ones.

If `match` command succeeded, your App Store cert repository should have content similar to this: ![fastlane match repo](/assets/post/fastlane-match-repo.png){:.width-80}
The reason why one might prefer git repository over other storage options is purely for historical purposes: it is easy to see who and when create/renewed certificates and profiles.

And that's all there's to it. No need to login to App Store developer account manually anymore to generate and download certificates and smaller possibility for someone revoke certificates by accident when more people join your team. In addition, if you need to share signing certificates with someone, simply share access to repository and decryption key which was used to encrypt files.