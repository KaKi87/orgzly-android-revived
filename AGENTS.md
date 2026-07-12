# AGENTS.md

## Cursor Cloud specific instructions

Orgzly Revived is an Android app (Kotlin/Java, Gradle). There is no server/backend to
run; the "application" is the Android APK. Standard build/test commands live in
`README.org` and `.github/workflows/test.yaml`; the notes below only cover
non-obvious, durable details for this VM.

### Environment (already provisioned by the update script + VM snapshot)
- JDK 21 is used (the CI in `.github/workflows/test.yaml` also builds with JDK 21, even
  though `app/build.gradle` targets source/target compatibility 17).
- The Android SDK is installed at `~/android-sdk`; `ANDROID_HOME`/`ANDROID_SDK_ROOT`
  are exported from `~/.bashrc`. Installed: `platform-tools`, `platforms;android-36`
  (compileSdk 36), `build-tools;35.0.0`.
- `app.properties` is required by the Gradle build and is `.gitignore`d. The update
  script copies it from `sample.app.properties`. The default (placeholder) values are
  enough to build and run all `fdroid` unit tests; only some Dropbox tests are skipped.

### Build / test / lint (use the `fdroid` flavor for development)
- Build debug APK: `./gradlew assembleFdroidDebug` (output:
  `app/build/outputs/apk/fdroid/debug/app-fdroid-debug.apk`).
- Unit tests (Robolectric, run on the JVM — the primary way to exercise core
  functionality here): `./gradlew testFdroidDebugUnitTest`.
- Lint: `./gradlew lintFdroidDebug`. Note: the codebase currently has pre-existing lint
  errors and lint is NOT run in CI, so this task fails on those existing issues; that is
  expected and unrelated to environment setup.

### Gotcha: JGit unit tests vs. this VM's git commit-signing config
`GitRepoTest` (the git-repo sync unit tests) uses JGit `5.13.x`, which throws
`IllegalArgumentException: Invalid value: gpg.format=ssh` while reading the global git
config. This VM's `~/.gitconfig` enables SSH commit signing (`gpg.format = ssh` +
`commit.gpgsign = true`) for the Cursor agent, so those 21 tests fail with the default
config even though the app code is fine (CI is green). To run the full unit-test suite
green, temporarily neutralize signing, run, then restore:

```bash
cp ~/.gitconfig /tmp/gitconfig.bak
git config --global --unset gpg.format
git config --global --unset commit.gpgsign
./gradlew testFdroidDebugUnitTest
cp /tmp/gitconfig.bak ~/.gitconfig   # restore so your own commits stay signed
```

Do NOT put this git-config mutation in the update script (it would disable the agent's
commit signing).

### Instrumented / emulator tests cannot run in this VM
`./gradlew connectedAndroidTest` (and the emulator-based jobs in `test.yaml`) require
KVM. There is no `/dev/kvm` in the Cloud VM and no ARM host, so the x86_64 emulator will
not start. Validate changes with the JVM unit tests and the debug build instead.
