# SIROS ID Wallet Wrapper App

A wrapper app for the SIROS ID EUDI wallet system.

## Features

- Allows the use of YubiKeys and similar hardware keys for Passkey storage.
- Supports Bluetooth credential presentation. (unfinished)
- Optional (proprietary) FaceTec SDK support to allow digital credentials creation from physical passports and ID cards.
- Contains an AutoFill Credential Provider Extension for Passkeys which are bound to your digital credentials.
- Will capture calls to [Config.xcconfig](Config.xcconfig)'s `BASE_DOMAINx`, if on the server side, `/.well-known/apple-app-site-association` is configured correctly. 

## Build

Note: This app needs a physical iOS device to develop with. A simulator **is not enough**, since simulators can not store Passkeys! (Which you need to unlock your wallet.)

Note: this is written with Xcode 27.0 in mind.


- Clone and open project:

```sh
git clone https://github.com/sirosfoundation/wallet-ios-wrapper.git
cd wallet-ios-wrapper
open wwWallet.xcodeproj
```

Dependencies are handled by Swift Package Manager. Xcode should resolve these automatically.


**DO NOT try to let Xcode fix signing issues automatically**. You will destroy the usage of the build configuration in [Config.xcconfig](Config.xcconfig) and later might accidentally check in your custom development configuration.

Instead create a configuration manually.

### App ID configuration

- Go here: https://developer.apple.com/account/resources/identifiers/list
- Create an app ID `com.example.mywallet` with capabilities "App Groups", "Associated Domains", "AutoFill Credential Provider", "NFC Tag Reading".
- Create another app ID `com.example.mywallet.eaf` for the extension with capabilities "App Groups", "AutoFill Credential Provider".
- Create an app group ID `group.com.example.mywallet`.
- Edit the app group assignment of your newly create app IDs so it contains your app group. 

Now add this app configuration to [Config.xcconfig](Config.xcconfig):

- There's 2 sections: 1 for development and one for production. If you're developing, you will only need to fill the section with the `[config=Debug]` postfix.

- Add the `APP_BUNDLE_ID` like `com.example.mywallet`.
- Add the `EAF_BUNDLE_ID` like `com.example.mywallet.eaf`.
- Add the `APP_GROUP` like `group.com.example.mywallet`.
- Add your `DEVELOPMENT_TEAM` ID like `A1BC234EF5`. Can be found [on this page](https://developer.apple.com/account/resources/certificates/list) in the top right corner.
- Add at least `BASE_DOMAIN1` with the location of your SIROS ID wallet installation. E.g. `mywallet.example.com`.

### Signing configuration
  
- Use the "Keychain Access" app on your Mac to create a certificate signing request like shown here: https://developer.apple.com/help/account/certificates/create-a-certificate-signing-request
- Create an "Apple Development" certificate using this CSR here: https://developer.apple.com/account/resources/certificates/list
- Download the certificate and double-click it. It will be added to your Keychain.

- Register your development device here: https://developer.apple.com/account/resources/devices/list
  You can find the necessary UDID in Xcode: Xcode -> Open Developer Tool -> Device Hub -> select your device -> Info button

- Create 2 profiles for the app and the autofill extension here: https://developer.apple.com/account/resources/profiles/list
  - Development, iOS App Development
  - For the app, select the `com.example.mywallet` ID. For the extension select the `com.example.mywallet.eaf` ID.
  - Don't use offline support, you will need a new profile every week, otherwise.
  - Select your development certificate.
  - Select your devices.
  - Give it a sensible name describing its function like e.g. "MyWallet Dev Myname" and "MyWallet AutoFill Dev Myname".
  
- Add these profile names in [Config.xcconfig](Config.xcconfig) under `APP_PROVISIONING_PROFILE_SPECIFIER` and `EAF_PROVISIONING_PROFILE_SPECIFIER`.
- Then let Xcode download the profiles: Xcode -> Settings -> Apple Accounts -> Your developer account (sign in, if you didn't, yet) -> Your Team -> Download Manual Profiles.

- Your code signing issues should be gone. Check the project configuration in the root object (called "wwWallet", currently) -> targets "wwWallet" and "AutoFillExtension" -> "Signing & Capabilities" tab. No red warnings should be seen in the "Debug" sections.
- If not, you might need to restart Xcode, so it will detect your newly created development certificate.
- If there is still a warning, check if capabilities are configured correctly. You will need to regenerate profiles and reload them through Xcode, if you messed them up earlier.
- There's also the `CODE_SIGN_IDENTITY` option, but for development purposes, the preconfigured `iPhone Developer` generic value should typically be fine. If not, you instead might want to use a full ID of the certificate instead, which you can see when you press the space bar (preview) while the downloaded cert file is selected in the Finder app. It should be named something like "Apple Development: My Team (A1BC234EF5)".

### Avoid accidental checkin of development configuration

```sh
git update-index --skip-worktree Config.xcconfig
```

### Server configuration

The server needs to contain a publicly accessible file at `/.well-known/apple-app-site-association` looking like this:

```json
{
    "applinks": {
        "details": [
            {
                "appIDs": [
                    "A1BC234EF5.com.example.mywallet",
                ],
                "components": [
                    {
                        "/": "/*",
                        "comment": "Matches any URL with a path that starts with /."
                    }
                ]
            }
        ]
    },
    "webcredentials": {
        "apps": [
            "A1BC234EF5.com.example.mywallet",
        ]
    }
}
```

This is needed for
1. the app capturing links which would otherwise send the user to the web version of the wallet.
2. Enabling Passkey login: Passkeys are always bound to domains. You will not be able to select a Passkey to unlock your wallet, if this association is not working.

This file is typically read by iOS when the app is installed. A read failure then will render the app non-functional!

To debug problems with AASA, follow this guide: https://developer.apple.com/documentation/technotes/tn3155-debugging-universal-links

- Main issues:
    - Make sure the file can be read from anywhere. It will be accessed through Apple's CDN to avoid disclosing user's IP addresses. They don't guarantee any IP address ranges.
    - JSON needs to be valid.
    - Content-Type: should be `application/json`.
    - No redirects allowed!

### Compile & Run

Make sure the "wwWallet" target is selected in the top center of Xcode and your development device next to it. Hit the run button. The app should compile and run on your device.

