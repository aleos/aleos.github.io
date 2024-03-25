---
title: Deep Links in SwiftUI
date: 2024-03-20
publishDate: 2024-03-25
tags: [swiftui, swift, ios, dev]
draft: false
---

Deep linking enables sources like emails or websites to direct users to specific content inside an app, enhancing the user experience by providing direct access to app content. This feature is particularly beneficial for marketing and user engagement.

# What is a deep link?

A deep link is a URL that directly opens a specific screen in your app. For instance, in a news app, a specific article could open when a user taps a link in an email or a message. Deep links offer a seamless user experience and can simplify development and testing, allowing you to directly open a specific screen from a browser or terminal.

# How to implement deep links in SwiftUI

SwiftUI's `onOpenURL` modifier allows your app to handle deep links. Here's how to implement it:

```swift
import SwiftUI

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    // Handle the deep link
                }
        }
    }
}
```

This example shows the `onOpenURL` modifier in action. When a deep link is activated, the closure associated with onOpenURL is executed, allowing you to manage the deep link.

# Setting Up Your App to Recognize Deep Links

Configure your app to recognize deep links in Xcode:

1. Open your project settings and select the app target.
2. In the Info tab, add a new URL Type.
3. Enter a unique URL Scheme for your app.

For different environments, you can define unique URL Schemes. For example, use `myapp-dev` for development and `myapp-prod` for production. To do this, set a User-Defined Build Setting in the Build Settings tab and use it for the URL Schemes.[^app-url-scheme]

![APP_URL_SCHEME](/docs/assets/deep-links/app-url-scheme.png)

# Testing Your Deep Links

To test deep links in the simulator, use the `xcrun simctl openurl` command. For instance, to open the URL <myapp://article/123>, run the following command:

```shell
xcrun simctl openurl booted myapp://article/123
```

# Wrapping Up

By adding deep links to your SwiftUI app, you make it easier for users to find what they're looking for and improve their overall experience. Plus, it's a big help during app development and testing.

# Read more:

- [Deep linking and URL scheme in iOS][deep-linking-and-url-schemes-in-ios]

[deep-linking-and-url-schemes-in-ios]: https://benoitpasquier.com/deep-linking-url-scheme-ios/#:~:text=Setting%20up%20URL%20Scheme&text=In%20Xcode%2C%20under%20your%20project,often%20reuse%20the%20app%20bundle. "Deep linking and URL scheme in iOS"

[^app-url-scheme]: Solution on Stack Overflow: [How to register different URL Scheme for each app configuration (Debug, Release, ..etc)?](https://stackoverflow.com/a/65418149/191945)
