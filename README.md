# Uni Links

A Mac OS implementation of [Avioli](https://github.com/avioli) Unilinks 
plugin for Flutter.


These links are simply web-browser-like-links that activate your app and may
contain information that you can use to load specific section of the app or
continue certain user activity from a website (or another app).

Universal Links are regular https links, thus if the app is not
installed (or setup correctly) they'll load in the browser, allowing you to
present a web-page for further action, eg. install the app.

Make sure you read both the Installation and the Usage guides, thoroughly,
especiallly for App/Universal Links (the https scheme).


## Installation

To use the plugin, add `uni_links_macos` as a
[dependency in your pubspec.yaml file](https://flutter.io/platform-plugins/).

This package requires the original [uni_links](https://pub.dev/packages/uni_links) to work.


### Permission

Mac OS require to declare links' permission in a configuration file.

The following steps are not Flutter specific, but platform specific. You might
be able to find more in-depth guides elsewhere online, by searching about 
Universal Links or Custom URL schemas.

There are two kinds of links in Mac OS: "Universal Links" and "Custom URL schemes".

  * Universal Links only work with `https` scheme and require a specified host,
  entitlements and a hosted file - `apple-app-site-association`. Check the Guide
  links below.
  * Custom URL schemes can have... any custom scheme and there is no host
  specificity, nor entitlements or a hosted file. The downside is that any app
  can claim any scheme, so make sure yours is as unique as possible,
  eg. `hst0000001` or `myIncrediblyAwesomeScheme`.

You need to declare at least one of the two.

--

For **Universal Links** you need to add or create a
`com.apple.developer.associated-domains` entitlement - either through Xcode or
by editing (or creating and adding to Xcode) `macos/Runner/DebugProfile.entitlements` and `macos/Runner/Release.entitlements`
file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- ... other keys -->
  <key>com.apple.developer.associated-domains</key>
  <array>
    <string>applinks:[YOUR_HOST]</string>
  </array>
  <!-- ... other keys -->
</dict>
</plist>
```

This allows for your app to be started from `https://YOUR_HOST` links.

For more information, read Apple's guide for
[Universal Links](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html).

--

For **Custom URL schemes** you need to declare the scheme in
`macos/Runner/Info.plist` (or through Xcode's Target Info editor,
under URL Types):

```xml
<?xml ...>
<!-- ... other tags -->
<plist>
<dict>
  <!-- ... other tags -->
  <key>CFBundleURLTypes</key>
  <array>
    <dict>
      <key>CFBundleTypeRole</key>
      <string>Editor</string>
      <key>CFBundleURLName</key>
      <string>[ANY_URL_NAME]</string>
      <key>CFBundleURLSchemes</key>
      <array>
        <string>[YOUR_SCHEME]</string>
      </array>
    </dict>
  </array>
  <!-- ... other tags -->
</dict>
</plist>
```

This allows for your app to be started from `YOUR_SCHEME://ANYTHING` links.

For a little more information, read Apple's guide for
[Inter-App Communication](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html).

I **strongly** recommend watching the [Apple WWDC 2015, session 509 - Seamless Linking to Your App](https://developer.apple.com/videos/play/wwdc2015/509/) to understand how the Universal Links work (and are setup).


## Usage

There are two ways your app will recieve a link - from cold start and brought
from the background. More on these after the example usage in
[More about app start from a link](#more-about-app-start-from-a-link).

### Initial Link (String)

Returns the link that the app was started with, if any.

```dart
import 'dart:async';
import 'dart:io';

import 'package:uni_links/uni_links.dart';
import 'package:flutter/services.dart' show PlatformException;

// ...

  Future<Null> initUniLinks() async {
    // Platform messages may fail, so we use a try/catch PlatformException.
    try {
      String initialLink = await getInitialLink();
      // Parse the link and warn the user, if it is not correct,
      // but keep in mind it could be `null`.
    } on PlatformException {
      // Handle exception by warning the user their action did not succeed
      // return?
    }
  }

// ...
```


### Initial Link (Uri)

Same as the `getInitialLink`, but converted to a `Uri`.

```dart
    // Uri parsing may fail, so we use a try/catch FormatException.
    try {
      Uri initialUri = await getInitialUri();
      // Use the uri and warn the user, if it is not correct,
      // but keep in mind it could be `null`.
    } on FormatException {
      // Handle exception by warning the user their action did not succeed
      // return?
    }
    // ... other exception handling like PlatformException
```

One can achieve the same by using `Uri.parse(initialLink)`, which is what this
convenience method does.


### On change event (String)

Usually you would check the `getInitialLink` and also listen for changes.

```dart
import 'dart:async';
import 'dart:io';

import 'package:uni_links/uni_links.dart';

// ...

  StreamSubscription _sub;

  Future<Null> initUniLinks() async {
    // ... check initialLink

    // Attach a listener to the stream
    _sub = getLinksStream().listen((String link) {
      // Parse the link and warn the user, if it is not correct
    }, onError: (err) {
      // Handle exception by warning the user their action did not succeed
    });

    // NOTE: Don't forget to call _sub.cancel() in dispose()
  }

// ...
```


### On change event (Uri)

Same as the `stream`, but transformed to emit `Uri` objects.

Usually you would check the `getInitialUri` and also listen for changes.

```dart
import 'dart:async';
import 'dart:io';

import 'package:uni_links/uni_links.dart';

// ...

  StreamSubscription _sub;

  Future<Null> initUniLinks() async {
    // ... check initialUri

    // Attach a listener to the stream
    _sub = getUriLinksStream().listen((Uri uri) {
      // Use the uri and warn the user, if it is not correct
    }, onError: (err) {
      // Handle exception by warning the user their action did not succeed
    });

    // NOTE: Don't forget to call _sub.cancel() in dispose()
  }

// ...
```

### More about app start from a link

If the app was terminated (or rather not running in the background) and the OS
must start it anew - that's a cold start. In that case, `getInitialLink` will
have the link that started your app and the Stream won't produce a link (at
that point in time).

Alternatively - if the app was running in the background and the OS must bring
it to the foreground the Stream will be the one to produce the link, while
`getInitialLink` will be either `null`, or the initial link, with which the
app was started.

Because of these two situations - you should always add a check for the
initial link (or URI) and also subscribe for a Stream of links (or URIs).


## Tools for invoking links

If you register a schema, say `unilink`, you could use these cli tools:

Assuming you've got Xcode already installed:

```sh
/usr/bin/xcrun simctl openurl booted "unilinks://host/path/subpath"
/usr/bin/xcrun simctl openurl booted "unilinks://example.com/path/portion/?uid=123&token=abc"
/usr/bin/xcrun simctl openurl booted "unilinks://example.com/?arr%5b%5d=123&arr%5b%5d=abc&addr=1%20Nowhere%20Rd&addr=Rand%20City%F0%9F%98%82"
```

If you've got `xcrun` (or `simctl`) in your path, you could invoke it directly.

The flag `booted` assumes an open simulator (you can start it via
`open -a Simulator`) with a booted device. You could target specific device by
specifying its UUID (found via `xcrun simctl list` or `flutter devices`),
replacing the `booted` flag.

### App Links or Universal Links

These types of links use `https` for schema, thus you can use above examples by
replacing `unilinks` with `https`.


## Contributing

For help on editing plugin code, view the
[documentation](https://flutter.io/platform-plugins/#edit-code).


## License

BSD 2-clause
