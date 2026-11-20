# Runtime profile for `PGO=use`

`runtime.profdata` is the profile that `make PGO=use` reads (change it with `PGO_PROFILE=<file>`). It only covers the
runtime (`runtime/`), never the generated game library, and holds execution counters keyed by function names
(`file.c:function`): no game data, no paths.

This fork's profile (RX580.7) was made with clang 23.1.5 (MSYS2 UCRT64) from the RX580.7 sources. One training run of
the instrumented build (`make PGO=gen`): `tools/perf_routes.py run` over the kakariko, field, castle and hateno routes
(and part of korok) at 1920x1080 with SMAA, effects at full resolution and shadows at console resolution, on an RX 580
8 GB with AMD's own driver and an i3-10105F: the AMD and 4-core paths these builds are played with. The profile it
replaced (clang 23.1.4, upstream `test` at 04aa642, TAA and high shadows) had gone stale for most of the render thread's
functions, edited since. RX580.6 (the old profile) against RX580.7 (this one, with that release's other changes) on
the Great Plateau, which is not in the training: render thread 9.16 -> 8.32 ms a frame, 106.1 -> 115.0 fps (two
interleaved passes). The file is about 860 KB; its size follows the number of counters in the code, not the play time.

The upstream profile, as measured upstream:

Measured on a 5700X3D + 3080 Ti (3 rotated rounds, TAA, high shadows, routes NOT in the training), against the plain
build: with `make dist`'s `PGO=use LTO=thin` plateau 1080p +33%, castle 1080p +23%, plateau 1440p +27%, castle 1440p +19%;
PGO alone +28 / +18 / +21 / +17%. The previous profile (AA off, older code) gave +1..+10% on the same routes and a merge of
the two was worse than the new one alone: regenerate instead of merging.

## Limits

- A profile only fits the compiler that wrote it. With another clang, `PGO=use` can fail with an "unsupported
  instrumentation profile format version" error: make a new one (below).
- It goes stale as the runtime changes. A function that was edited since loses its data and is built as usual; everything else keeps working. Regenerating every so often keeps the gain.

## Making a new profile

```
make PGO=gen B=build-gen RELEASE=1 build-gen/savagewind.exe
# run build-gen/savagewind.exe with LLVM_PROFILE_FILE=<dir>/rt-%p.profraw and play for a while (TAA on: the profile
# should match the settings the game is played with), or script it:
#   LLVM_PROFILE_FILE=<dir>/rt-%p.profraw python tools/perf_routes.py run --exe build-gen/savagewind.exe --routes kakariko field hateno korok rito ...
# The profile is written every 3000 frames and at a scripted SAVAGEWIND_EXIT_AT, so closing the window is fine.
llvm-profdata merge -o tools/pgo/runtime.profdata <dir>/*.profraw      # add the old file to the list to keep its data
make PGO=use B=build-pgo RELEASE=1 build-pgo/savagewind.exe
```
On MSYS2, `llvm-profdata` comes with `mingw-w64-ucrt-x86_64-llvm-tools` and the profile runtime with
`mingw-w64-ucrt-x86_64-compiler-rt`. clang 23.1.5 links `lib/clang/23/lib/x86_64-w64-windows-gnu/libclang_rt.profile.a`,
which that package installs as `lib/clang/23/lib/windows/libclang_rt.profile-x86_64.a`: copy it to the first name.
