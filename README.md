# Install Other OS (modified copy)

Run a full Linux inside an app on an **Xbox Series X|S in Developer Mode**.

This project is **based on [YoCanIGetUhhhh/XboxInstallOtherOS](https://github.com/YoCanIGetUhhhh/XboxInstallOtherOS)** (source snapshot version 1.2.42.0). It is a modified copy, not a GitHub fork: the code was cloned and uploaded to a separate repository. All the core work (the UWP app, the QEMU port, the JIT, the display and input code) comes from the original author. See [Credits](#credits-and-license).

The app uses [QEMU](https://www.qemu.org/) to emulate an x86_64 PC in software. The Xbox does not let apps use a hypervisor, so this is **much slower than a real PC**. It is good enough for a light desktop (XFCE), a terminal and small servers.

> **Status:** the package builds with GitHub Actions (QEMU 10.0.0 workflow, UWP audit: 0 blocked imports), installs on an **Xbox Series X** in Developer Mode, starts, and boots the **Debian 13 netinst installer** (see [Tested with](#tested-with)). Items marked *not tested* below are still unverified.

## What this version changes

| Change | Where | State |
|---|---|---|
| Two more CPU types in Settings: `Haswell` and `max` (default stays `qemu64`) | `MainPage.xaml.cs` | Builds and the app runs. `Haswell` / `max` not tested in a guest yet |
| Optional build against **QEMU 11.0.2** instead of 10.0.0 | `native/build_qemu_11.0.2.yml`, `native/patch_egl_for_angle_11x.py` | Patches tested on copies of the QEMU source; the build was **not confirmed** to finish and was **not run** on a console |

Everything else is unchanged from the original.

### Changelog

**1. CPU choices** (`MainPage.xaml.cs`, around line 422)

```csharp
// before
private static readonly string[] CpuChoices = { "qemu64", "Nehalem" };
// after
private static readonly string[] CpuChoices = { "qemu64", "Nehalem", "Haswell", "max" };
```

The Settings button **CPU type** cycles through the list in that order. The default is still `qemu64`. The choice is saved in `cpu.cfg` and passed to QEMU as `-cpu`. No other code was changed.

**2. QEMU 11.0.2 build (optional, not active by default)**

- Added `native/build_qemu_11.0.2.yml`: a copy of `build.yml` with two changed lines: `--branch v10.0.0` → `--branch v11.0.2`, and `patch_egl_for_angle.py` → `patch_egl_for_angle_11x.py`.
- Added `native/patch_egl_for_angle_11x.py`: the same ANGLE/EGL patches, but `ui/egl-headless.c` may contain 0 or more `eglMakeCurrent` calls (QEMU 10.0.0 has 1, 11.0.2 has 0, 11.1.x has 8). It still works on 10.0.0.

**3. Repository housekeeping**

- Extended `.gitignore` (macOS files, `*.orig` backups, Visual Studio and build output, packages, ISOs and the files the build workflow creates).
- Added this README.

The original `build.yml`, `native/patch_egl_for_angle.py` and all native sources are unchanged.

### CPU types

| Type | What it is |
|---|---|
| `qemu64` | Plain x86_64, basic features only. **Default and known to work.** |
| `Nehalem` | Intel feature set from 2008 (SSE4.2, POPCNT). |
| `Haswell` | Intel feature set from 2013 (AVX, AVX2, FMA, BMI). QEMU may print a harmless warning about TSX features it cannot emulate. |
| `max` | Every feature QEMU's translator supports. Newest feature set, least predictable. |

These only change which instructions the guest software may use. **They do not make the VM faster**, and emulated AVX2 code can even be slower. If a guest misbehaves, go back to `qemu64`.

### The QEMU 11.0.2 build

The original build clones QEMU `v10.0.0` and applies a set of patches. The new workflow file clones `v11.0.2` and uses an updated EGL patch (in 11.0.2, `egl-headless.c` no longer contains the call the old patch looked for). All other patches applied cleanly to a copy of 11.0.2.

It is **not active by default**. To try it, copy `native/build_qemu_11.0.2.yml` to `.github/workflows/` and run it manually. Keep the original `build.yml` as the fallback.

## Tested with

| Item | Result |
|---|---|
| Console | Xbox Series X, Developer Mode, Device Portal |
| Build | GitHub Actions `build.yml` (QEMU **v10.0.0**), `.msix` + 3 x64 `.appx` from the Release |
| App | Installs, starts, main menu shows |
| Guest ISO | Debian 13 "trixie" amd64 **netinst** (`debian-13.7.0-amd64-netinst.iso`), Xfce desktop chosen in the installer |
| Installer | Starts and runs on the Xbox (reached "Loading additional components") |
| Virtual disk | Created in Settings and stored in the app's `LocalState` |
| CPU type | `qemu64` (default) |
| Not confirmed yet | Finished install and first boot from disk, speed numbers, `Haswell` / `max`, the QEMU 11.0.2 build |

## Tools used

- **GitHub Actions** (`windows-latest`) with **MSYS2 / MinGW (UCRT64)** to build QEMU as a DLL
- **QEMU 10.0.0** (optional test: 11.0.2), **virglrenderer**, **ANGLE** (OpenGL ES on Direct3D 11), **libslirp**
- `uwp_audit.py` (Python + pefile) and the Windows SDK `WindowsApp.lib` to check UWP-allowed imports
- **Xbox Device Portal** (`https://<xbox-ip>:11443`) to deploy the package
- [xb-homebrew-vault](https://github.com/marcelofrau/xb-homebrew-vault) to send the Debian ISO to the Xbox and install the app
- MacBook with git and GitHub for editing and pushing
- **Claude Code** for code changes, review and documentation

## What you need

- An Xbox Series X|S with **Developer Mode** on and the **Device Portal** enabled
- A PC or phone on the same network
- A Linux ISO (a small *netinst* image is a good fit; Debian 13 was used by the original author)
- Optional: a USB keyboard. The Xbox pointer works as a mouse inside Linux.

## Install

1. Build the package with GitHub Actions (run the workflow from the **Actions** tab) or take it from **Releases**.
2. Open the Device Portal: `https://<your-xbox-ip>:11443`.
3. Go to **My games & apps** and use **Add / Deploy**.
4. Choose the `.msix` and add the three x64 `.appx` dependency files.

Updating keeps your virtual disk and saved files. Going back to an older version needs an uninstall first, **which deletes your virtual disk**.

## First start

1. Open **Install Other OS** and press **Settings**.
2. Pick a disk size and press **Create virtual disk**.
3. Upload your ISO with the Device Portal: **File explorer → LocalAppData → Microsoft.MicrosoftEdge.XboxVmHost_xkj04xcm6e7n4 → LocalState**. The file must sit directly in `LocalState` and its name must end in `.iso`.
4. Turn **Network** on if you want internet in Linux.
5. Press **Boot from ISO** and install your distro (a light, minimal install works best).
6. Afterwards use **Start Linux from disk**.

Change **one setting at a time**. The defaults are the recommended ones.

## Keys while Linux is running

Hold **Ctrl + Alt + Shift**, then press: **V** save state and return to the menu, **S** screenshot, **Q** stop the virtual PC, **D** show or hide the input counters.

## Limitations

- **Speed.** Software emulation only (QEMU TCG). There is no hypervisor or hardware virtualization for Xbox apps, so native-speed virtualization is not possible.
- **Heavy software does not run well.** Games and demanding 3D are out of reach. There is an experimental "GPU" screen mode (virtual graphics card drawn by the Xbox GPU through ANGLE and Direct3D 11), but do not expect more than basic OpenGL.
- **Screen size.** The app copies at most **1280 × 1024** pixels from the virtual screen.
- **Developer Mode only.** The app uses restricted capabilities (`codeGeneration`, `expandedResources`). Microsoft documents `expandedResources` as granted per title and rejected in Xbox certification, so this cannot go to the Store.
- **Resources.** Microsoft's documented limits for a normal UWP app on Xbox are about **1 GB of memory** and a **share of 2–4 CPU cores**. With `expandedResources` the app can get exclusive cores and a memory partition of at least 4 GB, **only while it is in the foreground**. How a Developer Mode sideload is treated is not documented, so test it.
- **One saved state at a time.** It is used up when you resume it and is only valid for the same app version, the same settings and an unchanged disk file.
- **Foreground only.** The virtual PC stops when the app leaves the foreground (Guide button, Home). Save state first.
- **QEMU runs once per app start.** The app restarts itself when the virtual PC ends.
- The **Pick ISO file** button in Settings is untested. Use the Device Portal upload instead.

## Possible crashes and problems

| Symptom | Likely cause | What to do |
|---|---|---|
| App closes or crashes during boot or install | Too many CPUs or too much RAM for the Xbox's memory limit | Use 1–2 CPUs and moderate RAM. Change one setting at a time. |
| App crashes soon after start with a large code cache | Code cache (64 / 128 / 256 MB) plus guest RAM exceeds the app's memory | Go back to 128 MB. |
| Very slow, or errors about generating code | The JIT is not available. The manifest comment says: if install fails with a capability error, delete the `codeGeneration` line and redeploy. Without it there is no JIT. | Keep the `codeGeneration` line if the install allows it. |
| Guest freezes or misbehaves with `max` or `Haswell` | The guest does not cope with the extra CPU features, or QEMU does not emulate one of them correctly | Switch back to `qemu64`. |
| Saved state will not resume | The app version, settings (including CPU type) or disk file changed since it was saved | Start Linux from disk. Change CPU type **before** saving state, not after. |
| No network after changing settings | The virtual network card was renamed (for example `ens3` → `ens4`) | Run `ip a` in Linux and fix the name in `/etc/network/interfaces`. |
| Virtual PC stops by itself | The app lost the foreground | Keep the app in front. Save state before leaving. |
| Blank or wrong colours in "GPU" screen mode | Experimental graphics path | Use another screen mode (Auto or a fixed size). |
| Linux disappeared after a reinstall | Uninstalling the app deletes the virtual disk | Back up before uninstalling or downgrading. |
| QEMU 11.0.2 build fails or the app will not start with it | The 11.0.2 build is untested | Use the original `build.yml` (QEMU 10.0.0). |

Under the Visual Studio debugger the memory limits do not apply, so something that works there may still crash when installed normally.

## Building

The workflow in `.github/workflows/build.yml` builds QEMU with MSYS2/MinGW as a DLL, applies the patches in `native/`, and packages the app. Main steps: get QEMU source → rename `main()` → skip the Direct3D device lookup → ANGLE patches for EGL/OpenGL → swap the JIT memory call for the Xbox-allowed one → configure with `--target-list=x86_64-softmmu --enable-tcg` → link as a DLL → audit imports against what UWP allows → package.

## Disclaimer

This is a hobby project. It is not affiliated with or endorsed by Microsoft. Use it at your own risk: a crash can lose the contents of your virtual disk. You are responsible for the Linux ISOs you install and their licenses.

## Credits and license

- Original project: [YoCanIGetUhhhh/XboxInstallOtherOS](https://github.com/YoCanIGetUhhhh/XboxInstallOtherOS) (GPL-2.0).
- [QEMU](https://www.qemu.org/) (GPL-2.0) and [libslirp](https://gitlab.freedesktop.org/slirp/libslirp) (BSD).
- This project is released under the **GNU General Public License v2**, the same license as the original (see `LICENSE`).
