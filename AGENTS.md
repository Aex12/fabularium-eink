# Fabularium (e-ink fork): notes for agents

This is a fork of Fabularium (an Android interactive-fiction player with
Glk/garglk interpreters in native code). Its main target is **e-ink Android
tablets**, with Android 11 as the reference device. Every UI change should keep
the screen high-contrast and static:

- Use pure black (`#000000`) on pure white (`#FFFFFF`). Avoid mid-greys,
  translucency and hues: e-paper dithers them into low-contrast grey. The only
  allowed grey is `eink_pressed` (`#BBBBBB`), used for pressed/checked fills.
- Use no animations, ripples, gradients or elevation shadows. They ghost and
  force slow refreshes. Separate regions with solid 1dp black rules instead.
- Make icons black PNGs (`*_black_24dp`). Don't use white icons, and don't rely
  on tinting.

## Toolchain (via mise)

`mise.toml` in the repo root pins the tools:

- `java = temurin-17`: AGP 7.4 needs JDK 11+. The system JDK 25 is too new for
  Gradle 7.6.
- `android-sdk` (vfox plugin): used only to bootstrap `sdkmanager`. It sets
  `ANDROID_HOME=~/.local/share/android-sdk` and puts the SDK's
  `cmdline-tools/latest/bin`, `platform-tools` and `emulator` on PATH.

To set up a fresh machine:

```sh
mise trust && mise install
SDK=~/.local/share/android-sdk
yes | sdkmanager --sdk_root=$SDK --licenses
sdkmanager --sdk_root=$SDK "cmdline-tools;latest" "platform-tools" \
  "platforms;android-33" "build-tools;33.0.2" "ndk;25.2.9519653" "cmake;3.22.1"
git submodule update --init --recursive   # garglk terps; the build fails without it
```

**Gotcha:** the mise-installed `avdmanager` resolves the SDK root from its own
install dir and can't see the system images. Use
`$SDK/cmdline-tools/latest/bin/avdmanager`. It comes first on PATH under mise.

## Build

```sh
mise exec -- ./gradlew assembleRelease   # app/build/outputs/apk/release/app-release.apk
mise exec -- ./gradlew assembleDebug
```

- Release builds are signed with the local **debug key** so they can be
  sideloaded. `minifyEnabled` is off because there are no ProGuard keep rules
  for the JNI callbacks.
- `targetSdkVersion` stays at **28** on purpose. That keeps legacy (non-scoped)
  external storage on Android 10/11, which the library and file browser need.
  Lint reports `ExpiredTargetSdkVersion`; that's expected and non-fatal.
- `android.useAndroidX=false`: the code still uses the `com.android.support`
  28.0.0 libraries. A migration to AndroidX hasn't been done yet.
- `jni/CMakeLists.txt` adds `-fcommon`. Clang 11+ defaults to `-fno-common`,
  and some terps (alan2, others) define globals in headers. Without the flag the
  link fails with "duplicate symbol: glkMainWin".
- A full native build of all 4 ABIs takes about 1–2 minutes.

## Testing

Prefer the emulator. Fall back to the real device over adb.

```sh
avdmanager create avd -n eink30 -k "system-images;android-30;google_apis;x86_64" -d pixel_c
emulator -avd eink30 -no-window -no-audio -no-boot-anim -gpu swiftshader_indirect -no-snapshot
adb install -r app/build/outputs/apk/release/app-release.apk
adb shell am start -n io.davidar.fabularium/com.luxlunae.fabularium.MainActivity
adb exec-out screencap -p > shot.png
```

Known issue (2026-10, Fedora 44, emulator from SDK): the emulator **segfaults
during boot** with `-gpu swiftshader_indirect`, `-gpu guest`, and
`-feature -Vulkan`. It wasn't fixed. Testing was done on the real e-ink tablet
via `adb` instead. If you try again, an older emulator build or a smaller/lower
density AVD than `pixel_c` (2560x1800) are the next things to try.

### Real device: Boox Tab Ultra (Android 11, 1404x1872, adb serial `BEBEB6B3`)

- Run `adb devices` first. The user plugs the tablet in when asked.
- **Screenshots:** Boox firmware prints `capture from screenshot!` into stdout,
  so `adb exec-out screencap -p > x.png` produces a corrupt file. Use this
  instead:
  `adb shell screencap -p /sdcard/s.png && adb pull /sdcard/s.png x.png && adb shell rm /sdcard/s.png`.
  Screenshots are landscape 1872x1404, and `adb shell input tap X Y` uses the
  same coordinates.
