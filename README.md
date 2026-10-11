# MotionForge 0.2

A native iOS / iPadOS motion-graphics editor (SwiftUI + Metal). Source project, not a compiled app:
an `.ipa` can only be built on macOS with Xcode, so this folder includes everything needed to produce one.

## Get the .ipa

Pick one. All three produce the same app.

### A. GitHub Actions (no Mac needed)
1. Create a new GitHub repository and push the contents of this folder to its `main` branch.
2. Open the repo's **Actions** tab, choose **Build unsigned IPA**, and press **Run workflow** (it also runs on every push).
3. When it finishes, download the **MotionForge-unsigned-ipa** artifact. The **build-log** artifact has the full compiler output.

The result is an *unsigned* `.ipa`. iOS refuses to install unsigned apps, so sign it with your own Apple ID or developer
certificate before installing: Sideloadly, AltStore, or any re-signing tool you trust. Free Apple IDs work; the app
expires after 7 days and has to be re-signed.

### B. Your own Mac
```
bash scripts/build_ipa.sh                                   # unsigned -> build/MotionForge-unsigned.ipa
TEAM_ID=ABCDE12345 bash scripts/build_ipa.sh                # signed for your registered devices
TEAM_ID=ABCDE12345 EXPORT_METHOD=ad-hoc bash scripts/build_ipa.sh
TEAM_ID=ABCDE12345 EXPORT_METHOD=app-store-connect bash scripts/build_ipa.sh   # then upload with Transporter / TestFlight
```
Needs Xcode 15 or newer. The script installs XcodeGen through Homebrew if it's missing.

### C. Xcode
`brew install xcodegen && xcodegen generate && open MotionForge.xcodeproj`, choose your team under
Signing & Capabilities, connect a device, press Run (or Product > Archive).

Use a physical device. The Simulator can run the app, but Metal video decoding behaves differently there.

## Payload and sideloading
An `.ipa` is a zip with one top-level folder named `Payload` that holds the compiled `MotionForge.app`. That folder
only exists after the app has been compiled, which needs a Mac, so it can't be shipped inside this source zip.
The workflow produces all of these for you:
- `MotionForge-unsigned.ipa`: give this to Sideloadly, AltStore, SideStore or any re-signing tool.
- `MotionForge-unsigned.tipa`: the identical file renamed for TrollStore users.
- `Payload/`: the raw `.app` folder, for tools that ask for it unzipped.

Already have a built `MotionForge.app` (from your own Mac or another CI)? Run
`bash scripts/make_ipa_from_app.sh path/to/MotionForge.app` to wrap it into a correct `.ipa`. It works on Linux too.

Sideloading notes: a free Apple ID can sideload up to 3 apps and the signature lasts 7 days (AltStore and SideStore
can refresh it automatically); a paid developer account lasts a year. The app needs no special entitlements.

## What works in this build
- Import video from Photos. The first clip sets the composition size (long edge capped at 1920).
- Layers: video, text, shapes. Reorder, duplicate, delete, hide, trim start and end, 4 blend modes (Normal, Add, Screen, Multiply).
- Transform (X, Y, Scale, Rotation, Opacity) with keyframes, auto-keying, and per-keyframe cubic-Bézier easing.
- Expressions per property: `time`, `index`, `value`, `wiggle()`, `linear()`, `ease()`.
- Composition effects on the GPU: displacement map, turbulent displace, radial chromatic aberration. Compare before/after with a draggable divider.
- Undo / redo for layer, effect and length edits.
- Spring-animated drawer and timeline: pinch-zoom, scrubbing, keyframe diamonds.
- Export to H.264 MP4 with the first clip's audio passed through, then share or save from the share sheet.

## Not in this build
- Neural restoration (face restore, frame interpolation, upscale): the Core ML bridge is in `NeuralStage.swift`, but no models are bundled and nothing calls it yet.
- 3D cameras and lights, track mattes, particles, shatter, puppet pins, project save/load, more than four blend modes.
- Audio from anything other than the first clip, and audio editing.

## Honest status
This code was written without a compiler or a device. The structure and brackets are checked, but the first CI build
may report compile errors that need small fixes. If it fails, open the **build-log** artifact and send me the first few
`error:` lines.
