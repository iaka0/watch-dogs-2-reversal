# Watch Dogs 2: reverse-engineering report

Target: `bin\Disrupt_64.dll` plus the `data_win64` archives. Final PC patch, **wd2-game-rel-pc-189** (binaries changelist 1088394).
All addresses are **RVAs** in `Disrupt_64.dll`. At runtime: `GetModuleHandleA("Disrupt_64.dll") + RVA`.
The DRM wrapper, the crack and the online/anti-cheat code were deliberately not analysed.

---

## 1. Binaries

| File | Role |
|---|---|
| `WatchDogs2.exe` (540 KB) | Thin launcher. It loads `Disrupt_64.dll` and starts it. No game code. |
| `Disrupt_64.dll` (139 MB) | The whole engine and game, built 2017-07-25. |

**Section layout.** The section names are scrambled by the protection wrapper, so go by the flags, not the names:

| Section | RVA | Size | Real role |
|---|---|---|---|
| `.reloc` | 0x1000 | 0x34F9000 | uninitialised data (BSS) |
| `.edata a` | 0x34FA000 | 0x8C5000 | **read-only data**: strings, `luaL_Reg` tables, vtables |
| `.idata` | 0x3DBF000 | 0x8961E0 | imports, RTTI, more data |
| `.pdata a` | 0x4656000 | 0x32BC00 | exception table: **277,071 function boundaries** (`functions_pdata.txt`) |
| `.tls em` | 0x4989000 | 0x76BB000 | **all game code** |
| `.code`, `.arch` | 0xC044000+ | small | protection stub (not touched) |

**Exports.** The only exports are the Wwise audio API (`AK::SoundEngine::*`, 247 of them).

