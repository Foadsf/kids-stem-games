# F-Droid submission drafts

The four `.yml` files here are metadata files for
[fdroiddata](https://gitlab.com/fdroid/fdroiddata), one per game. They live here only as drafts:
the copies F-Droid uses are the ones merged into fdroiddata. Nothing has been submitted.

| Game | Metadata file | `subdir` |
|---|---|---|
| Robot Path | [`io.github.foadsf.robotpath.yml`](io.github.foadsf.robotpath.yml) | `robot-path/app` |
| Pattern Machine | [`io.github.foadsf.patternmachine.yml`](io.github.foadsf.patternmachine.yml) | `pattern-machine/app` |
| Kangaroo Hops | [`io.github.foadsf.kangaroohops.yml`](io.github.foadsf.kangaroohops.yml) | `kangaroo-hops/app` |
| Balance Scale | [`io.github.foadsf.balancescale.yml`](io.github.foadsf.balancescale.yml) | `balance-scale/app` |

Each build pins the full commit hash of tag `v0.1.0` (`b86ed86`). The `prebuild` installs
platform `android-35` and build-tools `37.0.0`, the same pair CI installs. Gradle 9.8.1 is in
F-Droid's [gradle-transparency-log](https://gitlab.com/fdroid/gradle-transparency-log), so
`gradlew-fdroid` accepts it. `SourceCode` points at each game's folder (`/tree/HEAD/…`, which
`fdroid lint` asks for instead of `/tree/main/…`); all four share one issue tracker.

## Store text: why `fastlane/` moved into `app/`

Each game's store text and screenshots used to sit in `<game>/fastlane/`. They are now in
`<game>/app/fastlane/`. This was the only change needed outside `fdroid/`. The reason:

F-Droid builds each app from its own full checkout of this repository, so every checkout
contains all four games' `fastlane/` folders. `fdroid update` therefore has to decide which one
belongs to the app. In fdroidserver `master` (`insert_localized_app_metadata` in `update.py`) it
accepts:

- `fastlane/` at the repository root,
- `src/<flavor>/fastlane/` for a Gradle flavor in the build,
- `<subdir>/fastlane/`, where the path before `fastlane` must be exactly the build's `subdir`.

Our `subdir` is `robot-path/app` (it has to point at the module holding `versionCode`, or
`checkupdates` cannot read the version), so `robot-path/fastlane/` matched none of these. It was
skipped with the debug message *"Not scanning … with unknown subdir"*. Nothing was duplicated: the
files were moved with `git mv`, and the screenshot links in the root README were updated.

This was checked by running fdroidserver `master`'s own `insert_localized_app_metadata` (commit
`5806784`, 2026-10-12) on a checkout of each app, laid out as `fdroid update` sees it:

| Layout | name / summary / description / changelog | screenshots |
|---|---|---|
| `v0.1.0` (`<game>/fastlane/`) | none found, for all four | 0 |
| this branch (`<game>/app/fastlane/`) | all found, for all four | 3 each |

Every checkout holds all four games' folders, and each app still picked up only its own text
(Robot Path got *Robot Path*, and so on).

The released fdroidserver 2.4.5 cannot read `<subdir>/fastlane/` at all, only the root. That
does not matter for f-droid.org, whose fdroiddata CI runs fdroidserver from git `master`, but a
self-hosted repo on 2.4.5 would show these apps without descriptions.

**Consequence for the first build:** tag `v0.1.0` predates the move. If F-Droid builds
`b86ed86`, the apps would be published without store text or screenshots. Build the first
release from a commit that contains `<game>/app/fastlane/` instead (see *Submitting*).

## Tags: one tag per app, not one for the whole repo

**Recommendation: per-app tags `<game>-vX.Y.Z`**, for example `robot-path-v0.1.1`. Each YAML
has a matching `UpdateCheckMode: Tags ^robot-path-v\d+\.\d+\.\d+$` and
`AutoUpdateMode: Version`.

How fdroidserver uses tags (`checkupdates.py`, `check_tags`): it lists the repository's tags,
keeps those that match the `Tags` pattern, checks out at most the **five newest**, reads
`versionCode` and `versionName` from the app's `subdir`, and takes the highest `versionCode`.
With `AutoUpdateMode: Version`, a new build is added whose `commit` is that tag's commit hash.

