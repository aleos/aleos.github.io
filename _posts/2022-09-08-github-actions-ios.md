---
title: "GitHub Actions for iOS apps"
date: 2022-09-08T07:47:43Z
---

There is Fastlane to make CI/CD process easier but what if you don't want to have one more dependancy or want to have more control? There is `xcodebuild` utility from the Xcode Command Line Tools. Fastlane also uses it internally to build projects. It's not so hard to use `xcodebuild` directly as it looks like, but requires some knowledge. Let's dive into this topic and make GitHub Actions workflow for an iOS app project. The final workflow is [here](https://github.com/aleos/github-actions-ios) on GitHub.

# Build iOS app

`xcodebuild` is used to build iOS projects. To check its version run:

```zsh
xcodebuild -version
```

Output:

```zsh
> xcodebuild -version
Xcode 14.0
Build version 14A309
```

