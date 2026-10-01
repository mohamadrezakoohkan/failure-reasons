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

## [multiple-commands-produce] Multiple commands produce the same output

**Symptom**

```
error: Multiple commands produce '…/App.app/Info.plist'
    note: Target 'App' (project 'App') has copy command from '…/App/Info.plist'
    note: Target 'App' (project 'App') has process command with output '…/App.app/Info.plist'
```

**Cause** — two build steps write one file in the product. The new build
system (default since Xcode 10, the only one since Xcode 14) refuses this;
the legacy one silently let the last writer win. Usual culprits:

- `Info.plist` (or an `.entitlements` file) listed in *Copy Bundle Resources*
  while also processed via `INFOPLIST_FILE`.
- Two resources with the same file name in different folders — groups are
  flattened into the bundle root, so `A/Config.json` and `B/Config.json` collide.
- A framework embedded twice: by an *Embed Frameworks* phase and by CocoaPods'
  `[CP] Embed Pods Frameworks` script, or by two targets into one app.

**Fix** — the `note:` lines name both producers; remove one. Drop the plist
from Copy Bundle Resources, rename or use a folder reference for clashing
resources, embed each framework in exactly one place. `-UseModernBuildSystem=NO`
no longer exists, so there is no flag to hide it.

**Rule:** read the two `note:` lines — each names a producer, and the fix is always deleting one of them, never a build setting.

---

## [license-first-launch] Xcode license not accepted / first launch not run

**Symptom** — every command fails before touching the project, exit code `69`:

```
You have not agreed to the Xcode license agreements. Please run
'sudo xcodebuild -license' from within a Terminal window to review and agree
to the Xcode and Apple SDKs license.
```

It also breaks `git`, `clang` and `make`, since the `/usr/bin` shims route
through the active Xcode.

**Cause** — a newly installed or *upgraded* Xcode (minor updates included)
needs its license accepted and its "first launch" packages (MobileDevice, CoreSimulator
support) installed. Opening Xcode.app does both; a headless CI runner never opens it.

**Fix**

```bash
sudo xcodebuild -license accept
sudo xcodebuild -runFirstLaunch
xcodebuild -checkFirstLaunchStatus && echo ready   # non-zero exit = still pending
```

With several Xcodes installed, run both for each one (set `DEVELOPER_DIR`),
since acceptance is tracked per version.

**Rule:** exit 69 before any build output means the Xcode was never initialised — `-license accept` plus `-runFirstLaunch` belong in the image provisioning step, right after every Xcode install.

---

## [dependency-cycle] Cycle inside a target / between targets

**Symptom** — often appears only after adding an app extension, a watch app, or
a new run-script phase:

```
error: Cycle inside App; building could produce unreliable results.
Cycle details:
→ Target 'App': CodeSign /…/App.app
○ Target 'App' has process command with output '…/App.app/Info.plist'
○ That command depends on command in Target 'App': script phase "Crashlytics"
error: Cycle in dependencies between targets 'App' and 'AppTests'
```

**Cause** — the new build system orders tasks by their declared inputs and
outputs, not by where phases sit in the list, and it rejects a loop outright.
Usual culprits:

- A run-script phase whose *input* is a product file (`$(BUILT_PRODUCTS_DIR)/$(INFOPLIST_PATH)`,
  the dSYM) sitting *above* a phase that produces it — typical of Crashlytics
  upload and version-stamping scripts.
- *Embed App Extensions* / *Embed Watch Content* placed after a run-script
  phase, so the script both waits on and feeds the embedded product.
- A *Headers* phase after *Compile Sources* in a framework target.
- Across targets: a test or extension target listed as a dependency of its own host,
  or two frameworks that link each other.

**Fix** — read the `Cycle details` path, then reorder phases so the order is:
dependencies → headers → sources → resources → embed phases → run scripts that read
the product *last*. Give every run script explicit input/output files. For a
cross-target cycle, delete the backward edge in *Target Dependencies* or
*Link Binary With Libraries*. `-UseModernBuildSystem=NO` is gone; there is
nothing to switch off.

**Rule:** a cycle is a wrong phase order, not a flaky build — move scripts that read the built product to the end of the phase list and declare their inputs and outputs.

---

## [pods-out-of-sync] Sandbox not in sync with Podfile.lock

**Symptom** — fails in the first seconds, after a branch switch or on CI:

```
error: The sandbox is not in sync with the Podfile.lock. Run 'pod install' or
update your CocoaPods installation.
error: Unable to load contents of file list:
'/…/Target Support Files/Pods-App/Pods-App-frameworks-Debug-input-files.xcfilelist'
```

**Cause** — CocoaPods injects a `[CP] Check Pods Manifest.lock` phase that diffs
the committed `Podfile.lock` against `Pods/Manifest.lock` (written by the last
`pod install`). They differ when `Podfile.lock` changed (pull, branch switch)
but `pod install` was not re-run, or when a CI cache restored a `Pods/`
directory from another commit. The `.xcfilelist` variant means `Pods/` is
missing entirely. Building the `.xcodeproj` instead of the `.xcworkspace`
fails differently — `No such module` — for the same root reason.

**Fix**

```bash
bundle exec pod install           # the CocoaPods version pinned in Gemfile.lock
xcodebuild -workspace App.xcworkspace ...
```

On CI, key the `Pods/` cache on `hashFiles('Podfile.lock')` and run
`pod install` unconditionally — it is a no-op when the cache is correct.
Use `bundle exec` so the CocoaPods version matches the one that wrote the lock;
a different version can rewrite `Manifest.lock` and re-trigger the error.

**Rule:** this is a stale `Pods/` directory, never a code error — `pod install` after every pull that touches `Podfile.lock`, and cache `Pods/` by the lockfile hash.

---

## [script-path-missing] Run-script phase: command not found

**Symptom** — builds from Terminal but fails in Xcode.app (or on an Apple
Silicon machine / a CI runner, but not on an Intel Mac):

```
/…/Script-8F3A1C.sh: line 3: swiftlint: command not found
Command PhaseScriptExecution failed with a nonzero exit code
```

**Cause** — run-script phases do not source `~/.zshrc` or `~/.zprofile`.
Xcode.app is started by launchd and gives scripts a fixed `PATH`:
`$(DEVELOPER_DIR)/usr/bin`, `/usr/local/bin` and the system dirs. Homebrew on
Apple Silicon installs to `/opt/homebrew/bin`, and mise/asdf/rbenv shims live
under `~`, so neither is on that `PATH`. `xcodebuild` run from a shell inherits
that shell's `PATH`, which is why the same build passes on the command line.
`PhaseScriptExecution failed` is only the wrapper; the real error is the line
above it.

**Fix** — make the script find the tool itself, and fail loudly when it can't:

```bash
export PATH="/opt/homebrew/bin:$HOME/.local/share/mise/shims:$PATH"
if ! command -v swiftlint >/dev/null; then
  echo "error: swiftlint not installed (brew install swiftlint)"; exit 1
fi
swiftlint
```

Better still, drop the global tool: run it as an SPM build-tool plugin or call a
version-pinned binary in the repo (`mise exec -- swiftlint`, `Pods/SwiftLint/swiftlint`).
Do not symlink into `/usr/local/bin` to hide it — the next machine breaks again.

**Rule:** a build phase sees launchd's `PATH`, not your shell's — set `PATH` inside the script or call the tool by a pinned path.

---

## [dyld-rpath-missing] Builds fine, crashes at launch: Library not loaded

**Symptom** — `BUILD SUCCEEDED`, then the app or `xcodebuild test` dies before
any code runs:

```
dyld[4121]: Library not loaded: @rpath/Analytics.framework/Analytics
  Referenced from: <…> /…/App.app/App
  Reason: tried: '/…/App.app/Frameworks/Analytics.framework/Analytics' (no such file), …
The test runner exited with code -1 before finishing running tests.
```

**Cause** — linking and embedding are separate steps. The linker only records
"load `@rpath/X` at runtime"; it never checks the framework will be there. It
is missing when the dynamic framework is set to *Do Not Embed*, when a new
target (extension, test bundle, CLI tool) links it without its own copy phase,
or when `LD_RUNPATH_SEARCH_PATHS` lacks the directory it lives in. A SwiftPM
product switched to `type: .dynamic`, or a Tuist/XcodeGen change from static
to dynamic linking, causes the same crash with no source change.

**Fix** — see what the binary asks for, then compare with what the bundle holds:

```bash
otool -L App.app/App | grep @rpath             # what it needs
otool -l App.app/App | grep -A2 LC_RPATH        # where it looks
ls App.app/Frameworks                           # what was shipped
```

Set the framework to *Embed & Sign* in the app target only. Give runpath
`@executable_path/Frameworks` for apps and `@loader_path/Frameworks` for
frameworks and hostless test bundles. Extensions should load from the host app
(`@executable_path/../../Frameworks`), not carry their own copy.

**Rule:** a green link proves nothing about runtime — every `@rpath` dependency must be embedded once and reachable from an `LC_RPATH` entry.

---

## [undefined-symbols] Compiles, then the linker fails: Undefined symbols

