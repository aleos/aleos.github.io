---
title: GitHub Actions for iOS apps
date: 2022-09-08T07:47:43Z
---

Let's make GitHub Actions workflow for an iOS app project. The final workflows are on [GitHub][final-project].

## Build an iOS App

`xcodebuild` is used to build iOS projects. Its format can be found on [`man xcodebuild`][man-xcodebuild]:

> ```zsh
> xcodebuild [-project name.xcodeproj] -scheme schemename
>            [[-destination destinationspecifier] ...]
>            [-destination-timeout value] [-configuration configurationname]
>            [-sdk [sdkfullpath | sdkname]] [action ...]
>            [buildsetting=value ...] [-userdefault=value ...]
> ```

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

Check that it's the needed version. If there are a few Xcode versions installed on the machine, current Xcode version can be changed by [`xcode-select`][man-xcode-select]:

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

It was easy, right? By default `xcodebuild` automatically finds a project in the current directory and uses action `build` and configuration _Release_. It's interesting that while the build scheme isn't specified (as in this case), `xcodebuild` puts the build products in the `./build` folder. But otherwise it puts them in the `Derived Data` folder. to avoid accidentally committing its contents, it's a good idea to add `./build` directory to `.gitignore`.

## Create a Test Workflow

CI/CD consists of two parts. Let's tackle the first part: CI.

### Build

Create a file named `test.yml` in a `.github/workflows` directory in your repository.
Copy the following YAML contents into the `test.yml` file:

```yaml
name: "🧪 Test Workflow"
on: push
jobs:
  test:
    name: "🛠 Build and 🧪 Test"
    runs-on: macos-12
    steps:
      - name: Check out repository code
        uses: actions/checkout@v3
      - name: 🛠 Build
        run: |
          xcodebuild \
            -project App.xcodeproj \
            -scheme App \
            -destination 'platform=iOS Simulator,name=iPhone 14 Pro' \
            build-for-testing
```

For more information about GitHub Actions, see "[GitHub Docs][github-actions-docs]".

The first step of the new workflow is build the app to be able to run tests on the next step. `build-for-testing` action builds the app and associated tests. This requires specifying a scheme. To see available schemes run `xcodebuild -list`, and for available destinations `xcodebuild -showdestinations`. You can choose any other available simulator. At this step we don't specify provisioning profiles and sign keys yet, because simulator builds don't require them.

Commit and push these changes. You can see status of the workflow run on Actions tab of your repository on GitHub:

![Workflow Status](/docs/assets/github-actions-ios/build.png)

### Test

Now we are ready to add a test step and run unit and UI tests. It's very similar to build step:

```yaml
- name: 🧪 Test
  run: |
    xcodebuild \
      -project App.xcodeproj \
      -scheme App \
      -destination 'platform=iOS Simulator,name=iPhone 14 Pro' \
      test-without-building
```

It will run tests using the binaries that were built at the previous step.

## Archive an iOS App

It's time for the second part: CD. Create a new workflow file `distribute.yml` with the following content:

```yaml
name: 🚀 Distribute Workflow
on: push
jobs:
  distribute:
    name: 🚀 Distribute
    runs-on: macos-12
    steps:
      - name: Checkout
        uses: actions/checkout@v3
```

It's time to think about certificates and provisioning files because they are required to distribute iOS apps. There are a few different strategies to manage iOS apps' signing: [match][match] which Fastlane recommends to use, managed/manual signing. We'll use manual signing only for app's distribution. This approach gives more control over app's signing and it's really easy to switch to any another method.

### Create Signing Files

Firstly create a distribution certificate and provisioning profile at "[Certificates, IDs & Profiles][ Developer]" section of your  Developer account page. If you already have one skip this step.

1. Create a signing certificate on the "Certificates" section:
   1. Certificates ↣ Create a New Certificate ↣ Apple Distribution.
   2. Upload a [Certificate Signing Request][create-certificate].
   3. Download Your Certificate.
   4. Open the downloaded certificate: it'll be added to Keychain Access.
   5. Find the certificate in Keychain Access under "My Certificates" tab. Right click on this certificate and choose "Export ...". Keep default selected format: `.p12`. Set a password to protect it.
2. Create an identifier:
   1. Identifiers ↣ Register a new identifier ↣ App IDs ↣ Register a new identifier (App) ↣ Register an App ID
3. Create a provisioning profile:
   1. Profiles ↣ Register a New Provisioning Profile ↣ Distribution (Ad Hoc) ↣ Generate a Provisioning Profile
   2. Download the newly generated provisioning profile.
