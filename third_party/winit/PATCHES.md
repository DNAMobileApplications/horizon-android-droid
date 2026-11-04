# Winit Android input patch

Upstream: winit 0.30.13 from crates.io, original Apache-2.0 license retained.

Android-only mouse hover/buttons/wheel/relative events and modifier snapshots
are forwarded as normal winit events. Android touch and absolute mouse coordinates use the same safe-area/aspect
rectangle as presentation, including resolution scaling. New touches outside
that rectangle are ignored; existing gestures still receive their releases.
Desktop backends retain their upstream behaviour. A stock Android NativeActivity host is used for full
mouse input; no project Java/Kotlin host is included.

Host input (IronBridge Launcher): `platform::android::host_input` lets a host app's own
Android views feed the event loop. Pushed keys, text, pointer positions, buttons, wheel and
relative motion are emitted as ordinary `KeyboardInput`, `Ime::Commit`, `CursorMoved`,
`MouseInput`, `MouseWheel` and `DeviceEvent::MouseMotion` events from one pointer/keyboard
device (pointer positions get the same presentation mapping as a real mouse). When enabled
with `gamepads_to_host`, controller key and joystick events are left unhandled so Android
delivers them to the host activity's views. Disabled by default; the standalone app is unchanged.