**Symptom** — every file compiles and the build dies at the link step:

```
Undefined symbols for architecture arm64:
  "_OBJC_CLASS_$_SKStoreProductViewController", referenced from:
       in RateAppView.o
  "_$s10Networking9APIClientC5fetch10Foundation4DataVyYaKF", referenced from:
       in FeedLoader.o
ld: symbol(s) not found for architecture arm64
clang: error: linker command failed with exit code 1
```

**Cause** — the compiler only needs a declaration; the linker needs the code.
`import Foo` succeeds whenever `Foo.swiftmodule` or its headers sit in the
shared build folder, even when the target never links `Foo`. That is why it
works in one scheme and breaks in another. Usual reasons nothing defines the
symbol:

- The file that defines it is not in the target's *Compile Sources* (wrong
  target membership, or a Tuist/XcodeGen glob that skips it).
- The library, framework or SPM product is linked into a sibling target but not
  this one.
- A system library is not linked: `std::__1`/`___cxa_*` → `libc++.tbd`,
  `_sqlite3_*` → `libsqlite3.tbd`, `_OBJC_CLASS_$_SK…` → `StoreKit`. This shows
  up when autolinking is off, or the reference comes from a prebuilt static
  library.
- A test target uses `internal` API, but the app was built with
  `ENABLE_TESTABILITY = NO` (running tests against Release).
- A prebuilt `.xcframework` is older than the source: a signature changed, so
  the mangled name no longer matches.

**Fix** — demangle the name, then check who should define it:

```bash
grep -A4 "Undefined symbols" build.log | xcrun swift-demangle
nm -gU path/to/libFoo.a | grep APIClient        # does the binary export it?
```

