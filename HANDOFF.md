# HANDOFF — "Rooftop Peekaboo" (2P hide & seek)

Last updated: 2026-10-27, session 2. Read this first, then check `Content/Rooftop/` in the editor.

## The Game (agreed design)
Two skyscraper rooftops across a gap. Local 2-player hotseat on one PC, turn-based camera (no split-screen in v1):
- **Hider** (Player 1): during HIDE phase (~5s), moves with WASD to get behind one of **3 boxes** on their rooftop, then freezes.
- **Shooter** (Player 2): during SHOOT phase, sees a fixed first-person camera across the gap; mouse cursor + left-click picks which box to shoot.
- Right box → CAUGHT (shooter scores). Wrong box → ESCAPED (hider scores). First to **5** wins. Roles do NOT swap in v1 (crates exist on both towers for future swaps).

## What's DONE

### Session 1
1. **UnrealMCP wired end-to-end**: `firstgame.uproject` has plugins `ModelContextProtocol`, `AllToolsets`, `PythonScriptPlugin` enabled; `Config/DefaultEditorPerProjectUserSettings.ini` auto-starts the MCP server on editor launch (port 8000, path `/mcp`). Bionic MCP "Unreal" → streamableHttp `http://127.0.0.1:8000/mcp`. Verify with `netstat -ano | grep :8000` + curl initialize probe; if stale ECONNREFUSED, Reconnect in Bionic Settings.
2. **Map** `Content/Rooftop/Lvl_Rooftop` (duplicated from Lvl_ThirdPerson). Layout in cm:
   - `Tower_A_HiderSide`: SM_Cube scale (20,16,40) at (-2000, 0, 2000), top z=4000, spans x -3000..-1000.
   - `Tower_B_ShooterSide`: same at (+2000, 0, 2000). Gap = 20m (x -1000..+1000).
   - Crates on Tower A (3m SM_Cube, center z=4150): `Crate_A_0` (-1500,-700), `Crate_A_1` (-1500,0), `Crate_A_2` (-1500,+700). Hider hides on -x side (≈ x=-1950).
   - Crates on Tower B: `Crate_B_0/1/2` mirrored at (+1500, y) — future role swaps.
   - `PS_Hider`: PlayerStart (-2600, 0, 4050), yaw 0 → template GameMode spawns Player 0 (BP_ThirdPersonCharacter = HIDER pawn) here.
   - `ShooterCam`: CameraActor (1300, 0, 4250), yaw 180 (looks across gap at Tower A).

### Session 2 (this session)
1. **MCP verified live**: editor running with all toolsets; `Lvl_Rooftop` was loaded in the editor.
2. **Box art SVGs restored** — session-1 handoff claimed they were saved to `Content/Rooftop/Art/`, but that folder did not exist on disk (searched whole project, zero .svg files). User re-sent the originals this session; now in place:
   - `Content/Rooftop/Art/BoxL.svg` — wooden crate, yellow "Box L" label front + stencil top.
   - `Content/Rooftop/Art/BoxM.svg` — wooden crate, yellow oval sticker "Box M" top-left.
   - `Content/Rooftop/Art/BoxR.svg` — wooden crate, torn bottom-left corner (cardboard backing), "Box R" label.
   - They're detailed vector art (18–182KB each) — test the SVG→PNG conversion on one before doing all three.
3. **BP_RooftopGame created IN MEMORY ONLY — NOT SAVED TO DISK**: duplicated `/Game/LevelPrototyping/Interactable/Door/BP_DoorFrame` → `/Game/Rooftop/BP_RooftopGame` via `AssetTools.duplicate`. It still carries all the door leftovers (components + variables, listed in step 1a below). **If the editor was restarted before it got saved, re-duplicate and redo steps 1a–1c.**
4. **Node discovery done** — verified type_ids for this project (graph refPath format below):
   - `Camera|DeprojectScreentoWorld` — in: `Player` (PC ref), `ScreenPosition` (V2D byref); out: `WorldPosition`, `WorldDirection`, `ReturnValue`(bool). ⚠️ multi-output node — DSL handling untested.
   - `Collision|LineTraceByChannel` — in: Start, End, TraceChannel (ETraceTypeQuery enum; default literal was "TraceTypeQuery1" — verify the ECC_Visibility literal), bTraceComplex, ActorsToIgnore, DrawDebugType, bIgnoreSelf, TraceColor, TraceHitColor, DrawTime; out: **`then` (exec)**, `OutHit` (hit result struct), `ReturnValue` (bool) → use multi-exec continuation `(:then ...)` with `_out_hit`, `_return_value`.
   - `Utilities|FlowControl|Delay` — classic delay node.
   - `Game|Player|SetViewTargetwithBlend` — pins unverified.
   - `Game|GetPlayerController` — pins unverified (expect WorldContextObject + PlayerIndex).
   - `Class|PlayerController|SetShowMouseCursor` (bool) — use for cursor show/hide (no separate Show/Hide nodes found).
   - `Game|Player|WasInputKeyJustPressed` — exists; pins unverified. **Plan: poll it in EventTick** for LeftMouseButton (shoot) and R (restart after GameOver) — avoids Input Key event nodes entirely.
