# Sparkle-Itegration-macOS-

[![forthebadge](https://forthebadge.com/images/badges/built-with-love.svg)](https://forthebadge.com)
[![forthebadge](https://forthebadge.com/images/badges/made-with-swift.svg)](https://forthebadge.com)
## Description
This is an exemple to an integration of Sparkle updater engine in macOS app 

<img src="https://github.com/max21910/Sparkle-Itegration-macOS-/blob/main/Ressources/Screenshot.png?raw=true" width="732" alt="Sparkle shows familiar update window with release notes">

## How to Instal ? 
Follow the instalation guide from the sparkle website : 
* 1) add the Sparkle Frameworks

## Swift Package Manager

In your Xcode project: File › Add Packages…
Enter :
```
https://github.com/sparkle-project/Sparkle 
```
as the package repository URL
Choose the Package Options. The default options will let Xcode automatically update versions of Sparkle 2.
From Xcode’s project navigator, if you right click and show the Sparkle package in Finder, you will find Sparkle’s tools to generate and sign updates in ../artifacts/Sparkle/

## CocoaPods:

Add``` pod 'Sparkle'```
 to your Podfile.
Add or uncomment use_frameworks! in your Podfile.
## Carthage:

Add binary
``` "https://sparkle-project.org/Carthage/Sparkle.json"
``` to your Cartfile.
Run carthage update
Link the Sparkle framework to your app target:
Drag the built Carthage/Build/Mac/Sparkle.framework into your Xcode project.
Make sure the box is checked for your app’s target in the sheet’s Add to targets list.
Make sure the framework is copied into your app bundle:
Click on your project in the Project Navigator.
Click your target in the project editor.
Click on the General tab.
In Frameworks, Libraries, and Embedded Content section, change Sparkle.framework to Embed & Sign.
Sparkle’s tools to generate and sign updates are not included from Carthage and need to be grabbed from our latest release.

Sparkle only supports using a binary origin with Carthage because Carthage strips necessary code signing information when building the project from source.

## Sparkle manually:

Get the latest version of Sparkle.
Link the Sparkle framework to your app target:
Drag Sparkle.framework into your Xcode project.
Be sure to check the “Copy items into the destination group’s folder” box in the sheet that appears.
Make sure the box is checked for your app’s target in the sheet’s Add to targets list.
Make sure the framework is copied into your app bundle:
Click on your project in the Project Navigator.
Click your target in the project editor.
Click on the General tab.
In Frameworks, Libraries, and Embedded Content section, change Sparkle.framework to Embed & Sign.
In Build Settings tab set “Runpath Search Paths” to @loader_path/../Frameworks (for non-Xcode projects add the flags -Wl,-rpath,@loader_path/../Frameworks). By default, recent versions of Xcode set this to @executable_path/../Frameworks which is already sufficient for regular applications.
If you have your own process for copying/packaging your app make sure it preserves symlinks!
If you enable Library Validation, which is part of the Hardened Runtime and required for notarization, you will also need to either sign your application with an Apple Development certificate for development (requires being in Apple’s developer program), or disable library validation for Debug configurations only. Otherwise, the system may not let your application load Sparkle if you attempt to sign to run locally via an ad-hoc signature. This is not an issue for distribution when you sign your application with a Developer ID certificate.
```
* 
Sandboxed applications using Sparkle 2 require additional setup.
```
Pre-releases when available are published on GitHub. They are also available in Swift Package Manager, CocoaPods, and Carthage too by specifying the pre-release version in your project’s manifest.



## Set up a Sparkle updater object
in swift ui :
```
import SwiftUI
import Sparkle

// This view model class publishes when new updates can be checked by the user
final class CheckForUpdatesViewModel: ObservableObject {
    @Published var canCheckForUpdates = false

    init(updater: SPUUpdater) {
        updater.publisher(for: \.canCheckForUpdates)
            .assign(to: &$canCheckForUpdates)
    }
}

// This is the view for the Check for Updates menu item
// Note this intermediate view is necessary for the disabled state on the menu item to work properly before Monterey.
// See https://stackoverflow.com/questions/68553092/menu-not-updating-swiftui-bug for more info
struct CheckForUpdatesView: View {
    @ObservedObject private var checkForUpdatesViewModel: CheckForUpdatesViewModel
    private let updater: SPUUpdater
    
    init(updater: SPUUpdater) {
        self.updater = updater
        
        // Create our view model for our CheckForUpdatesView
        self.checkForUpdatesViewModel = CheckForUpdatesViewModel(updater: updater)
    }
    
    var body: some View {
        Button("Check for Updates…", action: updater.checkForUpdates)
            .disabled(!checkForUpdatesViewModel.canCheckForUpdates)
    }
}

@main
struct MyApp: App {
    private let updaterController: SPUStandardUpdaterController
    
    init() {
        // If you want to start the updater manually, pass false to startingUpdater and call .startUpdater() later
        // This is where you can also pass an updater delegate if you need one
        updaterController = SPUStandardUpdaterController(startingUpdater: true, updaterDelegate: nil, userDriverDelegate: nil)
    }
    
    var body: some Scene {
        WindowGroup {
        }
        .commands {
            CommandGroup(after: .appInfo) {
                CheckForUpdatesView(updater: updaterController.updater)
            }
        }
    }
}
```
##  security concerns
Because Sparkle is downloading executable code to your users’ systems, you must be very careful about security. To let Sparkle know that a downloaded update is not corrupted and came from you (instead of a malicious attacker), we recommend:

Serve updates over HTTPS.
Your app will not update on macOS 10.11 or later unless you comply with Apple’s App Transport Security requirements. HTTP requests will be rejected by the system.
You can get free certificates from Let’s Encrypt, and test server configuration with ssltest.
Sign the application via Apple’s Developer ID program.
Sign the published update archive with Sparkle’s EdDSA (ed25519) signature.
Updates using Installer package (.pkg) must be signed with EdDSA.
Binary Delta updates must be signed with EdDSA.
Updates of preference panes and plugins must be signed with EdDSA.
Updates to regular application bundles that are signed with Apple’s Developer ID program are strongly recommended to be signed with EdDSA for better security and fallback. Sparkle now deprecates not using EdDSA for these updates.
Please ensure your signing keys are kept safe and cannot be stolen if your web server is compromised. One way to ensure this for example is not having your signing keys accessible from the machine that is hosting your product.

EdDSA (ed25519) signatures
To prepare signing with EdDSA signatures:
```
Run ./bin/generate_keys tool (from the Sparkle distribution root). 
```
This needs to be done only once. This tool will do two things:

It will generate a private key and save it in your login Keychain on your Mac. You don’t need to do anything with it, but do keep it safe. See further notes below if you happen to lose your private key.
It will print your public key to embed into applications. Copy that key (it’s a base64-encoded string). You can run ./bin/generate_keys again to see your public key at any time.
Then add your public key to your app’s Info.plist as a SUPublicEDKey property. Note that for new projects created with Xcode 12 or later, this file may be in the Info tab under your target settings.
```
Here is an example run of ./bin/generate_keys:
```

A key has been generated and saved in your keychain. Add the `SUPublicEDKey` key to
the Info.plist of each app for which you intend to use Sparkle for distributing
updates. It should appear like this:
```
    <key>SUPublicEDKey</key>
    <string>pfIShU4dEXqPd5ObYNfDBiQWcXozk7estwzTnF9BamQ=</string>
    
    ```
You can use the -x private-key-file and -f private-key-file options to export and import the keys respectively when transferring keys to another Mac. Otherwise we recommend keeping the keys inside your Mac’s keychain. Be sure to keep them safe and not lose them (they will be erased if your keychain or system is erased).

If your keys are lost however, you can still sign new updates for Developer ID signed applications through key rotation. Note this will not work for Installer package based updates or for applications that are not code signed. In those cases you may lose the ability to sign new updates.

Please visit Migrating to EdDSA from DSA if you are still providing DSA signatures so you can learn how to stop supporting them.

## Apple code signing
If you are code-signing your application via Apple’s Developer ID program, Sparkle will ensure the new version’s author matches the old version’s. Sparkle also performs shallow (but not deep) validation for testing if the new application’s code signature is valid.

Note that embedding the Sparkle.framework into the bundle of a Developer ID application requires that you code-sign the framework and its helper tools with your Developer ID keys. Xcode should do this automatically if you create an archive via Product › Archive and Distribute App choosing Developer ID method of distribution.
You can diagnose code signing problems with codesign --deep -vvv --verify <path-to-app> for code signing validity, spctl -a -t exec -vv <path-to-app> for Gatekeeper validity, and by checking logs in the Console.app. See Apple’s Code Signing in Depth for more code signing details.
Rotating signing keys
For regular application updates, if you both code-sign your application with Apple’s Developer ID program and include a public EdDSA key for signing your update archive, Sparkle allows rotating keys by issuing a new update that changes either your Apple code signing certificate or your EdDSA keys.

We recommend rotating keys only when necessary like if you need to change your Developer ID certificate, lose access to your EdDSA private key, or need to change (Ed)DSA keys due to migrating away from DSA.

## Distributing your App
We recommend distributing your app in Xcode by creating a Product › Archive and Distribute App choosing Developer ID method of distribution. Using Xcode’s Archive Organizer will ensure Sparkle’s helper tools are code signed properly for distribution. In automated environments, this can instead be done using xcodebuild archive and xcodebuild -exportArchive.

If you distribute your app on your website as a Apple-certificate-signed disk image (DMG):

Add an /Applications symlink in your DMG, to encourage the user to copy the app out of it.
Make sure the DMG is signed with a Developer ID and use macOS 10.11.5 or later to sign it (an older OS may not sign correctly). Signed DMG archives are backwards compatible.
If you distribute your app on your website as a ZIP or a tar archive (due to app translocation):

Avoid placing your app inside another folder in your archive, because copying of the folder as a whole doesn’t remove the quarantine.
Avoid putting more than just the single app in the archive.
If your app is running from a read-only mount, you can encourage (if you want) your user to move the app into /Applications. Some frameworks, although not officially sanctioned here, exist for this purpose. Note Sparkle will not by default automatically disturb your user if an update cannot be performed.

Sparkle supports updating from ZIP archives, tarballs, disk images (DMGs), and installer packages. While you can reuse the same archive for distribution of your app on your website, we recommend serving ZIPs or tarballs (e.g. tar.xz) for updates because they are the fastest and most reliable formats for Sparkle. Disk images (DMGs) can be significantly slower to extract programmatically and sometimes be less reliable to attach/detach. Installer packages should rarely be used for distribution or updates (i.e. only for kexts, but not for installing daemons or installing system extensions).

## Publish your appcast
Sparkle uses appcasts to get information about software updates. An appcast is an RSS feed with some extra information for Sparkle’s purposes.

Add a SUFeedURL property to your Info.plist; set its value to a URL where your appcast will be hosted, e.g.
``` https://yourcompany.example.com/appcast.xml. 
```
We strongly encourage you to use HTTPS URLs for the appcast.
Note that your app bundle must have an incrementing and properly formatted CFBundleVersion key in your Info.plist. Sparkle uses this to compare and determine the latest version of your bundle.
If you update regular app bundles and you have set up EdDSA signatures, you can use a tool to generate appcasts automatically:

Build your app and compress it (e.g. in a ZIP/tar.xz/DMG archive), and put the archive in a new folder. This folder will be used to store all your future updates.
Run generate_appcast tool from Sparkle’s distribution archive specifying the path to the folder with update archives. Allow it to access the Keychain if it asks for it (it’s needed to generate signatueres in the appcast).
```
./bin/generate_appcast /path/to/your/updates_folder/
```
The tool will generate the appcast file (using filename from SUFeedURL) and also *.delta update files for faster incremental updates. Upload your archives, the delta updates, and the appcast to your server.
When generating the appcast, if an .html file exists with the same name as the archive, then it will added as the releaseNotesLink. Run generate_appcast -h for a full overview and list of supported options.

You can also create the appcast file manually (not recommended):

Make a copy of the sample appcast included in the Sparkle distribution.
Read the sample appcast to familiarize yourself with the format, then edit out all the items and add one for the new version of your app by following the instructions at Publishing an update.
6. Test Sparkle out
Use an older version of your app, or if you don’t have one yet, make one seem older by editing Info.plist and change CFBundleVersion to a lower version.
A genuine older version of the app is required to test delta updates, because Sparkle will ignore the delta update if the app doesn’t match update’s checksum.
Editing CFBundleVersion of the latest development version of the app is useful for testing the latest version of Sparkle framework.
Run the app, then quit. By default, Sparkle doesn’t ask the user’s permission for checking updates until the second launch, in order to make your users’ first-launch impression cleaner.
Run the app again. The update process should proceed as expected. Note by default, Sparkle checks for updates in the background once every 24 hours. To test automatic update checks immediately, run defaults delete my-bundle-id SULastCheckTime to clear the last update check time before launching the app. Alternatively, initiate a manual update check from the app’s menu bar.
The update process will be logged to Console.app. If anything goes wrong, you should find detailed explanation in the log.

Make sure to also keep Sparkle’s debug symbols files (.dSYM) around as they will be useful for symbolicating crash logs if something were to go wrong.

## Next steps
That’s it! You’re done! You don’t have to do any more. 