If nothing exports it, fix target membership or rebuild the stale binary. If
something does, add that product to *Link Binary With Libraries* (or the
target's SPM/Tuist dependencies) of the failing target itself. Do not count on
another target pulling it in.

**Rule:** an `import` that compiles proves visibility, not linkage — every target must declare its own dependencies.

---

## [duplicate-symbols] Linker fails: duplicate symbol, or two copies at runtime

**Symptom** — the link step finds the same symbol twice:

```
duplicate symbol '_OBJC_CLASS_$_Reachability' in:
    /…/libAnalytics.a[3](Reachability.o)
    /…/libNetworking.a[7](Reachability.o)
ld: 1 duplicate symbols
clang: error: linker command failed with exit code 1
```

Or the build succeeds and the console warns at launch:

```
objc[812]: Class _TtC4Core6Logger is implemented in both …/Core.framework/Core
and …/App.app/App. One of the two will be used. Which one is undefined.
```

**Cause** — two object files define the same global, and the linker is told
to load both. Usual sources:

- Two vendored static libraries each embed the same third-party code
  (Reachability, protobuf, a crash reporter). This usually goes unnoticed until
  `-ObjC` or `-all_load` in `OTHER_LDFLAGS` forces every archive member in.
- A source file is in *Compile Sources* twice, or is compiled in the app
  *and* also shipped inside a linked static library.
- A C/ObjC global is *defined* in a header (`NSString *kKey = @"…";`), so every
  file that includes it defines it again.
- The runtime variant: one static library is linked into both the app and a
  dynamic framework the app embeds. Each binary links cleanly, so the error
  only appears when both load into one process. Singletons, `+load` and
  type casts then quietly act on the wrong copy.

**Fix** — see which archives define it, then remove one definition:

```bash
grep -B1 -A3 "duplicate symbol" build.log
nm -gU libAnalytics.a libNetworking.a | grep Reachability
```

Drop the duplicate vendored copy, or ask for a build without it. Swap
`-all_load` for `-force_load path/to/libOne.a`. Headers hold only
`extern` declarations, with the definition in one `.m`/`.c` file. A static
library shared by the app and its frameworks goes into exactly one dynamic
framework, and everyone else links that framework (or make the library dynamic).

**Rule:** each symbol gets one definition per process — link any static library into exactly one binary, and never paper over it with `-all_load`.

---

## [type-check-timeout] Compiler gives up: unable to type-check this expression

**Symptom** — one Swift file never finishes compiling. It fails with no real
type error, or SwiftUI points at a whole `body`:

```
error: the compiler is unable to type-check this expression in reasonable time;
       try breaking up the expression into distinct sub-expressions
error: failed to produce diagnostic for expression; please submit a bug report
```

It can pass on a fast laptop and fail on a slower CI runner, or start after a
Swift upgrade with no source change.

**Cause** — Swift infers types by trying every combination of overloads.
Operators (`+`, `*`, `??`, `==`) have dozens of overloads, and untyped literals
(`0`, `1.5`, `"a"`) can be any `ExpressibleBy…` type. Closures with no
annotations and mixed `CGFloat`/`Double` math (implicitly converted since
Swift 5.5) add more. The number of combinations grows exponentially, and the
solver stops when it hits its time or memory limit. Long SwiftUI `body`s with
ternaries, `if`s and modifier chains are one big expression, so they hit this
often and the error lands on the whole view.

**Fix** — find the slow expressions, then give the solver fewer choices:

```bash
# OTHER_SWIFT_FLAGS, Debug only — warns with the time spent (ms)
-Xfrontend -warn-long-expression-type-checking=100
-Xfrontend -warn-long-function-bodies=200
```

Split the expression into `let`s with explicit types. Type the literals
(`let spacing: CGFloat = 8`). Annotate closure parameters and return types.
Use string interpolation instead of `+` chains, and keep one numeric type per
expression. In SwiftUI, move branches into small subviews or `@ViewBuilder`
properties. Keep the warning flags on, so new hot spots show up before CI hits
the limit.

**Rule:** a timeout here is a code smell, not a flaky build — make the expression cheap to infer rather than retrying on a faster machine.

---

## [test-host-missing] Tests never start: Could not find test host

**Symptom** — `xcodebuild build` passes, but `test` or `build-for-testing`
fails before any test runs:

```
error: Could not find test host for AppTests: TEST_HOST evaluates to
       "…/Debug-iphonesimulator/App.app/App"
```

Often it starts right after renaming the app, adding a configuration such as
`Staging`, or running the same tests on macOS.

**Cause** — a hosted unit-test bundle is injected into the app at runtime.
`TEST_HOST` is a hard-coded path to the app's executable, and `BUNDLE_LOADER`
(usually `$(TEST_HOST)`) is what the test bundle links against. The path goes
stale when the app's `PRODUCT_NAME` changes, or differs per configuration
(`App Staging.app`). It is also wrong on macOS, where the binary lives in
`App.app/Contents/MacOS/App`. And it can be right but empty: the scheme's Build
action does not build the app for *Test*, so the host is never produced.

**Fix** — compare what the setting says with what was built:

```bash
xcodebuild -showBuildSettings -scheme App -configuration Debug \
  | grep -E ' (TEST_HOST|BUNDLE_LOADER) = '
ls "$(dirname "<TEST_HOST value>")"
```

Use the template's platform-neutral form, with the real product name:

```
TEST_HOST     = $(BUILT_PRODUCTS_DIR)/App.app/$(BUNDLE_EXECUTABLE_FOLDER_PATH)/App
BUNDLE_LOADER = $(TEST_HOST)
```

If the name changes per configuration, set `TEST_HOST` per configuration too.
In the scheme, tick *Test* for the app target under Build, or set *Host
Application* on the test target's General tab. Tests that do not need the app
can drop the host: clear `TEST_HOST` and `BUNDLE_LOADER`, and move the code
under test into a framework. A wrong `BUNDLE_LOADER` fails at link time
instead → [undefined-symbols].

**Rule:** `TEST_HOST` is a path, not a reference — when the app's name or platform changes, update it on purpose.

---

## [non-modular-header] Could not build module: non-modular header include

**Symptom** — an Objective-C (or mixed) framework builds on its own, then any
`import` of it from Swift or `@import` from Objective-C fails:

```
error: include of non-modular header inside framework module 'Foo':
       '/…/Vendor/Bar.h' [-Werror,-Wnon-modular-include-in-framework-module]
error: could not build Objective-C module 'Foo'
```

Often starts after turning on `DEFINES_MODULE`, adding a Swift file to an
Objective-C target, or moving a pod to a static framework.

**Cause** — a framework module is only what its module map lists (usually the
umbrella header). Clang requires every header reached from it to belong to a
module too. A public header that does `#import "Bar.h"` from a plain header
search path, a non-modular library, or the app's own sources pulls in text no
module owns, and Clang refuses to build the module. The `In file included
from` lines above the error show the chain.

**Fix** — keep the module's public surface self-contained:

- Move the `#import` from the public `.h` into the `.m`; in the header, use
  `@class Bar;` / `@protocol Bar;` forward declarations.
- If the type must stay public, make the dependency a module: give it a
  `module.modulemap`, or with CocoaPods use `use_modular_headers!` (or
  `:modular_headers => true` on that pod).
- Headers only used inside the framework go in the *Project* header role, not
  *Public*, and never in the umbrella header.

`CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES = YES` turns the error
into a warning for Objective-C builds only. The Swift importer still fails, and
the module stays fragile.

**Rule:** a framework's public headers may import only other modules — anything else goes in the `.m` or behind a forward declaration.

---

## [entitlements-mismatch] Profile doesn't include / match the entitlement

**Symptom** — signing works for simulator and for other targets, then a device
build or archive fails right after someone adds a capability:

```
error: Provisioning profile "App AppStore" doesn't include the
       com.apple.developer.associated-domains entitlement.
error: Provisioning profile "App Dev" doesn't match the entitlements file's
       value for the aps-environment entitlement.
```

**Cause** — two sources must agree. The target's `.entitlements` file
(`CODE_SIGN_ENTITLEMENTS`) says what the app asks for. The provisioning profile
embeds what the App ID was allowed *when the profile was generated*. Ticking a
capability in Xcode edits only the file; a manual or `match`-managed profile
stays old until it is regenerated. Values count too: App Group and keychain
group IDs, or `aps-environment` `development` vs `production`, must be in the
profile. A per-configuration entitlements file can make only Release fail.

**Fix** — diff the two sides:

```bash
plutil -p App/App.entitlements
security cms -D -i App.mobileprovision | plutil -extract Entitlements xml1 -o - -
```

Enable the capability on the App ID in the developer portal, then regenerate
and reinstall the profile (`fastlane match <type> --force`, or let
`-allowProvisioningUpdates` do it for automatic signing). If the entitlement
was not meant to ship, remove it from the file instead.

Related: `Entitlements file "App.entitlements" was modified during the build`
means a script phase wrote to the file. Generate it into `$(DERIVED_FILE_DIR)`
and point `CODE_SIGN_ENTITLEMENTS` there; `CODE_SIGN_ALLOW_ENTITLEMENTS_MODIFICATION
= YES` only silences the check.

**Rule:** a new capability is a profile change, not just a project change — regenerate the profile in the same PR that edits `.entitlements`.

---

## [missing-input-file] Build input file cannot be found

**Symptom** — the build stops before compiling anything useful:

```
error: Build input file cannot be found: '/…/App/Features/Old/OldView.swift'.
       Did you forget to declare this file as an output of a script phase
       or custom build rule which produces it?
```

Often passes locally and fails on CI or a fresh clone, or right after a
rebase, a folder move, or a merge that touched `project.pbxproj`.

**Cause** — the project still lists a path that does not exist when the build
needs it. The usual cases:

- The file was moved, renamed, or deleted on disk, but the reference in
  `project.pbxproj` stayed (a red file in the navigator). Merges that resolve
  `pbxproj` conflicts by hand leave these behind.
- The file exists only on your machine: git-ignored, never committed, or a
  generated file (SwiftGen, protobuf, `.xcconfig`) that CI never generates.
- Case differs (`Foo.swift` vs `foo.swift`). The default macOS volume ignores
  case, so it works locally; a case-sensitive CI volume does not.
- A script phase produces the file, but runs after *Compile Sources* or does
  not list it in *Output Files*, so the build system looks for it too early.
- `INFOPLIST_FILE`, `CODE_SIGN_ENTITLEMENTS`, or a bridging header path in
  build settings points at an old location.

**Fix** — find where the stale path comes from, then fix that source:

```bash
grep -n "OldView.swift" App.xcodeproj/project.pbxproj
xcodebuild -showBuildSettings -scheme App | grep -E "INFOPLIST_FILE|ENTITLEMENTS|BRIDGING"
git ls-files | grep -i "oldview"
```

Remove and re-add the red reference, or regenerate the project if it comes
from Tuist/XcodeGen (the manifest is the source to fix). For generated files,
put the script phase before *Compile Sources* and declare the file in its
*Output Files* — see [script-sandbox]. Commit files that are really sources.

**Rule:** the project file must describe what is in git — after any move or `pbxproj` merge, do a clean build from a fresh clone before pushing.

---

## [result-bundle-exists] Existing file at -resultBundlePath

**Symptom** — xcodebuild quits in under a second, before it resolves packages
or compiles anything:

```
xcodebuild: error: Existing file at -resultBundlePath "/…/Build.xcresult"
```

The first run works. The second run on the same machine fails, and so does a
retry step on CI, or a local `make test` run twice in a row.

**Cause** — xcodebuild will not overwrite or append to a result bundle. If the
path already exists, it refuses to start. This path usually survives because:

- The script uses a fixed path (`Build.xcresult`) and never deletes it.
- A retry wrapper (fastlane `multi_scan`, a CI "retry on failure" step, a
  `for` loop that reruns flaky tests) reruns with the same path.
- A CI cache or a reused workspace brings back the bundle from an older job.
- A run that crashed or was killed still left a half-written bundle.

**Fix** — make each run's path unique, or delete it on purpose right before
the run:

```bash
BUNDLE="build/Test-$(date +%Y%m%d-%H%M%S).xcresult"
xcodebuild test -scheme App -destination "$DEST" -resultBundlePath "$BUNDLE"

# or, with a fixed path:
rm -rf build/Test.xcresult
xcodebuild test ... -resultBundlePath build/Test.xcresult
```

Give each retry its own suffix (`-attempt2`) so the first failure is still
there to read. You can combine them afterwards with
`xcrun xcresulttool merge a.xcresult b.xcresult --output-path all.xcresult`.
Do not add `*.xcresult` to CI caches.

**Rule:** a result bundle path must be new for every xcodebuild run — use a timestamp or retry suffix, never a shared fixed name.

---

## [compiler-crash] Swift compiler crashes: failed due to signal

**Symptom** — no `error:` in your code, just a crash dump from the compiler:

```
error: compile command failed due to signal 11 (use -v to see invocation)
Please submit a bug report (https://swift.org/contributing/#reporting-bugs)
Stack dump:
1.  Apple Swift version 6.x ...
2.  While evaluating request TypeCheckSourceFileRequest(source_file "/…/Feed/FeedView.swift")
3.  While type-checking 'FeedView' (at /…/Feed/FeedView.swift:12:8)
** BUILD FAILED **  — Command SwiftCompile failed with a nonzero exit code
```

Often Release-only (archive fails, Debug is fine), or starts right after an
Xcode update.

**Cause** — the compiler itself hit a bug. The signal says which kind:

- `signal 11` (segfault) or `signal 6` (abort / assertion) — a real compiler
  bug, triggered by one construct: nested generics, result builders, macros,
  `some`/`any` types, key paths, or code the optimizer inlines in Release.
- `signal 9` (killed) — not a bug: the OS killed the compiler for using too
  much memory. Common on small CI machines with many parallel jobs.
- Rarely, a module cache built by another compiler version — see
  [stale-derived-data].

**Fix** — the stack dump names the file and often the line. Start there:

```bash
grep -nE "While (type-checking|evaluating|emitting|silgen).*at /" build.log | head
# Release-only? Build one file at a time to find which file crashes:
xcodebuild archive ... SWIFT_COMPILATION_MODE=singlefile
# signal 9? Give each compiler more memory by running fewer jobs:
xcodebuild ... -jobs 2
```

Rewrite the named construct into something simpler: add explicit types, split
a large result builder or closure, move a generic helper out of the extension.
Test with `-Onone` to check if the optimizer causes it. If the crash only
happens in one optimized function, `@_optimize(none)` on that function is a
workaround. Then try the newest Xcode, and report the reduced case to
github.com/swiftlang/swift.

**Rule:** a compiler crash is not your error message — read the `While …` lines in the stack dump to find the construct, rewrite it simpler, and treat `signal 9` as memory, not a bug.

---

## [project-unreadable] Unable to read project: damaged or future format

**Symptom** — xcodebuild stops before it builds anything, with one of these:

```
xcodebuild: error: Unable to read project 'App.xcodeproj'.
  Reason: The project 'App' is damaged and cannot be opened. Examine the
  project file for invalid edits or unresolved source control conflicts.

xcodebuild: error: … cannot be opened because it is in a future Xcode
  project file format (77).
```

It works on one teammate's Mac and fails on another, or on CI, right after a
merge or right after someone opened the project in a newer Xcode.

**Cause** — `project.pbxproj` is an old-style plist, and xcodebuild will not
read it if any part is broken:

- Merge conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) were committed, or
  a hand-fixed conflict left a missing `;`, `}` or a duplicate object ID.
- A newer Xcode raised `objectVersion`. Xcode 16 writes `77` when the project
  uses synchronized folders (`PBXFileSystemSynchronizedRootGroup`), and older
  Xcode versions refuse to open it.
- A generator (Tuist, XcodeGen) or a script wrote a bad value, or a
  `.xcworkspace/contents.xcworkspacedata` has a conflict too.

**Fix** — find the broken line, then either fix it or regenerate the file:

```bash
grep -nE "^(<<<<<<<|=======|>>>>>>>)" App.xcodeproj/project.pbxproj
plutil -lint App.xcodeproj/project.pbxproj      # shows where parsing fails
grep -n "objectVersion" App.xcodeproj/project.pbxproj
```

For a future format, either use the same Xcode as CI (see
[wrong-developer-dir]), or go to File Inspector → Project Format in the newer
Xcode, choose an older format and commit. Synchronized folders need Xcode 16,
so convert them back to groups first. If the project is generated, do not fix
it by hand: run `tuist generate` / `xcodegen` again. Do not use
`merge=union` for `*.pbxproj` — it hides conflicts and can cause this error.

