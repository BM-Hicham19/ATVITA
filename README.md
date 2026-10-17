# Adventure Time (Android 1.0.8) → PS Vita

This is **not a rewrite of the game**. It is a small Dalvik virtual machine written in C (`src/vm.c`,
~800 lines) that runs the game's *own* `classes.dex` on the Vita, plus native implementations of the
~250 Android / Java calls the game makes (`src/jlib.c`, `src/droid.c`): a software canvas, bitmaps,
PNG decoding, streams, strings, input, file storage. Game code, art and levels come from **your own APK**;
nothing from the game is in this repository or in the VPK.

## Status (what has and hasn't been verified)

Verified on a Linux PC, running the real `09617-adventuretime-v1_0_8.apk` headless:
language select → title → main menu → New Game → overworld with Finn & Jake, dialogue, HUD,
walking with the virtual stick, screen transitions, save files; 5,000+ frames under AddressSanitizer/UBSan
with no errors, and the garbage collector stress-tested with a tiny heap threshold.
About 0.8 ms/frame on a desktop CPU; peak memory ~55 MB.

**Not verified:** anything on real Vita hardware. `src/main_vita.c` (display, pad, touch, threads) was only
syntax-checked against stand-in headers, so expect to fix small SDK-signature mistakes on first build.
Vita performance is an estimate (the Cortex-A9 should manage 30 fps, but I haven't seen it).

**Not implemented:** audio (no MP3/OGG/MIDI playback; `MediaPlayer` calls are accepted and ignored),
`Canvas.drawText` (the game draws its text with its own bitmap fonts, so this should rarely matter),
ads / analytics / "More Games" / "Invite" (switched off on purpose).

## Build without installing anything (GitHub Actions)

Create a new GitHub repository, upload the *contents* of this folder (not your APK), and open the
**Actions** tab: the "Build VPK" workflow compiles it inside the official `vitasdk/vitasdk` image and
attaches `AdventureTimeVita.vpk` as a downloadable artifact. (Written from memory, not run by me.)

## Build locally (needs vitasdk, see vitasdk.org for installing it)

    export VITASDK=/usr/local/vitasdk        # wherever yours is
    mkdir build && cd build && cmake .. && make
    # -> AdventureTimeVita.vpk

Install the VPK with VitaShell, then copy your APK to the Vita as

    ux0:data/AdventureTimeVita/game.apk

(Tested with v1.0.8, package `com.globalfun.adventuretime.free`.) Save data is written next to it.

## Controls

| Vita                      | Game                                   |
|---------------------------|----------------------------------------|
| D-pad / left stick        | move (drives the game's virtual stick) |
| Cross / Circle / R        | action button                          |
| Square / L                | switch character                       |
| Triangle                  | skip dialogue                          |
| Start                     | pause                                  |
| Start + Select            | quit                                   |
| Front touch screen        | works as on the phone (menus, etc.)    |

Button positions are in `src/buttons.c` if you want to move them.

## When something goes wrong

* A red screen for 5 s means the game failed to start; read `ux0:data/AdventureTimeVita/log.txt`.
* Create an empty file `ux0:data/AdventureTimeVita/debug` to also log the game's own messages.
* The log lists any Android call that isn't implemented (`unimplemented native: ...`).

## Desktop test harness

    make -f Makefile.linux
    ./atvm_test game.apk 1800 300 "130:960:256;400:960:540;700:960:316" "1300-1500:1:0:0"
    #           apk       frames dump  taps (frame:x:y, DISP coords 1920x1080)   pad (f0-f1:dx:dy:act)

Frames are written to `/tmp/atframes/*.ppm`. `AVM_TRACE=2` prints every method call, `AVM_LOG=1` the game's
log output, `AVM_THROWLOG=1` every exception, `AVM_GC_KB=2048` forces frequent garbage collection.

## Code map

    src/vm.c        dex loader, class model, GC, interpreter
    src/jlib.c      java.lang / java.util / java.io natives
    src/droid.c     android.* natives, software canvas, boot sequence
    src/res.c       zip + inflate, PNG decoder, resources.arsc parser
    src/input.c     multitouch -> Android MotionEvents      src/buttons.c  gamepad -> touch UI
    src/main_vita.c Vita frontend                           src/main_linux.c  test frontend

The game believes it runs on a 1920x1080 phone (so it picks its 640x360 asset set); the canvas renders at
960x540 and the Vita's 1920x1088 touch panel maps straight onto the game's coordinates.