This was tried with `fdroid checkupdates --auto` on a scratch clone with local tags (nothing was
pushed). The clone had `robot-path-v0.1.0` and `pattern-machine-v0.1.0`, then a commit bumping
Robot Path to 0.1.1 (versionCode 2) tagged `robot-path-v0.1.1` and `v0.2.0`, then a commit
bumping Pattern Machine to 0.2.0 tagged only with the repo-wide `v0.3.0`:

- Robot Path updated to 0.1.1 (2), and a new build entry was generated with
  `subdir: robot-path/app` and the commit of `robot-path-v0.1.1`.
- Pattern Machine stayed at 0.1.0 (1): its pattern ignores `v0.3.0`.
- Control: with a plain `UpdateCheckMode: Tags`, Pattern Machine took `v0.3.0` and moved to
  0.2.0 (2). So the pattern, not chance, is what kept the games apart.

Shared tags (`v0.2.0` for the whole repo) would work, but badly:

- **The games cannot be released on their own.** Fixing only Balance Scale still means tagging
  the whole repo. For the other three, `checkupdates` checks out the new tag and finds the same
  `versionCode`. Nothing breaks, but the tag name no longer tells you which app changed.
- **The tag says nothing about the app's version.** Robot Path may be at 0.3.0 while the repo
  tag says `v0.5.0`. Nothing enforces that the two match, and the store shows `versionName`.
- **The five-tag window is shared.** Five quick releases of one game push the others' last
  release out of the window. That is harmless today, but it is a trap.

Per-app tags avoid all three: each app sees only its own tags. fdroiddata already contains
monorepo apps that do this (for example `UpdateCheckMode: Tags ^android-v` with
`subdir: android/app`, and `Tags ^client-v[0-9.]+$`).

`v0.1.0` can stay as a repository-wide release tag on GitHub. The patterns ignore it.

## How it was checked (2026-10-18, fdroidserver 2.4.5 in `~/dev/envs/fdroidserver`)

- **`fdroid lint`** passes for all four in a current fdroiddata checkout, and
  `fdroid rewritemeta` leaves every file byte-for-byte unchanged. Lint did check them: the
  first draft used `SourceCode: …/tree/main/<game>` and lint rejected all four with
  *"Use /HEAD instead of /master or /main"*.
- **`fdroid build -l <appid>`** built all four from `b86ed86`:

  | App | Unsigned release APK | Label | versionCode | minSdk | Permissions | Locales |
  |---|---|---|---|---|---|---|
  | `io.github.foadsf.robotpath` | 54,343 B | Robot Path | 1 | 19 | none | en, nl, fa |
  | `io.github.foadsf.patternmachine` | 51,943 B | Pattern Machine | 1 | 19 | none | en, nl, fa |
  | `io.github.foadsf.kangaroohops` | 53,171 B | Kangaroo Hops | 1 | 19 | none | en, nl, fa |
  | `io.github.foadsf.balancescale` | 49,203 B | Balance Scale | 1 | 19 | none | en, nl, fa |

  The values were read from the APKs with `aapt2 dump badging` / `dump permissions`. The source
  scanner found nothing to flag. It only removed the four `gradle-wrapper.jar` files and
  stripped the signing configs from the four `build.gradle.kts`.
  **A zero exit code was not enough here.** The first local run of all four exited 0 and
  produced no APK: each had failed in `prebuild` with `sdkmanager: command not found`, because
  the Android SDK was not on `PATH` on this machine. With `ANDROID_HOME` and
  `cmdline-tools/latest/bin` on `PATH`, the same metadata built cleanly. On F-Droid's
  buildserver `sdkmanager` is on `PATH`, and that is where these files are meant to run.