**Rule:** `plutil -lint project.pbxproj` must pass before a merge is pushed, and the whole team must build with the same Xcode that sets `objectVersion`.

---

## [test-hang] Tests hang forever, or fail: exceeded execution time allowance

**Symptom** — `xcodebuild test` stops printing output in the middle of the
suite and stays there until the CI job is killed. Nothing is written to the
`.xcresult`, so no test is shown as the cause. With timeouts turned on, one
test fails instead and the run goes on:

```
Test Case '-[AppTests.SyncTests testRefresh]' started.
error: -[AppTests.SyncTests testRefresh] : Test exceeded execution time allowance
```

**Cause** — one test never finishes. Test timeouts are off by default, so
xcodebuild waits for it with no limit. Common reasons:

- A deadlock on the main thread: `DispatchQueue.main.sync` called from the
  main thread, or a semaphore waits on main for a callback that also runs on
  main.
- An `async` test awaits a continuation that is never resumed (look for
  `SWIFT TASK CONTINUATION MISUSE … leaked its continuation`), or a
  `wait(for:timeout:)` with a very large timeout.
- A real network call, keychain call, or system permission alert that never
  gets an answer on a headless CI simulator.
- A UI test waits on an element that never appears, with no timeout.

**Fix** — turn on timeouts so a hang becomes a normal test failure with a
name. When a test goes over its time, Xcode takes a spindump, fails that test,
restarts the test runner and runs the rest of the suite:

```bash
xcodebuild test -scheme App -destination "$DEST" \
  -test-timeouts-enabled YES \
  -default-test-execution-time-allowance 120 \
  -maximum-test-execution-time-allowance 600 \
  -resultBundlePath "build/Test-$(date +%s).xcresult"
```

The same setting is in the test plan (Configurations → Test Timeouts). One
slow test can ask for more time with `executionTimeAllowance` (XCTest, rounded
up to whole minutes) or `.timeLimit(.minutes(2))` (Swift Testing). Then open
the spindump for the failed test in the result bundle: it shows the stack of
the thread that is stuck. Fix that code — replace the real network call with a
stub, or remove the blocking wait on main.

**Rule:** CI always runs tests with `-test-timeouts-enabled YES`, so a hang fails one named test with a spindump instead of silently using the whole job timeout.

---

## [codesign-detritus] resource fork, Finder information, or similar detritus not allowed

**Symptom** — compile and link pass, then the last step, signing, fails:

```
/…/Build/Products/Debug-iphoneos/App.app: resource fork, Finder information, or similar detritus not allowed
Command CodeSign failed with a nonzero exit code
```

It works on CI and on other laptops, but fails on one machine. Often it starts
after the project moved to `~/Desktop` or `~/Documents`.

**Cause** — `codesign` refuses a bundle if any file in it has a resource fork
or Finder info stored as an extended attribute (xattr). Common sources:

- The project or DerivedData is in a folder synced by iCloud Drive (Desktop &
  Documents sync), Dropbox, or Google Drive. The sync tool adds
  `com.apple.FinderInfo` / `com.apple.fileprovider.*` to files, and they are
  copied into the `.app`.
- An image, font or `.xcframework` added from Finder, a zip, or an email keeps
  its Finder info.
- On newer macOS, `com.apple.provenance` on build products can trigger it too.

**Fix** — find which file has the attribute, then remove the source of it:

```bash
xattr -lr build/Products/Debug-iphoneos/App.app | grep -vE "com.apple.quarantine" | head
xattr -cr path/to/Assets  # clean the source files, not only the build
rm -rf ~/Library/Developer/Xcode/DerivedData/App-*
```

If the project is in a synced folder, `xattr -cr` does not last — the sync
tool adds the attributes again. Move the clone to a folder that is not synced
(`~/Developer`), and keep `-derivedDataPath` outside synced folders too.

**Rule:** never build from an iCloud/Dropbox-synced folder; on a detritus error run `xattr -lr` on the `.app` to find the file, and clean the source, not only the build.

---

## [device-not-ready] Physical device: Developer Mode disabled / unpaired

**Symptom** — building or testing on a real iPhone fails before install. The
device is plugged in, but xcodebuild lists it under *Ineligible destinations*
or says:

```
Developer Mode disabled. To use iPhone for development, enable Developer Mode in Settings → Privacy & Security.
iPhone 11 is not available because it is unpaired. Pair with the device in the Xcode Devices Window, and respond to any pairing prompts on the device.
The device "iPhone" is not available because it is not executable.
```

It often starts after an iOS or Xcode update, or on a CI device lab that
nobody touches.

**Cause** — the device is connected but not ready for development. Since iOS
16, a device needs three things: Developer Mode on, a trusted pairing with this
Mac, and a finished "Preparing device for development" step (Xcode copies
debug support to the device). An update, a reset, or a new Mac can undo any of
them. Also, Xcode (CoreDevice) sometimes reports Developer Mode as off when it
only failed to read the status.

**Fix** — check what the device state really is, then fix that one thing:

```bash
xcrun devicectl list devices                       # state: available / unavailable
xcrun devicectl device info details --device <UDID> | grep -iE "developerMode|pairing"
```

- Developer Mode off: on the device, Settings → Privacy & Security → Developer
  Mode → on, reboot, and confirm the prompt after reboot. It needs a person
  with the passcode — CI cannot do it.
- Unpaired: unlock the device, open Xcode → Window → Devices and Simulators,
  tap *Trust* on the device and enter the passcode.
- Not executable / preparing: keep the device unlocked and wait until the
  Devices window stops showing "Preparing"; then run again.
- It says off but the toggle is on: turn it off and on, reboot the device and
  the Mac.

**Rule:** before a device build, check `xcrun devicectl list devices`; after any iOS/Xcode update, someone must unlock each lab device and re-check Developer Mode and pairing by hand.

---

## [actool-runtime-mismatch] Asset catalog fails: No simulator runtime version available

**Symptom** — sources compile, then `CompileAssetCatalog` fails. It happens
even in a build for a device or an archive:

```
error: No simulator runtime version from [<DVTBuildVersion 22E238>, <DVTBuildVersion 22F77>] available to use with iphonesimulator SDK version <DVTBuildVersion 23B77>
** BUILD FAILED **  (CompileAssetCatalog … Assets.xcassets)
```

It often starts after an Xcode minor update or a new CI runner image, when
`xcrun simctl list runtimes` still shows iOS runtimes.

**Cause** — `actool` renders asset catalogs with a simulator runtime, and that
runtime must match the SDK build of the Xcode you are using. The runtimes that
are installed belong to a different SDK build (an older point release, or a
beta), so none of them match. This is not [missing-platform-runtime]: runtimes
are there, just not the right build. A macOS update can also turn off the
runtime disk images, and then the result is the same.

**Fix**

```bash
xcrun simctl runtime match list          # SDK build → runtime it uses (or none)
xcodebuild -downloadPlatform iOS         # get the runtime for THIS Xcode's SDK
# or point the SDK at a runtime that is already installed:
xcrun simctl runtime match set iphoneos<version> <runtime-build>
```

If `simctl runtime list` shows runtimes as *Unusable* after a macOS update,
restart the Mac (or `sudo killall -9 com.apple.CoreSimulator.CoreSimulatorService`)
so the runtime images mount again.

**Rule:** when Xcode changes, install its runtime too — a simulator runtime that
does not match the SDK build breaks asset catalogs, even in device builds.

---

## [scheme-not-testable] Scheme is not configured for the test action

**Symptom** — `xcodebuild build` passes, but `test` stops before anything is
built for testing:

```
xcodebuild: error: Scheme App is not currently configured for the test action.
xcodebuild: error: Scheme App does not have an associated test plan named "CI".
Tests in the target "AppTests" can't be run because "AppTests" isn't a member of the specified test plan or scheme.
```

**Cause** — the scheme exists ([scheme-not-found] is ruled out), but its Test
action is empty or points at something that is not there. One of:

- No test targets are listed in the scheme's Test action, so it has no
  `<Testables>`. This happens with auto-created schemes and after a generator
  (Tuist, XcodeGen) rebuilds the scheme without the test target.
- The scheme uses test plans, but the `.xctestplan` file was moved, renamed, or
  never committed. The `.xcscheme` still points at the old path.
- `-testPlan` names a plan that is not attached to *this* scheme, or
  `-only-testing:` names a target that is not in the active plan.

**Fix** — see what the scheme actually tests:

```bash
xcodebuild -showTestPlans -scheme App -workspace App.xcworkspace
grep -nE "<Testables|<TestPlans|reference =" \
  App.xcodeproj/xcshareddata/xcschemes/App.xcscheme
```

Add the test target under Edit Scheme → Test (or to the test plan), make sure
every `.xctestplan` it points to is in git, and pass `-testPlan` only with a
name from `-showTestPlans`. For generated projects, fix the scheme in the
manifest, not in Xcode.

**Rule:** a scheme that builds is not a scheme that tests — check `-showTestPlans` before adding `test` to CI.

