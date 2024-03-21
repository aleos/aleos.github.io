---
title: GitHub Actions for iOS apps 2
date: 2022-09-18
draft: true
---

==⚠️ Upload symbols==

### Apple Test Flight

xcrun altool \
 --upload-app \
 --type ios \
 --file "${IPA_PATH}" \
 --username ${APPSTORE_EMAIL} \
 --password ${APPSTORE_PASSWORD}

---

## ==Notes for an extended article==

### Upload DSYMs

### Use DEVELOPER_DIR variable to set active Xcode

See <x-man-pages://xcrun>

### Provide changelog for builds

```zsh
CHANGELOG=${RUNNER_TEMP}/CHANGELOG

echo "#${{ github.event.pull_request.number }} ${{ github.event.pull_request.title }}" >> $CHANGELOG
echo "" >> $CHANGELOG
echo "${{ github.event.pull_request.body }}" >> $CHANGELOG
```

### Cache package dependencies

To drastically speed up archive phase.

### Put all build products into a `.build` directory

Say 'no' to Derived Data
Make exactly the same structure inside `build` as xcodebuild does, running without a scheme
Research folders structure in Derived Data.

### How to deal with a few distribution profiles for different archive times (Ad Hoc / App Store)?

⚠️ Provisioning profile was set in xcode manually. But there is a problem to distribute the app to App Store, because it uses another provisioining!

### xcconfig

It is possible to override build options with providing an `.xcconfig`, i.e. add `⍺` to target's name and use a different icon for beta builds.

### A couple of words about Fastlane?

There is Fastlane to make CI/CD process easier but what if you don't want to have one more dependancy or want to have more control? There is `xcodebuild` utility from the Xcode Command Line Tools. Fastlane also uses it internally to build projects. It's not as hard to use `xcodebuild` directly as it looks like, but requires some knowledge.

### Useful xcodebuild info parameters:

- `-showBuildTimingSummary`
  Display a report of the timings of all the commands invoked during the build.
- `-enableCodeCoverage [YES | NO]`
  Turns code coverage on or off during testing. This overrides the setting for the test action of a scheme in a workspace.

### xc|pretty and xc|beautify

`xcodebuild` prints to the terminal all commands it runs and this output is really long and nearly unreadable especially for big projects. There is a utility to fix this: [xcpretty](xcpretty).

### Compile and deploy the Swift-DocC documentation?

### Use xcrun simctl to speed up tests on CI

### Use environment variables in `env` section of yaml

### Use build artefacts

- to make available
  - test reports
  - build log produced by `| tee`
  - ❓
- to break jobs into parts
  - make upload to Firebase App Distribution in a separate job using Linux image
  - parallelise some parts like uploading artefacts?

### How to reuse commands?

Between:

- workflows
- jobs
- steps

### Use symctl to speed builds up

Preheat simulators?

[xcpretty]: https://github.com/xcpretty/xcpretty "xc|pretty"

Link to Build settings reference: [build-settings-reference]: https://help.apple.com/xcode/mac/current/#/itcaec37c2a6 "Build settings reference"
