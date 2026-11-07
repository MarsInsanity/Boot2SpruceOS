# Boot2SpruceOS (Anbernic RG552)

The Android app, labelled "SpruceOS" on the device, that takes an Anbernic
RG552 from its stock Android (eMMC) back to dArkMoss and spruce on the TF1 card.
It replaces the "RG552 Boot Switch" app, and keeps its package name
(`org.darkmoss.rg552bootswitch`), so the two never sit side by side.

The `rg552` branch is the RG552 app. It used to live in dArkMoss's RG552 fork
([MarsInsanity/BobMoss](https://github.com/MarsInsanity/BobMoss), branch `rg552`,
`android/rg552-bootswitch/`); its history came along.

Download: the release tagged [`RG552`](https://github.com/MarsInsanity/Boot2SpruceOS/releases/tag/RG552), which always
holds the newest build. Older builds are kept as `rg552-<date built>`.

## How the switch works

- **dArkMoss → Android**: spruce's *Boot to Android* app runs
  `rg552-bootswitch android`. It saves the first sector of each boot loader
  stage on the TF1 card into its unused sectors from 40 (marker in 39), blanks
  them, and reboots. With no loader on TF1, the RG552 starts Android.
- **Android → dArkMoss**: this app runs the same script with `linux`, which
  copies the saved sectors back, then reboots.

The script is dArkMoss's own: `scripts/rg552/rg552-bootswitch` in
MarsInsanity/BobMoss, branch `rg552`, also installed on the card as
`/usr/local/sbin/rg552-bootswitch`. `build.sh` downloads it and bundles it as an
asset, so the app and spruce's *Boot to Android* always agree on how the saved
loader is marked. Each release names the commit it took the script from.

## Using it

One screen, made for the controller and laid out like spruce's own (spruce's
tree, font and theme colours):

- *...Checking if SpruceOS can be booted into*: it reads the TF1 card.
- *Press the A button to boot into SpruceOS*: A puts the loader back and
  reboots into dArkMoss and spruce.
- *dArkMoss is not bootable, ...*: no dArkMoss card with a saved loader in TF1.
  Insert one and press A to scan again (coming back to the app scans too).

B or Back leaves. A tap anywhere does what A does. If the RG552's A button
does nothing, `adb logcat -s SpruceOS` shows each key's code and scan code.

The app needs **root** (the firmware's su, or Magisk) because it writes raw
sectors of the SD card. Without root, the same can be done with
`adb root` + `adb shell sh rg552-bootswitch.sh linux`, or from a Linux/macOS PC
with the card in a reader: see `rg552-bootswitch/README.txt` on the card's
BOOT partition.

## Icon

spruce's tree (spruceOS `spruce/imgs/tree_sm_close_crop.png`), traced to
straight edges by `tools/make-icons.py`, which writes:

- `res/mipmap-*/ic_launcher.png`: the round icon Android 7.1 (the RG552's own
  Android) and older use;
- `res/drawable/ic_launcher_foreground.xml`: the adaptive icon's foreground
  (Android 8+) and its monochrome layer, which Android 13+ tints to the
  wallpaper's colours when themed icons are on. The tree is 48 dp tall on the
  108 dp canvas, inside the 66 dp circle every launcher mask keeps.

The font is spruce's theme font (`Themes/SPRUCE/nunwen.ttf`, merged Nunito,
SIL Open Font License: `assets/fonts/OFL.txt`), cut down to Latin characters.

## Building

GitHub Actions builds it on every push to `rg552` (`build-rg552` workflow) and
publishes the `RG552` release. Locally, with an Android SDK (a platform of API
33 or later), a JDK, git and curl:

    ANDROID_HOME=~/Android/Sdk ./build.sh

No Gradle: `build.sh` drives aapt2, javac, d8, zipalign and apksigner directly.
`BOOTSWITCH_SCRIPT=path` bundles a local copy of the script instead of
downloading it; `BOOTSWITCH_REPO` and `BOOTSWITCH_REF` pick where it is
downloaded from (default `MarsInsanity/BobMoss`, `rg552`).

## Signing key (so updates install over the previous app)

Android only installs an update over an app signed with the same key. The
workflow signs with the keystore in this repository's secret
`RG552_APK_KEYSTORE` (base64), password `RG552_APK_KEYSTORE_PASS`, alias
`RG552_APK_KEY_ALIAS` (optional, default `rg552`); without the secret it falls
back to a throwaway key and warns. To create it once, on any PC with a JDK and
the GitHub CLI:

    keytool -genkeypair -keystore rg552.keystore -alias rg552 -keyalg RSA \
      -keysize 2048 -validity 10000 -dname "CN=dArkMoss RG552"
    base64 -w0 rg552.keystore | gh secret set RG552_APK_KEYSTORE -R MarsInsanity/Boot2SpruceOS
    gh secret set RG552_APK_KEYSTORE_PASS -R MarsInsanity/Boot2SpruceOS

(keytool asks for the password; give the same one to the second `gh secret
set`. On macOS, `base64 -i rg552.keystore` instead of `base64 -w0`.) Keep
`rg552.keystore` somewhere safe: an app signed with a different key cannot
update this one. The first build signed with it still needs the old app
uninstalled once. Version codes are minutes since the epoch, so every build is
newer than the last. Locally, set `KEYSTORE`, `KEYSTORE_PASS` and `KEY_ALIAS`
for `build.sh` to sign with the same key.