---

## [embedded-binary-mismatch] Embedded binary not prefixed / not signed like the parent app

**Symptom** — every target compiles, then the host app fails in the
`ValidateEmbeddedBinary` step, right after the extension is copied in:

```
error: Embedded binary's bundle identifier is not prefixed with the parent app's bundle identifier.
error: Embedded binary is not signed with the same certificate as the parent app.
warning: The CFBundleVersion of an app extension ('1') must match that of its containing parent app ('412').
```

**Cause** — Xcode checks that each embedded extension, widget, or watch app
matches the app that contains it. The settings drift apart because each target
has its own build settings:

- The extension's `PRODUCT_BUNDLE_IDENTIFIER` is not `<app id>.<suffix>`. This
  is common when the app id changes per configuration (`.dev`, `.staging`) but
  the extension id is hard-coded.
- `DEVELOPMENT_TEAM`, `CODE_SIGN_IDENTITY`, or `CODE_SIGN_STYLE` differ between
  targets. CI often overrides these for the app only.
- `CURRENT_PROJECT_VERSION` / `MARKETING_VERSION` are bumped on the app only.
  Xcode only warns, but App Store Connect rejects the upload.

**Fix** — compare the settings of both targets for the same configuration:

```bash
for t in App Widget; do echo "== $t"; xcodebuild -showBuildSettings \
  -project App.xcodeproj -target "$t" -configuration Release | grep -E \
  " (PRODUCT_BUNDLE_IDENTIFIER|DEVELOPMENT_TEAM|CODE_SIGN_IDENTITY|CODE_SIGN_STYLE|CURRENT_PROJECT_VERSION|MARKETING_VERSION) ="; done
```

Build extension ids from the app id (for example `$(APP_BUNDLE_ID).widget`) and
keep team, signing, and version settings in one shared `.xcconfig` for all
targets. When CI passes signing overrides on the command line
(`DEVELOPMENT_TEAM=…`), they apply to every target — prefer that to per-target
edits.

**Rule:** an extension is signed and versioned *with* its host — derive its settings from the app, never copy them.

---

## [module-redefinition] Redefinition of module — two module maps claim one name

**Symptom** — Clang stops while building or importing a module:

```
error: redefinition of module 'DoubleConversion'
  module DoubleConversion {
         ^
/…/Pods/Headers/Public/DoubleConversion/module.modulemap:1:8: note: previously defined here
error: could not build module 'Darwin'
```

The follow-on `could not build module 'Darwin'` / `'Foundation'` errors are
noise. The first `redefinition` line and its `previously defined here` note
name the two files.

**Cause** — Clang loads every `module.modulemap` it finds on the header and
framework search paths. Two maps that declare the same module name collide:

- The same library arrives twice: from CocoaPods *and* SPM, or through two
  transitive dependencies that each vendor it.
- A framework's source directory holds a `module.modulemap`, and so does its
  built or installed copy. Both are on the search path.
- A recursive search path (`$(SRCROOT)/**`) or a stale `HEADER_SEARCH_PATHS`
  entry reaches a copied `include/` folder or old build products.
- Several Xcodes are installed and a path points into the wrong one's SDK.

**Fix** — find every map that declares the module:

```bash
grep -rl --include=module.modulemap "module DoubleConversion" \
  Pods Packages ~/Library/Developer/Xcode/DerivedData 2>/dev/null
```

Keep one provider for the library and remove the other. Replace recursive
search paths with exact ones. For a framework's own map, rename the source copy
(for example `Foo.modulemap`) and point `MODULEMAP_FILE` at it, so only the
built product is found by the default name. After the change, wipe DerivedData
(→ [stale-derived-data]).

**Rule:** one module name, one module map on the search path — find the second map, never rename the module to hide it.

---

## [module-cache-mtime] Header modified since the module file was built

**Symptom** — a CI build that passed yesterday fails straight away on clean
code, often in an SDK header nobody touched:

```
fatal error: file '/Applications/Xcode.app/…/usr/include/os/object.h' has been modified since the module file '/…/ModuleCache.noindex/…/Darwin-3FJ2K9.pcm' was built: mtime changed
error: could not build module 'Foundation'
```

**Cause** — each Clang module file (`.pcm`) records the path and mtime of every
header it was built from. If a header's mtime changes, Clang rejects the `.pcm`.
This happens when an old `ModuleCache.noindex` sits on top of new headers:

- CI restores a cached `DerivedData` (or `-derivedDataPath`) after the runner
  image updated Xcode. The SDK headers are new, but the cache is old.
- A fresh `git checkout` resets source mtimes, but the restored cache still
  holds `.pcm` files built against the older timestamps.
- Two Xcode builds share one cache folder (the same `-derivedDataPath`) and
  overwrite each other's modules.

This is different from [stale-derived-data]. That entry is about the Swift
*version*. This one is about header *timestamps*, and it happens even with the
same Xcode version.

**Fix** — delete the module cache. Rebuilding it only takes seconds:

```bash
rm -rf ~/Library/Developer/Xcode/DerivedData/ModuleCache.noindex \
       "$DERIVED_DATA_PATH"/ModuleCache.noindex
```

In CI, leave `ModuleCache.noindex` out of the cache and keep caching the rest of
DerivedData and SwiftPM. Put the Xcode version (`xcodebuild -version`) in the
cache key, so each Xcode version gets its own cache.

**Rule:** the module cache belongs to one Xcode and one checkout — never restore it across runs; key every build cache on the Xcode version.

---

## [pif-transfer] Build service could not create build operation: unable to load transferred PIF

**Symptom** — the build stops before any compiling starts:

```
xcodebuild: error: Build service could not create build operation:
  unable to load transferred PIF: The workspace contains multiple references
  with the same GUID 'PACKAGE:1YUBE3S3A43QP92YAIBS0U5OJYK9I2OP2::MAINGROUP'
```

A related message comes from the same step:
`could not generate PIF because the workspace has not finished loading or is
still waiting for package resolution`.

**Cause** — Xcode turns the workspace into a PIF (Project Information File)
and gives it to the build service. Every project and Swift package in the PIF
gets a GUID. If two entries get the same GUID, or package resolution is still
running, the build service rejects the PIF:

- The same package appears twice. For example, a local package overrides a
  remote one with the same identity, or two projects each add the same package.
- `SourcePackages` still has old checkouts after a branch switch changed
  `Package.resolved`.
- `~/.gitconfig` has `[safe] bareRepository = explicit`. Some SourceTree
  updates add it. SwiftPM keeps its package cache as bare git repos, so this
  setting breaks the checkouts, and the error shows up as a GUID clash.
- `xcodebuild` is run without resolving packages first, so the workspace has
  not finished loading.

**Fix**

```bash
git config --global --get-all safe.bareRepository   # "explicit"? remove it:
git config --global --unset-all safe.bareRepository
rm -rf ~/Library/Caches/org.swift.swiftpm \
       ~/Library/Developer/Xcode/DerivedData/*/SourcePackages
xcodebuild -resolvePackageDependencies -workspace App.xcworkspace -scheme App
```

If it still fails, look for a package that appears twice. Check for the same
package URL, or a local path with the same name, in `project.pbxproj` and in
every `Package.swift`. In the Xcode app, use File → Packages → Reset Package
Caches.

This is different from [spm-resolution]. There, the dependencies cannot be
fetched. Here, they were fetched, but the build service cannot load the
workspace.

**Rule:** a PIF/GUID error is a package-graph problem, not a code problem — check `safe.bareRepository`, make every package appear exactly once, and resolve before building.

---

## [toolchains-override] A leftover `TOOLCHAINS` variable swaps Xcode's compiler

**Symptom** — `xcode-select -p` is correct, but the build uses a different
Swift, or fails at package resolution or linking:

```
xcodebuild: error: Could not resolve package dependencies:
  error: link command failed with exit code 1
clang: error: unable to execute command: posix_spawn failed: No such file or directory
```

Other symptoms: iOS builds fail with `no such module 'Swift'`. Or the build
passes in one terminal, fails in another, and always fails in CI.

**Cause** — `xcrun` and `xcodebuild` read the `TOOLCHAINS` environment variable.
When it names an `.xctoolchain` in `/Library/Developer/Toolchains`, they use
that toolchain instead of the one in Xcode. `xcode-select` has no effect on
this. Common ways it gets set:

- `swiftly` (1.2+) or a snapshot install exports `TOOLCHAINS` in the shell
  profile or the CI environment.
- A test harness or script passes its whole environment on to `xcodebuild`.

swift.org toolchains are built for macOS and Linux. They have no iOS standard
library, and some tools Xcode needs are missing. Package manifests always
load with Xcode's own SwiftPM, so the tools version can also disagree. A
`TOOLCHAINS` value for a toolchain that is not installed is ignored without
any warning, so the same shell can work on one machine and fail on another.

This is different from [wrong-developer-dir]. There, the wrong *Xcode* is
selected. Here, the right Xcode is selected, but its compiler is replaced.

**Fix**

```bash
echo "TOOLCHAINS=$TOOLCHAINS"; xcrun --find swiftc   # must be inside Xcode.app
env -u TOOLCHAINS xcodebuild ...                     # build without it
```

