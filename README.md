# horizon-engine

**The Rust engine inside DroidBridge Launcher.**

This is DroidBridge Launcher's fork of [Horizon Android](https://github.com/Vasysik/horizon-android),
which is the Android port of [Horizon](https://github.com/horizon-rust/horizon): Minecraft: Java
Edition 26.3, rewritten in Rust. DroidBridge builds this fork into its APK and offers it per
instance as **Rust (experimental)**, next to its regular Java engine.

It is not a standalone app. There is no separate APK to install: DroidBridge starts the engine,
hands it the game folder and the player's verified account, and runs its own controls over it.
For the standalone Android app, see [Horizon Android](https://github.com/Vasysik/horizon-android);
for the PC game, see [Horizon](https://github.com/horizon-rust/horizon).

> Unofficial. Not affiliated with or endorsed by Mojang or Microsoft, nor by the Horizon or
> Horizon Android projects. Playing still needs a Microsoft account that owns Minecraft: Java
> Edition; DroidBridge checks ownership before the engine starts.

## What this fork adds

Everything from Horizon Android (Vulkan rendering, the native Minecraft 26.3 client, `.hmod`
mods, resource packs, Java multiplayer), plus what DroidBridge needs to host it:

- **Launched by DroidBridge.** The engine skips its own launcher screens and goes straight into
  the game, using DroidBridge's instance folder and the account session DroidBridge already
  verified. Leaving the game saves the world and returns to DroidBridge.
- **DroidBridge's controls.** DroidBridge's touch layouts, virtual mouse, controller mapper and
  gyroscope camera drive the game as keyboard and mouse input, Shift and Ctrl state included.
  The engine's own touch UI and keypad stay off.
- **Physical mouse and keyboard.** Both *Native Mouse* and *Android Virtual Mouse* work. With
  Native Mouse, the game's window captures the mouse in gameplay and menus, so Android's
  pointer never reaches the screen edges (and the system bars). DroidBridge's hardware mouse
  DPI scale and hide-cursor setting apply.
- **DroidBridge owns the Android keyboard.** The engine never opens or closes it. While it is
  open, the whole picture is lifted above it, with chat sitting just above the keyboard.
- **Vulkan driver choice.** The engine can use the Vulkan driver DroidBridge loaded (Turnip,
  PanVK...) instead of the phone's own.
- **Add-ons screen.** DroidBridge's per-instance Add-ons screen (mods, shader packs, mod catalog)
  asks the engine's own file checks, through a content provider in the engine's process.
- **Built-in native mods.** `.hnmod` native mods are compiled into the engine at build time and
  installed per instance from the Add-ons screen. The engine's own bundled mods (Zoom, Minimap,
  Recipes, SmartF3) show up there too.
- **Open-source licenses page.** Each build writes the notices for the engine, the native mods
  and every Rust library compiled in, and DroidBridge shows them.
- **Shader packs.** Horizon's own `.hshader` packs, and Iris/OptiFine `shaders/` ZIPs
  (experimental: heavy on phones, and some effects may be missing).

## Building

You normally don't build this repository by hand: DroidBridge's Gradle build does it when you
press Run in Android Studio. It calls

```sh
python tools/android_build.py --public --host-libs
```

which builds `libhorizon_android.so` and the engine's data for DroidBridge to pack into its APK.
With `--host-libs`, the script builds only what DroidBridge bundles (no standalone APK).

**You need:**

- Rust, installed with [rustup](https://rustup.rs). The version comes from `rust-toolchain.toml`
  (1.95.0) and installs by itself on the first build. Add the phone target once, from inside
  this folder so it goes to that version: `rustup target add aarch64-linux-android`.
- The Android SDK and NDK (Android Studio's SDK Manager), and Android Studio's JDK.
- Python 3.10 or newer.
- Git, and internet on the first build: Cargo downloads the Rust libraries, and the native mods
  step builds its mixin tool once.

**Output** (in `.local/android/`, which is gitignored):

| Path | What |
| --- | --- |
| `gradle-jni/<abi>/libhorizon_android.so` | The whole engine, loaded by DroidBridge's `HorizonActivity` |
| `gradle-assets/` | Engine data, `LICENSE` and the generated `THIRD_PARTY_NOTICES.md` (the licenses page) |

The first build compiles everything and takes a while; later builds only rebuild what changed.
Deleting `target/` or `.local/` is safe and only costs one full build.

**Useful switches:**

| Switch | Effect |
| --- | --- |
| `--abi arm64-v8a` | Phone CPU to build for (the default) |
| `--no-native-mods` | Build the plain engine, without native mods |
| `HORIZON_NATIVE_MODS_STRICT=1` | Stop the build when a native mod fails, instead of leaving it out |
| `HORIZON_NATIVE_MODS_FROM_PHONE=0` | Don't pick up `.hnmod` files imported on a connected phone |
| `ANDROID_SERIAL=<id>` | Which phone to use when several are connected over adb |
| `HORIZON_NATIVE_WORK=<folder>` | Where native mods are staged (default: `%LOCALAPPDATA%\DroidBridge\horizon-native-mods` on Windows) |

## Native mods

Native mods (`.hnmod`) are Rust source patched into the engine, so they are compiled into the
APK; a phone can't add one by itself (and Google Play doesn't allow downloading code that runs
outside a sandbox). Put `.hnmod` files in [`native-mods/`](native-mods/), or list catalog mod
ids in [`native-mods/registry.txt`](native-mods/registry.txt). Each starts out **not installed**
in every instance; players install it per instance in DroidBridge's Add-ons screen.

A mod that doesn't fit is left out with the reason in the build log: it needs newer hook points
than this engine has (`NATIVE_HOOKS_API` in `crates/horizon-client/src/game/native_hooks.rs`),
it has no on/off switch, it duplicates one of the engine's bundled mods, or it doesn't compile.
Details: [`native-mods/README.txt`](native-mods/README.txt).

Sandboxed `.hmod` mods and shader packs need no build: players add them on the phone.

## Repository layout

| Folder | What it is |
| --- | --- |
| `crates/` | Horizon's Rust code, one crate per folder. `horizon-client` is the game client (its `src/android/` holds the Android code), `horizon-client-render` the renderer. `horizon-android` is what becomes `libhorizon_android.so`: it builds `horizon-client/src/android_main.rs` as a shared library. |
| `third_party/` | Patched copies of `winit`, `wgpu-hal` and `naga`, used instead of the crates.io versions (`[patch.crates-io]` in `Cargo.toml`). |
| `tools/` | Python build scripts: `android_build.py`, `android_native_mods.py`, `android_notices.py`. |
| `native-mods/` | Native mods built into DroidBridge's engine. |
| `docs/` | Horizon's and Horizon Android's documentation. |
| `launcher/`, `packaging/` | Horizon's desktop launcher and Horizon Android's standalone app; not used by DroidBridge. |

### Where DroidBridge's changes live

Most are marked with a "DroidBridge" or "host" comment.

| File | Change |
| --- | --- |
| `crates/horizon-client/src/android_main.rs` | Starting under DroidBridge, with its controls |
| `crates/horizon-client/src/android/host_handoff.rs` | Launch settings from DroidBridge, and what the game reports back |
| `crates/horizon-client/src/android/host_input_jni.rs` | JNI for `dev.horizon.android.HorizonInput`: keys, text, mouse, keyboard lift, Native Mouse |
| `crates/horizon-client/src/android/host_addons_jni.rs` | JNI for `dev.horizon.android.HorizonAddonsProvider`: the Add-ons screen's checks |
| `crates/horizon-client/src/android/host_install.rs` | Downloading Minecraft 26.3's files for the engine |
| `crates/horizon-client/src/android/launcher.rs`, `platform.rs` | Straight into the game; DroidBridge owns the Android keyboard |
| `crates/horizon-client/src/game.rs`, `game/chat.rs`, `game/android_input.rs` | Small marked blocks: cursor/FPS reporting, controller camera, chat above the keyboard, returning to DroidBridge |
| `crates/horizon-client-render/src/gfx.rs`, `gfx/frame_scale.rs`, `android_runtime.rs` | Keyboard lift, GPU info, DroidBridge's input taking over |
| `third_party/winit/src/platform_impl/android/` | DroidBridge's input as keyboard/mouse events, Native Mouse menu cursor, DPI scale |
| `third_party/wgpu-hal/src/vulkan/instance.rs` | Using DroidBridge's chosen Vulkan driver |
| `tools/android_build.py`, `android_native_mods.py`, `android_notices.py`, `native-mods/` | `--host-libs`, native mods, the licenses page |

The Java side of these bridges (`dev.horizon.android.*`, the Add-ons screen and the controls
host) lives in DroidBridge Launcher's own project, not here. The JNI function names tie the two
together, so the Java classes keep the `dev.horizon.android` package.

## Keeping up with upstream

This fork follows Horizon Android's `android` branch, which in turn merges Horizon.

1. Merge the latest `android` branch from `https://github.com/Vasysik/horizon-android`.
2. Conflicts, if any, are almost always in the files listed above; keep both sides.
3. Watch for upstream doing something DroidBridge already does, then let DroidBridge's way win
   when DroidBridge is hosting (`host_handoff::get().host_controls`). Past examples: the engine
   lifting chat above the keyboard itself, and reading Shift/Ctrl only from modifier events.
4. Rebuild, then check on a phone: launching from DroidBridge, touch controls, a controller
   (sprint, Y quick move), chat with the keyboard open, Native and Virtual Mouse, a shader pack,
   and the built-in native mods.

## Licence and credits

- Horizon is © 2026 Ludovic Marie and the Horizon contributors, under the [MIT License](LICENSE).
  The Android port is by Vasysik and the Horizon Android contributors. This fork keeps their
  authorship and history.
- Code from other projects included in Horizon is listed in
  [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). Each build also generates the full notices
  for the Rust libraries and native mods it compiles in, which DroidBridge shows as its
  open-source licenses page.
- Native mods keep their own licences (see each mod's source).
- Minecraft is a trademark of Mojang Synergies AB. Its code and assets belong to Mojang and
  Microsoft under the [Minecraft EULA](https://www.minecraft.net/en-us/eula) and are not part of
  this repository or covered by its licence. Game files, prepared resources, APKs, signing keys
  and `local.properties` stay out of Git.
- DroidBridge Launcher itself is a separate app by DNA Mobile Applications; its own code is not
  in this repository.