5. **Design correction vs session-1 plan**: do NOT call `DisableInput(PlayerController)` to freeze the hider. PC0 is also the shooter's controller; disabling its input likely stops ALL key state updates on that PC, which would break click detection via `WasInputKeyJustPressed` on the same PC. Instead **freeze the hider by movement mode**: at hide-end set CharacterMovementComponent → `SetMovementMode("MOVE_None")` + `StopMovementImmediately()`; restore `"MOVE_Walk"` in StartRound. (Discover exact node type_ids: search "movementmode", "stopmovement".) Verify in PIE that WASD truly can't move the pawn during SHOOT phase.

## What's NOT done yet (next session, in order)

### 1a. Clean up BP_RooftopGame
- `BlueprintTools.get_parent` first; if not already Actor, `set_parent` → `/Script/Engine.Actor`.
- Remove leftover components via `ActorTools.remove_component`: `box` (BoxComponent), `door Control` (TimelineComponent), `door`, `door2` (StaticMeshComponents). **Keep `defaultSceneRoot`** (root SceneComponent — actor needs a transform to be placed in level).
- Remove leftover variables via `BlueprintTools.remove_variable` (exact names, note the spaces): `"door Size"`, `"closed Position"`, `"door Frame"`, `"door Frame Scale"`, `"door Detection Adjust"`, `"door Thickness"`, `"split Door"`, `"door Frame Overide Material"`, `"add Door"`, `"closed Position_0"`.
- Do cleanup BEFORE writing the new graph so the final compile is clean.

### 1b. Add variables (BlueprintTools)
- `add_variable`: RoundState int, ShooterScore int, HiderScore int, WinScore int, HideTime float, ResultTime float, HidingCrateIndex int.
- `add_object_variable` with `container_type: "ARRAY"`: CratesA of `/Script/Engine.Actor`.
- `add_object_variable`: ShooterCam (`/Script/Engine.CameraActor`), HiderPawn (`/Script/Engine.Character`), HiderSpawn (`/Script/Engine.PlayerStart`) ← new vs session-1 plan, explicit spawn ref for robust reset.
- `set_variable_instance_editable true` on: CratesA, ShooterCam, HiderSpawn (HiderPawn is runtime-only; scores/state are internal).
- Set CDO defaults WinScore=5, HideTime=5.0, ResultTime=2.5, HidingCrateIndex=-1 via `ObjectTools.set_properties`. ⚠️ untested which instance refPath targets the CDO — try asset path `/Game/Rooftop/BP_RooftopGame.BP_RooftopGame`, then generated class `_C`. Fallback: set them in BeginPlay graph code.

### 1c. Build EventGraph with `BlueprintTools.write_graph_dsl`
- **Graph refPath**: `/Game/Rooftop/BP_RooftopGame.BP_RooftopGame:EventGraph` (format `AssetName:ObjectName`; `:ScriptBlueprint.EventGraph` is REJECTED).
- Call `get_graph_dsl_docs()` first. Key rules: quote class paths & enum literals; `(bind var ...)` before reusing any node output; `self` auto-bound in event/fn bodies; multi-exec continuations `(:PinName stmts...)` expose data outputs as `_underscored_pinname`.
- ⚠️ The graph still contains the old door logic. **Verify whether write_graph_dsl clears existing nodes first** (write a tiny test script, then `read_graph_dsl`). If it appends, delete leftover nodes first (`find_nodes` title="" + `delete_node`, or just re-duplicate the BP and redo 1a/1b).
- ⚠️ **Test risky DSL constructs with tiny scripts BEFORE writing the full graph** (each write replaces+compiles):
  1. `(fn StartRound ())` definition + calling it from an event body — call syntax unverified! Try bare `(StartRound)`; if that fails, after the fn exists use `find_node_types "startround"` to get its callable type_id.
  2. Multi-output node via bind (DeprojectScreentoWorld) — see fallback below.
  3. Array element access CratesA[i] — discover: `find_node_types "arrays"`.
