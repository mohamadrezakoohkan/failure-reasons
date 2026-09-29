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
it by hand: run `tuist generate` / `xcodegen` again. Add
`*.pbxproj merge=union` only if you accept that you must check each merge.

**Rule:** `plutil -lint project.pbxproj` must pass before a merge is pushed, and the whole team must build with the same Xcode that sets `objectVersion`.

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
