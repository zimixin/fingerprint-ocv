# fingerprint-ocv (zimixin fork)

**USB fingerprint sensor driver** for the FPC 9201 controller used in Redmi Book 16 and compatible laptops.

- Device: `FPC Sensor Controller L:0001` (USB `10a5:9201`, FW `021.26.2.x`)
- This is a **fork** of [vrolife/fingerprint-ocv](https://github.com/vrolife/fingerprint-ocv)
  (AGPL-3.0) with the changes below to build and run robustly on modern Arch/
  Rolling with system OpenCV 5 and to fix template persistence.

> Upstream does **not** support this device through libfprint/fprintd. This
> driver talks the vendor bulk protocol directly and exposes a standard
> fprintd-style D-Bus API (`net.reactivated.Fprint`).

---

## What's different from upstream

| Change | File | Why |
|---|---|---|
| Removed `CMAKE_TOOLCHAIN_FILE` (vcpkg) | `CMakeLists.txt` | Use system OpenCV instead of the 2022-era pinned vcpkg snapshot |
| `OpenCV_DIR` -> `/usr/lib/cmake/opencv5` | `src/CMakeLists.txt` | Arch ships OpenCV 5; point CMake at it |
| Renamed OpenCV targets `opencv_features2d`→`opencv_features`, `opencv_calib3d`→`opencv_calib` | `src/CMakeLists.txt` | OpenCV 5 changed module/target names |
| `::umask(0600)` → `::umask(0077)` | `src/main.cpp` | **Persistence fix.** Old mask turned `0666 → 0066`, so the daemon could not read its own template file → `load()` failed silently → the enrolled print was "lost" after a restart. With `0077` templates are written `0600` and survive restarts. |

Everything else matches upstream. Compatible with the original build too — the
patches only relax the toolchain and fix the persistence bug.

---

## Build (Arch, system OpenCV 5)

```bash
sudo pacman -S libusb libevent libdbus openssl opencv make cmake pkg-config gcc

git clone git@github.com:zimixin/fingerprint-ocv.git
cd fingerprint-ocv
git submodule init
git submodule update            # pulls jinx, asyncusb, asyncdbus (git@ github paths)
cmake -S . -B build            # uses /usr/lib/cmake/opencv5
cmake --build build
```

Binary: `build/src/fingerprint-ocv`.

> Note: the upstream `vcpkg` submodule is **not needed** — we build against
> system OpenCV. You can skip `git submodule update` for `vcpkg/`.

---

## Install

### udev rule — give the sensor USB access

The USB device is root-only by default. `/etc/udev/rules.d/90-fpc-fingerprint.rules`:

```
SUBSYSTEM=="usb", ATTR{idVendor}=="10a5", ATTR{idProduct}=="9201", MODE="0660", GROUP="wheel"
```

Then:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=usb
```

### systemd user service

`~/.config/systemd/user/fingerprint-ocv.service`:

```ini
[Unit]
Description=FPC 9201 fingerprint sensor driver (fingerprint-ocv)
After=graphical-session.target

[Service]
Type=dbus
BusName=net.reactivated.Fprint
UMask=0077
WorkingDirectory=/var/lib/fprint
ExecStart=/home/<you>/fingerprint-ocv/build/src/fingerprint-ocv --bus=session --data-path=/var/lib/fprint --min-area=60000
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
```

- `--min-area=60000`: enrollment calibration. The driver default `120000` was
  tuned for a larger scanner; the FPC 9201 frame is 112×88 = 9856 px, so each
  enrollment stage needs ~6000 px of new coverage (about one good press).
  `60000` makes enrollment finish in ~10–15 discrete presses. Verification
  quality does not depend on this (see `min-score`).
- Storage dir `/var/lib/fprint` must be owned by the user.

Enable & start:

```bash
systemctl --user daemon-reload
systemctl --user enable --now fingerprint-ocv.service
```

---

## Using the D-Bus API

Namespace `net.reactivated.Fprint`, device object `/net/reactivated/Device/0`.

```bash
# list enrolled prints (user name = login user)
busctl --user call net.reactivated.Fprint /net/reactivated/Device/0 \
  net.reactivated.Fprint.Device ListEnrolledFingers s $USER
# → as 1 "primary"   (print names, not the user) | Error.NoEnrolledPrints (none)

# enroll a NEW print (must use a fresh name — same name OVERWRITES)
busctl --user call net.reactivated.Fprint /net/reactivated/Device/0 \
  net.reactivated.Fprint.Device EnrollStart s secondary

# verify against any enrolled print
busctl --user call net.reactivated.Fprint /net/reactivated/Device/0 \
  net.reactivated.Fprint.Device VerifyStart s any
```

Important quirks:

- **Names, not the user.** `EnrollStart(name)` / `VerifyStart(name)` take the
  fingerprint NAME; `is_any()` treats empty or `"any"` as "check every print".
- **Same name overwrites.** Prints are keyed by (user, name); re-enrolling
  `primary` replaces the previous `primary` print. Use a fresh name per finger.
- **Enrollment needs discrete presses.** Touch ~1 s, lift, pause ~2 s, repeat.
  Continuous pressure keeps scanning the same region and `merge()` gets no new
  area → stages do not advance (`enroll-remove-and-retry`). 10 stages.
- **`finger-present` latches.** There is no finger-up event; after the first
  touch of a scan the property stays `true` until the next scan starts. Don't
  use it as "finger is touching now".

---

## Limits

- No PAM integration by default (this is a raw D-Bus daemon; wire it to
  `pam_fprintd` / a PAM module separately if needed).
- Not concurrency-safe on D-Bus: concurrent queries can abort the daemon
  (`AsyncDBusSendWithReply`). Avoid background polling.

---

## License

AGPL-3.0 — same as upstream. See `LICENSE`.