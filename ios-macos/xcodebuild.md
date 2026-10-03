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

## [pipe-masks-exit] CI is green, but the build failed

**Symptom** — the log shows `** BUILD FAILED **` or `** TEST FAILED **`,
but the CI step passes and the next steps run with no app or no results.
Or the opposite: the build passes and the step fails with a Ruby error from
the formatter:

```
** BUILD FAILED **
… step finished, exit code 0

xcpretty: invalid byte sequence in US-ASCII (ArgumentError)
```

**Cause** — a shell pipeline returns the exit code of its *last* command.
In `xcodebuild … | tee build.log | xcpretty`, that is `xcpretty`, which
exits 0 after printing the failure. `set -e` does not help, because it only
checks that same last code. GitHub Actions adds `-o pipefail` only when the
step says `shell: bash`; the default `bash -e {0}` does not. The Ruby error
is a separate bug: `xcpretty` crashes on non-ASCII log text (emoji, accented
paths) when the runner has no UTF-8 locale.

This is not a real build error. Read the raw log
(see "Before anything else") to find the real one.

**Fix** — make the pipeline fail when `xcodebuild` fails:

```bash
set -o pipefail
export LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8   # stops the xcpretty crash
xcodebuild … 2>&1 | tee build.log | xcbeautify
```

- On GitHub Actions, set `shell: bash` on the step, or add `set -o pipefail`
  at the top of the script.
- To keep the pipe and still check the code: `${PIPESTATUS[0]}` in bash,
  `$pipestatus[1]` in zsh.

**Rule:** every `xcodebuild | formatter` line needs `set -o pipefail`; a green step with `BUILD FAILED` in the log is the pipe, not the build.

---

## [rosetta-shell] Simulator build picks x86_64 on an Apple silicon Mac

**Symptom** — the same command builds in Xcode but fails from a terminal or
CI agent. The build targets `x86_64` even though the Mac is Apple silicon:

```
error: Could not find module 'Foo' for target 'x86_64-apple-ios-simulator';
found: arm64-apple-ios-simulator
ld: building for 'iOS-simulator', but linking in object file built for … arm64
```

`-showdestinations` lists simulators with `arch:x86_64`.

**Cause** — the shell that ran `xcodebuild` is running under Rosetta, so
`xcodebuild` runs as an x86_64 process. With `ONLY_ACTIVE_ARCH=YES` the
"active" arch is now `x86_64`. Prebuilt modules and XCFrameworks that ship
only an `arm64` simulator slice then do not match. Common sources: a
Terminal app set to "Open using Rosetta", an x86_64 Homebrew in `/usr/local`
whose `bash`/`ruby` starts the build, or an x86_64 Java for the CI agent
(Jenkins, TeamCity).

This is not [simulator-arch] (that is a device build linked into a simulator
app). Here the build is a simulator build for the wrong CPU.

**Fix** — check, then run the build natively:

```bash
sysctl -n sysctl.proc_translated   # 1 = this shell is under Rosetta
uname -m                           # x86_64 here on Apple silicon = Rosetta
arch -arm64 xcodebuild …           # one-off native run
```

- Turn off "Open using Rosetta" for Terminal/iTerm; install arm64 Homebrew
  in `/opt/homebrew`; use an arm64 JDK for the CI agent.
- To force the arch, pin it in the destination:
  `-destination 'platform=iOS Simulator,name=iPhone 16,arch=arm64'`.
- Do not "fix" it by adding `EXCLUDED_ARCHS[sdk=iphonesimulator*]=arm64`;
  that hides the problem and breaks native builds.

**Rule:** a simulator build that targets `x86_64` on Apple silicon means the shell is under Rosetta — check `sysctl.proc_translated`, then run `arch -arm64`.

---

## [privacy-manifest] Upload accepted, then rejected: ITMS-91053 / ITMS-91061

**Symptom** — `archive`, `-exportArchive` and the upload all succeed. Minutes
later App Store Connect emails that the build is invalid:

```
ITMS-91053: Missing API declaration - Your app's code in the "Foo" file
references one or more APIs that require reasons, including …
NSPrivacyAccessedAPICategoryUserDefaults
ITMS-91061: Missing privacy manifest - … "Frameworks/Bar.framework/Bar" …
```

**Cause** — Apple scans the uploaded binary, not your project. Two checks:

