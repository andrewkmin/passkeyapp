# Passkey app

This repo contains a sample passkey app which creates/uses passkeys. It it built with Expo.

The functionality is simple:
* "Sign Up" creates a new passkey and an associated Turnkey sub-organization
* "Sign In" gets a passkey signature and uses Turnkey's "Who am I?" endpoint to retrieve the sub-organization ID

Here's a video of it in action on iOS:

https://github.com/r-n-o/passkeyapp/assets/104520680/9fabf71c-d88a-4631-8bfa-14b55c72967b

## Running the app locally (iOS)

### Provisioning profile

To trigger passkey prompts you will need a way to sign the app with a proper provisioning profile. Follow the steps outlined in https://docs.expo.dev/app-signing/app-credentials/#provisioning-profiles. TL;DR: run `eas credentials`.

### Turnkey setup

Sign up for a new Turnkey organization at app.turnkey.com and create a user able to create sub-organizations (this can be done with policies):
```json
{
  "effect": "EFFECT_ALLOW",
  "consensus": "approvers.any(user, user.id == '<user id goes here>')",
  "condition": "activity.resource == 'ORGANIZATION' && activity.action == 'CREATE'"
}
```

Then generate a fresh API key with the `turnkey` CLI (`turnkey gen -k new-api-key-name --organization <org id>`). You can also visit https://r-n-o.github.io/p256-keygen/ and get values from there if it's simply for testing purposes.

Once you have org ID, public and private API key:
```sh
cp .env.template .env
```
Then insert your values in the new `.env` file

### Expo build / run

* `npx expo run:ios` will start the app using expo on a simulator
* To run with expo on your iOS device, connect the device and run `npx expo run:ios --device` (the device needs to have developer mode enabled and be connected to your Mac)
* `eas build -p ios --profile preview` will trigger a build through expo (online CI). This gets you a QR code to install on any device covered by the provisioning profile
* `npx expo prebuild --platform ios` will "prebuild" and let you build locally with xcode. Then open the project with the `PasskeyApp.xcworkspace` file to build with xcode (this is useful)
* `eas build --platform ios --local --profile preview` can be used to run a local build without xcode, and will produce a `.ipa` file. The `.ipa` can be dropped on the device through xcode: "Window" -> "Devices and Simulators", then drop the app under the "Installed Apps" section.

## `http` folder

In the HTTP folder you'll find a folder with what's hosted at https://passkeyapp.tkhqlabs.xyz. It contains a Cloudflare worker function to give apple-app-site-association the right MIME type.

To run it locally: `npx wrangler pages dev http`.

This is a necessary step 


## Troubleshooting

### com.apple.AuthenticationServices.AuthorizationError Code=1004

```
[AuthenticationServices] ASAuthorizationController credential request failed with error: Error
Domain=com.apple.AuthenticationServices.AuthorizationError Code=1004 "(null)"
```

This happens when the RPID is incorrect. I have no idea why Apple doesn't return a proper error here. The RPID should be the domain name (inverse of the bundle ID). E.g. `passkeyapp.tkhqlabs.xyz`

### This app cannot be installed because its integrity could not be verified

If you're trying to installed a `.ipa` file on a device without signing it with a provisioning profile linked to this device, that's the message you get. Make sure you select the right profile when building (this also happened to me if I do not select a profile at all: `eas build --platform ios --local` produces a by-default unsigned `.ipa` file!)

### My apple-app-site-association-file (AASA file) isn't updated? What is the app actually loading?

This is so frustrating. AASA files aren't loaded directly by apps, they're loaded from a special-purpose Apple CDN which caches these files.

Supposedly it's refreshed on install, but I have seen evidence to the contrary in my testing. Solutions:
* use `mode=developer` ([docs](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_developer_associated-domains)): this lets you bypass SSL restrictions and use a self-signed cert. But you might as well use ngrok, it'll provide a valid cert. Developer mode also causes fetching to go straight from app to server instead of using the Apple CDN. This might work for you!
* check `curl -v https://app-site-association.cdn-apple.com/a/v1/<your-domain>` to see what your app is "seeing". This is useful if you're trying to debug production-like setups where developer mode isn't an option

Another tool that can be useful: https://branch.io/resources/aasa-validator. It's a validator for AASA files. You do not need "applinks" if all you're doing in passkeys so don't trust 100% of the things it says. But it's useful to rule out basic issues like invalid JSON or MIME type. [This article](https://towardsdev.com/swift-associated-domains-universal-links-aasa-webcredentials-c66900df7b7e) is how I found this tool.

### `{"error": "Native error", "message": "The"}`

This is because `react-native-passkey` isn't loaded in your package.json. I'm not sure why but requiring it from `@turnkey/react-native-passkey-stamper` isn't sufficient. You have to require it from the react-native project itself. Is there something we can do in the `@turnkey` package to remove this requirement? Please open an issue or reach out if you know of something!
