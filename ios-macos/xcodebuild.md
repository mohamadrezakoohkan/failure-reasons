# xcodebuild — failure reasons

Tooling failures hit when driving `xcodebuild` on iOS/macOS, and what actually
fixes them. Each entry is symptom → cause → fix → a one-line rule.

Entries are addressed by slug (`[stale-derived-data]`) so rules elsewhere can
cite them.

---

## Before anything else: find the real error

`** BUILD FAILED **` is the *last* line, never the cause. The real error is
often hundreds of lines above, buried in compiler invocations.

```bash
xcodebuild ... 2>&1 | tee build.log | xcbeautify
grep -nE "error:|fatal error:|❌" build.log | head -30
```

Two habits that pay for themselves:

- Pipe through `xcbeautify` (or `xcpretty`) for reading, but always `tee` the
  raw log — the pretty printers drop lines you will need.
- Add `-resultBundlePath Build.xcresult`. Test failures and crash logs live in
  the result bundle, not in stdout.

**Rule:** never diagnose from the tail of an xcodebuild log; grep the raw log for `error:` first.

---

## [wrong-developer-dir] xcodebuild requires Xcode, not Command Line Tools

**Symptom**

```
xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer
directory '/Library/Developer/CommandLineTools' is a command line tools instance
```

**Cause** — the active developer directory points at the standalone Command
Line Tools. Common after a macOS upgrade, after installing CLT for Homebrew, or
when Xcode was moved/renamed.

**Fix**

```bash
xcode-select -p                                              # see where it points
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

Per-invocation override without sudo, useful when several Xcodes are installed:

```bash
DEVELOPER_DIR=/Applications/Xcode-16.4.app/Contents/Developer xcodebuild ...
```

**Rule:** check `xcode-select -p` before blaming the project; pin multi-Xcode machines with `DEVELOPER_DIR`.

---

## [no-matching-destination] Unable to find a destination matching the specifier

**Symptom**

```
xcodebuild: error: Unable to find a destination matching the provided
destination specifier:
        { platform:iOS Simulator, OS:latest, name:iPhone 15 Pro }
```

**Cause** — the named device does not exist on this machine, the runtime is not
installed, or the name drifted (Xcode ships new device names every release, and
CI images rarely match a laptop).

**Fix** — ask the toolchain what it actually has, then pin by UDID:

```bash
xcodebuild -scheme MyScheme -showdestinations
xcrun simctl list devices available
xcodebuild ... -destination 'id=<UDID>'
```

For builds that do not need to run, skip device selection entirely:

```bash
xcodebuild ... -destination 'generic/platform=iOS Simulator'
```

**Rule:** never hardcode a simulator *name* in a script; resolve it from `-showdestinations`, or use a generic destination when you only need to compile.

---

## [missing-platform-runtime] iOS platform/runtime not installed

**Symptom** — destination lookup lists only macOS, or the error names a runtime
that "is not installed". Fresh Xcode installs since Xcode 15 ship *no* device
runtimes.

**Fix**

```bash
xcodebuild -downloadPlatform iOS          # or: xcodebuild -downloadAllPlatforms
xcrun simctl list runtimes
```

**Rule:** a fresh Xcode has no simulator runtimes — `-downloadPlatform iOS` is part of machine setup, not troubleshooting.

---

## [scheme-not-found] Workspace does not contain that scheme

**Symptom**

```
xcodebuild: error: The workspace named "App" does not contain a scheme named "App".
The "-list" option can be used to find the names of the schemes in the workspace.
```

**Cause** — one of:

- The scheme is not *shared* (`xcshareddata/xcschemes/`), so it exists only in
  the author's `xcuserdata` and never reached git.
- The project is generated (Tuist, XcodeGen, SPM) and was not regenerated after
  a manifest change.
- `-project` was passed where `-workspace` is required (CocoaPods, or any
  multi-project setup) — the scheme lives in the workspace.

**Fix**

```bash
xcodebuild -workspace App.xcworkspace -list
```

Then share the scheme in Xcode (Manage Schemes → Shared) and commit it, or
regenerate the project with the generator the repo uses.

**Rule:** `-list` first; an unshared scheme is a git problem, not a build problem.

---

## [codesign-no-profile] No signing certificate / no provisioning profile

**Symptom**

```
error: No signing certificate "iOS Development" found
error: No profiles for 'com.example.app' were found
```

**Cause** — signing identity or profile is absent, which is the normal state on
a clean CI runner.

**Fix** — for simulator builds and most CI compile checks, do not sign at all:

```bash
xcodebuild ... CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO CODE_SIGN_IDENTITY=""
```

For a real archive, authenticate properly rather than disabling signing —
either an App Store Connect API key or a pre-imported profile:

```bash
xcodebuild -allowProvisioningUpdates \
  -authenticationKeyPath "$KEY_PATH" \
  -authenticationKeyID "$KEY_ID" \
  -authenticationKeyIssuerID "$ISSUER_ID" ...
