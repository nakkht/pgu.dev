---
layout: post
title: "iOS: using App Store Connect API with fastlane"
author: Paulius Gudonis
---

Back in 2018 Apple introduced App Store Connect API and with that came App Store Connect API Keys. These keys essentially are combination of:

* Issuer ID
* Key ID
* Private key

And are normally used together. By simply visiting [App Store Connect: Users and Access](https://appstoreconnect.apple.com/access/api) you can generate keys right away. However, the limit as of now for total number of keys is 50 (you can of course revoke keys at any time later and generate new ones) and furthermore, only user accounts with Admin role can generate the keys.

The same way your account has a role in App Store Connect, API keys have it too. These roles determine which API endpoints will be accessible. Further information about the roles can be found on [Apple's help page](https://help.apple.com/app-store-connect/#/deve5f9a89d7).

> **Note**: An API key's access cannot be limited to specific apps.

Since 2021 March Apple requires two-factor authentication for developer program accounts to sign in to their Apple Developer account. This brings more hassle to automate tasks such as deploying app to App Store, generating and accessing certificates/profiles, inviting people to TestFlight, adding test devices, etc. So if you did not migrate already to API keys, it is about time.

To begin, first, key has to be generated. Simply visit [App Store Connect: Users and Access](https://appstoreconnect.apple.com/access/api) and select `Keys` tab and follow instruction there to generate a key. For simple app binary upload to the App Store Connect, roles such as `App Manager` or `Developer` for API key should suffice.

After creating the key, you will have a chance to download it once. So if you loose it or it becomes compromised, you will have to revoke it and generate a new one. Downloaded file will be with `.p8` extension and content will look like this:

```
-----BEGIN PRIVATE KEY-----
MIGTAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBHkwdwIBAQQgx/elP2iGhBcUtD2B
I4AcMkR7x6YsoqwQu/1rdV4J+QmgCgYIKoZIzj0DAQehRANCAAQP/zl0hkl45Lri
eQnfJ+c+wLGGfd7ovYPoxR6ce3L3wwVKo/gmxLLVQdQEeacYW0PPPsvcKUUzxWtm
hPRRCDBE
-----END PRIVATE KEY-----
```

As it says, it is a private key, so keep it in a secure and safe place. The part between the `-----BEGIN PRIVATE KEY-----` and `-----END PRIVATE KEY-----` is a `base64` formatted `ASN.1 PKCS#8` representation of the key itself.
 
You should be able to see `Key ID` and `Issuer ID` on the same page `Keys` tab page. Since we have all the API key materials, let's integrate those into fastlane workflow to upload iOS build to App Store.

There are few ways you might want to introduce API key into your project:
* Using global environment variables set in your `.zshenv` or `.bash_profile` (could be useful if you share keys among multiple projects on your machine)
* Setting values straight in `app_store_connect_api_key` action (this will end up in repository and can be a security hazard)
* Using `.env` file in `fastlane` folder (this makes variables accessible only to particular project. It is important to add `.env` file to `.gitignore` so it does not end up in repository)

Personally, I prefer to use `.env` file as it isolates the key to a single project and is less likely to leak in some unrelated logs. Running the following fastlane command: `fastlane action app_store_connect_api_key` shows us documentation and which fastlane environment variables represent which keys: ![app_store_connect_api_key documentation](/assets/post/app-store-connect-api-key-content.png)
So far, we are interested in the following environment variables: 
* `APP_STORE_CONNECT_API_KEY_KEY_ID`
* `APP_STORE_CONNECT_API_KEY_ISSUER_ID`
* `APP_STORE_CONNECT_API_KEY_KEY`
* `APP_STORE_CONNECT_API_KEY_IS_KEY_CONTENT_BASE64`

The reasons we are interested in setting `APP_STORE_CONNECT_API_KEY_IS_KEY_CONTENT_BASE64` is due to the fact that `.p8` file content is multi-line. This will cause issues as value content in new lines will be missed. Thus, we are going to encode all `.p8` content using `base64`. By simply running `cat key.p8 | base64` in terminal we will get nice single line value which we can use. Example `base64` value for the key content shown earlier:

```javascript
LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JR1RBZ0VBTUJNR0J5cUdTTTQ5QWdFR0NDcUdTTTQ5QXdFSEJIa3dkd0lCQVFRZ3FEVkF1UEVITHNQenFhSzYKaVpsR3N1MnY1eEZzVERTTUF6eWJvSnhDbkhLZ0NnWUlLb1pJemowREFRZWhSQU5DQUFUd0t2Ym5va2l0SnNaSQpkMVRWSFhvdytCQXNMTDJ2d1NBK0lwSG50YW85V05DVjZ1dlhMNWZ3am9kUk9nQ05PNm10YnVWZ3h2QUJPMDJMCkxlc0VYaEpjCi0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0=
```

Based on values we have, we can create `.env` file in fastlane folder with the following key/value pairs:

```javascript
APP_STORE_CONNECT_API_KEY_KEY_ID="key id value"
APP_STORE_CONNECT_API_KEY_ISSUER_ID="issuer id value"
APP_STORE_CONNECT_API_KEY_KEY="single line base64 representation of .p8 file contents"
APP_STORE_CONNECT_API_KEY_IS_KEY_CONTENT_BASE64=true
```

And that is all there is to it. Now we can use `app_store_connect_api_key` directly in our custom lanes without explicitly setting any values as they will be loaded from environment variables. For example, this is how simplified upload to TestFlight could look like:

```ruby
lane :upload do
	app_store_connect_api_key
	build_app
	upload_to_testflight
end
```

As long as API keys are set in the environment, the execution of the script will require no Apple ID or 2 step verification. This sets a strong precedence for automating more tasks which require communication with App Store Connect API. Especially when more people in the team start use the scripts or other environments, such as CI, are introduced.