4. Set up the project:
   1. Open the downloaded provisioning profile or download via Xcode ↣ Preferences… ↣ Accounts ↣ Download Manual Profiles
   2. In the Xcode in the section Signing & Capabilities of your target change filter from _All_ to _Release_ and uncheck _Automatically manage signing_ and select the provisioning profile.

That's it. As a result you should have two files:

1. A distribution certificate in `.p12` format.
2. A distribution provisioning profile.

### Add GitHub Repository Secrets

You can store the certificate and the provisioning profile file in the same repository, but there are safer methods, especially if it's a public repository. [match][match] recommends to use another private repository for it. But there is another option: GitHub Repository Secrets. it's located on your project on GitHub: Settings ↣ Secrets ↣ Actions. But there is a problem with such a method: it doesn't support files. To store files there convert their content to `base64` format. Additionally you can encrypt them if you want even more security.

Following command encode file to `base64` format and put the result in your clipboard:

```zsh
base64 <file> | pbcopy
```

Use it for the `.p12` certificate file. Create a new repository secret with value from your pasteboard and name it `DISTRIBUTION_CERTIFICATE_BASE64`. Create another secret `DISTRIBUTION_CERTIFICATE_PASSWORD` with value of certificate's password (you set the password when exported the certificate from the Keychain Access app). Create one more secret for the provisioning profile. Name it `ADHOC_PROVISIONING_PROFILE_BASE64`.

![Repository Secrets](/docs/assets/github-actions-ios/repository-secrets.png)

### Import the Signing Certificate

The signing certificate should be imported to Keychain Access. It's not an easy task and it requires a deep understanding how keychain works and how to control it using shell interface. The power of GitHub Actions is that there are a lot of ready-to-use actions that can solve the most of common CI/CD tasks.

Instead of doing it ourselves we'll use the `import-codesign-certs` action from "[Apple Github Actions][apple-github-actions]" collection that implements the most common iOS/macOS apps building tasks. So add a new step to `distribute` workflow:

```yaml
- name: 📜 Import certificate
  uses: Apple-Actions/import-codesign-certs@v1
  with:
    p12-file-base64: ${{ secrets.DISTRIBUTION_CERTIFICATE_BASE64 }}
    p12-password: ${{ secrets.DISTRIBUTION_CERTIFICATE_PASSWORD }}
```

### Install the Provisioning Profile

Add a new step for importing the provisioning profile. How to read it from GitHub Secrets? Read the secret into an environment variable:

```yaml
env:
  ADHOC_PROVISIONING_PROFILE_BASE64: ${{ secrets.ADHOC_PROVISIONING_PROFILE_BASE64 }}
```

Then make a couple of variables in a `run` section (replace provisioning filename with yours):

```zsh
PROVISIONING_PROFILE_FILENAME=App_Ad_hoc.mobileprovision
PROVISIONING_PROFILE_PATH=${RUNNER_TEMP}/${PROVISIONING_PROFILE_FILENAME}
```

Import provisioning profile from secrets:

```zsh
echo -n "${ADHOC_PROVISIONING_PROFILE_BASE64}" | base64 --decode --output ${PROVISIONING_PROFILE_PATH}
```

Rename profile using its UUID and put it into the folder where Xcode reads provisioning profiles from:

```zsh
PROVISIONING_PROFILE_UUID=$(grep UUID -A1 -a ${PROVISIONING_PROFILE_PATH} | grep -io "[-A-Z0-9]\{36\}")
mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
cp ${PROVISIONING_PROFILE_PATH} ~/Library/MobileDevice/Provisioning\ Profiles/${PROVISIONING_PROFILE_UUID}.mobileprovision
```

That's it. The whole step:

```yaml
- name: 📱 Install provisioning profile
  env:
    ADHOC_PROVISIONING_PROFILE_BASE64: ${{ secrets.ADHOC_PROVISIONING_PROFILE_BASE64 }}
  run: |
    # create variables
    PROVISIONING_PROFILE_FILENAME=App_Ad_hoc.mobileprovision
    PROVISIONING_PROFILE_PATH=${RUNNER_TEMP}/${PROVISIONING_PROFILE_FILENAME}

    # import provisioning profile from secrets
    echo -n "${ADHOC_PROVISIONING_PROFILE_BASE64}" | base64 --decode --output ${PROVISIONING_PROFILE_PATH}

    # apply provisioning profile
    PROVISIONING_PROFILE_UUID=$(grep UUID -A1 -a ${PROVISIONING_PROFILE_PATH} | grep -io "[-A-Z0-9]\{36\}")
    mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
    cp ${PROVISIONING_PROFILE_PATH} ~/Library/MobileDevice/Provisioning\ Profiles/${PROVISIONING_PROFILE_UUID}.mobileprovision
```

