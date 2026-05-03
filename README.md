# kompy-cs2

Internal-style overlay and gameplay hooks for **Counter-Strike 2** on Windows. The cheat runs as a **64-bit DLL**, draws with **Dear ImGui** on **Direct3D 11**, and taps game memory through `client.dll` offsets and a signature-based hook on input history.

This repo is aimed at people reverse-engineering or learning how these pieces fit together. **Using it on VAC-secured servers violates Valve’s rules and risks a ban.** The authors are not responsible for what you do with the code.

---

## Why OBS is involved

kompy does **not** talk to OBS over a plugin API or WebSocket. It piggybacks on something OBS already injects when you capture the game with **Game Capture**.

OBS ships a module called **`graphics-hook64.dll`**. When Game Capture hooks a title, that DLL ends up loaded in the game process and sits in the graphics path so OBS can read frames. Inside that DLL, OBS keeps pointers to the “real” DXGI **`Present`** and **`ResizeBuffers`** implementations it wrapped.

kompy **`GetModuleHandleA("graphics-hook64.dll")`**, then reads function pointers at **fixed offsets** from the module base (`real_present_offset` / `real_resize_offset` in `workspace/render/present.hxx`). It swaps those pointers for its own `hk_present` / `hk_resize`, which:

1. Lets the game and OBS keep doing their job after kompy chains back to the original functions.
2. Gives kompy a stable per-frame callback where it can run entity logic, aim, and ImGui without setting up a separate dummy swap chain.

**Important:** Those offsets are **not** universal. They belong to a particular build of OBS’s hook DLL. After an OBS update, the layout inside `graphics-hook64.dll` can move; then Present is wrong, initialization fails, or you crash. Treat the constants in `present.hxx` like any other fragile offset—re-dump or re-analyze the DLL when you update OBS.

Console messages spell out the dependency, e.g. missing module → *“is OBS game capture active?”* and a null original Present → *“OBS hook not active yet.”*

---

## What else is hooked

Besides the DXGI path through OBS’s hook DLL, kompy installs a **trampoline** on a function in **`client.dll`** located by **byte pattern** (`populate_history_entry` in `workspace/engine/hooks.hxx`). That path feeds silent-aim / history-related behavior configured from the menu.

Everything else leans on **static offsets** and **netvars** in `workspace/engine/sdk/` (e.g. `offsets.hxx`, `netvars.hxx`). Game patches routinely change those; the project will silently break or crash until someone updates the numbers and re-validates patterns.

---

## Features (from the in-game UI)

Rough map of what the ImGui menu exposes:

| Tab | Notes |
|-----|--------|
| **Visuals** | ESP (boxes, skeleton, head marker, snap lines), health/armor bars, name/weapon, team filters, map radar, dormant players |
| **Aim** | Aimbot, silent aim, RCS, FOV, smoothing, bone selection, visibility check |
| **Misc** | No flash/smoke tweaks, grenade prediction, no recoil, rapid fire, hit sound, backtrack, spectator list, anti-aim |

Toggle the menu with **F8**. Unload with **END** (main thread waits on that key, then tears down hooks and ImGui).

---

## Requirements

- **Windows x64** (targets desktop Win32 APIs and DXGI/D3D11).
- **Visual Studio 2022** (project uses **Platform Toolset v143**, **C++20**).
- **Counter-Strike 2** 64-bit process.
- **OBS Studio** configured to use **Game Capture** on CS2 so `graphics-hook64.dll` is loaded *before* kompy tries to patch Present. Capture does not need to be streaming to Twitch—local preview is enough as long as the hook is active.
- A **DLL injector** you trust (not included). How you inject is your problem; `DllMain` only spawns the init thread.

The `.vcxproj` currently hard-codes extra include paths on one configuration (`Release|x64`). If your machine paths differ, adjust **Project → Properties → VC++ Directories** or edit the XML so includes resolve (`dependencies/ImGui`, etc.).

---

## Build

1. Open `kompy-cs2.vcxproj` in Visual Studio (or generate a solution that references it).
2. Select **Release** and **x64** (this is the sensible target for CS2).
3. Build. Output directory is set to **`../builds`** for Release|x64—your DLL lands there unless you change `OutDir`.

Dependencies bundled in-tree include **Dear ImGui** and Win32/DX headers from the Windows SDK. No separate CMake step.

---

## Runtime flow (typical)

1. Start **OBS**, add a **Game Capture** source targeting CS2 (or launch CS2 after the scene is ready so the hook attaches).
2. Launch **CS2** and get in a state where capture is actually hooking (if `RealPresent` is still null, kompy exits early—usually timing or OBS not capturing that window).
3. **Inject** the built DLL into the `cs2.exe` process.
4. Watch the **console** window kompy allocates for status (`[kompy] …` lines).
5. Use **F8** for the menu, **END** to unload.

If initialization fails immediately, check: Game Capture on, 64-bit OBS hook present, matching OBS build vs. hard-coded hook offsets, and that game/offset data matches the live CS2 build.

---

## Project layout (short)

- `dllmain.cpp` — entry, console, init/shutdown, input hook install/remove.
- `workspace/render/present.hxx` — OBS hook discovery, DXGI hooks, ImGui + Win32 WndProc forwarding.
- `workspace/engine/` — game context, classes, pattern scanner, offsets, netvars, input-history hook.
- `workspace/entities/visuals/` — frame update, ESP/aim/misc logic tied to the menu config.

---

## License

No license file is shipped in this snippet of the repo; add one if you publish publicly. Until then, assume **all rights reserved** unless the repository owner states otherwise.