Remove the `export TOOLCHAINS=…` line from the shell profile or CI step. If
`swift` must stay on a snapshot, put that toolchain's `usr/bin` first on
`PATH` and do not set `TOOLCHAINS`, so `xcodebuild` keeps Xcode's toolchain.
Never ship an App Store build made with a non-Xcode toolchain.

**Rule:** `xcodebuild` should never see `TOOLCHAINS` — unset it for every Xcode build and check `xcrun --find swiftc` whenever the compiler seems wrong.

---

## [explicit-modules] Unable to find module dependency after moving to Xcode 26

**Symptom** — code that built on Xcode 16 now fails before it compiles:

```
error: Unable to find module dependency: 'Networking'
error: Compilation search paths unable to resolve module dependency: 'Testing'
```

The `import` is fine and the module exists. Often it only fails for
`build-for-testing`, for a device, or for macOS.

**Cause** — Xcode 26 turns on *explicitly built modules* for Swift
(`SWIFT_ENABLE_EXPLICIT_MODULES`); Xcode 16 already did it for C/ObjC
(`CLANG_ENABLE_EXPLICIT_MODULES`). Before compiling, the build scans every
`import` and builds each module from the target's own declared dependencies
and search paths. Before this, the compiler found modules in whatever
DerivedData already held, so these problems were hidden:

- A target imports a module it does not depend on. It worked because another
  target built that module first (transitive or lucky build order).
- `import Testing` or `import XCTest` in an app or library target.
- CocoaPods or hand-made framework search paths that point at a module
  without a module map the scanner can find. Newer clang also stops looking
  for module maps in SDK subdirectories.

**Fix**

```bash
# Confirm it is this: the build passes with explicit modules off
xcodebuild ... SWIFT_ENABLE_EXPLICIT_MODULES=NO CLANG_ENABLE_EXPLICIT_MODULES=NO
```

Then fix the graph, not the setting. Add the missing module to the target's
dependencies (Xcode target, or `dependencies:` in `Package.swift`). Remove test
framework imports from non-test targets. For pods, run `pod deintegrate && pod
install` on a current CocoaPods. Turn the setting off in the project only as a
short-term workaround.

This is different from [non-modular-header]. There, a module is found but
cannot be built. Here, the scanner cannot find the module at all.

**Rule:** after an Xcode upgrade, "unable to find module dependency" means an undeclared dependency — declare every module a target imports instead of relying on build order.

---

## [libarclite-missing] Linker fails: File not found libarclite_*.a

**Symptom** — every file compiles. Then the link step fails on a pod or old
framework target:

```
ld: file not found: /Applications/Xcode.app/Contents/Developer/Toolchains/
  XcodeDefault.xctoolchain/usr/lib/arc/libarclite_iphonesimulator.a
clang: error: linker command failed with exit code 1
```

It can also say `libarclite_iphoneos.a` for device builds, or
`libarclite_macosx.a` for Mac builds.

**Cause** — `libarclite` was a small helper library that let ARC code run on
very old OS versions. The linker adds it on its own when a target's
deployment target is old enough, which usually means iOS 8 or lower. Xcode
14.3 removed the `usr/lib/arc` folder, so the file the linker asks for is
gone. The app target is usually fine. The problem is a pod or vendored
project that still declares an ancient deployment target.

This is different from [deployment-target]. There, Xcode shows a warning
about the range. Here, the same old setting stops the build at link time.

**Fix**

```bash
# Find the targets that still declare an old minimum
grep -n "IPHONEOS_DEPLOYMENT_TARGET = [0-9]\." Pods/Pods.xcodeproj/project.pbxproj | sort -u
```

Raise only those targets in the Podfile `post_install` hook. Leave pods that
already declare a higher value alone:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |t|
    t.build_configurations.each do |c|
      if c.build_settings['IPHONEOS_DEPLOYMENT_TARGET'].to_f < 12.0
        c.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '12.0'
      end
    end
  end
end
```

Then run `pod install`. For a vendored `.xcodeproj`, raise its deployment
target directly. Do not copy the `arc` folder from an old Xcode into the new
one. The next Xcode update removes it again, and a CI image never has it.

**Rule:** a missing `libarclite` means a dependency still targets iOS 8 — raise that target's minimum, never bring the library back.

---

## [spm-unsafe-flags] Package product uses unsafe build flags

**Symptom** — package resolution succeeds, then the build (or Xcode's
package graph) refuses one dependency:

```
error: The package product 'Foo' cannot be used as a dependency of this
target because it uses unsafe build flags.
```

It often appears right after bumping a package to a new tag, while the old
tag built fine.

**Cause** — the package's `Package.swift` sets `unsafeFlags(...)` in
`swiftSettings`, `cSettings` or `linkerSettings`. SwiftPM allows those flags
only in the root package, a local (path) package, or a remote package pinned
by branch or revision. A remote package required **by version** (`from:`,
`exact:`, a range) is rejected. The maintainer usually added a flag such as
`-enable-testing` or `-warnings-as-errors` and tagged a release by mistake.

This is different from [spm-resolution]. There, SwiftPM cannot pick a version.
Here, the version resolves; the manifest is then refused.

**Fix**

```bash
# Confirm which dependency declares the flags
grep -rn "unsafeFlags" ~/Library/Developer/Xcode/DerivedData/*/SourcePackages/checkouts/*/Package.swift
```

Pin the previous tag that has no `unsafeFlags`, or wait for a patch release
and report it upstream. As a stopgap, pin by `revision:` to the same commit.
If you own the package, move the flag behind a condition or drop it before
tagging. Do not vendor the package just to get around the check.

**Rule:** `unsafeFlags` are for local work — a tagged release that ships them breaks every version-based consumer.

---

## [swift-version-unsupported] SWIFT_VERSION missing or no longer supported

**Symptom** — the build stops before any Swift file compiles, naming one
target (often a pod):

```
error: SWIFT_VERSION '3.0' is unsupported, supported versions are: 4.0, 4.2, 5.0, 6.0.
  (in target 'Foo' from project 'Pods')
error: The "Swift Language Version" (SWIFT_VERSION) build setting must be
  set to a supported value for targets which use Swift.
```

**Cause** — every Swift target needs a `SWIFT_VERSION` language mode that the
current compiler still accepts. The first error means the target asks for a
mode that was dropped (3.x went with Xcode 10.2). The second means the target
has no value at all. Common sources: an old pod whose podspec has no
`swift_version`, a leftover `.swift-version` file, or a value set only in an
xcconfig that `pod install` did not carry into `Pods.xcodeproj`.

This is not a compiler mismatch like [stale-derived-data]. The setting is
checked before compiling, so wiping DerivedData does nothing.

**Fix**

```bash
# See which targets have a bad or empty value
xcodebuild -showBuildSettings -workspace App.xcworkspace -scheme App 2>/dev/null \
  | grep -E "TARGET_NAME|SWIFT_VERSION"
```

For your own targets, set the value in the project or xcconfig. For pods,
prefer a release that declares `swift_version`. Otherwise, set it in the
Podfile and run `pod install`:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |t|
    next unless t.name == 'Foo'
    t.build_configurations.each { |c| c.build_settings['SWIFT_VERSION'] = '5.0' }
  end
end
```

Use the oldest mode the code builds in (usually `5.0`). Do not set `6.0`
everywhere to silence the error; Swift 6 mode turns on strict concurrency
checking and adds new errors.

**Rule:** every Swift target must name a supported language mode — pin it per target, never trust a pod to inherit yours.

---

## [spm-platform-minimum] Package requires a higher minimum platform than the target

**Symptom** — resolution succeeds, then the build fails on the target that
links the package:

```
error: The package product 'Foo' requires minimum platform version 16.0 for
  the iOS platform, but this target supports 15.0 (in target 'Widget' from project 'App')
```

For a prebuilt module (an `.xcframework` or binary target) it shows up at
compile time instead:

```
error: compiling for iOS 15.0, but module 'Foo' has a minimum deployment target of iOS 16.0
```

**Cause** — a package's `platforms:` line in `Package.swift` sets a minimum OS,
and every target that links one of its products must meet it. This usually
starts after a package bump that raised the floor, even as a minor version. It
also happens when the app is fine but a smaller target is not. An extension,
widget, or test target that kept an older `IPHONEOS_DEPLOYMENT_TARGET` is
checked on its own.

This is the reverse of [deployment-target]. There, your minimum is too old for
Xcode. Here, your minimum is too old for the package. Clearing caches does not help.

**Fix**

```bash
# Which targets link the package, and at what minimum?
xcodebuild -showBuildSettings -workspace App.xcworkspace -scheme App 2>/dev/null \
  | grep -E "TARGET_NAME|IPHONEOS_DEPLOYMENT_TARGET"
# What the resolved package actually asks for
grep -A4 "platforms" ~/Library/Developer/Xcode/DerivedData/App-*/SourcePackages/checkouts/Foo/Package.swift
```

