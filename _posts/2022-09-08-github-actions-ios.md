---
title: "GitHub Actions for iOS apps"
date: 2022-09-08T07:47:43Z
---

{{TOC}}

Let's make GitHub Actions workflow for an iOS app project. The final workflow is on [GitHub][final-project].

# Build an iOS App

`xcodebuild` is used to build iOS projects. Its format can be found on `man xcode`:

```zsh
xcodebuild [-project name.xcodeproj] -scheme schemename
           [[-destination destinationspecifier] ...]
           [-destination-timeout value] [-configuration configurationname]
           [-sdk [sdkfullpath | sdkname]] [action ...]
           [buildsetting=value ...] [-userdefault=value ...]
```

To check Xcode's version run:

```zsh
xcodebuild -version
```

Output:

```zsh
% xcodebuild -version
Xcode 14.0
Build version 14A309
```

Check that it's the needed version. If there are a few Xcode versions installed on the machine, current Xcode version can be changed by `xcode-select`:

```zsh
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

Run `xcodebuild` to build your project (it have to be run from the project's directory):

```zsh
> xcodebuild
Command line invocation:
...
** BUILD SUCCEEDED **
```

It was easy, right? By default `xcodebuild` automatically finds a project in the current directory and uses action `build` and configuration *Release*. It's interesting that while the build scheme isn't specified (as in this case), `xcodebuild` puts the build products in the `./build` folder. But otherwise it puts them in the `Derived Data` folder. to avoid accidentally committing its contents, it's a good idea to add `./build` directory to `.gitignore`.

# Create a Test Workflow

CI/CD consists of two parts. Let's tackle the first part: CI.

## Build

Create a file named `test.yml` in a `.github/workflows` directory in your repository. 
Copy the following YAML contents into the `test.yml` file:

```yaml
name: "🧪 Test Workflow"
on: [push] # on every push
jobs:
  test:
    name: "🛠 Build and 🧪 Test"
    runs-on: macos-12 # there is also an option "macos-latest"
    steps:
      - name: Check out repository code
        uses: actions/checkout@v3
      - name: 🛠 Build
        run: |
          xcodebuild \
            -project App.xcodeproj \
            -scheme App \
            -destination "platform=iOS Simulator,name=iPhone 14 Pro" \
            build-for-testing
```

For more information about GitHub Actions, see "[GitHub Docs][github-actions-docs]".

The first step of the new workflow is build the app to be able to run tests on the next step. `build-for-testing` action builds the app and associated tests. This requires specifying a scheme. To see available schemes run `xcodebuild -list`, and for available destinations `xcodebuild -showdestinations`. You can choose any other available simulator. At this step we don't specify provisioning profiles and sign keys yet, because simulator builds don't require them.

Commit and push these changes. You can see status of the workflow run on Actions tab of your repository on GitHub:

![Workflow Status](/docs/assets/build.png)

## Test

Now we are ready to add a test step and run unit and UI tests. It's very similar to build step:

```yaml
- name: 🧪 Test
  run: |
    xcodebuild \
      -project App.xcodeproj \
      -scheme App \
      -destination "platform=iOS Simulator,name=iPhone 14 Pro" \
      test-without-building
```

It will run tests using the binaries that were built at the previous step.

# Archive an iOS App

Create a new workflow yaml: `deploy.yml` with the following content:

```yaml
name: Distribute

on:
  push:
    branches: [ $default-branch ]
  pull_request:
    branches: [ $default-branch ]

env:
  DEVELOPER_DIR: /Applications/Xcode_14.0.app/Contents/Developer

jobs:
  distribute:
    name: Distribute
    runs-on: macos-12
    steps:
      - name: Checkout
        uses: actions/checkout@v3
```

It's time to think about certificates and provisioning files because they are required to distribute iOS apps. There are a few different strategies to manage iOS apps' signing: [match][match] which Fastlane recommends to use, managed/manual signing. We'll use manual signing only for app's distribution. This approach gives more control over app's signing and it's really easy to switch to any another method.

## Create Signing Files

Firstly create a distribution certificate and provisioning profile at "[Certificates, IDs & Profiles][ Developer]" section of your  Developer account page. If you already have one skip this step.

1. Create a certificate on "Certificates" section:
	1. Certificates ↣ Create a New Certificate ↣ Apple Distribution.
	2. Upload a [Certificate Signing Request][create-certificate].
	3. Download Your Certificate.
	4. Open the downloaded certificate.
	5. Find the certificate in Keychain Access under "My Certificates" tab. Right click on this certificate and choose "Export ...". Keep default selected format: `.p12`. Set a password to protect it.
2. Create an identifier:
	1. Identifiers ↣ Register a new identifier ↣ App IDs ↣ Register a new identifier (App) ↣ Register an App ID
3. Create a provisioning profile:
	1. Profiles ↣ Register a New Provisioning Profile ↣ Distribution (Ad Hoc) ↣ Generate a Provisioning Profile
	2. Download the newly generated provisioning profile.

That's it. As a result you should have two files:
1. A distribution certificate in `.p12` format.
2. A distribution provisioning profile.




---

###### ==About Fastlane?==

There is Fastlane to make CI/CD process easier but what if you don't want to have one more dependancy or want to have more control? There is `xcodebuild` utility from the Xcode Command Line Tools. Fastlane also uses it internally to build projects. It's not as hard to use `xcodebuild` directly as it looks like, but requires some knowledge.


###### ==xcconfig==

It is possible to override build options with providing an `.xcconfig`, i.e. add `⍺` to target's name and use a different icon for beta builds.


###### ==Useful xcodebuild info parameters:==
- `-showBuildTimingSummary`
           Display a report of the timings of all the commands invoked during the build.
- `-enableCodeCoverage [YES | NO]`
           Turns code coverage on or off during testing. This overrides the setting for the test action of a scheme in a workspace.

###### ==xc|pretty and xc|beautify==

`xcodebuild` prints to the terminal all commands it runs and this output is really long and nearly unreadable especially for big projects. There is a utility to fix this: [xcpretty](xcpretty).

###### ==Compile and deploy the Swift-DocC documentation?==

###### ==Use xcrun simctl to speed up tests on CI==


[final-project]: https://github.com/aleos/github-actions-ios "GitHub Actions for iOS"
[github-actions-docs]: https://docs.github.com/actions "GitHub Actions"
[match]: https://codesigning.guide "codesigning.guide concept"
[ Developer]: https://developer.apple.com/account/resources " Developer"
[create-certificate]: https://help.apple.com/developer-account/#/devbfa00fef7 "Create a certificate signing request"


[xcpretty]: https://github.com/xcpretty/xcpretty "xc|pretty"

==Link to Build settings reference:== [build-settings-reference]: https://help.apple.com/xcode/mac/current/#/itcaec37c2a6 "Build settings reference"