- **91053** — some code (yours or an SDK's) calls a "required reason" API
  (`UserDefaults`, file timestamps, system boot time, disk space, active
  keyboards) and no `PrivacyInfo.xcprivacy` in the bundle declares a reason.
- **91061** — an SDK on Apple's list of common third-party SDKs is embedded
  without its own privacy manifest (and signature), usually an old version.

A manifest that exists but never reaches the bundle fails the same way: a
static library or static framework drops its manifest unless it ships it in
a resource bundle, and a file not in a target's Copy Bundle Resources is not
copied.

This is not [export-archive]: nothing fails locally, so CI stays green.

**Fix** — see what the archive really contains, then fill the gaps:

```bash
# Every manifest that made it into the built app
find Build.xcarchive/Products/Applications -name PrivacyInfo.xcprivacy
```

- Xcode Organizer → right-click the archive → **Generate Privacy Report**
  shows the merged result; any API category missing there is what 91053 names.
- Your code: add `PrivacyInfo.xcprivacy` to the app target with
  `NSPrivacyAccessedAPITypes` and a reason code (e.g. `CA92.1` for
  `UserDefaults` used only by the app).
- SDKs: upgrade to a version that ships a manifest. For a static pod, the
  manifest must be in `resource_bundles`, not `resources`.

**Rule:** the App Store checks the binary — confirm every `PrivacyInfo.xcprivacy` is inside the `.xcarchive` before uploading, not just in the repo.

---

## [swift-tools-version] Package needs a newer Swift than this Xcode has

**Symptom** — resolution stops before anything compiles:

```
xcodebuild: error: Could not resolve package dependencies:
  package 'swift-foo' is using Swift tools version 6.0.0 but the installed
  version is 5.10.0 in https://github.com/acme/swift-foo
```

**Cause** — the first line of a dependency's `Package.swift`
(`// swift-tools-version:6.0`) is the minimum SwiftPM that may read it, and
SwiftPM is tied to Xcode (Xcode 15.3 → 5.10, Xcode 16 → 6.0). It usually
appears without anyone touching Xcode:

- The package raised its tools version in a minor or patch release, and an
  unpinned resolve (CI, `rm Package.resolved`, "Update to Latest") pulled it.
- A developer on a newer Xcode resolved and committed `Package.resolved`; CI
  still runs the older Xcode.

Packages can keep older Xcodes working with `Package@swift-5.10.swift`; most
do not.

This is not [spm-resolution] (cache, auth, network) and not
[toolchains-override] (a swapped compiler): the manifest is fine, this Xcode
is just too old to read it.

**Fix**

```bash
xcrun swift --version          # the SwiftPM this Xcode really has
# Which checked-out package asks for more?
head -1 ~/Library/Developer/Xcode/DerivedData/*/SourcePackages/checkouts/*/Package.swift
```

Either move CI to the Xcode that matches the team, or pin the package to the
last release with a supported tools version (`.upToNextMinor(from:)` or
`exact:`) and keep `-onlyUsePackageVersionsFromResolvedFile` on CI.

**Rule:** treat `Package.resolved` and the CI Xcode version as one unit — bump them in the same PR, never one alone.

---

## [spm-fingerprint-mismatch] Revision does not match previously recorded value

**Symptom** — resolution fails on one machine (or one runner), works on a fresh one:

```
xcodebuild: error: Could not resolve package dependencies:
  Revision 39abfc9... for swift-foo remoteSourceControl https://github.com/acme/swift-foo
  version 2.0.0 does not match previously recorded value ccf49c3...
```

**Cause** — SwiftPM is trust-on-first-use: the first time it resolves a
version it records that tag's commit under
`~/Library/org.swift.swiftpm/security/fingerprints/`. If the maintainer later
deletes and re-pushes the tag at another commit, every machine that already
saw the old one refuses the new one. Fresh CI runners and new laptops have no
record, so it "only fails for some people".

This is not [spm-resolution] (network, auth, cache) and not
[macro-plugin-trust] (the same `security/` folder, but a different record):
the tag itself changed under you.

**Fix**

```bash
# Which package, and what did it record?
ls ~/Library/org.swift.swiftpm/security/fingerprints/ | grep -i swift-foo
# Drop only that package's record, then resolve again
rm ~/Library/org.swift.swiftpm/security/fingerprints/swift-foo-*.json
xcodebuild -resolvePackageDependencies -scheme App
```

Check the new commit is one you expect before trusting it — a moved tag is
exactly what the check exists to catch. Ask the maintainer to cut a new
version instead of re-tagging; if they will not, pin by `revision:`.

**Rule:** a released tag is immutable — when it moves, verify the new commit, clear that one fingerprint, and never wipe the whole `security/` folder by reflex.

---

## [xcresulttool-legacy] Tests pass, then the report step fails: --legacy flag is required

**Symptom** — the build and tests succeed after an Xcode 16 upgrade, but the
step that reads the result bundle fails (or posts an empty report):

```
Error: This command is deprecated and will be removed in a future release,
--legacy flag is required to use it.
Usage: xcresulttool get object [--legacy] --path <path> ...
```

**Cause** — Xcode 16 deprecated the old `xcresulttool get --format json`
object graph and `export` form. They now refuse to run without `--legacy`.
Your own scripts and older report tools (fastlane `trainer`, Danger plugins,
`xcparse`, test-report actions) call the old form, so the step after the tests
breaks, not `xcodebuild`.

This is not [result-bundle-exists] (writing the bundle) and not
[pipe-masks-exit] (here the report step exits non-zero; the tests were fine).

**Fix**

```bash
# Unblock today: keep the old command, add the flag
xcrun xcresulttool get --legacy --format json --path Build.xcresult
# Move to the supported commands
xcrun xcresulttool get test-results summary --path Build.xcresult
xcrun xcresulttool get test-results tests   --path Build.xcresult
xcrun xcresulttool get build-results        --path Build.xcresult
```

Bump report tools to a version that supports Xcode 16 before adding
`--legacy` by hand. The flag is a stopgap — Apple says the old form will be
removed.

**Rule:** when a CI report step breaks after an Xcode upgrade, check the result-bundle reader first — move to `get test-results`, and use `--legacy` only as a stopgap.

---

## [build-number-reused] Upload rejected: Redundant Binary Upload / train closed

**Symptom** — archive and export succeed, the upload step is refused:

```
ERROR ITMS-4238: "Redundant Binary Upload. You've already uploaded a build
  with build number '412' for version number '1.2.0'."
ERROR ITMS-90186: "Invalid Pre-Release Train. The train version '1.2.0' is
  closed for new build submissions"
ERROR ITMS-90062: "The value for key CFBundleShortVersionString [1.2.0] in
  the Info.plist file must contain a higher version than that of the
  previously approved version [1.2.0]."
```

**Cause** — App Store Connect keys every upload by (version, build). 4238:
that build number was already used for this version — often a re-run CI job,
or two branches counting from the same base. 90186 / 90062: that version is
already released, so it takes no new builds at all. The sneaky case is a
bump that never lands: an Info.plist with a literal `CFBundleVersion` instead
of `$(CURRENT_PROJECT_VERSION)` ignores the setting you passed to
`xcodebuild`, so the archive keeps the old number.

This is not [export-archive] (signing / export options; here the upload
itself is refused) and not [embedded-binary-mismatch] (extension and app
disagree; here the whole app's number is the problem).

**Fix**

```bash
# What actually went into the archive?
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' -c 'Print :CFBundleVersion' \
  App.xcarchive/Products/Applications/App.app/Info.plist
# Bump at archive time — applies to every target, extensions included
xcodebuild archive -scheme App -archivePath App.xcarchive \
  CURRENT_PROJECT_VERSION="$CI_BUILD_NUMBER"
```

If the printed number did not change, replace the literals in every
Info.plist with `$(CURRENT_PROJECT_VERSION)` / `$(MARKETING_VERSION)`. Use a
number that only grows (CI run number, or latest TestFlight build + 1), never
a git commit count. For 90186 / 90062, bump `MARKETING_VERSION` instead.

**Rule:** read the version and build from the archive, not the project — derive the build number from something that only grows, and pass it in at archive time.

---

## [sdk-too-old] Upload rejected: ITMS-90725 SDK version issue

**Symptom** — archive, export and signing all pass; App Store Connect warns,
then (after Apple's deadline) refuses the build:

```
ITMS-90725: SDK version issue - This app was built with the iOS 18.5 SDK.
All iOS and iPadOS apps must be built with the iOS 26 SDK or later,
included in Xcode 26 or later, in order to be uploaded to App Store Connect.
```

**Cause** — Apple raises the minimum *build* SDK every spring (iOS 18 SDK from
April 2025, iOS 26 SDK / Xcode 26 from April 28, 2026). The SDK comes from
whichever Xcode ran `xcodebuild`, so a CI image or `xcode-select` still on the
old Xcode keeps shipping old-SDK builds. Your deployment target does not
matter here: you can build with the new SDK and still support old iOS.

This is not [deployment-target] (minimum OS the app runs on) and not
[build-number-reused] (the number is refused, not the SDK).

**Fix**

```bash
# Which SDK and Xcode built this archive?
/usr/libexec/PlistBuddy -c 'Print :DTSDKName' -c 'Print :DTXcode' \
  App.xcarchive/Products/Applications/App.app/Info.plist
# Point the runner at the new Xcode, then archive again
sudo xcode-select -s /Applications/Xcode_26.app && xcodebuild -version
```

Pin the Xcode version in CI config (runner image, `.xcode-version`, fastlane
`xcversion`) so a cached old image cannot win. Expect new warnings and
dependency bumps on the first build with the new SDK — upgrade before the
warning becomes a rejection.

**Rule:** when ITMS-90725 appears as a warning, plan the Xcode upgrade then — check `DTSDKName` in the archive, not the Xcode you have on your laptop.

---

## [notarization-invalid] macOS: notarytool says Invalid, stapler fails

**Symptom** — a Developer ID macOS build archives, exports and signs fine,
then notarization rejects it and stapling fails:

```
Processing complete
  status: Invalid
...
CloudKit query for App.app (2/...) failed due to "Record not found".
The staple and validate action failed! Error 65.
```

**Cause** — the notary service checks every Mach-O inside the bundle, not
only the app. Notarization needs the hardened runtime, a Developer ID
Application certificate, a secure timestamp, and no
`com.apple.security.get-task-allow` entitlement. Usually one of these is
missing: a Debug build was exported, `ENABLE_HARDENED_RUNTIME` is off, or a
nested helper, framework or CLI tool was copied in after signing or signed
with `--timestamp=none`. Stapler error 65 just means there is no accepted
ticket to staple — fix notarization first.

This is not [timestamp-unavailable] (signing itself fails) and not
[export-archive] (nothing is exported).

**Fix**

```bash
# The real reasons are in the log, not in "Invalid"
xcrun notarytool log <submission-id> --keychain-profile notary
# Check each binary: flags must include "runtime", authority "Developer ID Application"
codesign -dvv --verbose=4 App.app/Contents/MacOS/App
codesign -d --entitlements - App.app | grep get-task-allow
```

Set `ENABLE_HARDENED_RUNTIME = YES` on every target, archive the Release
configuration, and export with `method` = `developer-id`. Sign anything you
add by hand inside-out with `codesign --force --options runtime --timestamp`.
Then submit again with `notarytool submit --wait` and run
`xcrun stapler staple App.app`.

**Rule:** when notarization says Invalid, read `notarytool log` first — each issue names one binary path; fix that binary, not the whole app.

---

## [purpose-string-missing] Upload rejected: ITMS-90683 Missing purpose string

**Symptom** — archive, export and upload succeed; App Store Connect then
emails that the build is invalid:

```
ITMS-90683: Missing purpose string in Info.plist - Your app's code references
one or more APIs that access sensitive user data, or the app has one or more
entitlements that permit such access. The Info.plist file for the "App.app"
bundle should contain a NSCameraUsageDescription key …
```

**Cause** — Apple scans the linked symbols, not what your app actually calls.
An SDK that links camera, photos, location, contacts or Bluetooth APIs (WebRTC,
a payments or support SDK) is enough, even if that code never runs. The other
common case: the string exists only in `InfoPlist.strings` — that file only
translates a key; the key must also be in the bundle's own Info.plist. With
`GENERATE_INFOPLIST_FILE = YES`, the key comes from an
`INFOPLIST_KEY_NS…UsageDescription` build setting, which may be set only for
Debug.

This is not [privacy-manifest] (required-reason APIs in
`PrivacyInfo.xcprivacy`; here it is a `NS…UsageDescription` key) and not
[infoplist-missing] (no Info.plist at all).

**Fix**

```bash
# Which purpose strings actually shipped?
plutil -p App.xcarchive/Products/Applications/App.app/Info.plist | grep UsageDescription
# Which embedded binary links the camera / photos API?
nm -u App.app/Frameworks/*.framework/* 2>/dev/null | grep -E 'AVCaptureDevice|PHPhotoLibrary'
```

Add the key named in the email to the app target's Info.plist (or
`INFOPLIST_KEY_…` for every configuration) with a real sentence, not a
placeholder. Do the same for extensions that embed the SDK. A string is
fine even if the feature is never used; dropping the SDK is the only way to
avoid it.

**Rule:** a purpose string is required for what the binary *links*, not what it calls — check the archived Info.plist for every key ITMS-90683 names.

---

## [nested-frameworks] Upload rejected: ITMS-90205 / 90206 Frameworks inside an extension

**Symptom** — archive and export succeed; the upload (or validation) fails:

```
ERROR ITMS-90205: "Invalid Bundle. The bundle at 'App.app/PlugIns/Widget.appex'
contains disallowed nested bundles."
ERROR ITMS-90206: "Invalid Bundle. The bundle at 'App.app/PlugIns/Widget.appex'
contains disallowed file 'Frameworks'."
```

**Cause** — on iOS only the app's own `Frameworks/` folder may hold
frameworks; an `.appex` or a framework must not carry its own. Usual sources:
the extension (or a framework) has a dependency set to **Embed & Sign**, a
CocoaPods/Carthage copy-frameworks script runs on the extension target, or
`ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES = YES` on the extension copies Swift
libs into it. The debug build runs fine — only the store checks this.

This is not [embedded-binary-mismatch] (bundle ID / signing of an embedded
binary); here the binary is just in the wrong place.

**Fix**

```bash
# Which bundles carry their own Frameworks folder?
find App.xcarchive/Products/Applications/App.app -path '*/PlugIns/*/Frameworks' -o -path '*.framework/Frameworks'
```

Set those dependencies to **Do Not Embed** in the extension and framework
targets, embed them once in the app target, and set
`ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES = NO` on every target except the app.
The extension still links them and finds them via
`@executable_path/../../Frameworks` in `LD_RUNPATH_SEARCH_PATHS`.

**Rule:** only the app embeds frameworks — extensions and frameworks link, never embed.

---

## [app-icon-missing] Upload rejected: ITMS-90713 / 90022 / 90717 app icon

**Symptom** — the app runs (maybe with a blank icon); the upload fails:

```
ERROR ITMS-90713: "Missing Info.plist value. A value for the Info.plist key
'CFBundleIconName' is missing in the bundle 'com.example.app'."
ERROR ITMS-90022: "Missing required icon file. The bundle does not contain an
app icon for iPhone / iPod Touch of exactly '120x120' pixels."
ERROR ITMS-90717: "Invalid App Store Icon. The App Store Icon in the asset
catalog in 'App.app' can't be transparent nor contain an alpha channel."
```

**Cause** — `actool` only writes `CFBundleIcons`/`CFBundleIconName` into the
built Info.plist when `ASSETCATALOG_COMPILER_APPICON_NAME` names an icon set
that exists in a catalog the target actually compiles. It goes silent when the
setting is empty (new target, Release-only override), the set was renamed
(`AppIcon` → `AppIcon-Prod`), the `.xcassets` is missing from Copy Bundle
Resources, or a hand-written `CFBundleIcons` in the source Info.plist hides
it. 90717 is different: the 1024 pt marketing icon has an alpha channel.

**Fix**

```bash
# Did the icon keys reach the built app?
plutil -p App.xcarchive/Products/Applications/App.app/Info.plist | grep -A4 CFBundleIcon
# Which icon name does each configuration ask for?
xcodebuild -showBuildSettings -scheme App -configuration Release | grep APPICON_NAME
# Does the 1024 icon carry alpha?
sips -g hasAlpha App/Assets.xcassets/AppIcon.appiconset/*.png
```

Set `ASSETCATALOG_COMPILER_APPICON_NAME` to the exact set name for every
configuration, add the catalog to the target, and delete any manual
`CFBundleIcons` keys. Flatten an alpha icon by exporting it without
transparency (e.g. `sips -s format jpeg icon.png --out i.jpg && sips -s format png i.jpg --out icon.png`).

**Rule:** the icon set name in build settings must match the catalog, in every configuration.

---

## [framework-min-os] Upload rejected: ITMS-90208 / 90530 / 90360 framework MinimumOSVersion

**Symptom** — the app builds, runs and archives; the upload fails on one
embedded framework:

```
ERROR ITMS-90208: "Invalid Bundle. The bundle App.app/Frameworks/Vendor.framework
does not support the minimum OS Version specified in the Info.plist."
ERROR ITMS-90530: "Invalid MinimumOSVersion. ... MinimumOSVersion in
'App.app/Frameworks/Vendor.framework' is ''."
ERROR ITMS-90360: "Missing Info.plist value. A value for the key 'MinimumOSVersion'
in bundle App.app/Frameworks/Vendor.framework is required."
```

**Cause** — every embedded `.framework` carries its own Info.plist, and the
store checks its `MinimumOSVersion`: it must be present and no newer than the
app's. Xcode fills it for frameworks it builds, but not for prebuilt binaries
(vendor `.xcframework`s from SPM, Flutter's `App.framework`, React Native's
`hermes.framework`) whose plist was written by hand or by a script — it ships
empty, missing, or newer than the app's deployment target (the app was
lowered, or the vendor raised its floor).

This is not [deployment-target] (a target's own setting out of range) or
[spm-platform-minimum] (caught at build time); here only the store reads the plist.

**Fix**

```bash
# MinimumOSVersion of the app and of every embedded framework
for p in App.xcarchive/Products/Applications/App.app/Info.plist \
         App.xcarchive/Products/Applications/App.app/Frameworks/*.framework/Info.plist; do
  echo "$p: $(/usr/libexec/PlistBuddy -c 'Print :MinimumOSVersion' "$p" 2>&1)"
done
```

Update the dependency to a release that sets the key, or raise the app's
deployment target to at least the framework's. For your own prebuilt framework
(Flutter: `ios/Flutter/AppFrameworkInfo.plist`), set `MinimumOSVersion` to the
app's target. Last resort for a vendor binary: a Run Script after Embed that
runs `PlistBuddy -c "Set :MinimumOSVersion $IPHONEOS_DEPLOYMENT_TARGET"` (or
`Add ... string`) on the embedded plist, then re-signs that framework with
`codesign -f -s "$EXPANDED_CODE_SIGN_IDENTITY"`.

**Rule:** every embedded framework must state a MinimumOSVersion no newer than the app's.

---

## [interface-type-shadows-module] Shipped .swiftinterface fails: X is not a member type of X.X

**Symptom** — the framework builds and archives fine. But once it ships as an
`.xcframework`, or a newer Xcode opens it, the code that imports it fails:

```
error: 'Direction' is not a member type of class 'Compass.Compass'
error: failed to build module 'Compass' for importation due to the errors above;
the textual interface may be broken by project issues or a compiler bug
```

**Cause** — with `BUILD_LIBRARY_FOR_DISTRIBUTION=YES`, the compiler writes a
text `.swiftinterface` that spells every type out in full
(`Compass.Direction`). Say the module declares a public type with its own name
(`class Compass` in module `Compass`), or imports a module that does. Then
`Compass.` points to the *type*, not the module, and the interface can no
longer be read. Your own build uses the binary `.swiftmodule`, so it never sees
this. Only consumers, or a compiler that rebuilds from the interface, hit it.

This is not [xcframework-create]. There, the bundle cannot be made at all. Here,
the bundle is made, but its interface is broken.

**Fix**

```bash
# Catch it when the interface is written, not in the consumer's build
xcodebuild archive ... BUILD_LIBRARY_FOR_DISTRIBUTION=YES \
  OTHER_SWIFT_FLAGS='$(inherited) -verify-emitted-module-interface'
# Does a public type have the module's own name?
grep -nE "^public .*(class|struct|enum|protocol|actor) Compass\b" \
  Compass.xcframework/*/Compass.framework/Modules/Compass.swiftmodule/*.swiftinterface
```

The only lasting fix is to rename the type or the module (e.g. `Compass` →
`CompassView`). Until then, add `-Xfrontend -alias-module-names-in-module-interface`
to `OTHER_SWIFT_FLAGS`. It writes module names as aliases (`Compass__`) that
cannot be confused with types. Apple does not officially support this flag, so
check it again after every Xcode update. Do not hand-edit the shipped
`.swiftinterface`.

**Rule:** a module built for distribution must not have a public type with its own name.

---

## [missing-required-module] Importing a framework fails: Missing required module 'X'

**Symptom** — the framework builds on its own. The app or test target that
imports it fails on the `import` line:

```
error: Missing required module 'CSQLite'
```

**Cause** — the framework imports a private Clang module (a C target, or an
internal `module.modulemap`) with a plain `import CSQLite`. That import is then
written into the framework's `.swiftmodule` / `.swiftinterface`. So every
consumer must also find `CSQLite`, but its module map is not shipped in the
`.xcframework` and is not on the consumer's search paths. Test targets and
xcframeworks hit this most. Turning on Swift 6 mode can also bring it out.

This is not [non-modular-header]. There, the module cannot be *built*. Here, it
was built, but the consumer cannot *find* it.

**Fix**

```bash
# Which private modules leak into the shipped interface?
grep -h "^import" Foo.xcframework/*/Foo.framework/Modules/Foo.swiftmodule/*.swiftinterface
```

Best: hide the import so it stays out of the interface. Write
`internal import CSQLite` (Swift 5.9+, SE-0409). On older compilers, write
`@_implementationOnly import CSQLite`. Do this in *every* file that imports
it. No public API may then use a `CSQLite` type. For a target that is not
shipped (e.g. a unit-test target), a quick fix is to add the folder with the
module map to the consumer's `SWIFT_INCLUDE_PATHS`.

**Rule:** a private C module must never appear in a framework's public imports.

---

## [ipad-orientations] Upload rejected: ITMS-90474 iPad Multitasking requires all orientations

**Symptom** — the archive builds, exports and signs. The upload is refused:

```
ERROR ITMS-90474: "Invalid Bundle. iPad Multitasking support requires these
orientations: 'UIInterfaceOrientationPortrait,UIInterfaceOrientationPortraitUpsideDown,
UIInterfaceOrientationLandscapeLeft,UIInterfaceOrientationLandscapeRight'.
Found 'UIInterfaceOrientationPortrait' in bundle 'com.example.app'."
```

**Cause** — the app runs on iPad (`TARGETED_DEVICE_FAMILY` has `2`), so App
Store Connect assumes it supports Split View and Stage Manager. Those need all
four orientations for iPad. Someone locked the app to portrait for iPhone, and
the same list was used for iPad. Often a phone-only change to
`INFOPLIST_KEY_UISupportedInterfaceOrientations` did it, with no `~ipad`
variant set. ITMS-90475 is the same check failing on a missing launch
storyboard.

The old escape hatch was `UIRequiresFullScreen = YES`. It is deprecated in
iPadOS 26, and Apple says a future release will ignore it (TN3192). Do not add
it now.

This is not [purpose-string-missing] or [app-icon-missing]. Those are missing
keys. Here, the key is there, but its iPad value is too narrow.

**Fix**

```bash
# What the built app actually declares, for iPhone and for iPad
/usr/libexec/PlistBuddy -c 'Print :UISupportedInterfaceOrientations' \
  -c 'Print :UISupportedInterfaceOrientations~ipad' \
  -c 'Print :UIDeviceFamily' "$APP/Info.plist"
```

Set the iPad list on its own, and leave the iPhone list as it is:
`INFOPLIST_KEY_UISupportedInterfaceOrientations_iPad = UIInterfaceOrientationPortrait
UIInterfaceOrientationPortraitUpsideDown UIInterfaceOrientationLandscapeLeft
UIInterfaceOrientationLandscapeRight`. If the app truly cannot work on iPad,
remove `2` from `TARGETED_DEVICE_FAMILY`. Then it runs as an iPhone app on iPad.

**Rule:** an app that targets iPad must list all four orientations for iPad.

---

## [swift-support-missing] Upload rejected: ITMS-90426 / 90424 Invalid Swift Support

**Symptom** — the archive is fine and has a `SwiftSupport` folder. The `.ipa`
you upload is refused:

```
ERROR ITMS-90426: "Invalid Swift Support. The SwiftSupport folder is missing.
Rebuild your app using the current public (GM) version of Xcode and resubmit it."
ERROR ITMS-90424: "Invalid Swift Support. The SwiftSupport folder is empty."
```

**Cause** — the app embeds Swift runtime dylibs (`libswift*.dylib` in
`Frameworks/`). This happens with a deployment target below iOS 12.2, or with
`ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES = YES`. App Store Connect then wants a
matching copy in a top-level `SwiftSupport/iphoneos/` folder of the `.ipa`.
Only `xcodebuild -exportArchive` with `method` = `app-store-connect` writes
that folder. It is lost when the `.ipa` is made another way:

- zipping `Products/Applications/App.app` into `Payload/` by hand;
- exporting `ad-hoc` or `enterprise` and uploading that file;
- re-signing the `.ipa` with a script or a separate signing team.

90424 is the same check when the folder exists but is empty, or no longer
matches the re-signed dylibs.

This is not [export-archive]. There, the export fails. Here, the export
"worked", but the file was not made for the App Store.

**Fix**

```bash
# Does the ipa have the folder, and does the app embed Swift dylibs?
unzip -l App.ipa | grep -E "SwiftSupport/|Frameworks/libswift"
```

Export again from the `.xcarchive` with `method` = `app-store-connect`, and
upload that `.ipa` without changing it. If signing happens elsewhere, hand
over the `.xcarchive`, not an `.ipa`. If the deployment target is 12.2 or
higher, set `ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES = NO` on the app. Then no
Swift dylibs are embedded, and the folder is not needed.

**Rule:** the `.ipa` you upload must come straight from an App Store export —
never zip it or re-sign it by hand.

---

## [metal-toolchain-missing] Xcode 26: cannot execute tool 'metal' — Metal Toolchain missing

**Symptom** — the build stops at the first `.metal` file (`CompileMetalFile`).
This can be your own shader, a Core Image kernel, or a file inside a package.
SwiftUI previews of that target fail too:

```
error: cannot execute tool 'metal' due to missing Metal Toolchain;
use: xcodebuild -downloadComponent MetalToolchain
```

**Cause** — since Xcode 26, the Metal compiler is no longer inside Xcode.app.
It is a separate download (about 700 MB), like the platform runtimes in
[missing-platform-runtime]. The usual triggers are:

- a fresh Xcode on a laptop or CI image, where only the iOS platform was added;
- a hosted runner (for example GitHub's `macos-26`) that does not include it;
- a toolchain installed for an older Xcode or beta build that no longer matches
  the selected Xcode (`xcode-select`).

Nothing is wrong with your code. Projects with no `.metal` file never need the
toolchain, which is why only some apps fail on the same machine.

This is not [script-path-missing]. That is a run-script tool missing from PATH.
Here, Xcode's own compiler step is missing a component it no longer ships.

**Fix**

```bash
# Is the compiler there for the Xcode that is selected?
xcode-select -p && xcrun -f metal && xcrun metal --version
# Install it for that Xcode (run after any xcode-select switch)
xcodebuild -downloadComponent MetalToolchain
```

On CI, add the download as a step before the build, after the Xcode is
selected. To avoid downloading it on every run, export it once with
`xcodebuild -downloadComponent MetalToolchain -exportPath <dir>`, cache that
folder, and restore it with `xcodebuild -importComponent MetalToolchain
-importPath <bundle>`. Key the cache on the Xcode build number, because a
toolchain from another build is "missing" again. If it still fails after an
upgrade from a beta, delete the old component and download it again.

**Rule:** if the app has any `.metal` file, install the Metal Toolchain for the
exact Xcode that builds it.

---

## [bundle-id-collision] Upload rejected: ITMS-90685 CFBundleIdentifier Collision

**Symptom** — archive and export succeed. The upload (or Validate App) fails:

```
ERROR ITMS-90685: "CFBundleIdentifier Collision. There is more than one bundle
with the CFBundleIdentifier value 'com.vendor.SDK' under the iOS application 'App.app'."
```

**Cause** — every bundle inside the `.app` (each `.framework`, `.appex`,
`.bundle`, and the app itself) must have its own `CFBundleIdentifier`. Two of
them share one. Usual sources:

- the same framework is copied twice: embedded by the app *and* by a framework
  or extension, or added once by hand and once by CocoaPods/SPM;
- a vendored SDK ships a framework and a resource `.bundle` whose `Info.plist`
  hardcodes the same ID;
- a framework or extension `Info.plist` has a literal ID, not
  `$(PRODUCT_BUNDLE_IDENTIFIER)`, copied from another target or from the app.

Simulator and debug builds run fine. Only the store checks this.

This is not [nested-frameworks]. That one is about *where* a framework sits.
This one is about two bundles, anywhere, with *the same name*. A framework
embedded twice can trigger both.

**Fix**

```bash
# Every bundle ID in the archive, duplicates only
find App.xcarchive/Products/Applications/App.app -name Info.plist -exec \
  /usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" {} \; 2>/dev/null | sort | uniq -d
# Where does that ID live?
grep -rl --include=Info.plist "com.vendor.SDK" App.xcarchive/Products/Applications/App.app
```

If one framework appears twice, embed it only in the app target and set it
to **Do Not Embed** everywhere else. If two different bundles carry the ID,
set `CFBundleIdentifier` to `$(PRODUCT_BUNDLE_IDENTIFIER)` and give each
target a unique value. For a vendor `.bundle` you cannot rebuild, update the
SDK or report it to the vendor. Do not patch the plist after signing.

**Rule:** one bundle ID per bundle — check the archive with `uniq -d` before
you upload.

---

## [testing-search-paths] Helper framework fails: No such module 'XCTest' / 'Testing'

**Symptom** — test bundles build fine. A shared test-helper target (mocks,
fixtures, a `TestSupport` framework) fails as soon as it imports XCTest or
Swift Testing:

```
error: No such module 'XCTest'
error: Unable to find module dependency: 'Testing'
ld: framework 'XCTest' not found
```

A variant: the helper builds, but the app links it by mistake and crashes at
launch with `Library not loaded: @rpath/XCTest.framework/XCTest`.

**Cause** — `XCTest.framework` and `Testing.framework` are not in the SDK. They
live in `$(PLATFORM_DIR)/Developer/Library/Frameworks`. Xcode adds that folder
to the search paths only when `ENABLE_TESTING_SEARCH_PATHS = YES`. Unit-test
and UI-test bundles get it by default. A plain framework or static library does
not. Usual triggers:

- a new helper target created as a normal framework, or generated by
  Tuist/XcodeGen without the setting;
- the setting is on for Debug only, so `build-for-testing` with Release fails;
- the helper is a dependency of the app target, not only of the test bundles.

This is not [missing-required-module]. That is a shipped interface naming a
private module. Here, the module exists but is outside the search path.

**Fix**

```bash
# Is the setting on for the helper, in every configuration?
xcodebuild -showBuildSettings -scheme App -configuration Release \
  | grep -E "TARGET_NAME|ENABLE_TESTING_SEARCH_PATHS"
# Where the frameworks actually are
ls "$(xcrun --sdk iphonesimulator --show-sdk-platform-path)/Developer/Library/Frameworks"
```

Set `ENABLE_TESTING_SEARCH_PATHS = YES` on the helper target, for all
configurations (in Tuist: `settings: .settings(base: ["ENABLE_TESTING_SEARCH_PATHS": "YES"])`).
Do not hardcode the platform path in `FRAMEWORK_SEARCH_PATHS`; it breaks on the
next Xcode. Link the helper only from test targets. If the app needs part of
it, move that part to a target that does not import XCTest.

**Rule:** any non-test target that imports XCTest or Testing needs
`ENABLE_TESTING_SEARCH_PATHS = YES` — and must never be linked by the app.

---

## [sdk-signature-missing] Upload rejected: ITMS-91065 Missing signature

**Symptom** — the archive and upload succeed. Then App Store Connect sends:

```
ITMS-91065: Missing signature - Your app includes "Frameworks/Vendor.framework/Vendor",
which includes Alamofire, an SDK that was identified in the documentation as a
commonly used third-party SDK. If a new app includes a commonly used third-party
SDK, or an app update adds a new commonly used third-party SDK, the SDK must
include a signature file.
```

Often it shows up only after a dependency bump, or on one app but not a sibling
app that ships the same framework.

**Cause** — Apple keeps a list of commonly used SDKs (Alamofire, SDWebImage,
OpenSSL/BoringSSL, Firebase, OneSignal…). Since February 2025, a *binary* copy
of one of them must carry the vendor's signature. Usual triggers:

- the vendor ships an unsigned `.xcframework`;
- only the xcframework wrapper is signed, while its slices are unsigned or
  ad-hoc;
- you rebuilt or re-wrapped the binary yourself (lipo, a custom script, a
  CocoaPods `vendored_frameworks` repack). That removes the vendor's signature;
- an unsigned framework of yours links a listed SDK statically, so the SDK
  is hidden inside a binary with no signature.

Code built from source (plain SwiftPM / CocoaPods source pods) is not affected.
The check only applies the first time the SDK is added to an app, which explains
why one app passes and another fails.

This is not [embedded-binary-mismatch]. That one is your own re-signing at
export. Here, the SDK's *vendor* signature is missing.

**Fix**

```bash
# Is the xcframework signed, by whom, with a timestamp?
codesign -dvv Vendor.xcframework 2>&1 | grep -E "Authority|Timestamp|not signed"
# Which embedded binary hides the listed SDK?
nm -gU Payload/App.app/Frameworks/Vendor.framework/Vendor | grep -i alamofire | head
```

Upgrade to a signed vendor release, or switch the SDK to a source dependency.
If you control the framework, sign it with an Apple Distribution or Developer ID
certificate and a secure timestamp
(`codesign --timestamp -s "Apple Distribution: …" Vendor.xcframework`).
If you cannot get a signed build, stop linking that SDK statically.

**Rule:** a listed SDK ships as source or as a vendor-signed xcframework. Never
re-wrap a vendor binary.

---

## [stray-binary-in-bundle] Upload rejected: ITMS-90171 Invalid Bundle Structure

**Symptom** — the archive builds and runs. Then the upload, or a later email, says:

```
ITMS-90171: Invalid Bundle Structure - The binary file 'App.app/libVendor.a'
is not permitted. Your app can't contain standalone executables or libraries,
other than a valid CFBundleExecutable of supported bundles.
```

The path in the message is the file to remove. It can also point inside a
`.bundle` (`App.app/Vendor.bundle/Contents/MacOS/Vendor`).

**Cause** — an iOS app may carry Mach-O code in only two kinds of place: the main
executable, and the executable of a real bundle (`Frameworks/*.framework`,
`PlugIns/*.appex`). Any other Mach-O file is rejected. Usual ways one gets in:

- a `.a`, `.dylib`, CLI tool or `.dSYM` sits in *Copy Bundle Resources*, often
  because it came in through a folder reference or a resources glob;
- a vendor SDK ships a helper binary (`dump_syms`, an upload tool) inside its
  `.bundle`, and the whole bundle is copied;
- a resource `.bundle` target was built with the macOS SDK, so it has a
  `Contents/MacOS` executable;
- a loose `.dylib` is embedded instead of a `.framework`.

Simulator and Debug builds never check this, so it only shows up at upload.

This is not [nested-frameworks]. That one is a real framework in the wrong
folder. Here, the file should not be in the app at all.

**Fix**

```bash
# Every Mach-O file in the app that is not a bundle executable
find Payload/App.app -type f -exec file {} + | grep Mach-O \
  | grep -vE "App\.app/App:|\.framework/[^/]+:|\.appex/[^/]+:"
```

Remove the file from *Copy Bundle Resources* (or from the resources glob, or the
pod's `resources`). For a vendor bundle, delete the helper in a run-script phase
after the copy, or use a vendor release that leaves it out. For a `.bundle`
target, set `SDKROOT = iphoneos`. Wrap a loose dylib in a framework.

**Rule:** only bundle executables may be Mach-O. Run the `find` above on every
archive before you upload.

---

## [non-public-api] Upload rejected: ITMS-90338 Non-public API usage

**Symptom** — the archive builds, validates locally and runs. After upload, App
Store Connect sends:

```
ITMS-90338: Non-public API usage - The app references non-public selectors
in App: _setBackgroundColor:, initWithURLStrings:. If method names in your
source code match the private Apple APIs listed above, altering your method
names will help prevent this app from being flagged in future submissions.
```

Variants of the same check say "contains or inherits from non-public classes"
or "links to non-public libraries".

**Cause** — Apple scans the strings in every Mach-O file for selector, class and
library names that match its private API list. It does not know who owns the
method. So it fires on:

- your own Objective-C method (or `@objc` Swift method) that happens to share a
  name with a private one;
- a vendor static library or framework that really calls a private API (or has
  a matching name), often after an SDK update;
- `NSSelectorFromString` / `perform(_:)` / KVC on a private key, which leaves the
  name as a plain string.

The line after "in" names the binary — `App` means the main executable, which
also holds every static library linked into it.

This is not [privacy-manifest] (missing reasons for *public* APIs) or
[sdk-signature-missing]. This one is about names on the private list.

**Fix**

```bash
# Which binary in the app carries the flagged name?
find Payload/App.app -type f -exec file {} + | grep Mach-O | cut -d: -f1 \
  | while read f; do strings - "$f" | grep -q 'initWithURLStrings:' && echo "$f"; done
# Main executable: which linked .a / .o brought it in?
grep -rl 'initWithURLStrings' Pods/ Carthage/ ~/Library/Developer/Xcode/DerivedData/*/SourcePackages 2>/dev/null
```

Your code: rename the method (add a prefix) and drop any string-built selector.
Vendor code: update to a release that fixed it, or ask the vendor — do not patch
their binary. Upload again; the check runs on every build.

**Rule:** prefix your own Objective-C selectors, never reach private API through
strings, and grep new SDK versions for the flagged names before you ship them.

---

## [version-string-format] Upload rejected: ITMS-90060 / 90058 version must be integers

**Symptom** — the archive builds and exports. The upload is refused:

```
ERROR ITMS-90060: "This bundle is invalid. The value for key
  CFBundleShortVersionString '3.0.0-beta.1' in the Info.plist file must be a
  period-separated list of at most three non-negative integers."
ERROR ITMS-90058: "This bundle is invalid. The value for key CFBundleVersion
  '412-dev' in the Info.plist file must be a period-separated list of at most
  three non-negative integers."
```

**Cause** — App Store Connect accepts only `1`, `1.2` or `1.2.3` in these keys:
no letters, no `-beta`, no fourth part. Apple checks every bundle, not just the
app. So the bad value is often not yours:

- an embedded framework (Pod, Carthage, binary SPM) ships its own Info.plist
  with a pre-release version like `3.0.0-beta.1`;
- a CI step writes a branch name, a git hash or `git describe` output into
  `MARKETING_VERSION` or `CURRENT_PROJECT_VERSION`;
- a fourth part (`1.2.3.4`) was added for internal builds.

This is not [build-number-reused] (the number is valid but already used or too
low). This one is about the format.

**Fix**

```bash
# Every version in the app, the bad ones only
find Payload/App.app -name Info.plist | while read f; do
  for k in CFBundleShortVersionString CFBundleVersion; do
    v=$(/usr/libexec/PlistBuddy -c "Print :$k" "$f" 2>/dev/null) || continue
    echo "$v" | grep -qE '^[0-9]+(\.[0-9]+){0,2}$' || echo "$f $k=$v"
  done
done
```

Your own target: pass clean numbers at archive time and keep tags or hashes in
a custom key. Vendor framework: move to a final release, or ask the vendor —
a pre-release build cannot go to the store.

**Rule:** version keys are up to three integers in every bundle — check the
built app with the loop above before you upload.

---

## [modified-after-signing] Upload rejected: ITMS-90035 A sealed resource is missing or invalid

**Symptom** — the archive builds and exports. The upload, or installing on a
device, fails:

```
ERROR ITMS-90035: "Invalid Signature. A sealed resource is missing or invalid.
  The file at path [App.app/App] is not properly signed."
ERROR ITMS-90035: "Invalid Signature. Invalid Info.plist (plist or signature
  have been modified)."
```

On a device it shows up as `0xe8008001` or "The code signature of the app is
invalid".

**Cause** — the signature seals a hash of every file in the bundle. If any file
changes after signing, the seal breaks. Usual ways this happens:

- a CI step edits the built `.app` or `.ipa` (PlistBuddy sets a version, a
  config file is swapped, a framework is stripped), then zips it again without
  signing it again;
- a run-script phase changes an embedded framework after *Embed Frameworks*
  has already signed it;
- a resource name has a non-ASCII character (`Café.png`, an umlaut in the
  product name). Zipping or copying can change its Unicode form, so the name
  in the seal no longer matches the file.

This is not [sdk-signature-missing] (a vendor SDK has no signature at all).
Here a signature exists, but it does not match the files any more.

**Fix**

```bash
# Which file breaks the seal? Run on the exported app
codesign --verify --deep --strict --verbose=4 Payload/App.app
# Names with non-ASCII characters
find Payload/App.app | LC_ALL=C grep -n '[^ -~]'
```

Make every change before signing: set versions with build settings, pick
configs per build configuration. If you must change the app after export,
sign every changed bundle again, inside-out, with the same identity and
entitlements. Rename resources to plain ASCII.

**Rule:** nothing touches the bundle after it is signed. Run
`codesign --verify --deep --strict` on the final `.ipa` before you upload.

---

## [unsupported-architectures] Upload rejected: ITMS-90087 Unsupported Architectures

**Symptom** — the archive builds, exports and runs on a device. The upload fails:

```
ERROR ITMS-90087: "Unsupported Architectures. The executable for
  App.app/Frameworks/Vendor.framework contains unsupported architectures
  '[x86_64, i386]'."
```

It often comes with ITMS-90209 (Invalid Segment Alignment) and ITMS-90125
(LC_ENCRYPTION_INFO missing) for the same framework. All three have one cause.

**Cause** — an embedded framework is a "fat" binary: device and simulator slices
merged with `lipo`. The *Embed Frameworks* phase copies the binary as it is; it
does not remove slices. The store accepts device slices only, so any
`x86_64` or `i386` slice in the app is rejected. Usual sources:

- an old vendor SDK shipped as a plain `.framework`, not an `.xcframework`;
- a framework built with an old Carthage, or a custom "universal" script;
- a strip step that used to run (Carthage `copy-frameworks`, a CocoaPods embed
  script) was removed when the project changed tools.

This is not [xcframework-create] (building the xcframework fails). Here a fat
framework was never turned into one.

**Fix**

```bash
# Which embedded binaries carry simulator slices?
for f in Payload/App.app/Frameworks/*.framework; do
  lipo -info "$f/$(basename "$f" .framework)"
done | grep -E 'x86_64|i386'
```

Best: replace the fat framework with an `.xcframework` (ask the vendor, or
build one per platform). Short term: in a run-script phase after *Embed
Frameworks*, run `lipo -remove x86_64 -remove i386` on the binary, then sign it
again with `$EXPANDED_CODE_SIGN_IDENTITY` — removing slices breaks the
signature (see [modified-after-signing]).

**Rule:** only device slices go into a store build. Run `lipo -info` on every
embedded framework, and ship vendor code as `.xcframework`.

---

## [profile-cert-mismatch] Provisioning profile doesn't include signing certificate

**Symptom** — signing worked last week. Now archive or export fails:

```
error: Provisioning profile "App Store com.example.app" doesn't include
  signing certificate "Apple Distribution: Example Ltd (ABCDE12345)".
error: No signing certificate "iOS Distribution" found: No "iOS Distribution"
  signing certificate matching team ID "ABCDE12345" with a private key was found.
```

**Cause** — a profile lists the exact certificates it trusts (by serial). The
keychain has a certificate with the same *name* but a different serial, or no
private key for it. Usual ways this happens:

- someone created a new distribution certificate (the team limit is small, so
  an old one was revoked), and the profile in the repo or on CI was not
  regenerated;
- the certificate expired and was renewed — same name, new serial;
- CI imported the `.cer` only. Without the private key from the `.p12` the
  identity does not exist for `codesign`.

This is not [codesign-no-profile] (nothing installed) or
[entitlements-mismatch] (the profile lacks a capability). Here both files are
present, but they do not belong together.

**Fix**

```bash
# Identities codesign can really use (certificate + private key)
security find-identity -v -p codesigning
# Which certificate does the profile trust, and until when?
security cms -D -i App.mobileprovision \
  | plutil -extract DeveloperCertificates.0 raw -o - - \
  | base64 -D | openssl x509 -inform DER -noout -subject -enddate -fingerprint -sha1
```

The SHA-1 must match one line of `find-identity` (ignore the colons). If not,
regenerate the profile for the current certificate (or let
`-allowProvisioningUpdates` with an API key do it), and import the `.p12`, not
the `.cer`. With match or a shared certs repo, update the repo once — not each
runner.

**Rule:** a profile and a certificate are a pair. When a certificate is
renewed or revoked, regenerate every profile that uses it the same day.

---

## [beta-toolchain-upload] Upload rejected: ITMS-90111 Unsupported SDK or Xcode version

**Symptom** — archive and export work, TestFlight even accepts the build, but
App Store upload or review submission fails:

```
ITMS-90111: Unsupported SDK or Xcode version - App submissions must use the
  latest Xcode and SDK Release Candidates (RC).
```

**Cause** — something in the build chain was a beta. App Store Connect reads
the build stamps that Xcode writes into the app's `Info.plist`. A beta build
number ends in a lowercase letter (`17A5241e`); GM/RC builds do not. Usual ways
this happens:

- the archive was made with a beta Xcode or beta SDK (the "try the new Xcode"
  runner image, or `xcode-select` left pointing at `Xcode-beta.app`);
- the Xcode is a release, but the **Mac** runs a beta macOS. `BuildMachineOSBuild`
  carries that beta stamp, and Apple rejects it with the same code — changing
  Xcode, SDK or deployment target does nothing;
- rarely, the opposite: an Xcode so old that Apple no longer accepts it.

This is not [sdk-too-old] (ITMS-90725, the SDK is below the minimum). Here the
tools are too new — or not released yet.

**Fix**

```bash
# Which tools stamped this build? A trailing lowercase letter means beta
plutil -p App.xcarchive/Products/Applications/*.app/Info.plist \
  | grep -E 'DTXcodeBuild|DTSDKBuild|DTPlatformBuild|BuildMachineOSBuild'
# On the runner: is the OS or the selected Xcode a beta?
sw_vers -buildVersion; xcodebuild -version
```

Archive again with a released Xcode on a released macOS — a second Mac, a
stable CI image, or Xcode Cloud. Keep beta Xcode and beta macOS on separate
runners that never run the release lane.

**Rule:** release archives come only from GA Xcode on GA macOS. Check the
`DT*` and `BuildMachineOSBuild` stamps before you upload.

---

## [swift6-language-mode] Concurrency warnings became errors: risks causing data races

**Symptom** — code that built yesterday now fails, with no change to the code
itself. Often it happens after a package update or a `swift-tools-version` bump:

```
error: sending 'value' risks causing data races
error: static property 'shared' is not concurrency-safe because it is
  nonisolated global shared mutable state
error: main actor-isolated property 'x' can not be referenced from a
  nonisolated context
```

**Cause** — the target now compiles in **Swift 6 language mode**, where data-race
checks are errors, not warnings. Same compiler, different mode. Usual triggers:

- a package moved to `// swift-tools-version:6.0`. Every target in it now
  defaults to Swift 6 mode — your local package, or a dependency you bumped;
- `SWIFT_VERSION` changed from `5.0` to `6.0` in a target or xcconfig;
- `SWIFT_STRICT_CONCURRENCY = complete` together with
  `SWIFT_TREAT_WARNINGS_AS_ERRORS` (or `-warnings-as-errors`) on CI only;
- a new target from an Xcode 26 template comes with
  `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`. Old code moved into it changes
  isolation.

This is not [swift-version-unsupported]. That error happens before compiling.
This one is real diagnostics in your source. Wiping DerivedData does nothing.

**Fix**

```bash
# Which mode did each failing target really compile in?
grep -oE -- '-swift-version [0-9]+|-strict-concurrency=[a-z]+|-default-isolation[= ][A-Za-z]+' build.log | sort | uniq -c
# Which packages ask for tools 6?
grep -l 'swift-tools-version: *6' Package.swift */Package.swift \
  SourcePackages/checkouts/*/Package.swift 2>/dev/null
```

To unblock, pin the mode instead of hiding the diagnostics. In a package:
`swiftLanguageModes: [.v5]`, or `.swiftLanguageMode(.v5)` on a single target.
In Xcode: `SWIFT_VERSION = 5.0` plus `SWIFT_STRICT_CONCURRENCY = targeted`.
Then move to Swift 6 one module at a time, leaves first. For a dependency you
don't own, pin the last release that still builds, and report the issue.

**Rule:** the language mode is a build setting — set it on purpose for each
target. Don't let a tools-version bump or a template default change it for you.

---

## [testability-disabled] Tests fail to compile: Module 'X' was not compiled for testing

**Symptom** — the app builds, but the test target stops at its first import:

```
error: module 'App' was not compiled for testing
@testable import App
```

**Cause** — `@testable import` works only if the module was built with
`-enable-testing`. Xcode adds that flag only when `ENABLE_TESTABILITY = YES`,
and that is on only in Debug by default. Usual triggers:

- the scheme's Test action uses `Release`, or a `Staging`/`Beta` config that was
  copied from Release;
- a Tuist/XcodeGen custom configuration is a release-type config, so it gets
  release defaults;
- `build-for-testing` ran with one `-configuration` and `test-without-building`
  with another, or CI reused a Release build from the same DerivedData;
- the module is a prebuilt binary (xcframework, binary SPM target). It was never
  built for testing and can't be fixed from your side.

This is not the `ENABLE_TESTABILITY` case in [undefined-symbols]. That one is
a link error. This one is a compile error at the `import` line.

**Fix**

```bash
# Which configuration does the Test action use?
grep -o 'TestAction[^>]*buildConfiguration = "[^"]*"' *.xcodeproj/xcshareddata/xcschemes/*.xcscheme
# Is testability on for that configuration?
xcodebuild -showBuildSettings -scheme App -configuration Staging | grep ENABLE_TESTABILITY
# Did the module really compile with it?
grep -c -- '-enable-testing' build.log
```

Run tests in Debug. If tests must run against a release-like config, set
`ENABLE_TESTABILITY = YES` only in a test-only config, never in the one you
archive. It exports internal symbols and blocks some optimizations. For a
binary module, drop `@testable` and test its public API. For SwiftPM, use
`swift test -c release -Xswiftc -enable-testing`.

**Rule:** decide the test configuration once, in the scheme. Build and run
the tests with that same configuration.

---

## [asset-symbol-collision] Generated asset symbols clash: Invalid redeclaration / Ambiguous use

**Symptom** — the asset catalog compiles. Then Swift fails, often in a file
you never wrote, or at a call site that worked before:

```
GeneratedAssetSymbols.swift: error: invalid redeclaration of 'brand'
error: ambiguous use of 'brand'
warning: The "Primary" color asset name resolves to a conflicting Color
  symbol "primary". Try renaming the asset.
```

**Cause** — since Xcode 15, `actool` writes a Swift file with one symbol for
each color and image: `ColorResource.brand`, `ImageResource.logo`, and, when
`ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS = YES`, also
`Color.brand`, `UIColor.brand` and `UIImage.logo`. Those names collide with:

- your own `extension Color { static let brand = … }`, or SwiftGen / R.swift
  output that declares the same names;
- built-in members. An asset named `Primary`, `Background` or `Red` maps to
  `Color.primary` / `.red`. The symbol is skipped with a warning, and the
  call site picks up the system color without telling you;
- a `public` extension in another module with the same name. Both are visible
  at the call site, so the use is ambiguous.

It often shows up when an old project is opened in a newer Xcode, or the
setting is turned on there. The asset catalog didn't change.

**Fix**

```bash
# What did actool generate, and does it clash with your code?
f=$(find ~/Library/Developer/Xcode/DerivedData -name GeneratedAssetSymbols.swift -path '*App*' | head -1)
grep -oE 'static let [A-Za-z0-9_]+' "$f" | sort | uniq -d
grep -rnE 'static (let|var) (brand|logo)\b' --include='*.swift' Sources/
xcodebuild -showBuildSettings -scheme App | grep ASSETCATALOG_COMPILER_GENERATE
```

Choose one way to get typed assets. To keep Apple's: delete the hand-written
or SwiftGen extensions and use `Color(.brand)` / `Image(.logo)`. To keep your
own: set `ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS = NO`.
`ColorResource` stays, and it does not clash. Rename assets that match system
names (`Primary` → `TextPrimary`).

**Rule:** one source of asset symbols per module. Treat the "conflicting
symbol" warning as an error: a color that is silently replaced is a UI bug.

---

## [extension-version-mismatch] Upload flagged: ITMS-90473 extension version does not match the app

**Symptom** — the archive builds, exports and uploads. Then App Store Connect
sends an email, as a warning or as a rejection:

```
ITMS-90473: CFBundleShortVersionString Mismatch - The CFBundleShortVersionString
  value '2.3.0' of extension 'Widget.appex' does not match the
  CFBundleShortVersionString value '2.4.0' of its containing iOS application 'App.app'.
ITMS-90473: CFBundleVersion Mismatch - The CFBundleVersion value '118' of
  extension 'NotificationService.appex' does not match ... '121' ...
```

**Cause** — every `.appex` (widget, notification service, share, watch app)
must carry the same `CFBundleShortVersionString` and `CFBundleVersion` as the
app that embeds it. The values are valid, but they drifted apart:

- the extension's Info.plist has a literal (`1.0`, `1`) instead of
  `$(MARKETING_VERSION)` / `$(CURRENT_PROJECT_VERSION)`. So an archive-time
  override (`CURRENT_PROJECT_VERSION=121`) reaches the app but not the extension;
- `MARKETING_VERSION` is set at target level on the app only. The extension
  falls back to the project value or the template's `1.0`;
- a bump step (fastlane with a `target:`, a `sed`/PlistBuddy script, Tuist/
  XcodeGen `settings` on one target) edits only the app target;
- a new extension was added from the Xcode template after the last bump.

This is not [version-string-format] (a value is not a valid number) and not
[build-number-reused] (the number is already used). Here every number is valid
on its own; they just don't match.

**Fix**

```bash
# App and each extension, side by side — every line must be the same
for f in Payload/App.app/Info.plist Payload/App.app/PlugIns/*.appex/Info.plist \
         Payload/App.app/Watch/*.app/Info.plist; do
  [ -f "$f" ] || continue
  printf '%s  %s (%s)\n' \
    "$(/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' "$f")" \
    "$(/usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' "$f")" "$f"
done
# Which targets hard-code a version, or set their own?
grep -rnE '<string>[0-9]+(\.[0-9]+)*</string>' --include=Info.plist -B1 . | grep -A1 -E 'CFBundle(ShortVersionString|Version)<'
grep -nE '(MARKETING_VERSION|CURRENT_PROJECT_VERSION) = ' App.xcodeproj/project.pbxproj | sort | uniq -c
```

Replace literals with `$(MARKETING_VERSION)` and `$(CURRENT_PROJECT_VERSION)`.
Set those two once, at project level (or in a shared xcconfig), and delete the
target-level copies. Then one bump, or one override on the `xcodebuild archive`
line, sets the whole archive.

**Rule:** the version lives in one place for the whole app. Every extension
reads it from there; none has its own.

---

## [automation-mode-timeout] UI tests never start: Timed out while enabling automation mode

**Symptom** — unit tests pass. The UI test bundle waits about 90 seconds, then
fails before the first test runs:

```
Testing failed:
  The test runner failed to initialize for UI testing.
  (Underlying Error: Timed out while enabling automation mode.)
```

It works when someone runs the tests at the Mac, and fails on CI or over SSH.

**Cause** — since macOS 13, UI tests (and WebDriverAgent/Appium) must turn on
Automation Mode first. Turning it on asks for an admin password or Touch ID. On
a runner nobody can answer, so `testmanagerd` waits and gives up:

- the job runs from SSH, `launchd` or a service user, with no logged-in GUI
  session to show the prompt;
- a fresh or re-imaged Mac never had automation allowed without a password;
- a macOS update reset the setting;
- the screen is locked or the Mac is asleep, so the prompt can't appear.

This is not [test-hang] (tests run, then stall) and not [simulator-wedged] (the
simulator itself won't boot). Here the runner never gets permission to drive
the UI.

**Fix**

```bash
# Current state: is a password needed?
automationmodetool
# Once per runner, as an admin: allow it without a password
sudo automationmodetool enable-automationmode-without-authentication
# Older macOS / helper tools: dev tools without a prompt
sudo DevToolsSecurity -enable
```

Put the command in the runner's provisioning script, so a new image or a macOS
update can't drop it. Run the agent in a logged-in GUI session (auto-login, a
user LaunchAgent, not a LaunchDaemon), and turn off screen lock and sleep.
Allowing automation without a password lets any local process drive the UI:
do it on CI Macs only, never on a laptop.

**Rule:** UI tests on CI need a GUI session and automation already allowed. Set
both when the runner is built, not when the test fails.

---

## [cloud-signing-permission] Export fails: no access to cloud-managed distribution certificates

**Symptom** — the archive succeeds. `-exportArchive` with
`-allowProvisioningUpdates` then stops:

```
error: exportArchive: Cloud signing permission error
  You haven't been given access to cloud-managed distribution certificates.
  Please contact your team's Account Holder or an Admin.
```

It exports fine from an Admin's Xcode, and fails on CI.

**Cause** — automatic signing at export does not use a certificate on the
runner. It asks Apple to sign with a cloud-managed distribution certificate,
and only some accounts may do that:

- the App Store Connect API key passed with `-authenticationKeyPath` has the
  Developer or App Manager role; cloud distribution signing needs an Admin key;
- the Apple ID signed into the runner's Xcode is not an Admin and was never
  given "Access to Cloud Managed Distribution Certificate";
- the export method is `developer-id`: Apple does not cloud-sign Developer ID,
  even with an Admin key.

This is not [export-archive] (bad options plist) and not [codesign-no-profile]
(nothing installed). Here Xcode can reach Apple, but the account may not sign.

**Fix**

```bash
# Which key and signing style does the export use?
/usr/libexec/PlistBuddy -c Print ExportOptions.plist
# Option A: an Admin API key (App Store / TestFlight only)
xcodebuild -exportArchive -archivePath App.xcarchive -exportPath ./export \
  -exportOptionsPlist ExportOptions.plist -allowProvisioningUpdates \
  -authenticationKeyPath "$KEY_PATH" -authenticationKeyID "$KEY_ID" \
  -authenticationKeyIssuerID "$ISSUER_ID"
# Option B: no cloud at all — installed cert + profile, manual signing
#   ExportOptions.plist: signingStyle=manual, signingCertificate,
#   provisioningProfiles = { bundle ID : profile name }
xcodebuild -exportArchive -archivePath App.xcarchive -exportPath ./export \
  -exportOptionsPlist ExportOptions.plist
```

For a person, an Admin turns on "Access to Cloud Managed Distribution
Certificate" in App Store Connect → Users and Access. For Developer ID, always
use option B. An Admin key can do anything on the team: keep it in the CI
secret store, scoped to the release job.

**Rule:** cloud signing is a permission, not a file. Give CI an Admin key, or
give it the certificate and profile and turn cloud signing off.

---

## [pla-not-accepted] Signing or upload fails: PLA Update available

**Symptom** — the code is the same as yesterday's green run. Now every step
that talks to Apple fails at once: `-allowProvisioningUpdates` builds, export,
and upload, on every runner and every laptop on the team:

```
error: exportArchive: Unable to process request - PLA Update available
  You currently don't have access to this membership resource. To resolve
  this issue, agree to the latest Program License Agreement in your
  developer account.
```

API-key uploads show the same thing as `403 FORBIDDEN.REQUIRED_AGREEMENTS_MISSING_OR_EXPIRED`.

**Cause** — Apple published a new Program License Agreement. Until the team's
**Account Holder** accepts it, the developer portal blocks the whole team:
no new profiles or certificates, no cloud signing, no uploads. Builds that
need no network (installed cert, manual signing, no upload) still work, so the
failure often appears only in the release job. Admins cannot accept the PLA,
and API keys cannot either. Only the Account Holder can.

This is not [license-first-launch] (the Xcode licence on the Mac) and not
[cloud-signing-permission] (one account's role). Here the account is fine, but
the whole team is blocked.

**Fix**

```bash
# Confirm: does the App Store Connect API refuse because of an agreement?
curl -s -H "Authorization: Bearer $ASC_JWT" \
  https://api.appstoreconnect.apple.com/v1/apps?limit=1 | grep -o '"code" *: *"[^"]*"'
# → "FORBIDDEN.REQUIRED_AGREEMENTS_MISSING_OR_EXPIRED" means: go accept it
```

The Account Holder signs in at developer.apple.com → Account and accepts the
banner. They also check App Store Connect → Business (Agreements) for an
expired Paid Apps agreement. Wait a few minutes, then re-run the job. Do not
change the code or the signing settings.

**Rule:** if every Apple-facing step fails at once with nothing changed, read
the first line for "PLA". The fix belongs to the Account Holder, not to CI.

---

## [only-testing-no-match] Tests "pass" in 0 seconds: Executed 0 tests

**Symptom** — the test job is green and fast, too fast. The log ends with:

```
Test Suite 'Selected tests' passed at ...
	 Executed 0 tests, with 0 failures (0 unexpected) in 0.000 (0.000) seconds
** TEST SUCCEEDED **
```

**Cause** — `-only-testing:` (or a test plan's "selected tests") names
something that does not exist. If the target exists but the class or method
does not, xcodebuild runs nothing, prints no warning, and exits 0. Usual ways:

- a test class or method was renamed, and the CI script still lists the old one;
- the module name is used where the **target** name is expected (they differ
  when the product name is changed);
- a Swift Testing test is written like XCTest. Swift Testing names the
  function with its parentheses, `Target/Suite/test()`, and uses the
  `@Suite` type name, not a display name.

This is not [scheme-not-testable] (the scheme has no test action) or
[pipe-masks-exit] (the build failed and the exit code was lost). Here every
step really succeeded. It just did nothing.

**Fix**

```bash
# How many tests really ran?
xcrun xcresulttool get test-results summary --path Test.xcresult \
  | plutil -extract totalTestCount raw -o - -
# Fail the job on zero
[ "$(xcrun xcresulttool get test-results summary --path Test.xcresult \
  | plutil -extract totalTestCount raw -o - -)" -gt 0 ] || exit 1
# The real identifiers, to copy into -only-testing
xcodebuild test-without-building -xctestrun App.xctestrun -enumerate-tests \
  -test-enumeration-style flat -destination "$DEST"
```

Copy the identifier from the enumeration, not from memory. Prefer filtering by
test plan or by target over listing single methods in CI scripts.

**Rule:** a test step that ran 0 tests is a failed step. Check the count, not
only the exit code.

---

## [binary-target-checksum] SPM: checksum of downloaded artifact of binary target does not match

**Symptom** — package resolution fails before anything compiles:

```
xcodebuild: error: Could not resolve package dependencies:
  checksum of downloaded artifact of binary target 'Vendor' (3f1a…)
  does not match checksum specified by the manifest (9c2e…)
```

**Cause** — a `.binaryTarget(url:checksum:)` pins a zip by its SHA-256. The
bytes at that URL are not the bytes the manifest expects. Usual ways:

- the vendor re-uploaded the zip under the same version (a hot-fix, a
  re-signed xcframework) and did not change the manifest or the tag;
- a proxy, CDN or mirror served an HTML error page, a login page, or a
  half-downloaded file with status 200;
- a stale or corrupt copy sits in the SwiftPM artifact cache and is reused.

This is not [spm-fingerprint-mismatch] (a git tag moved to another commit). Here
the source checkout is fine; only the downloaded binary differs.

**Fix**

```bash
# Which binary target, which URL, which checksum does the manifest expect?
grep -rn -A3 --include=Package.swift 'binaryTarget' SourcePackages/checkouts
# What does the URL really serve today?
curl -sSL -o /tmp/art.zip "$URL" && file /tmp/art.zip && swift package compute-checksum /tmp/art.zip
# Drop the cached copies, then resolve again
rm -rf ~/Library/Caches/org.swift.swiftpm/artifacts DerivedData/*/SourcePackages/artifacts
xcodebuild -resolvePackageDependencies -clonedSourcePackagesDirPath SourcePackages
```

If `file` says HTML, fix the network or auth, not the package. If it is a real
zip with a new checksum, the vendor changed it: pin the previous version or
wait for a new tag. Never edit the checksum in a dependency's manifest.

**Rule:** a binary target is pinned by bytes. Same version with new bytes is
the vendor's bug — pin an older version and report it, do not work around it.

---

## [device-not-in-profile] Install on device fails: A valid provisioning profile for this executable was not found (0xe8008015)

**Symptom** — the build and the signing succeed, then the install fails:

```
Unable to install "App"
  ERROR: A valid provisioning profile for this executable was not found.
  (0xe8008015)
```

Testers on an Ad Hoc or Firebase / Diawi build see "Unable to Install App"
or "This app cannot be installed because its integrity could not be verified".

**Cause** — the profile inside the app does not allow *this* device. Usual
ways:

- the device UDID is not in the profile's `ProvisionedDevices`: a new phone,
  a new tester, or the device was added in the portal but the profile was not
  regenerated (adding a device never updates existing profiles);
- the app was signed with an App Store profile, which has no device list and
  installs only through TestFlight or the store;
- the profile expired, or the device list was reset at the yearly membership
  renewal.

This is not [codesign-no-profile] (no profile was found, signing failed) or
[profile-cert-mismatch] (the certificate is not in the profile). Here signing
worked; the device check at install time refuses it.

**Fix**

```bash
# Which profile type, which devices, until when?
security cms -D -i App.app/embedded.mobileprovision > /tmp/p.plist
plutil -extract ProvisionedDevices xml1 -o - /tmp/p.plist
plutil -extract ExpirationDate raw -o - /tmp/p.plist
# The device's real UDID (not the CoreDevice identifier devicectl lists)
xcrun devicectl device info details --device "$DEVICE" | grep -i udid
# Development builds: let Xcode register the device and refresh the profile
xcodebuild ... -allowProvisioningUpdates -allowProvisioningDeviceRegistration
```

No `ProvisionedDevices` key means an App Store profile: use TestFlight, or
export with `method` = `release-testing` (Ad Hoc). For Ad Hoc, register the
UDID, regenerate the profile, download it to CI, and re-sign.

**Rule:** adding a device is two steps — register it, then regenerate every
Development and Ad Hoc profile that should include it.

---

## [coverage-not-collected] Tests pass but there is no coverage: No profiles could be merged

**Symptom** — the tests are green, then the coverage step fails or reports
nothing:

```
warning: .../Build/ProfileData/<UUID>/<id>.profraw: Invalid instrumentation
  profile data (file header is corrupt)
error: No profiles could be merged.
```

Or `xcrun xccov view --report Test.xcresult` says the bundle has no coverage
data, or the report is empty / 0%, and Codecov or Sonar rejects the upload.

**Cause** — coverage is made in two halves: the compiler instruments the
binary at **build** time, and the test run writes `.profraw` files that
`llvm-profdata` then merges. Usual ways one half goes missing:

- `-enableCodeCoverage YES` was passed to `test-without-building` but not to
  `build-for-testing`, so the binaries were never instrumented;
- the test plan or scheme has coverage off, or limits it to targets that
  were renamed;
- a test process crashed or was killed mid-run, leaving a truncated
  `.profraw` ("file header is corrupt");
- the binaries came from another toolchain than the `llvm-profdata` doing the
  merge (prebuilt instrumented frameworks, a `TOOLCHAINS` override, or an
  Xcode bug, as in 26.1): "raw profile version mismatch".

This is not [only-testing-no-match] (no tests ran) or [xcresulttool-legacy]
(the report tool's flags changed). Here tests ran; only the coverage is
missing.

**Fix**

```bash
# Is coverage in the result bundle at all?
xcrun xccov view --report --only-targets Test.xcresult
# Build and test both need the flag
xcodebuild build-for-testing ... -enableCodeCoverage YES
xcodebuild test-without-building ... -enableCodeCoverage YES -resultBundlePath Test.xcresult
# Which raw profile is bad, and why?
for f in DerivedData/Build/ProfileData/*/*.profraw; do
  xcrun llvm-profdata show "$f" >/dev/null || echo "BAD: $f"
done
```

"Corrupt header": find the test that crashed and fix that first. "Version
mismatch": rebuild every instrumented binary with the selected Xcode, and
unset `TOOLCHAINS`.

**Rule:** coverage starts at compile time — pass the flag to the build, not
just to the test run, and fail CI when the report has no targets.

---

## [xcstrings-invalid] String Catalog fails to compile: CompileXCStrings ... isn't in the correct format

**Symptom** — after a merge, the build stops in the localization step:

```
CompileXCStrings .../en.lproj/Localizable.strings .../Localizable.xcstrings
error: The data couldn't be read because it isn't in the correct format.
** BUILD FAILED **
```

Xcode may also refuse to open the catalog, or open it with strings missing.

**Cause** — a `.xcstrings` file is one large JSON document that Xcode
rewrites in full (keys sorted, every language nested inside each key). Two
branches that add strings touch the same lines, and git puts `<<<<<<<`
markers, a dropped comma or a duplicate key into the JSON. `xcstringstool`
cannot parse it, so the whole catalog fails, not one string. A
`merge=union` driver (often suggested) can cause the same thing: it keeps
both sides' lines, braces included.

This is not [multiple-commands-produce] (a `.strings` and a `.xcstrings`
of the same name in one target). Here the one catalog is broken.

**Fix**

```bash
# Leftover markers, and is it still valid JSON?
git grep -nE '^(<<<<<<<|=======|>>>>>>>)' -- '*.xcstrings'
for f in $(git ls-files '*.xcstrings'); do jq empty "$f" || echo "BAD: $f"; done
# Same key twice (jq keeps only the last, so check the raw text)
grep -oE '^    "[^"]+" : \{' Localizable.xcstrings | sort | uniq -d
```

Fix the JSON by hand (keep both sides' keys), then open the catalog in Xcode
once so it rewrites it in its own order. Better: take one side
(`git checkout --theirs`), build, and let Xcode extract your new strings again.

**Rule:** treat the catalog as generated JSON — check it with `jq` in CI and
before merging, and never let a line-based merge driver touch it.

---

## [swift-header-not-found] Objective-C can't see Swift: 'App-Swift.h' file not found

**Symptom** — a mixed Objective-C / Swift target fails in an `.m` file,
often only on a clean build or on CI:

```
CompileC .../LegacyViewController.o LegacyViewController.m
error: 'MyApp-Swift.h' file not found
#import "MyApp-Swift.h"
** BUILD FAILED **
```

**Cause** — Xcode generates `<ProductModuleName>-Swift.h` while compiling
the Swift files; it is never a file in the repo. The import breaks when the
name or the place does not match what was generated:

- the module name is not the target name: `PRODUCT_MODULE_NAME` was set, or
  the product name has spaces or hyphens (`My-App` → `My_App-Swift.h`);
- `SWIFT_OBJC_INTERFACE_HEADER_NAME` was changed, or
  `SWIFT_INSTALL_OBJC_HEADER = NO`, so no header is made at all;
- inside a framework the header must be `#import <MyKit/MyKit-Swift.h>`,
  not the quoted form;
- the header is imported from another target (a test or an extension) with
  no build dependency, or from a `.h` that Swift itself reads through the
  bridging header — a cycle, so it only "works" when an older copy is still
  in DerivedData.

If the header is found but says `unknown type name`, the Swift type is not
`@objc` / not a `NSObject` subclass, or is not `public` in a framework.

**Fix**

```bash
# The name Xcode will really generate, and whether it generates one
xcodebuild -showBuildSettings -scheme App | grep -E "PRODUCT_MODULE_NAME|SWIFT_OBJC_INTERFACE_HEADER_NAME|SWIFT_INSTALL_OBJC_HEADER"
# Every import of a generated header, to compare against that name
git grep -nE '#import [<"].*-Swift\.h[>"]'
# Did the last build produce it, and where?
find ~/Library/Developer/Xcode/DerivedData -name '*-Swift.h' -path '*App*' | head
```

Import exactly that name (`<MyKit/MyKit-Swift.h>` for frameworks), only from
`.m` files, and use `@class Foo;` in headers instead. Add a target dependency
when another target needs it.

**Rule:** treat `-Swift.h` as a build output — import it by the generated
name, only from implementation files, and prove it with a clean build.

---

## [sanitizer-runtime-missing] Sanitizer build fails: ___asan_init undefined / libclang_rt.asan dylib not loaded

**Symptom** — with Address (or Thread / Undefined Behavior) Sanitizer in
play, the link or the launch breaks, often only on CI or after a cache hit:

```
Undefined symbols for architecture arm64:
  "___asan_init", referenced from: _asan.module_ctor in libCore.a(Cache.o)
ld: symbol(s) not found for architecture arm64
```

or, at test launch:

```
dyld: Library not loaded: @rpath/libclang_rt.asan_iossim_dynamic.dylib
  Referenced from: .../MyApp.app/MyApp
```

**Cause** — instrumented code needs the sanitizer runtime that ships inside
the *selected* Xcode's clang, and Xcode only links and embeds it into the
app when the sanitizer is on for *that* build. It breaks when the two
halves disagree:

- a prebuilt `.a` / `.xcframework` (vendored, or restored from a build
  cache) was compiled with `-fsanitize=…`, but the app linking it was not;
- `build-for-testing -enableAddressSanitizer YES` ran with one Xcode and
  `test-without-building` with another, or the `.xctestrun` was moved
  without the products — the runtime path baked in points nowhere;
- the scheme turns ASan on for Run but the Test action does not reuse
  Run's diagnostics (or the reverse), so targets build differently;
- ASan and TSan are both requested — only one sanitizer can be on at a time.

**Fix**

```bash
# Which binaries are instrumented? Any hit in a prebuilt lib is the culprit
for f in $(find . -name '*.a' -o -name '*.framework' -prune -o -name '*.dylib'); do nm -m "$f" 2>/dev/null | grep -q '___asan_init\|___tsan_init' && echo "$f"; done
# Is the runtime embedded in the built app, and from which Xcode?
ls MyApp.app/Frameworks | grep clang_rt; xcode-select -p
# Same flag on both steps, same Xcode on both machines
xcodebuild build-for-testing -scheme App -enableAddressSanitizer YES
xcodebuild test-without-building -xctestrun App.xctestrun -enableAddressSanitizer YES
```

Rebuild any cached or vendored library without `-fsanitize` (sanitize only
your own code), keep sanitizers out of cache keys that Release builds share,
and run ASan and TSan as separate jobs.

---

## [test-bootstrap-crash] Tests never start: Early unexpected exit, operation never finished bootstrapping

**Symptom** — no test runs at all. The host app dies before XCTest can
connect, so the log shows no assertion, only:

```
Testing failed:
	MyAppTests:
		MyApp (12345) encountered an error (Early unexpected exit, operation
		never finished bootstrapping - no restart will be attempted.
		(Underlying Error: Test crashed with signal abrt before establishing connection.))
```

The signal varies (`abrt`, `segv`, `trap`, `kill`), and so does the wording:
"Crash: MyApp at …", or "The test runner exited with code -1 before checking
in". It often fails only on CI, or only in the first parallel clone.

**Cause** — unit tests run *inside* the host app, so the app's whole launch
path runs first. Anything that crashes during launch kills the run before
the test bundle loads:

- launch code needs something CI does not have: a `GoogleService-Info.plist`
  or secrets file left out of git, a `fatalError` on a missing environment
  value, keychain or network access, or a bad force-unwrap of a config value;
- a `+load` / static initializer in a linked library traps;
- dyld cannot load the test bundle into the host (missing `@rpath` framework,
  wrong architecture) → see [dyld-rpath-missing];
- `signal kill`: the system ended it — too many parallel simulator clones for
  the runner's memory, or a launch that hung until the watchdog fired.

**Fix** — find the real crash first; it is in the result bundle, not in the summary:

```bash
# Crash reports and the host app's console output from the run
xcrun xcresulttool export diagnostics --path Tests.xcresult --output-path diag
grep -rl -e "Exception Type" -e "Fatal error" diag | head; find diag -name 'StandardOutputAndStandardError*'
# Reproduce outside XCTest: launch the same build on the same simulator
xcrun simctl launch --console-pty booted com.example.MyApp
# Too many clones? Run with fewer, or none
xcodebuild test -scheme App -destination "$DEST" -parallel-testing-worker-count 2
```

Then make launch safe under test. Detect XCTest
(`ProcessInfo.processInfo.environment["XCTestConfigurationFilePath"] != nil`)
and skip SDK setup, or give the test run a minimal `AppDelegate` from
`main.swift`. Commit a fake config file for CI. Pure logic tests can drop the
host app entirely (Host Application: None).

**Rule:** the host app's launch is part of every unit test — keep it free of secrets, network and force-unwraps, and read the crash log before touching the tests.

---

## [spm-static-linked-twice] SPM package linked into two targets: linked as a static library by 'App' and 'X'

**Symptom** — the build stops before compiling, or the app launches with
duplicate classes:

```
error: Swift package target 'Realm' is linked as a static library by 'App'
  and 'Core', but cannot be built dynamically because there is a package
  product with the same name.
error: Swift package product 'Alamofire' is linked as a static library by
  'App' and 'Core'. This will result in duplication of library code.
objc[123]: Class _TtC9Alamofire7Session is implemented in both .../Core.framework/Core
  and .../App. One of the two will be used. Which one is undefined.
```

**Cause** — an SPM product with automatic linkage is static. Link it to two
targets that each end up as their own binary, like the app and an embedded
framework or extension, and each one gets its own copy. Xcode 13 and later try
to fix this by building the package target as a dynamic framework instead.
That fails when a target has the same name as a product (Realm 10.49+, for
example). Older Xcode versions fail with "duplication of library code". If the
build gets through, the run fails instead: two copies of every singleton,
`is`/`as?` checks failing across the boundary, and the "implemented in both"
warning above.

This is not [duplicate-symbols] (one binary, two copies seen by the linker) or
[embedded-binary-mismatch]. Here each binary links cleanly. The problem is the
package being in both.

**Fix** — the package should be linked in one place only:

```bash
# Which targets link which package products?
grep -n "productName = " App.xcodeproj/project.pbxproj | sort | uniq -c
# Tuist / XcodeGen: who depends on the package directly?
grep -rn '\.package(product: "Realm"' Project.swift Tuist/ 2>/dev/null
# After the build: is the code in more than one binary?
for b in "$APP"/MyApp "$APP"/Frameworks/*.framework/*[!.]*; do
  nm -U "$b" 2>/dev/null | grep -q 'RealmSwift' && echo "$b"; done
```

- Link the product only to the framework (`Core`). Remove it from the app's
  *Frameworks, Libraries…* list. The app imports it through `Core`.
- If you own the package, declare `type: .dynamic` on the product and embed it
  once in the app.
- For a third-party product whose target has the same name: use its prebuilt
  dynamic xcframework, or pin the last version that still builds.

**Rule:** one static package, one binary. When two targets need it, route it
through a framework or make it dynamic.

---

## [spm-resource-bundle] Package resources missing: Type 'Bundle' has no member 'module' / unable to find bundle named

**Symptom** — the package target fails to compile, or compiles and then
crashes the first time it loads an image, string or JSON file:

```
error: type 'Bundle' has no member 'module'
warning: 'feature': found 2 file(s) which are unhandled; explicitly declare
  them as resources or exclude from the target
    /Sources/Feature/Resources/config.json
Fatal error: unable to find bundle named MyPackage_Feature
```

**Cause** — SwiftPM only creates `Bundle.module` for a target that has
resources. Xcode-type files (`.xcassets`, `.xib`, `.storyboard`, `.lproj`,
`.xcstrings`) count by themselves. Anything else, like JSON, fonts or loose
PNGs, has to be listed under `resources:`. If it is not, you get the "unhandled"
warning, no bundle and no accessor. `Bundle.module` also does not exist in app
or Xcode framework targets. It is a package-only symbol. When the target does
compile, the resources go in a separate `<Package>_<Target>.bundle`. The
accessor looks for it next to the main bundle, next to the binary that loaded
it, and at an absolute DerivedData path baked in at build time. Tests and
previews often only work because of that last path. The crash happens when the
bundle never gets into the product: a prebuilt or static xcframework that ships
without it, a command-line tool installed without it, or a framework copied out
of the build directory. It works on the machine that built it and fails
everywhere else.

This is not [infoplist-missing] or [missing-input-file]. The build finds every
file. SwiftPM was never told to package them, or the package went missing.

**Fix** — declare the resources, then check the bundle ships with the product:

```bash
# What does SwiftPM treat as a resource? Empty means no Bundle.module
swift package describe --type json | jq '.targets[] | {name, resources}'
grep -i "unhandled" build.log
# Was the bundle built, and is it in the shipped product?
find "$BUILT_PRODUCTS_DIR" -maxdepth 1 -name '*_*.bundle'
find "$APP" -name '*_*.bundle'
```

- Add `resources: [.process("Resources")]` to the target (or `.copy` to keep a
  folder's layout). This needs `swift-tools-version` 5.3 or later.
- Only use `Bundle.module` inside the package. To share it, expose
  `public let featureBundle = Bundle.module`. Don't use it from app code.
- Prebuilt binaries: a binary target can't carry a loose `.bundle`. Build the
  package as a dynamic framework with its resources inside, or ship the bundle
  and copy it into the app yourself.
- CLI tools: install the `.bundle` next to the executable.

**Rule:** a resource is only shipped if SwiftPM was told about it. Check for
the `_<Target>.bundle` in the product, not in DerivedData.

---

## [argument-list-too-long] Build step can't start: unable to spawn process (Argument list too long)

**Symptom** — a script phase, the compiler or the linker never runs. Often
only on CI, or only after adding pods or moving the checkout:

```
error: unable to spawn process '/bin/sh' (Argument list too long)
clang: error: unable to execute command: posix_spawn failed: Argument list too long
Command PhaseScriptExecution failed with a nonzero exit code
```

**Cause** — the kernel limits how much a new process can receive, arguments
and environment together (`getconf ARG_MAX`). Xcode passes a lot. Every run
script phase gets all build settings as environment variables, plus one
`SCRIPT_INPUT_FILE_n` / `SCRIPT_OUTPUT_FILE_n` per listed input and output.
The compiler gets every search path as a flag. So the limit is hit by the
size of the project, not by your script: hundreds of pod frameworks listed as
inputs of "[CP] Embed Pods Frameworks", recursive search paths (`/**`) that
Xcode expands into every subfolder, `$(inherited)` chains that repeat the same
paths in `OTHER_CFLAGS` or `OTHER_LDFLAGS`, and a long checkout or
DerivedData path that is repeated in every one of them. It builds on a laptop
and fails on CI because the CI path is longer.

This is not [script-sandbox] or [script-path-missing]. The script is never
started, so nothing it does matters.

**Fix** — find which setting is huge, then shrink it:

```bash
# The limit, and how big the build settings already are
getconf ARG_MAX
xcodebuild -showBuildSettings -scheme App | wc -c
# The biggest settings, largest first
xcodebuild -showBuildSettings -scheme App | awk '{ print length, $1 }' | sort -rn | head
# Recursive search paths that expand to every subfolder
grep -n '/\*\*' App.xcodeproj/project.pbxproj Pods/Target\ Support\ Files/*/*.xcconfig
```

- Move script inputs and outputs into `.xcfilelist` files instead of listing
  them one by one. With CocoaPods, or to drop them entirely, set
  `install! 'cocoapods', :disable_input_output_paths => true` in the Podfile.
- Replace `/**` search paths with the folders that really hold headers.
- Remove duplicate entries from `$(inherited)` chains.
- Shorten the paths: check out to a short folder and pass a short
  `-derivedDataPath`.

**Rule:** the limit is the size of everything Xcode passes, not the size of
your command. Measure `-showBuildSettings` before blaming the script.

---

## [extension-unsafe-api] Extension build fails: 'shared' is unavailable in application extensions for iOS

**Symptom** — the app built fine until a widget, share or notification
extension started using a shared module. Now that module fails to compile,
often inside a package you did not change:

```
error: 'shared' is unavailable in application extensions for iOS: Use view controller based solutions where appropriate instead.
error: 'sharedApplication' is unavailable: not available on iOS (App Extension)
ld: warning: linking against a dylib which is not safe for use in application extensions: .../Analytics.framework/Analytics
```

**Cause** — extension targets build with `APPLICATION_EXTENSION_API_ONLY = YES`.
With it on, the compiler hides every API marked unavailable for extensions:
`UIApplication.shared`, `openURL`, background tasks and similar. The setting
spreads: a framework or Swift package that an extension links is compiled
the same way, so code that was fine in the app alone now breaks. A prebuilt
framework without the flag is not an error, only the linker warning above,
until it calls such an API at runtime from the extension.

This is not [undefined-symbols]. The symbol exists. The compiler refuses it
on purpose.

**Fix** — find who calls what, then keep that code out of the extension:

```bash
# Which targets build extension-safe?
xcodebuild -showBuildSettings -scheme App | grep -E '^\s*(TARGET_NAME|APPLICATION_EXTENSION_API_ONLY) '
# The calls the compiler refuses
grep -rnE 'UIApplication\.shared|sharedApplication' Sources Packages
# Is a prebuilt framework marked extension-safe? (look for APP_EXTENSION_SAFE)
otool -hv Analytics.framework/Analytics
```

- Mark the calling code `@available(iOSApplicationExtension, unavailable)`
  (Objective-C: `NS_EXTENSION_UNAVAILABLE_IOS`). Then the module compiles,
  and only the extension is stopped from calling that code.
- Better: pass what the module needs (an `open(url:)` closure, a scene)
  from the app instead of reaching for `UIApplication.shared` inside it.
- Split the module: an extension-safe core, and an app-only part that the
  extension does not link.
- Do not hide the call with `value(forKey: "sharedApplication")`. It compiles,
  then breaks at runtime and in review.

**Rule:** anything an extension links must be extension-safe. Pass app-only
things in from the app; don't reach for them from inside shared code.

---

## [installed-app-id-mismatch] Install on device fails: application-identifier entitlement does not match the installed application

**Symptom** — the build and signing succeed, then the install onto a physical
device stops. The same build installs fine on a device that never had the app:

```
This application's application-identifier entitlement does not match that of the installed application. These values must match for an upgrade to be allowed.
MismatchedApplicationIdentifierEntitlement
```

**Cause** — the device already has an app with this bundle ID, signed by a
different team. The `application-identifier` entitlement is
`<App ID prefix>.<bundle ID>`, and iOS only upgrades an app in place when
that full value matches. Common sources: a TestFlight or App Store build
(team prefix, or an old legacy App ID prefix), a teammate's build signed with
a personal team, or a switch of `DEVELOPMENT_TEAM` in the project.

This is not [device-not-in-profile] or [entitlements-mismatch]. The new
build is valid. It is the copy on the device that blocks it.

**Fix** — compare the two values, then remove the old copy:

```bash
# Prefix the new build will install with
codesign -d --entitlements - MyApp.app 2>/dev/null | grep -A1 application-identifier
# Is the bundle ID already on the device?
xcrun devicectl device info apps --device "$DEVICE_ID" | grep com.example.app
# Remove it (deletes its data), then install again
xcrun devicectl device uninstall app --device "$DEVICE_ID" com.example.app
```

- Uninstalling deletes the app's local data. Export anything you need first.
- On shared test devices, uninstall in the CI step before install, or give
  debug builds their own bundle ID suffix (`com.example.app.dev`) so they
  never collide with TestFlight builds.
- Keep one `DEVELOPMENT_TEAM` for everyone. Do not remove the
  `application-identifier` key from the entitlements file. The profile sets it.

**Rule:** "does not match that of the installed application" means a different team signed the app already on the device; uninstall it (or use a separate debug bundle ID), don't touch signing.

---

## Fast triage

Work down this list before deep-diving a log:

1. `xcode-select -p` — right Xcode? → [wrong-developer-dir]; `echo $TOOLCHAINS` empty? → [toolchains-override]; `uname -m` says x86_64 on Apple silicon? → [rosetta-shell]; `xcrun swift --version` older than a package needs? → [swift-tools-version]
2. `xcodebuild -list` — scheme actually exists and is shared? → [scheme-not-found]
3. `xcodebuild -showdestinations` — destination actually exists? → [no-matching-destination]
4. `df -h` — disk not full? → [disk-space]; `ulimit -n` only 256? → [too-many-open-files]
5. Wipe DerivedData, retry once. → [stale-derived-data]; "does not match previously recorded value"? → [spm-fingerprint-mismatch]; space in `pwd`? → [path-with-spaces]; binaries are "ASCII text"? → [lfs-pointer]
6. Still failing? `grep -nE "error:" build.log | head -30` and read the *first* error. Green CI but `BUILD FAILED` in the log? → [pipe-masks-exit]; tests green but the report step says "--legacy flag is required"? → [xcresulttool-legacy]; upload says "Redundant Binary Upload"? → [build-number-reused]; upload says "ITMS-90725"? → [sdk-too-old]; notarytool says "Invalid"? → [notarization-invalid]; email says "ITMS-90683"? → [purpose-string-missing]; upload says "ITMS-90205" or "90206"? → [nested-frameworks]; upload says "ITMS-90713", "90022" or "90717"? → [app-icon-missing]; upload says "ITMS-90208", "90530" or "90360" on a framework? → [framework-min-os]; importing a shipped framework says "is not a member type of"? → [interface-type-shadows-module]; importing a framework says "Missing required module"? → [missing-required-module]; upload says "ITMS-90474" or "90475"? → [ipad-orientations]; upload says "ITMS-90426" or "90424"? → [swift-support-missing]; "cannot execute tool 'metal'"? → [metal-toolchain-missing]; upload says "ITMS-90685"? → [bundle-id-collision]; helper target says "No such module 'XCTest'"? → [testing-search-paths]; upload says "ITMS-91065"? → [sdk-signature-missing]; upload says "ITMS-90171"? → [stray-binary-in-bundle]; email says "ITMS-90338"? → [non-public-api]; upload says "ITMS-90060" or "90058"? → [version-string-format]; upload says "ITMS-90035"? → [modified-after-signing]; upload says "ITMS-90087"? → [unsupported-architectures]; "doesn't include signing certificate"? → [profile-cert-mismatch]; upload says "ITMS-90111"? → [beta-toolchain-upload]; "risks causing data races" or "is not concurrency-safe" as errors? → [swift6-language-mode]; "was not compiled for testing"? → [testability-disabled]; "invalid redeclaration" in `GeneratedAssetSymbols.swift`? → [asset-symbol-collision]; email says "ITMS-90473"? → [extension-version-mismatch]; UI tests say "Timed out while enabling automation mode"? → [automation-mode-timeout]; export says "Cloud signing permission error"? → [cloud-signing-permission]; "PLA Update available" or "REQUIRED_AGREEMENTS_MISSING"? → [pla-not-accepted]; green tests but "Executed 0 tests"? → [only-testing-no-match]; "checksum of downloaded artifact of binary target"? → [binary-target-checksum]; device install says "0xe8008015"? → [device-not-in-profile]; green tests but "No profiles could be merged" or empty coverage? → [coverage-not-collected]; "CompileXCStrings" says "isn't in the correct format"? → [xcstrings-invalid]; `.m` file says "-Swift.h' file not found"? → [swift-header-not-found]; "___asan_init" undefined or "libclang_rt.asan" not loaded? → [sanitizer-runtime-missing]; "operation never finished bootstrapping" or "crashed with signal … before establishing connection"? → [test-bootstrap-crash]; "is linked as a static library by" two targets, or "Class … is implemented in both"? → [spm-static-linked-twice]; "Bundle' has no member 'module'" or "unable to find bundle named"? → [spm-resource-bundle]; "unable to spawn process" or "posix_spawn failed" with "Argument list too long"? → [argument-list-too-long]; "is unavailable in application extensions"? → [extension-unsafe-api]; device install says "does not match that of the installed application"? → [installed-app-id-mismatch]

If steps 1–5 change the outcome, it was the environment. If they do not, it is
the code — and only then is the diff worth reading.