Either raise the failing target's deployment target to the package's floor, or
pin the package to the last release that still supports your minimum
(`exact:` or `.upToNextMinor(from:)`) and commit `Package.resolved`. Keep all
targets that share packages on the same minimum, ideally set once in a shared
xcconfig.

**Rule:** a package bump can raise your minimum OS — check its `platforms:` before accepting the update, and keep every linking target on one floor.

---

## [xcframework-create] -create-xcframework rejects the archived frameworks

**Symptom** — each archive succeeds, then packaging them fails:

```
error: the path does not point to a valid framework: /…/ios.xcarchive/Products/Library/Frameworks/Foo.framework
error: Both 'ios-arm64-simulator' and 'ios-x86_64-simulator' represent two equivalent library definitions.
error: binaries with multiple platforms are not supported '/…/Foo.framework/Foo'
```

Or it passes, but apps on a newer Xcode fail with "module compiled with
Swift X cannot be imported" (→ [stale-derived-data], the prebuilt case).

**Cause** — an XCFramework holds one slice per *platform variant* (device,
simulator, Mac Catalyst), and each slice must be a real, single-platform
framework. The usual mistakes:

- `SKIP_INSTALL` is `YES` (the default for frameworks), so the archive has no
  `Products/Library/Frameworks` folder. The path in the first error is empty.
- Old fat-framework scripts: one archive per *architecture*, or a `lipo`-merged
  device + simulator binary. Arm64 and x86_64 simulator builds are one variant
  and must be one binary.
- `BUILD_LIBRARY_FOR_DISTRIBUTION` is off, so no `.swiftinterface` is shipped.
  It builds, but only on the exact Swift compiler that made it.

**Fix** — one archive per platform, then combine:

```bash
for p in "iOS" "iOS Simulator"; do
  xcodebuild archive -scheme Foo -destination "generic/platform=$p" \
    -archivePath "build/$p.xcarchive" \
    SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES
done
xcodebuild -create-xcframework \
  -framework "build/iOS.xcarchive/Products/Library/Frameworks/Foo.framework" \
  -framework "build/iOS Simulator.xcarchive/Products/Library/Frameworks/Foo.framework" \
  -output build/Foo.xcframework
```

A generic simulator destination already builds both simulator architectures
into one binary. Add `-debug-symbols <absolute path to .dSYM>` after each
`-framework` to ship dSYMs; relative paths are rejected.

**Rule:** one archive per platform with `SKIP_INSTALL=NO` and `BUILD_LIBRARY_FOR_DISTRIBUTION=YES` — never `lipo` device and simulator together.

---

## [infoplist-missing] Cannot code sign: target has no Info.plist

**Symptom** — a target fails at the signing step, often a pod, a test
bundle, or a target made by a generator (CocoaPods, Tuist, XcodeGen):

```
error: Cannot code sign because the target does not have an Info.plist file
  and one is not being generated automatically. Apply an Info.plist file to
  the target using the INFOPLIST_FILE build setting or generate one
  automatically by setting the GENERATE_INFOPLIST_FILE build setting to YES
  (recommended). (in target 'Foo' from project 'Pods')
```

**Cause** — every signed bundle needs an `Info.plist`. It comes from one of
two settings: `INFOPLIST_FILE` (a file on disk) or
`GENERATE_INFOPLIST_FILE = YES` (Xcode writes one). Here the target has
neither for this configuration. Common ways to get there:

- The target was made before Xcode 13 and never had `INFOPLIST_FILE`
  set. Older Xcode did not complain; Xcode 14+ does. `pod lib lint` test
  and app-host targets are a well-known case.
- `INFOPLIST_FILE` is set only in some configurations. A new `Staging`
  configuration copied from nothing, or an xcconfig not attached to it,
  leaves it empty.
- A generator spec lost the setting when it was upgraded.

This is not [missing-input-file]. There, `INFOPLIST_FILE` points to a file
that is gone. Here the setting is empty.

**Fix**

```bash
# Which targets and configurations have neither setting?
xcodebuild -showBuildSettings -workspace App.xcworkspace -scheme App \
  -configuration Staging 2>/dev/null \
  | grep -E "TARGET_NAME|INFOPLIST_FILE|GENERATE_INFOPLIST_FILE"
```

For your own targets, set `GENERATE_INFOPLIST_FILE = YES`. You can keep
`INFOPLIST_FILE` too; Xcode merges the file with the generated keys. For
pods, first update to a release that fixes it. Otherwise, set it in
`post_install`:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |t|
    t.build_configurations.each do |c|
      c.build_settings['GENERATE_INFOPLIST_FILE'] ||= 'YES'
    end
  end
end
```

`INFOPLIST_KEY_*` settings (for example `INFOPLIST_KEY_CFBundleDisplayName`)
only work when `GENERATE_INFOPLIST_FILE = YES`. With it off they are
silently ignored.

**Rule:** every signed target needs an Info.plist in every configuration — set `GENERATE_INFOPLIST_FILE = YES` unless a checked-in file is required.

---

## [bitcode-rejected] Upload rejected: the executable contains bitcode

**Symptom** — the archive builds, but App Store Connect refuses it. You see
this from `-exportArchive` with `destination = upload`, or from
`altool`/Transporter:

```
ITMS-90482: Invalid Executable - The executable
  'App.app/Frameworks/Foo.framework/Foo' contains bitcode.