```

**Rule:** simulator builds should never need a signing identity — if signing fails on a simulator destination, turn signing off rather than provisioning the runner.

---

## [keychain-locked] User interaction is not allowed

**Symptom** — signing fails on CI with `errSecInteractionNotAllowed`, or:

```
Command CodeSign failed with a nonzero exit code
security: SecKeychainItemImport: User interaction is not allowed.
```

**Cause** — the keychain holding the identity is locked, or it is not in the
search list for the non-interactive session.

**Fix**

```bash
security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
security set-key-partition-list -S apple-tool:,apple: -s -k "$KEYCHAIN_PASSWORD" build.keychain
```

**Rule:** on CI, unlock the keychain *and* set the partition list — importing the identity is only half the job.

---

## [stale-derived-data] Module compiled with a different Swift version

**Symptom**

```
error: module compiled with Swift 6.0 cannot be imported by the Swift 6.1 compiler
error: could not build module 'Foundation'
```

**Cause** — cached modules from a previous toolchain after an Xcode upgrade or
an Xcode switch.

**Fix**

```bash
xcodebuild clean -workspace App.xcworkspace -scheme App
rm -rf ~/Library/Developer/Xcode/DerivedData/App-*
rm -rf ~/Library/Caches/org.swift.swiftpm
```

If it survives a clean, the offending module is a prebuilt binary
(`.xcframework`) genuinely compiled by an older Swift — it must be rebuilt by
its vendor, or built with library evolution enabled. No amount of cleaning
fixes that one.

**Rule:** after any Xcode upgrade, wipe DerivedData before reporting a build failure; if it survives a clean wipe, suspect a prebuilt binary, not the cache.

---

## [db-locked] Accessing build database — database is locked

**Symptom**

```
error: unable to attach DB: accessing build database
"…/XCBuildData/build.db": database is locked
```

**Cause** — two builds share one DerivedData path: Xcode.app open on the same
project while `xcodebuild` runs, or parallel CI jobs on one runner.

**Fix** — give each build its own derived data:

```bash
xcodebuild ... -derivedDataPath "$PWD/.build/DerivedData-$JOB_ID"
```

**Rule:** one DerivedData per concurrent build; close Xcode or pass `-derivedDataPath` before running xcodebuild by hand.

---

## [spm-resolution] Package resolution failed

**Symptom**

```
error: Package.resolved file is corrupted or malformed
error: failed to resolve dependencies: … Authentication failed
xcodebuild: error: Could not resolve package dependencies
```

**Cause** — a corrupt resolution cache, an unreachable or private repository
with no credentials in the non-interactive session, or a `Package.resolved`
format version newer than the Xcode in use.

**Fix**

```bash
xcodebuild -resolvePackageDependencies -workspace App.xcworkspace -scheme App
rm -rf ~/Library/Caches/org.swift.swiftpm ~/Library/Developer/Xcode/DerivedData/*/SourcePackages
```

To keep CI reproducible and stop it silently upgrading dependencies:

```bash
xcodebuild ... -onlyUsePackageVersionsFromResolvedFile -disableAutomaticPackageResolution
```

For private repos over SSH, ensure the agent is loaded — Xcode's resolver uses
the same SSH config as git, and a passphrase prompt in a non-interactive shell
surfaces only as a resolution timeout.

**Rule:** resolve packages as an explicit step before building, and pin CI to the resolved file so dependency drift fails loudly instead of silently.

---

## [macro-plugin-trust] Macro / plugin must be enabled before it can be used

**Symptom** — builds in Xcode.app, fails on CI or a fresh clone:

```
error: Macro "CasePathsMacros" from package "swift-case-paths" must be enabled before it can be used
error: Plugin "SwiftLintBuildToolPlugin" from package "SwiftLint" must be enabled before it can be used
```

**Cause** — since Xcode 15, SwiftPM macros and build-tool plugins run only after
a user clicks "Trust & Enable" in the IDE. That approval is stored per machine
(`~/Library/org.swift.swiftpm/security/macros.json`, `plugins.json`), keyed by
package fingerprint — so it never exists on a CI runner, and it is invalidated
whenever the package version changes.

**Fix** — skip the trust prompt for that invocation:

```bash
xcodebuild ... -skipMacroValidation -skipPackagePluginValidation
```

The flags apply only to the `xcodebuild` they are passed to — fastlane `gym`,
`scan`, and `-resolvePackageDependencies` steps each need them too. The
machine-wide equivalent is
`defaults write com.apple.dt.Xcode IDESkipMacroFingerprintValidation -bool YES`.
Where executing unreviewed macro code on CI is a concern, commit the trusted
fingerprints and copy them into `~/Library/org.swift.swiftpm/security/` before
building instead of skipping validation.

**Rule:** "must be enabled" is a missing per-machine trust record, not a build error — pass the skip flags on *every* xcodebuild call, or ship the fingerprints.

---

## [script-sandbox] Sandbox: deny file-write-create in a build phase

**Symptom**

```
Sandbox: bash(4242) deny(1) file-write-create /Users/…/Generated/Strings.swift
```

**Cause** — `ENABLE_USER_SCRIPT_SANDBOXING` (default `YES` since Xcode 15) blocks
a run-script phase writing outside its declared outputs. Code generators,
SwiftGen, Phrase pulls, and version-stamping scripts all trip this.

**Fix** — the correct fix is to declare the script's input and output files so
the sandbox permits them (and the build stays incremental). The escape hatch:

```bash
xcodebuild ... ENABLE_USER_SCRIPT_SANDBOXING=NO
```

**Rule:** a sandbox denial means an undeclared script output — declare the outputs; disable sandboxing only as a stopgap.

---

## [simulator-wedged] Simulator will not boot / test runner never starts

**Symptom**

```
Unable to boot device because we cannot determine the runtime bundle
The request was denied by service delegate (SBMainWorkspace)
Test runner never began executing tests after launching
```

**Cause** — a wedged CoreSimulator service, or devices left booted by a killed
previous run.

**Fix**

```bash
xcrun simctl shutdown all
xcrun simctl erase all          # destructive: wipes simulator state
sudo pkill -9 -f CoreSimulator  # last resort
```

`Test runner never began executing tests` specifically usually means the test
host crashed at launch — read the crash log in the `.xcresult`, do not keep
rebooting simulators.

**Rule:** a wedged simulator is fixed by `simctl shutdown all`; a runner that launches and dies is an app crash — go to the result bundle.

---

## [simulator-arch] Built for iOS, not iOS Simulator

**Symptom**

```
building for iOS Simulator, but linking in object file built for iOS
ld: symbol(s) not found for architecture arm64
```

**Cause** — a dependency ships a device-only slice, or a legacy
`EXCLUDED_ARCHS[sdk=iphonesimulator*] = arm64` workaround (from the Intel era)
is still in the project and now breaks Apple Silicon.

**Fix** — delete stale `EXCLUDED_ARCHS` entries; replace fat frameworks with a
proper `.xcframework` containing a simulator slice.

**Rule:** `EXCLUDED_ARCHS = arm64` is an Intel-era relic — on Apple Silicon it is the bug, not the fix.

---

## [export-archive] exportArchive fails after a successful archive

**Symptom**

```
error: exportArchive: exportOptionsPlist error for key "method": expected one of {…}
error: exportArchive: No signing certificate "Apple Distribution" found
```

**Cause** — the export options plist does not match the archive's signing style,
or `method` uses a value retired by the current Xcode (`app-store` →
`app-store-connect`, `ad-hoc` → `release-testing` in Xcode 15+).

**Fix**

```bash
xcodebuild -exportArchive \
  -archivePath App.xcarchive \
  -exportPath ./export \
  -exportOptionsPlist ExportOptions.plist \
  -allowProvisioningUpdates
```

**Rule:** archive and export are separate signing contexts — an archive succeeding says nothing about export succeeding.

---

## [deployment-target] Deployment target out of supported range

**Symptom**

```
The iOS deployment target 'IPHONEOS_DEPLOYMENT_TARGET' is set to 11.0, but the
range of supported deployment target versions is 15.0 to 18.5
```

**Cause** — a dependency (usually a CocoaPods pod) still declares an ancient
minimum that the current Xcode refuses.

**Fix** — raise the target in the offending target's build settings, or in a
Podfile `post_install` hook. Warnings here become errors after an Xcode major
upgrade, so treat them as a deadline, not noise.

**Rule:** deployment-target warnings are pre-announced build failures — fix them before the next Xcode upgrade, not after.

---

## [disk-space] No space left on device

**Symptom** — clang or the linker dies with `No space left on device`, or the
build fails in a different place on every run.

**Cause** — DerivedData, archives, simulator runtimes and device support files
grow without bound. Multi-gigabyte builds fail non-deterministically once the
disk is near full.

**Fix**

```bash
du -sh ~/Library/Developer/Xcode/DerivedData ~/Library/Developer/Xcode/Archives \
       ~/Library/Developer/CoreSimulator/Devices
rm -rf ~/Library/Developer/Xcode/DerivedData/*
xcrun simctl delete unavailable
```

**Rule:** a build that fails in a different place each run is an environment problem — check disk space before reading the diff.

---

## Fast triage

Work down this list before deep-diving a log:

1. `xcode-select -p` — right Xcode? → [wrong-developer-dir]
2. `xcodebuild -list` — scheme actually exists and is shared? → [scheme-not-found]
3. `xcodebuild -showdestinations` — destination actually exists? → [no-matching-destination]
4. `df -h` — disk not full? → [disk-space]
5. Wipe DerivedData, retry once. → [stale-derived-data]
6. Still failing? `grep -nE "error:" build.log | head -30` and read the *first* error.

If steps 1–5 change the outcome, it was the environment. If they do not, it is
the code — and only then is the diff worth reading.