### Add an Archive Step

The hardest part was completed. The next step is really simple now:

```yaml
- name: 🏛 Archive
  run: |
    xcodebuild \
      -project App.xcodeproj \
      -scheme App \
      -destination 'generic/platform=iOS' \
      -archivePath ./build/App.xcarchive \
      archive
```

It's worth mentioning here that a special destination was used: `generic/platform=iOS`. It specifies _any_ Apple iOS device instead of targeting a certain device.

### Export

Exporting an archive requires export options plist. The easiest way to make it is to let Xcode to generate it:

1. Select Product ↣ Archive in Xcode.
2. As it completes click _Distribute App_ in Organizer. Select method of distribution ↣ Ad Hoc and provide any information it asks about. Then Export to any location and open result folder.
3. Inside the folder there is the needed file: `ExportOptions.plist`. Put it inside your iOS project.

To get more information about parameters in export options, see `xcodebuild --help`.
Description of the `-exportArchive` parameter in [`man xcodebuild`][man-xcodebuild]:

> ```zsh
> -exportArchive
>      Specifies that an archive should be distributed. Requires
>      -archivePath and -exportOptionsPlist. For exporting, -exportPath is
>      also required. Cannot be passed along with an action.
> ```

Add an export path to the workflow:

```yaml
- name: 📦 Export
  run: |
    xcodebuild \
      -exportArchive \
      -archivePath ./build/App.xcarchive \
      -exportPath ./build \
      -exportOptionsPlist ExportOptions.plist
```

This step exports the archive got on the previous step to `./build` directory.

That was a last step in building the app for distribution.

![Archive](/docs/assets/github-actions-ios/archive.png)

## Publish

### Firebase App Distribution

Read "[Distribute iOS apps to testers using the Firebase CLI][firebase-distribute-cli]" to get more info about using Firebase CLI tools. Google recommends using service accounts to authenticate in a CI environment. Follow instructions on "[Authenticate with a service account][auth-service-account]" and create a private key in JSON format. Encode this key in `base64` format as we did before:

```zsh
base64 app-github-actions-d059546774f3.json | pbcopy
```

And set it to the GitHub Repository secret using a name `GOOGLE_APPLICATION_CREDENTIALS_BASE64`.

```yaml
- name: 🦊 Upload to Firebase App Distribution
  env:
    GOOGLE_APPLICATION_CREDENTIALS_BASE64: ${{ secrets.GOOGLE_APPLICATION_CREDENTIALS_BASE64 }}
  run: |
    # Install Firebase CLI (https://firebase.google.com/docs/cli?authuser=1#mac-linux-auto-script)
    curl -sL https://firebase.tools | bash

    # App ID from GoogleService-Info.plist
    GOOGLE_APP_ID=1:873990114327:ios:40d53aa2956a563d09e052

    # Set env for Firebase tools authentication
    export GOOGLE_APPLICATION_CREDENTIALS=${RUNNER_TEMP}/google-service-account.json

    # Import Google service account credentials from the secrets
    echo -n "${GOOGLE_APPLICATION_CREDENTIALS_BASE64}" | base64 --decode --output ${GOOGLE_APPLICATION_CREDENTIALS}

    # Upload to Firebase App Distribution
    firebase appdistribution:distribute ./build/App.ipa \
      --app ${GOOGLE_APP_ID} \
      --testers "tester@company.com"
```

[final-project]: https://github.com/aleos/github-actions-ios "GitHub Actions for iOS"
[man-xcodebuild]: x-man-page://xcodebuild "man xcodebuild"
[man-xcode-select]: x-man-page://xcode-select "man xcode-select"
[github-actions-docs]: https://docs.github.com/actions "GitHub Actions"
[match]: https://codesigning.guide "codesigning.guide concept"
[ Developer]: https://developer.apple.com/account/resources " Developer"
[create-certificate]: https://help.apple.com/developer-account/#/devbfa00fef7 "Create a certificate signing request"
[apple-github-actions]: https://github.com/Apple-Actions "Apple Github Actions"
