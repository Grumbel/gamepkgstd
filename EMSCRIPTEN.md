# WebAssembly (Emscripten)

## Main loop and Asyncify

**WEB-1.** The game runs one frame per browser callback:
`emscripten_set_main_loop_arg(frame, ctx, 0, true)` (0 = requestAnimationFrame).
No `while (running)` loop, no `SDL_Delay()`, no busy waiting; without Asyncify
these freeze the tab.

**WEB-2.** Build without Asyncify. Code that blocks (nested menu loops,
`sleep`, waiting for a file or network load) is rewritten as state that
advances each frame. This also makes the native main loop simpler.

**WEB-3.** Asyncify is allowed only when the blocking code cannot reasonably be
restructured (typically a legacy port). Then:

- `wasm.nix` takes `enableAsyncify ? false`; the game's `<game>-wasm` output
  sets it, with a comment saying which code blocks.
- Restrict instrumentation with `-sASYNCIFY_ONLY=[...]` or
  `-sASYNCIFY_REMOVE=[...]`; whole-program Asyncify grows the binary
  substantially and slows it down.
- Do not use JSPI until all target browsers support it.

## Build

**WEB-4.** Link options:

```
-sALLOW_MEMORY_GROWTH=1
-sGROWABLE_ARRAYBUFFERS=0     # Emscripten 6: TextDecoder rejects resizable buffers → black page
-sFORCE_FILESYSTEM=1
-lidbfs.js
-sENVIRONMENT=web
-sEXPORTED_RUNTIME_METHODS=FS,addRunDependency,removeRunDependency
--shell-file mk/wasm/shell.html
```

SDL2 comes from the flake (`sdl2-wasm`, FLAKE-2), not from an Emscripten port
downloaded at build time (STACK-8).

**WEB-5.** The `<game>-wasm` output contains `index.html`, `<game>.js`,
`<game>.wasm` and `<game>.data` (if any), using only relative URLs so it works
under any path prefix. `nix run .#<game>-wasm` serves it (FLAKE-5).

## Storage

**WEB-6.** The game uses its normal Linux paths (DIR-5); `HOME` is
`/home/web_user`. The shell page mounts IDBFS on
`/home/web_user/.config` and `/home/web_user/.local/state` in `preRun`, and
loads them with `FS.syncfs(true, …)` guarded by
`addRunDependency`/`removeRunDependency`, so `main()` starts with the stored
files present.

**WEB-7.** After every config or savegame write, the game calls
`FS.syncfs(false, cb)`. Errors are logged to the console and shown to the
player.

**WEB-8.** The page offers "Export saves" (download the state directory as
`<game>-saves.json`) and "Import saves", because the browser may delete
IndexedDB data.

## Page

**WEB-9.** `mk/wasm/shell.html` provides: the canvas, a loading progress
indicator, a readable error message if loading fails, a short controls list,
and a link to the source and revision.

**WEB-10.** The game exports `<game>_pause()` (`-sEXPORTED_FUNCTIONS=_main,_<game>_pause`);
the page calls it on `visibilitychange` when the tab is hidden (KEY-7).

**WEB-11.** While the canvas has focus, arrow keys and Space do not scroll the
page. F11 uses SDL's fullscreen (browser fullscreen API).