- **Signature conflict:** the user may have the F-Droid build of
  `io.davidar.fabularium` installed, which is signed with a different key.
  `adb install` fails until it's uninstalled. Ask before uninstalling, because
  that wipes the app's data.
- Grant storage without the UI:
  `adb shell pm grant io.davidar.fabularium android.permission.WRITE_EXTERNAL_STORAGE`
  (and `READ_EXTERNAL_STORAGE`).
- After a fresh install, the on-device `/sdcard/Fabularium/fab.ini` may still
  be the old upstream copy. Back it up, then `adb push app/src/main/assets/fab.ini /sdcard/Fabularium/fab.ini`.
- The Boox per-app **Customize** panel (floating ball → Refresh / Display /
  Color / Others) can apply a hollow/outlined-text enhancement to the app. That
  is a device setting, not an app bug, and it shows up in screenshots. Make
  sure it's off before judging rendering.
- Test game: `Advent.z5` from
  `https://ifarchive.org/if-archive/games/zcode/Advent.z5`. Push it to
  `/sdcard/Fabularium/Games/`, then tap refresh in the Play tab.

Package/application id: `io.davidar.fabularium`. Activities live in
`com.luxlunae.fabularium` (library UI) and `com.luxlunae.glk.GLKActivity` (the
game, which runs in the separate `:GLKActivity` process).

## Where colours come from (theme map)

Four separate layers set colours. A change usually touches more than one.

1. **App theme** (library, file browser, IDE, editor, dialogs):
   `res/values/styles.xml` (`Theme.Fabularium` extends
   `Theme.AppCompat.Light.NoActionBar`, plus `Theme.Fabularium.Dialog`,
   `ThemeOverlay.Fabularium.Toolbar`, `Widget.Fabularium.TabLayout`) and
   `res/values/colors.xml` (`eink_*` palette). Legacy colour names such as
   `light_font`, `gold` and `background_material_dark` are re-pointed at the
   e-ink palette rather than renamed.
2. **Layouts**: the toolbars in `main_layout.xml`, `ide_layout.xml` and
   `text_editor_layout.xml` use the e-ink toolbar overlay and have a 1dp black
   divider under them. `glk_display.xml` holds the built-in keyboard colours,
   and `res/drawable/key_back*.xml` the outlined key shapes.
3. **Icons**: `res/drawable-*/ic_*_black_24dp.png` and `sym_keyboard_*.png`
   are all black-on-transparent.
   Dialog icons use `R.drawable.ic_warning_black_24dp` (a vector). Never use
   `android.R.drawable.ic_dialog_*`: those are grey gradients that dither badly
   on e-ink.
   - **Dialogs and popups** get a 2dp black border and no shadow or dim. See
     `Theme.Fabularium.Dialog` (`eink_dialog_background.xml`) and
     `Widget.Fabularium.PopupMenu*` / `ListPopupWindow`
     (`eink_popup_background.xml`). Most dialogs are AppCompat `AlertDialog`s
     that pick this up through `alertDialogTheme`. `UrlDialogFragment` passes
     the style explicitly.
4. **In-game (Glk) colours**:
   - Java defaults are in `glk/GLKConstants.java` (`GLK_DEFAULT_*_COLOUR`,
     `EMPTY_SCREEN_COLOR`) and in `glk/model/GLKModel.java` (fallbacks for the
     border and background preferences).
   - User-facing defaults are in `assets/fab.ini`: `tcolor`/`gcolor` per style,
     `windowcolor`, `linkcolor`, `tads_*`, and the per-terp `[ %adrift% ]`
     section.
   - **`fab.ini` is copied to the device's storage only on first run.**
     Existing installs keep their old copy until the user picks *Reset config files*
     in the game library's menu, or deletes the copy in the `Fabularium`
     folder.
   - Some games set their own colours. `stylehint 0` in fab.ini ignores them.
   - Only the keys handled in `GLKModel`'s config parser work in fab.ini. Some
     keys copied from Gargoyle's `garglk.ini` (for example `caretshape`) are
     **ignored**. Grep for `case "<key>"` before relying on one.
   - Echoed input in Z-code games is *not* bold, even with `tfont 8 propb`.
     bocfel resets `stylehint_Weight` for every style, Input included
     (`garglk/terps/bocfel/screen.c:2320`), and Fabularium honours game style
     hints. Possible fix: ignore weight hints for `style_Input`.
   - The built-in keyboard's hint characters use a paint colour hardcoded in
     `GLKKeyboardView` (`mHintPaint`), which is now black.
   - The in-app help is `assets/help.html` (plain CSS).