**Imports worth knowing for injection.** The DLL imports `DINPUT8.dll!DirectInput8Create` and `VERSION.dll`, so a proxy `dinput8.dll` or `version.dll` placed in `bin\` is the natural way to load your own code. It renders with D3D11 (`D3D11CreateDevice` / `CreateDXGIFactory1`), so ImGui overlays can hook `IDXGISwapChain::Present`.

**Middleware linked in:** Havok physics (`hk*`, `hknp*`), Crypto++, Wwise, Bink 2, NVIDIA HBAO+/TXAA/ShadowLib, AMD AGS, Ubisoft "Storm" networking, and an embedded **Lua 5.1**.

## 2. Engine module map

The game code was compiled without C++ RTTI. The 1,683 RTTI types that do exist all belong to Havok, Crypto++ and a Bloomberg library (`rtti_classes.txt`). The engine layout comes from the 1,915 leftover source paths instead (`strings_source_paths.txt`, rooted at `K:\Code\...`). The engine descends from **Driver: San Francisco**, hence the `Driver*` names:

- `Engine/` (entity, components, events, network, script, UI) and `EngineServices/`
- `Game/` (AI, camera, dialog, dynamic environment, persistence, vehicles, triggers)
- `DriverGame/`: **Hacking** (`Gameplays/Hacking`), LivingCity, Civilians, Emergency (police), ElectricSystem, Spawning, MissionSystem, RemoteControl (drone, jumper), Robot, Session/Coop, ProfilerSystem
- `DriverGame2/`: PhotoCamera, Cinema, ContactList, Leaderboards, script
- `DriverGameMissions/`: missions built from **Bricks** (`Brick/GameplayBricks/*`), CityActivities (racing), DynamicObjectives, FAA, PGT (procedural gameplay tool), Benchmark
- `DriverAI/BehaviorTree`, `AI/Pilot`, `OnlineSession/` (not analysed), `GraphicsRenderer`, `platforms/d3d11`

## 3. Native Lua API (the main find)

- **Lua 5.1**, with 64-bit `size_t` and **`lua_Number` = 32-bit float** (a nonstandard Ubisoft build).
- **1,138 functions** were recovered from 21 `luaL_Reg` tables. Each has its name and exact native address. See **`LUA_API.md`** (categorised) and `lua_api.txt` (raw).
- The game tables (`0x378CE60`, `0x378DF00`, `0x378E640`, `0x378EFD0`, `0x378F060`) are registered together at around `0x5FFA150`, all as **globals**.
- The stock libraries are linked in: `base`, `string`, `table`, `math`, `os`, `io`, `debug`, `package`, `coroutine`.

Highlights:

| Area | Examples |
|---|---|
| Player | `TeleportEntity`, `ActivateInvincibility`, `RemoveInvincibility`, `SetLocalPlayerClothing`, `ChangePlayerGkModelFromHumanConfig`, `AddItem`, `EquipItem`, `GetLocalPlayerCash`, `AddPerkWithDbObjToPlayer` |
| Spawning | `SpawnNPC`, `SpawnAIVehicleAndPassengerOnRoad`, `SpawnFieldAI`, `AddProp`, `RemoveEntity`, `UnspawnAllPedestriansInRadius`, `SetDensityContextOverride` |
| Police | `FelonySystemEnable`, `FelonySetMinHeat`, `FelonyStartChase`, `FelonyOverrideHeatParams`, `FelonyOverrideSWATParams`, `FelonyOverrideChopperParams`, `ForceArrest` |
| Hacking | `ForceHackOn`, `ForceHackOff`, `SetHackable`, `StartRandomHackingFromPlayer`, `OverrideHackingCategorie`, `EnableFireWallBreachinOnHackingSelectorEntity` |
| Vehicles and drones | `GetVehicleSpeed`, `ShutdownVehicleEngine`, `VehicleLightAndSiren`, `UnlockAndBuyCarOnDemand`, `ConnectToFlyingDrone`, `SetToyCarAvailable` |
| World | `EnvironmentManagerSetWeather`, `SetSandstormIntensity`, `SetSlomoFactor`, `LoadMissionLayer`, `SetHubLocked` |
| AI | `SendAICommand_*`, `CreateSquad`, `StartSquad`, `ReinforcementAddScriptedAgent` |
| Debug | `ExecuteConsoleCommand`, `DrawTextToScreen(text, duration, color)`, `LogConsole`, `EnableDebugAnnotationEntity`, `SpawnDebugHelper_SpawnVehicleInFrontOfPlayer` (string) |

**Second binding layer.** Lua also sees C++ singletons with methods, such as `CDominoManager_GetInstance()`, `CScriptCallbackSystem_GetInstance()` and `CDominoConsoleCommandManager_GetInstance():RegisterConsoleCommand(name, obj, "Method")`. These aren't in the `luaL_Reg` tables. They're the next thing to map.

## 4. Developer command-line switches (from the binary)

`-nointrovideo -nosplash -nofullscreen -xpos -ypos -resolution -world -spawnpos -exec -runscriptindebug -nosave -logFile -nologfile -nosound -norumble -nomouse -nopad -txtlang -localizationdisplayid -demo -marketing -fakeTrialMode -splitscreen -forcedrive -enablevedit -vehiclecolordebug -profileLoading -gpucapture -enablememtracer -unittest -automation`

The random-looking `-QTUQUe`-style strings are protection junk. Which switches survive in the retail build isn't verified yet.

## 5. Archive format (fully solved)

**`.fat` header** (28 bytes): magic `5TAF`, `u32 version = 11`, `u32 flags`, 8×`0xFF`, `u32 0`, `u32 entryCount`. 20-byte entries follow, sorted by hash, then a 4-byte tail.

| Field | Bits |
|---|---|
| `u64 hash` | **FNV-1 64** of the lowercase backslash path, `& 0x1FFF…F | 0xA000…0` |
| `u32 a` | `[0:30]` compressed size, `[30:32]` low 2 bits of the offset |
| `u32 b` | offset >> 2 (34-bit offsets) |
| `u32 c` | `[0:2]` scheme (0 = stored, 2 = LZ4 variant), `[2:32]` uncompressed size (0 when stored) |

**Compression.** Reversed from the decompressor at `Disrupt_64.dll+0x4B201A0`:

- A LEB128 varint prefix gives the **phase-switch point** for the in-place decoder.
- Standard LZ4 tokens and 255-run length extensions.
- **Extended offsets:** if the u16 offset is `>= 0xE000`, one extra byte follows and `offset += byte << 13` (about a 2 MB window).
- **In-place termination:** once the output passes the phase point and catches up with the input, the rest of the output is the raw remaining input.
- Validated with 0 errors on every entry of `common.dat` and the first 8,000 entries of `patch.dat`.

**Contents.** 594,407 entries across all archives. File types seen include `TBX` textures (with DDS inside), `nbCF` binary objects (FCB), `BKHD` Wwise banks, `MOEG` geometry, XML and RML, and Lua.

## 6. Game scripts

- **1,732 compiled Lua 5.1 chunks** were extracted (`extracted/lua/`), plus the plain-text `engine\scripts\missiontools.lua`.
- Every archive hash matches the source path embedded in its chunk, so every script has its real name.
- They were generated by **Domino**, Ubisoft's visual scripting system:
  - `domino\system\*` (434): reusable blocks such as `FelonySystemController`, `TeleportEntity`, `TimeScaleController`, `ConsoleCommand_v2`, `UIController`, `HackableController`
  - `domino\system\lualibraries\*`: the runtime (`LuaLibCore`, `LuaLibEvents`, `LuaLibFire`, `LuaLibAI`)
  - `domino\library\*` (226): taxi, racing, prepare and co-op player-count modules
  - `domino\user\san_francisco\missions\*` (about 950): every story mission and world activity
- Full annotated disassembly is in `lua_disassembly/` (per script, mirroring the game paths), with `_call_frequency.txt` showing which functions the game actually uses.
- The engine accepts **plain-text Lua** as well as bytecode, so mods can ship source.

## 7. Tools (all in `tools/`)

| Tool | Use |
|---|---|
| `wd2lz.py` | Decompressor (pure Python) |
| `wd2fat.py` | `index` (CSV of every entry), `scanlua` (parallel Lua extraction), `extract <fat> <out> [--magic] [--name]` |
| `build_names.py` | Resolves archive hashes to filenames (`filenames.txt`); give it more string sources to grow the list |
| `lua51dis.py` | Disassembler for the float32 Lua 5.1 build, with constant and call annotations |
| `wd2pack.py` | **Reversible** mod installer: appends files to `patch2.dat`, rewrites the `.fat`, backs up first; `uninstall` restores byte-for-byte (tested on a scratch archive) |
| `make_hook.py` | Wraps an original compiled script inside a text Lua file plus your code, so you can extend game scripts without decompiling |
| `api_doc.py` | Regenerates `LUA_API.md` |

An example mod (built, **not installed**) is `mods/hello_hook/`. It hooks `LuaLibCore`, draws on-screen text, and registers a `wd2mod` console command with `slomo`, `normal`, `rain`, `noheat` and `heat` options.

```
python tools\wd2pack.py install mods\hello_hook     # back up + install
python tools\wd2pack.py uninstall                    # restore originals
```

## 8. Open questions and next steps

1. **Archive priority (untested).** `lualibcore.lua` lives in `worlds\san_francisco\san_francisco.fat`. It isn't confirmed that an entry in `patch2` overrides a world-archive entry. If the hello mod shows nothing, try installing into the world archive (`--archive ..\worlds\san_francisco\san_francisco`). That is reversible the same way.
2. **Map the C++ singleton bindings** (`*_GetInstance` objects and their method tables). They are probably exposed through a reflection binder near the `luaL_Reg` registration code.
3. **Grow `filenames.txt`** by harvesting paths from the extracted XML, FCB and Lua data. Only about 2,000 of 594,000 names are resolved so far.
4. **FCB (`nbCF`) object format.** Decoding it unlocks gameplay tuning such as heat, vehicles and weapons.
5. **Native scripting layer.** A proxy `dinput8.dll` that finds `lua_State` (by hooking the registration function at `0x5FFA150`, or `luaL_loadbuffer`) would allow loading arbitrary Lua from disk at runtime, giving a ScriptHook-style framework.