- **Full flow design**:
  - `EventBeginPlay` → Delay 0.5s (pawn spawn grace) → StartRound()
  - `fn StartRound`: pc = GetPlayerController(self, 0); pawn = pc.GetPawn; CastToCharacter → HiderPawn (multi-exec `:then`); teleport HiderPawn to HiderSpawn loc+rot (`Transformation|SetActorLocation/SetActorRotation` — verify type_ids); restore movement mode Walk + zero velocity; SetViewTargetwithBlend(pawn) [back to hider view]; SetShowMouseCursor false; RoundState=0; PrintString "HIDE PHASE"; Delay(HideTime) → OnHideEnd inline
  - OnHideEnd (continuation): hiderLoc = HiderPawn loc; d0/d1/d2 = `Math|Vector|Distance`(hiderLoc, CratesA[i] loc); HidingCrateIndex = argmin via nested `(select ...)`; freeze: SetMovementMode MOVE_None + StopMovementImmediately on HiderPawn's CharacterMovementComponent (get comp node — search "charactermovement"); SetViewTargetwithBlend(ShooterCam, ~0.3s); SetShowMouseCursor true; RoundState=1; PrintString "SHOOT PHASE"
  - `EventTick`: if RoundState==1 and WasInputKeyJustPressed(LeftMouseButton): mousePos (get PlayerCameraManager from PC first — discover getter node, then `GetMousePosition` → V2D); deproject → origin+dir; end = origin + dir*6000; LineTraceByChannel(origin, end, visibility channel, bIgnoreSelf=true) `(:then`: if `_return_value`: hitActor = extract actor from `_out_hit` (discover "hitresult" break/getter node); find idx in CratesA by == compare over 0..2; if found and idx==HidingCrateIndex → ShooterScore+1, PrintString "CAUGHT!" else HiderScore+1, PrintString "ESCAPED!"; RoundState=2; Delay(ResultTime) → OnResultEnd inline). Also in Tick: if RoundState==3 and WasInputKeyJustPressed(R): scores→0, StartRound()
  - OnResultEnd (continuation): if ShooterScore>=WinScore or HiderScore>=WinScore → RoundState=3, PrintString "SHOOTER WINS"/"HIDER WINS — press R to restart"; else StartRound()