```

**Cause** — Xcode 14 deprecated bitcode, and uploads made with Xcode 16 or
later reject any binary that still has it. Your own targets are almost never
the problem: Xcode 14+ ignores `ENABLE_BITCODE = YES` and only warns. The
bitcode comes from a **prebuilt** binary — a vendored `.framework` or
`.xcframework` (an old analytics or ads SDK, Hermes in older React Native)
that was compiled with bitcode years ago.

**Fix**

```bash
# Find which embedded binaries still have a bitcode section
for f in App.xcarchive/Products/Applications/App.app/Frameworks/*.framework; do
  n=$(basename "$f" .framework)
  otool -l "$f/$n" | grep -q __LLVM && echo "bitcode: $n"
done
```

The best fix is to update that SDK to a release built without bitcode. If
you cannot, strip it before the binary is embedded and signed (stripping a
signed binary breaks its signature):

```bash
xcrun bitcode_strip -r Foo.framework/Foo -o Foo.framework/Foo
```

With CocoaPods, run that on the vendored binary under `Pods/` in
`post_install`. Also set `ENABLE_BITCODE = NO` there to silence the warning.

**Rule:** only prebuilt binaries still carry bitcode — check them with `otool -l | grep __LLVM` before you upload, then update or strip them.

---

## [tcc-protected-folder] Operation not permitted when run from launchd, cron or SSH

**Symptom** — the same command works in Terminal but fails when a
LaunchAgent, a cron job, a self-hosted CI runner or an SSH session runs it.
It fails before anything compiles:

```
xcodebuild: error: Unable to read project 'App.xcodeproj'.
  Reason: The file "project.pbxproj" couldn't be opened because you don't
  have permission to view it.
shell-init: error retrieving current directory: getcwd: cannot access
  parent directories: Operation not permitted
```

**Cause** — macOS privacy protection (TCC). `~/Desktop`, `~/Documents`,
`~/Downloads`, iCloud Drive and external volumes need a consent grant for
the app that started the process. Terminal already has that grant, but a
process started by launchd, cron or `sshd` does not. There is no one to
click the prompt, so the read fails with `EPERM`. `chmod`, `sudo` and
running as root do not help. The Unix permissions are fine; TCC is what
blocks it. This is not [script-sandbox], which only blocks writes from
build-phase scripts.

**Fix** — confirm it from the same context, then move the checkout or
grant access:

```bash
# Run this from the agent / cron / ssh context, not from Terminal
ls ~/Desktop/App >/dev/null && echo ok
# See what TCC denied
log show --last 5m --predicate 'subsystem == "com.apple.TCC"' | grep -i deny
```

- Best: keep the checkout outside the protected folders, for example
  `~/Developer/App` or `~/src/App`. No grant is needed, and it keeps working
  after OS updates.
- If you cannot move it, go to System Settings → Privacy & Security → Full
  Disk Access and add the binary that starts the job: `/usr/sbin/cron`,
  `/bin/bash`, `/bin/zsh`, the runner's own binary or its bundled `node`.
  Runner updates change that path, so you will have to add it again.
- For SSH, turn on System Settings → General → Sharing → Remote Login (i) →
  "Allow full disk access for remote users".

**Rule:** background builds never touch `~/Desktop`, `~/Documents` or `~/Downloads` — keep checkouts in `~/Developer`, and do not rely on a Full Disk Access grant that the next update can break.

---

## [wwdr-chain-broken] Unable to build chain to self-signed root for signer

**Symptom** — the identity is in the keychain and the keychain is unlocked,
but signing still fails:

```
Warning: unable to build chain to self-signed root for signer "Apple Distribution: Acme (ABCDE12345)"
/…/App.app: errSecInternalComponent
Command CodeSign failed with a nonzero exit code
```

Keychain Access shows the certificate as "not trusted" or "issued by an
unknown authority". Often on a new CI runner or a new Mac, or after someone
changed trust settings by hand.

**Cause** — `codesign` must build the chain leaf → Apple WWDR intermediate →
Apple Root CA. Apple now issues certificates from several intermediates
(WWDR G3–G6), and a fresh machine or a minimal CI keychain often has only an
old one, or none. The chain breaks the same way if someone set the leaf,
intermediate or root to *Always Trust*: custom trust settings make the
chain invalid for code signing. This is not [keychain-locked]: the same
`errSecInternalComponent` appears there, but without the "build chain"
warning.

**Fix** — see which intermediate signed the certificate, then install it:

```bash
security find-identity -v -p codesigning                       # identity listed, but "0 valid"?
security find-certificate -c "Apple Distribution: Acme" -p | openssl x509 -noout -issuer
curl -sO https://www.apple.com/certificateauthority/AppleWWDRCAG3.cer   # match the issuer's Gn
sudo security add-certificates -k /Library/Keychains/System.keychain AppleWWDRCAG3.cer
```

On CI, import the intermediate into the same keychain as the identity (or
into System). In Keychain Access, set every Apple certificate in the chain
back to *Use System Defaults*. Never set them to *Always Trust*.

**Rule:** "build chain to self-signed root" means the Apple intermediate is missing or its trust was changed — install the issuer's WWDR certificate during machine setup, and leave trust settings at system defaults.

---

## [too-many-open-files] Random files fail to open: Too many open files

**Symptom** — a large build fails at a different file on every run, usually
on a CI runner or after adding many packages:

```
error: unable to open output file '/…/Foo.o': 'Too many open files'
fatal error: error opening input file '/…/Bar.swift' (Too many open files)
error: accessing build database: disk I/O error
```

A clean retry, or a build with fewer packages, gets further and then fails
somewhere else.

**Cause** — the per-process limit on open file descriptors. macOS gives
shells a soft limit of 256 (`ulimit -n`), and launchd jobs inherit
`launchctl limit maxfiles`, which is also low by default. With parallel
jobs, many SPM modules, module caches and the build database all open at
once, the build passes that limit. The disk and the files are fine. This is
not [disk-space], and not [db-locked]: the database error is only one more
open that failed.

**Fix** — check the limit in the context that runs the build, then raise
it there:

```bash
ulimit -n; launchctl limit maxfiles      # 256 here is the problem
ulimit -n 65536 && xcodebuild ...         # this shell and its children only
sudo launchctl limit maxfiles 65536 524288   # whole system, until reboot
```

To keep it after a reboot, add a LaunchDaemon in `/Library/LaunchDaemons`
that runs `launchctl limit maxfiles 65536 524288` at load. Restart the CI
runner service afterwards, because a running job keeps its old limit. As a
stopgap, `-jobs 4` opens fewer files at once.

**Rule:** "Too many open files" is a limit, not a broken file — raise `ulimit -n` in the CI runner's start script and never debug the file the error names.

---

## [path-with-spaces] Build breaks only when the checkout path has a space

**Symptom** — the same commit builds on one machine and fails on another,
or fails only on CI jobs whose names contain a space:

```
/…/Script-1A2B.sh: line 2: /Users/ci/My: No such file or directory
Command PhaseScriptExecution failed with a nonzero exit code
fatal error: 'Foo/Foo.h' file not found
ld: warning: directory not found for option '-LApp'
```

`pwd` shows something like `/Users/ci/My App/` or `~/Desktop/Work Projects/`.

**Cause** — Xcode itself handles spaces, but two things around it do not:

- a run-script phase that uses a path without quotes, like
  `${SRCROOT}/scripts/lint.sh`. The shell splits it at the space and runs
  `/Users/ci/My`.
- a search-path setting (`HEADER_SEARCH_PATHS`, `FRAMEWORK_SEARCH_PATHS`,
  `LIBRARY_SEARCH_PATHS`, `OTHER_LDFLAGS`) in an `.xcconfig` written as
  `$(SRCROOT)/Vendor/My Lib`. These settings are lists split at spaces, so
  Xcode searches `…/Vendor/My` and `Lib`.

This is not [script-path-missing] (the tool exists) and not
[missing-input-file] (the file exists). The path was split in two.

**Fix** — confirm it first: clone into a path with no space and build
again. If that passes, quote every path:

```bash
"${SRCROOT}/scripts/lint.sh" "${BUILT_PRODUCTS_DIR}/${WRAPPER_NAME}"
```

```
HEADER_SEARCH_PATHS = $(inherited) "$(SRCROOT)/Vendor/My Lib"
```

To find the scripts that need quotes, run
`grep -n 'shellScript' *.xcodeproj/project.pbxproj` and look for `${SRCROOT}`
or `${PROJECT_DIR}` with no `\"` before it.
On CI, set the workspace to a path with no spaces, so a branch or job name
never ends up in the path.

**Rule:** quote every `${…}` path in build scripts and xcconfig lists, and give CI a workspace path with no spaces.

---

## [lfs-pointer] Binaries are Git LFS pointer files, not the real files

**Symptom** — the code compiles, then a vendored library or asset fails as
if it were broken. This often happens on a new clone or a CI runner:

```
ld: warning: ignoring file …/Vendor/libFoo.a, building for iOS-arm64 but
attempting to link with file built for unknown-unsupported file format
( 0x76 0x65 0x72 0x73 0x69 0x6F 0x6E 0x20 … )
Undefined symbols for architecture arm64: "_OBJC_CLASS_$_Foo"
```

Images or `.mlmodel` files in the same repo can fail too, with errors like
"not a valid PNG" or "could not be opened".

**Cause** — the repo stores large files in Git LFS, but the checkout never
downloaded them. Each file on disk is a small text pointer (under 200 bytes)
that starts with `version https://git-lfs…`. The bytes `0x76 0x65 0x72 0x73`
are the ASCII for "vers". This happens when `git-lfs` is not installed on
the machine, when CI checks out with LFS off (`actions/checkout` has
`lfs: false` by default), or when `GIT_LFS_SKIP_SMUDGE=1` is set.

This is not [undefined-symbols] (the symbols are in the real library) and
not [simulator-arch] (the file has no architecture at all; it is text).

**Fix** — check that it is a pointer, then download the real files:

```bash
file Vendor/libFoo.a     # "ASCII text" means it is a pointer
git lfs ls-files         # a "-" after the hash means not downloaded
git lfs install && git lfs pull
```

On GitHub Actions use `actions/checkout` with `lfs: true`. On other CI, add
`git lfs pull` after the clone. If the file comes from a Swift package that
uses LFS, SPM does not download LFS files. Ship it as a `binaryTarget` zip
instead.

**Rule:** when the linker sees a library as "unknown file format", run `file` on it first; an ASCII-text binary means `git lfs pull`.

---

## [timestamp-unavailable] codesign fails: The timestamp service is not available

**Symptom** — compiling and linking pass, then the signing step fails. It
happens most with archives, Developer ID builds and macOS apps, and often on
CI runners with locked-down network access:

```
…/MyApp.app: The timestamp service is not available.
Command CodeSign failed with a nonzero exit code
```

It can pass on a rerun without any change.

**Cause** — signing for distribution adds a secure timestamp
(`codesign --timestamp`). To do that, `codesign` must reach Apple's
timestamp server, `timestamp.apple.com`, over the network. The step fails
when the machine is offline, when a firewall or proxy blocks that host
(`codesign` does not read `HTTP_PROXY`), when IPv6 to the host is broken,
or when Apple's server is briefly down.

This is not [keychain-locked] (the key was found and used) and not
[wwdr-chain-broken] (the certificate chain is fine). It is only the network
call for the timestamp.

**Fix** — check that the host is reachable from the machine that builds:

```bash
curl -sI http://timestamp.apple.com/ts01 | head -1   # any HTTP reply = reachable
```

- On CI, allow `timestamp.apple.com` through the firewall. If IPv6 is the
  problem, force IPv4 for that host.
- If Apple's server is down, wait a few minutes and retry. A CI script can
  retry the sign step 2–3 times.
- For local or debug builds only, you can skip the timestamp with
  `OTHER_CODE_SIGN_FLAGS="--timestamp=none"`. Never do this for a build you
  ship: notarization rejects code with no secure timestamp.

**Rule:** "timestamp service is not available" is a network problem — check that `timestamp.apple.com` is reachable and retry; use `--timestamp=none` only for builds you never ship.

---

## Fast triage

Work down this list before deep-diving a log:

1. `xcode-select -p` — right Xcode? → [wrong-developer-dir]; `echo $TOOLCHAINS` empty? → [toolchains-override]
2. `xcodebuild -list` — scheme actually exists and is shared? → [scheme-not-found]
3. `xcodebuild -showdestinations` — destination actually exists? → [no-matching-destination]
4. `df -h` — disk not full? → [disk-space]; `ulimit -n` only 256? → [too-many-open-files]
5. Wipe DerivedData, retry once. → [stale-derived-data]; space in `pwd`? → [path-with-spaces]; binaries are "ASCII text"? → [lfs-pointer]
6. Still failing? `grep -nE "error:" build.log | head -30` and read the *first* error.

If steps 1–5 change the outcome, it was the environment. If they do not, it is
the code — and only then is the diff worth reading.