- **The release APKs were played on an Android 4.4 emulator.** F-Droid ships the
  R8-minified release build, while CI and all earlier testing used debug builds. So the four
  unsigned APKs from `fdroid build` were signed with a throwaway key and installed on an API 19
  emulator shaped like the Galaxy S4 mini (540×960). Each install was confirmed with `pm path`.
  Then each game's own `tools/verify.py` played it:

  | Game | Checks passed | Failed |
  |---|---|---|
  | Robot Path | 11 of 11 | none |
  | Pattern Machine | 23 of 23 | none |
  | Kangaroo Hops | 27 of 28 | stage 5 "which is more?" |
  | Balance Scale | 17, then stopped | stage 6 bananas |

  Both failures come from the test script, not the app. To jump ahead to a later stage,
  `verify.py` writes the saved progress with `run-as`, and its own docstring says that step needs
  a debug build. On these release APKs the emulator answers
  `run-as: Package '…' is not debuggable`. So the game stayed at stage 0 or 1, and the checks for
  the later stage failed. Everything before that seeding step passed. That covers normal play,
  hints, stickers, *More!*, the language chip and Farsi digits.
  **Not covered on the release build:** Kangaroo Hops' later stages and Balance Scale from
  stage 6 on (bananas, mixed fruit). The same scripts cover them on a debug build, and R8 only
  renames classes here. The app uses no reflection; the one `getIdentifier` call looks up the Android
  framework's `status_bar_height`, which shrinking does not touch.

## Inclusion criteria

- **Licence:** GPL-3.0 (`LICENSE`, and a copy in each game folder). No file says "or later", so
  the SPDX tag is `GPL-3.0-only`. `fdroid lint` accepts it.
- **No dependencies:** each game's `gradle/libs.versions.toml` contains only the Android Gradle
  Plugin. There are no libraries at all, so no non-free ones. Repositories are `google()` and
  `mavenCentral()` only, for the plugin. `dependenciesInfo` is off, so the APK has no
  Google-encrypted dependency block.
- **No network, no tracking:** no manifest declares a permission, and no source file uses
  `java.net`, sockets or `WebView`. No anti-features apply.
- **Bundled asset:** `app/src/main/assets/fonts/fredoka_semibold.ttf` in each game is Fredoka
  under SIL OFL 1.1, recorded in each game's `licenses/`. It is byte-identical (SHA-256
  `c773adb7…a335`) to the font in Kid Launcher, whose `tools/make_font.sh` rebuilds it from
  upstream google/fonts against a pinned SHA-256.
- **Fastlane:** `title`, `short_description` (71 to 75 characters, under F-Droid's 80-character
  cut-off), `full_description`, `changelogs/1.txt` and three phone screenshots per game,
  English only. The apps themselves are in English, Dutch and Farsi.

## What a reviewer may ask about

- **The monorepo itself.** Four apps share one git repository, so each fdroiddata entry clones
  the whole repository. A checkout is small (2.7 MB of files plus 1.3 MB of git history). The source
  scanner checks the whole checkout and only removes the four `gradle-wrapper.jar` files.
- **Copied code.** The games share some code by copying, not through a library module:
  `Backdrop` and `StickerView` are identical in all four apart from the package line, while
  `CandyButton` and `Sounds` started that way and now differ per game. That keeps each Gradle
  project self-contained and the build trivial, at the price of duplication. The four are
  different games, not one app published four times.
- **Category.** All four are listed as `Educational Game`. Robot Path and Pattern Machine could
  also go under `Puzzle Game`.
- **Debug signing hook.** Each `app/build.gradle.kts` looks for an optional `debug.keystore`,
  including a path outside the repository. That only affects debug builds; fdroidserver strips
  `signingConfigs` and signs the release APK itself.
- **Release build.** F-Droid builds the R8-minified release variant. CI builds debug only.

## Submitting (Foad's decision; nothing here has been sent)

1. Merge this branch. Then tag the merge commit once per app: `robot-path-v0.1.0`,
   `pattern-machine-v0.1.0`, `kangaroo-hops-v0.1.0`, `balance-scale-v0.1.0`, and push the tags.
   The code is the same as in `v0.1.0`; only the store text moved. Keeping versionCode 1 is fine,
   because nothing has been published yet.
2. In each YAML, set `commit:` to that commit's full hash.
3. Fork https://gitlab.com/fdroid/fdroiddata and copy the four files to `metadata/` on a new
   branch.
4. In the fdroiddata checkout, run `fdroid lint` and `fdroid rewritemeta` on the four app IDs.
   Both should leave the files as they are. `fdroid checkupdates` should now find each app's tag.
5. Push the branch and open one merge request for all four, using the *App inclusion* template.
   Four apps from one repository are easier to review together.

## Releasing a new version of one game

Bump `versionCode` and `versionName` in `<game>/app/build.gradle.kts`, add
`<game>/app/fastlane/metadata/android/en-US/changelogs/<versionCode>.txt`, commit, and push a
`<game>-vX.Y.Z` tag. F-Droid picks it up by itself; the other three games are not affected.