- **Deproject fallback** (if multi-output bind fails): manual ray math with single-output nodes only: camLoc/camRot from ShooterCam actor transform; aspect = `Class|PlayerCameraManager|GetAspectRatio`; fov = `GetFOVAngle`; dir = normalize(fwd + (2mx−1)·tan(fov/2)·aspect·right + (1−2my)·tan(fov/2)·up); mx,my = mousePos / viewport size (discover "viewportsize").
- **Hit-actor fallback** (if can't extract actor from `_out_hit`): per-crate ray-AABB slab test against crate GetActorLocation ± 150cm half-extents (all single-output math), or project each crate center to screen and compare with mouse pixel.

### 1d. Save + place instance
- `AssetTools.save_assets ["/Game/Rooftop/BP_RooftopGame"]`.
- `SceneTools.add_to_scene_from_class` — class refPath = generated class (get via `ObjectTools.get_class` on the asset, or try `/Game/Rooftop/BP_RooftopGame.BP_RooftopGame_C`); name "RooftopGame"; xform anywhere out of the way, e.g. (-2000, 5000, -100).
- `ObjectTools.set_properties` on the **instance**: CratesA = [Crate_A_0, Crate_A_1, Crate_A_2] (**order matters: index = L/M/R, must match texture step**), ShooterCam, HiderSpawn. ⚠️ verify JSON format for array-of-object-refs (try `[{"refPath": "..."}...]`, else `["..."]`).
  - Level actor refPaths look like `/Game/Rooftop/Lvl_Rooftop.Lvl_Rooftop:PersistentLevel.Crate_A_0` — confirm with `SceneTools.find_actors` (requires name/tag/collision_channels keys all present, empty values OK).
- Save the level too: `AssetTools.save_assets ["/Game/Rooftop/Lvl_Rooftop"]`.

### 2. HUD (UMG)
Minimal UMG: phase label + timer, scores, big CAUGHT!/ESCAPED! banner. Workflow per UMGToolSet docs: create widget → `ObjectTools.list_properties`/get/set with EXACT names (never guess). BP_RooftopGame will need CreateWidget+AddToViewport in BeginPlay once the HUD exists — or keep PrintString-only feedback for v1 and add HUD after core loop works.

### 3. Box textures
Original SVGs restored to `Content/Rooftop/Art/` this session. Host has no ImageMagick/Inkscape → try `uvx cairosvg` or pip in a venv; else ask user for PNG exports. Import as textures (`TextureTools`) → material per crate (`MaterialInstanceTools`), apply: Crate_A_0=L, _1=M, _2=R (matches CratesA array order!).

### 4. PIE test loop
`EditorAppToolset.StartPIE` (bSimulate=false, playMode=PlayMode_InViewPort, warmupSeconds≈3) → read log via `LogsToolset` → `StopPIE`. Fix, repeat. Verify: pawn spawns at PS_Hider; hide timer fires; camera switches to ShooterCam; click registers ONLY during SHOOT phase; scores increment correctly; hider can't move during SHOOT; R restarts after someone hits 5.

### 5. Optional later
Role swaps per round (crates already on both towers), split-screen, online multiplayer, sounds/VFX, kill-floor respawn if hider walks off roof (check pawn z < 3500 in Tick).

## MCP gotchas learned the hard way
- Top-level MCP tools are only `list_toolsets` / `describe_toolset` / `call_tool`. Everything else goes through `call_tool` with ALL of: `toolset_name`, `tool_name`, `arguments`. Tool names inside a toolset (e.g. get_graph_dsl_docs) are NOT top-level MCP tools.
- **Graph refPath format**: `/Game/Path/BP_X.BP_X:EventGraph` (`AssetName:ObjectName`). `:ScriptBlueprint.EventGraph` is rejected as "not valid EdGraph".
- **"Required but can be empty" params**: `find_node_types` requires the `context_pins` key (pass `[]`); `SceneTools.find_actors` requires name/tag/collision_channels keys (empty values OK). Omitting them errors even when you don't need filtering.
- **ObjectTools param names**: `instance` (not object), and `set_properties` takes `values` as a JSON *string*.
- `CaptureViewport` requires explicit `captureTransform` AND `annotations` objects even though "optional". Its base64 PNG is ~1MB → don't route through context. Direct HTTP trick: curl initialize (grab `mcp-session-id` header) → notifications/initialized → tools/call, save response to file, decode with python.
- Batch ops via `ProgrammaticToolset.execute_tool_script`: define `run()` returning dict; call other tools with `execute_tool(full_name, json_string)`; only json/math/time/copy/datetime/re importable. Call `get_execution_environment` once first.
- New maps: no create-level tool → `AssetTools.duplicate` an existing .umap to new path, `save_assets`, then `SceneTools.load_level`. (Duplicate leaves it dirty in memory — save before load.) Same trick works for Blueprints (no create-blueprint tool exists).
- Editor must be RESTARTED after .uproject plugin changes; MCP server only lives inside the editor process.
- `AssetTools.write_file` can write text files under /Game/, plugin Content, or Saved/.

## Repo state
Git repo on branch `master`, remote → GitHub (created 2026-10-27). `.gitignore` excludes ALL *.uasset/*.umap (+ .meta) — only .uproject + Config/ are tracked. **Uncommitted as of session 2**: modified `firstgame.uproject` + `Config/DefaultEditorPerProjectUserSettings.ini` (session-1 MCP wiring), untracked `HANDOFF.md`, and new `Content/Rooftop/Art/*.svg`. Commit these before starting risky work. If user wants content in git later, they must change the .gitignore policy deliberately.
