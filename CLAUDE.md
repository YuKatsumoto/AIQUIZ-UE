# UE 5.7 + Claude Code + VibeUE — Working Agreement (portable template)

> Copy this file into a new Unreal project root as **`CLAUDE.md`**, then add a short
> project-specific header above this line (what you're building, where content lives, the player
> pawn, the default map, how to launch the editor). Everything below is engine/tooling guidance that
> carries over between projects — keep it verbatim.
>
> **Assumed setup:** a **UE 5.7 Blueprint** project (gameplay in Blueprints, no game C++ module) with
> **VibeUE** (in-editor MCP, HTTP :8088) and **UnrealClaude** (HTTP :3000) declared in `.mcp.json`;
> open the editor first (a safe-DDC launcher avoids the UE 5.7 ZenServer DDC hang), then start Claude
> Code from the project folder so the MCP servers connect.

---

## 0. Golden rule: USE the VibeUE skill library — do not re-derive it

VibeUE ships a **lazy-loading skill library** at `Plugins/VibeUE/Content/Skills/` (≈23 skills:
`blueprints`, `blueprint-graphs`, `materials`, `umg-widgets`, `enhanced-input`, `data-tables`,
`data-assets`, `enum-struct`, `level-actors`, `animation-blueprint`, `niagara-systems`,
`landscape`, `screenshots`, `sound-cues`, `state-trees`, …). These are battle-tested and cover
most UE-tooling gotchas. **They are reachable the whole time the VibeUE MCP (:8088) is connected
— no manual activation is needed; you just have to load them.** Nobody injects them for you, so
loading is YOUR responsibility.

**MANDATORY before touching any of those domains:**

1. If VibeUE is connected, run `manage_skills(action="list")` **once per session** to see what exists.
2. Before the FIRST edit in a domain, `manage_skills(action="load", skill_name="<name>")`.
3. Use method names/params from the skill response's **`vibeue_apis`** block (auto-discovered, real
   signatures) — **NOT** from memory or from the example code in the skill's `content`.
4. Never guess a VibeUE/Unreal method name. If it isn't in `vibeue_apis` or discovery, it doesn't exist.

**Domain → skill mapping (load before working):**

| Trigger / asset prefix | Load skill |
|---|---|
| BP_, Blueprint, variables, components | `blueprints` |
| node, graph, wire, connect, pin, timer, event/function graph | `blueprint-graphs` |
| M_, MI_, material, material instance | `materials` |
| WBP_, widget, UMG, HUD | `umg-widgets` |
| IA_, IMC_, input action, Enhanced Input | `enhanced-input` |
| DT_/DA_, data table / data asset | `data-tables` / `data-assets` |
| ST_, StateTree, state machine | `state-trees` |
| screenshot / capture / vision | `screenshots` |
| level actor, place/spawn actor | `level-actors` |
| skeleton / anim BP / montage | `skeleton` / `animation-blueprint` / `animation-montage` |
| landscape / terrain | `landscape` |
| Niagara / VFX | `niagara-systems` / `niagara-emitters` |

**Do NOT re-derive these — the skills already document them correctly:**
- UE 5.7 math is **doubles** (`Add_DoubleDouble`, `Multiply_DoubleDouble` — NOT `_FloatFloat`).
- Blueprint node IDs are **32-char GUID strings**, not small ints.
- Branch pins are **`then`/`else`**, not `true`/`false`.
- **Compile the BP after structural changes** (vars/components/functions) before adding nodes.
- Don't guess function node names (`Get Actor Location` → `K2_GetActorLocation`); use
  `discover_nodes()` + `create_node_by_key()`.
- Cross-class member access → `add_member_get_node(bp, graph, "OtherClass_C", "Member", x, y)`.
- Failed-graph recovery → read `get_connections()`, remove orphans with `delete_node()` /
  `disconnect_pin()`; don't invent helper names.
- `ScreenshotService.capture_viewport()` from Python **never works** (async, file never lands) →
  use `capture_editor_window()`.
- `EditorLevelLibrary` / `spawn_actor_from_class` are **deprecated** in 5.7 → `EditorActorSubsystem`.
- Verification gate: after any graph edit, re-read nodes + connections + compile before claiming done.

---

## 1. Gotchas NOT covered by VibeUE skills (our hard-won delta)

These are the things we had to discover ourselves. Keep them here.

### 1.1 ⭐ Blueprints CANNOT be edited while PIE is running
Any BP graph/variable edit during Play returns an empty GUID / "Editor is currently in a play mode".
**End PIE and wait 2–3 s** before editing. PIE control:
```python
les = unreal.get_editor_subsystem(unreal.LevelEditorSubsystem)
les.editor_request_begin_play()      # start; then wait 5–8 s before observing
les.editor_request_end_play()        # stop; then wait 2–3 s before editing BPs
les.is_in_play_in_editor()           # check state
```

### 1.2 ⭐ Runtime `SpawnActorFromClass` driven through tooling crashes the editor
Spawning actors at runtime via the tooling bridge can hard-crash the editor. For content you'd
otherwise spawn/destroy on the fly, prefer a **pool of pre-placed actors** you reposition/recycle
instead — more stable, and the standard approach for repeated runtime content (see §1.13 for the
actor-vs-`ChildActorComponent` choice).

⚠️ **Not just spawning:** any **world-lifecycle** call from `execute_python_code` **during PIE** —
`GameplayStatics.open_level`, level travel, `quit_editor` — tears the world out from under the running
script and **hard-crashes the editor**. Trigger restarts/level transitions from inside the game's **own
Blueprint** (e.g. a button `OnClicked → OpenLevel`), never from editor Python. Recover per §1.7.

### 1.3 Cross-tool: screenshots & actor rotation go through UnrealClaude, not VibeUE/Python
- **Viewport screenshots that you can SEE inline** → UnrealClaude `unreal_capture_viewport`
  (returns base64) for the 3D scene. ⚠️ **UMG/Slate is NOT in viewport screenshots**, and
  `get_all_widgets_of_class` can't be called from Python (§1.10) — that route is a dead end. To actually
  **see the HUD/menus**, use VibeUE `ScreenshotService.capture_editor_window("<abs>.png")` then **`Read`
  the PNG file** (it renders inline in this harness). ⚠️ That capture **swaps R/B channels** (red↔blue,
  gold→cyan) — colors are correct in-game; confirm a true value with
  `WidgetService.get_property(p, comp, "ColorAndOpacity")`. (`capture_viewport` from Python never lands a file.)
- **Yaw / actor rotation:** Python `set_actor_rotation` hits gimbal lock — use UnrealClaude
  `unreal_move_actor` with a `rotation` param instead.

### 1.4 VibeUE `execute_python_code` swallows output on exception
Start every script with `import warnings; warnings.simplefilter("ignore")` or an uncaught exception
can return empty output. Also **print after every create/modify** (`CREATED:`/`MODIFIED:`/`DELETED:`
+ full asset path) — there's no auto-rollback, so the log is your only undo trail.

### 1.5 Save & persistence
After edits, save with `unreal.EditorAssetLibrary.save_asset(path)` per asset (verified). ⚠️ There is
**no `EditorAssetLibrary.save_dirty_packages_with_dialog`** (AttributeError) — to save everything use
`unreal.EditorLoadingAndSavingUtils.save_dirty_packages(True, True)` (a different class) or just loop
`save_asset` over the assets you touched. Editing a **BP component template** propagates
collision/material to `ChildActorComponent` children, but **transforms do NOT** — set scale/position
on the CAC itself. Template edits go through
`SubobjectDataSubsystem.k2_gather_subobject_data_for_blueprint(bp)` →
`k2_find_subobject_data_from_handle(h)` → `SubobjectDataBlueprintFunctionLibrary.get_object(data)`,
then compile + `save_asset`.

### 1.6 ⭐ UMG layout & widget-tree traversal via tooling
- `WidgetService.set_property(path, comp, prop, value)` takes **strings** (ImportText), e.g.
  `set_property(p, "Title", "Font.Size", "72")`, `set_property(p, "BG", "BrushColor", "(R=0,G=0,B=0,A=0.6)")`.
- **Slot layout (anchors/offsets/alignment) is NOT a `set_property` / `set_editor_property` thing** —
  `slot.set_editor_property("anchors", …)` fails ("Failed to find property 'anchors'"). Use the
  `CanvasPanelSlot` **setter methods** — these DO work, so the old "anchors can't be set, use
  `RenderTransform`" advice is now just a last resort:
  `set_anchors(unreal.Anchors(min,max))`, `set_offsets(unreal.Margin(l,t,r,b))`,
  `set_alignment(unreal.Vector2D)`, `set_auto_size(bool)`, `set_position()`, `set_size()`.
- **Reaching the slot objects:** `WidgetBlueprint` does NOT expose `widget_tree` (no editor property,
  no attribute), and the `WidgetTree` itself has NO `get_all_widgets()`/`find_widget()`. Walk it manually:
  ```python
  wb   = unreal.EditorAssetLibrary.load_asset(path)
  tree = unreal.find_object(wb, "WidgetTree")        # find the tree as a subobject of the WBP
  w    = unreal.find_object(tree, "RestartButton")    # each widget is a named subobject of the tree
  slot = w.get_editor_property("slot")                # CanvasPanelSlot → use the setter methods above
  ```
- `WidgetService.add_component(path, type, name, parent, is_variable)` returns a result with only
  `success` — there is NO `message`/`error` field (reading it raises). Add the **root first**
  (`set_as_root`/empty parent), parents before children.
- **`set_brush`/`set_font` signatures differ from the skill's example prose** — `set_brush(path, comp,
  slot_name, brush_info)` is **4 args** (`slot_name="Brush"` for an Image); the skill body shows 3. Per §0,
  take signatures from `discover_python_class('unreal.WidgetService', method_filter=…)`, not the prose.
- TextBlock color is an `FSlateColor`: `set_property(p, comp, "ColorAndOpacity",
  "(SpecifiedColor=(R=…,G=…,B=…,A=…))")` (Image `ColorAndOpacity` is a plain `FLinearColor`
  `"(R=…,G=…,B=…,A=…)"`). For a solid dim overlay use an Image + `set_brush` with
  `resource_path="/Engine/EngineResources/WhiteSquareTexture.WhiteSquareTexture"` and a tinted color.

### 1.7 Editor restart / never nuke build artifacts
For plugin/C++ changes: save → `unreal.SystemLibrary.quit_editor()`, relaunch
`UnrealEditor.exe "<...>.uproject"`, wait 60–120 s, confirm with `ue-python.exe exec "print('ready')"`.
On a crash relaunch with `-ddc=InstalledNoZenLocalFallback` (use the project's safe-DDC `.bat`). After a
crash, first **kill the frozen `UnrealEditor` + `CrashReportClientEditor` processes**, then relaunch; it's
booted once the process memory climbs past ~1.8 GB, and the VibeUE MCP **auto-reconnects** (no manual step).
**Never delete `Binaries/`/`Intermediate/`** — triggers a long missing-modules rebuild.

### 1.8 General editor timing
After any scene change wait **4–5 s** before screenshots (viewport/HISM refresh).

### 1.9 ⭐ Showing a widget at runtime, and the "no member-set" ceiling
- The **Create Widget** node (`K2Node_CreateWidget`) is **NOT** returned by `discover_nodes` (any
  search term, any category). Build it via `build_graph` with `spawner_key="NODE K2Node_CreateWidget"`,
  then set its `Class` pin to the WBP's generated class:
  `set_node_pin_value(bp, "EventGraph", id, "Class", "/Game/UI/WBP_Menu.WBP_Menu_C")`.
  Until `Class` is set the node is titled **"Construct NONE"**; after, `ReturnValue` becomes the typed
  widget ref. Then wire `CreateWidget.ReturnValue → AddToViewport.self` (a `function_call`, class
  `UserWidget`, function `AddToViewport`). Creating the widget *asset* is separate — that's the
  `WidgetBlueprintFactory` pattern in the `umg-widgets` skill.
- ⛔ **No member-SET node for EXTERNAL objects — but you CAN set your OWN (incl. inherited native) vars.**
  `add_member_get_node` reads another object's member; there's no setter analogue, `build_graph` has
  `member_get` but no `member_set`, and `discover_nodes` surfaces functions/events only (never VariableSet
  spawners) ⇒ you can't set a property on **another** object via tooling nodes. **HOWEVER**
  `add_set_variable_node(bp, graph, "VarName", …)` DOES write the BP's **OWN** vars, **including inherited
  native properties**. So a custom PlayerController CAN set its own cursor:
  `add_set_variable_node(".../BP_RunnerController", "EventGraph", "bShowMouseCursor", …)` returns a real Set
  node and works at runtime (read-back pin is `bShowMouseCursor`; the Python prop strips the `b` →
  `show_mouse_cursor`). Verified pattern for a runtime mouse cursor: make a PlayerController BP, on its Tick
  read the controlled pawn's state and Set its own `bShowMouseCursor` (cursor shows only on death → menus
  clickable), then register it as the GameMode's `player_controller_class` (§1.10). The old "keyboard-only /
  bake-on-CDO" workaround is no longer required.

### 1.10 Python-binding & process gotchas (not in any skill)
- **`unreal.Vector` has no `.size()`** (`'Vector' object has no attribute 'size'`) — the C++ `Size()`
  isn't bound. Compute manually: `(v.x*v.x + v.y*v.y + v.z*v.z) ** 0.5`.
- **`unreal.WidgetBlueprintLibrary` is not a module attribute** (`module 'unreal' has no attribute …`):
  you can't call/introspect it from Python, but its functions ARE reachable as graph nodes via
  `build_graph`/`add_function_call_node` (`SetInputMode_UIOnlyEx`, `Create`, …).
- **CDO writes work via `get_default_object` (root AND subobjects).** `get_default_object(gc).set_editor_property(
  "player_controller_class", unreal.EditorAssetLibrary.load_blueprint_class(path))` returns OK and persists —
  the old "CDO **root** raises `PYTHON_UNSAFE_CODE`" claim is **stale** (VibeUE relaxed it). ⚠️ After a CDO
  **class-default** change you MUST `compile_blueprint(<bp>)` + `save_asset`, or PIE keeps the old value —
  this is exactly what makes a GameMode keep spawning the engine `PlayerController` instead of your custom one.
  Default **subobjects** are writable too — the no-C++ way to raise an inherited native component's default:
  `cdo.character_movement.set_editor_property("max_walk_speed", 2200.0)`. Note `set_variable_default_value`
  returns **False** for class-typed inherited props (use the CDO write instead); plain
  `set_component_property(bp, "CharMoveComp", "MaxWalkSpeed", …)` also returns **False** silently.
- **A malformed `WidgetService` / `execute_python_code` call can HARD-CRASH the editor** — e.g. a
  duplicate component name or a re-entrant `add_component` → `0xC00000FD` (stack overflow) then
  `0xC0000005`, NOT a returned error. Recover per §1.7 (relaunch via the safe-DDC launcher, wait,
  reconnect MCP).
- **`manage_skills(action="load")` with several skills at once** can return ~68k chars (over the tool's
  output cap) and get dumped to a file. Load skills **one at a time**, or grep the saved file for the
  `vibeue_apis` / pattern blocks you need.
- **`set_component_property` silently fails for FName/class props.** Setting `CollisionProfileName`
  returns **False**; setting `ChildActorClass` returns **True** but does NOT reach pre-placed level
  instances (they keep the old value). Set these on the component **template subobject** instead
  (§1.5 gather pattern): `mesh.set_collision_profile_name("OverlapAllDynamic")`,
  `mesh.set_material(0, mat)`, then compile + `save_asset`.
- **`set_generate_overlap_events` is NOT bound** on components (`AttributeError`) → use
  `comp.set_editor_property("generate_overlap_events", True)`.
- **A BP variable is NOT instance-editable by default** — `set_editor_property` on a live PIE instance
  raises *"Property '…' cannot be edited on instances"*. Flip it at edit-time (PIE off) with
  `BlueprintService.modify_variable(bp, var, set_instance_editable=1)`, then it's drivable for testing.
- **`build_graph` can return `None` outright** if ANY node spec is invalid (e.g. `function_call` with
  `class:"Self"`, or `event:"ReceiveActorHit"` — the actor hit event is **`ReceiveHit`**, which exposes
  `MyComp/Other/OtherComp/Hit` directly, so you rarely need Break Hit Result). Then `.nodes_created` throws
  `AttributeError`, yet it still **partially created** the valid nodes — so a naive retry also duplicates
  event nodes (Event Tick etc.). Recover: find orphans (in `get_nodes_in_graph` but absent from
  `get_connections`, excluding comments), `delete_node` them, finish with `add_*_node` + `connect_nodes`.
  When it DOES return a result it has no `b_success` — check `nodes_created`/`connections_made`/`errors`/`warnings`.
  To call your **own** function/custom event in a batch: `class:"Self"` fails inside `build_graph`, but
  `add_function_call_node(bp, graph, "Self", "Fn")` works as a standalone node. Always pull event names from
  `list_overridable_functions` — never guess (`ReceiveActorHit`, etc.).

### 1.11 ⭐ Graph node creation: pick the right method, and math is float-only
- `add_math_node(op, value_type)` does **arithmetic only** — `op` ∈ Add/Subtract/Multiply/Divide/Clamp/
  Min/Max/Abs. `value_type="Integer"` is **rejected** (returns an empty id). Use `"Float"` and let the
  schema auto-insert an int→float cast on connect (e.g. `RandomIntegerInRange.ReturnValue (int) →
  Float-Multiply.A` just works).
- **Comparisons are NOT math ops.** `Greater`/`Less`/`GreaterEqual`/`Equal` via `add_math_node` fail →
  use `add_comparison_node(comparison_type, value_type)` (pins `A`, `B`, `ReturnValue` bool).
- An **empty-string return** from any `add_*_node` = the node failed to spawn; check it before wiring.

### 1.12 ⭐ Enhanced Input consumes mapped keys — raw InputKey won't fire
A `K2Node_InputKey` for a key already bound in an **active** mapping context never fires (Enhanced
Input consumed it); a key left **unmapped** in every active context still fires a raw `InputKey`. To
add a new discrete control: `InputService.create_action("IA_X", path, "Boolean")`, **remove the key
from any other action first** (`remove_mapping(imc, idx)` — indices shift, so remove high→low),
`add_key_mapping(imc, ia, "A")` + `add_trigger(imc, idx, "Pressed")` (fires once on press), then
`BlueprintService.add_input_action_node(bp, graph, ia_path)` and wire its `Triggered` pin. To remove an
unwanted movement binding, **delete its IA-driven `AddMovementInput` nodes**.

### 1.13 ⭐ ChildActorComponent pitfalls → prefer standalone actors for repeated content
Two traps when you attach a `ChildActorComponent` (CAC) to a parent actor whose root mesh is scaled:
- A child added with **no explicit parent** attaches under the **scaled** root mesh and **inherits that
  scale** (squashed). Worse, `get_component_hierarchy` reports `attach_parent: ""` for both, so it
  *hides* the inheritance — only the live **world transform** reveals it.
- `set_component_property(actor, "Child", "ChildActorClass", "/Game/…/BP_X.BP_X_C")` returns True and the
  **template** reads back the class, but **pre-placed level instances** keep `child_actor_class=None` and
  spawn nothing (see §1.10 FName/class write).

So for repeated content, make each item its **own actor** (owning its collision) and place N copies at
**edit-time** via `EditorActorSubsystem.spawn_actor_from_class`, rather than nesting CACs under a scaled
parent. If items must recycle at runtime, drive it from the actor's own Tick (reposition when it passes a
threshold). This dodges both pitfalls above.

### 1.14 ⭐ UMG Button `OnClicked` — `WidgetService.bind_event` is a silent NO-OP
`bind_event(wbp, "MyButton", "OnClicked", "MyHandler")` **returns `True` but does nothing** — it creates
NO `ComponentBoundEvent` node and NO working binding, so the button silently does nothing when clicked
(this bit us: the runner's RETRY button never fired even though its `OnRetryClicked` function was correctly
wired to `OpenLevel` — *nothing ever called it*). A UMG Button's multicast `OnClicked` needs a real
bound-event node; the legacy function-binding route `bind_event` uses doesn't hook it up. Don't trust its
`True`, and don't claim a button works until the binding is verified.

**Working fix — bind at runtime in `Event Construct`** (the standard UMG pattern for a widget created via
CreateWidget+AddToViewport):
```python
btn  = add_get_variable_node(wbp, "EventGraph", "RetryButton", x, y)         # the Button is a variable
bind = add_delegate_bind_node(wbp, "EventGraph", "Button", "OnClicked", x, y) # ⚠ "Button" NOT "UButton"
crd  = add_create_delegate_node(wbp, "EventGraph", "OnRetryClicked", x, y)    # wraps your handler fn
connect_nodes(wbp, "EventGraph", btn,  "RetryButton",     bind, "self")       # Target = the button
connect_nodes(wbp, "EventGraph", crd,  "OutputDelegate",  bind, "Delegate")   # handler delegate
connect_nodes(wbp, "EventGraph", last, "then",            bind, "execute")    # chain off end of Construct
```
- ⚠️ `add_delegate_bind_node` `target_class` must be **`"Button"`** (or `"/Script/UMG.Button"`); the API
  doc's example **`"UButton"` returns an empty id (FAILS)**. Bind-node pins are `execute / then / self / Delegate`.
- The `Create Event` (`add_create_delegate_node`) wraps a **same-signature** function (here no-param/void to
  match `OnClicked`); leave its `self` pin unconnected (defaults to self). Compile must show 0 errors.
- Belt-and-suspenders: set a full-screen dim overlay Image to **`Visibility="HitTestInvisible"`** so it can't
  eat clicks meant for the button.

**Verifying a runtime binding without crashing:** `WidgetService.spawn_widget_in_pie(path)` FAILS mid-PIE
(`valid:False`, "Widget Blueprint not found" — the load_asset-None-during-PIE issue, §3; note the handle
field is **`valid`**, not `b_valid`). Instead trigger the real flow on the live instance:
`pawn.call_method("Die")` runs the actual BP function (no `OpenLevel` → safe), then grep `Saved/Logs/*.log`
for a temp `PrintString` placed after the bind node to confirm the bind executed. UE dynamic delegates expose
`is_bound()` / `contains_function()` if you can get the instance. ⛔ NEVER `broadcast()` a delegate whose
handler calls `OpenLevel`/level-travel during PIE (hard-crash, §1.2) — let a human click it in-game instead.

### 1.15 ⭐ Heavy asset ops (FBX import) HARD-CRASH if called synchronously in `execute_python_code`
Importing an FBX with `AssetToolsHelpers.get_asset_tools().import_asset_tasks([task])` **directly** inside
`execute_python_code` crashes the editor with a TaskGraph assertion (NOT a returned error):
```
Assertion failed: ++Queue(QueueIndex).RecursionGuard == 1  ...\Async\TaskGraph.cpp  Line: 689
```
**Why:** `execute_python_code` runs inside a TaskGraph *task*. The static-mesh build (combine meshes,
collision, lightmap UVs) dispatches parallel work and then **synchronously pumps/flushes the task-graph
queue** — re-entering a queue already being processed one level up the stack → RecursionGuard fails →
fatal assertion. The first call returns a `PYTHON_RUNTIME_ERROR` "assertion failure"; the editor then
goes down and the **next MCP call can't connect**. Recover per §1.7 (no hung process to kill — it exits
clean; just relaunch via the safe-DDC `.bat`, wait, MCP auto-reconnects). This happens at **edit-time**
(not just PIE) — same root cause family as §1.2's world-lifecycle crashes: these engine ops assume the
normal main-loop tick, not a nested task context.

**Working fix — defer the import onto a one-shot slate post-tick callback** (runs from the main-loop tick,
exactly like clicking *Import* in the UI, so the internal flush is top-of-stack):
```python
import unreal, warnings; warnings.simplefilter("ignore")
_g = globals(); _g["_imp"] = {"ran": False, "h": None}
def _do(delta):
    st = _g["_imp"]
    if st["ran"]: return            # one-shot guard
    st["ran"] = True
    try:
        t = unreal.AssetImportTask()
        t.filename = r"C:\path\model.fbx"; t.destination_path = "/Game/Runner/Meshes"
        t.destination_name = "SM_X"; t.replace_existing = True; t.automated = True; t.save = True
        ui = unreal.FbxImportUI(); ui.import_mesh = True; ui.import_as_skeletal = False
        ui.set_editor_property("mesh_type_to_import", unreal.FBXImportType.FBXIT_STATIC_MESH)
        ui.static_mesh_import_data.set_editor_property("combine_meshes", True)  # merge sub-meshes → 1 SM
        t.options = ui
        unreal.AssetToolsHelpers.get_asset_tools().import_asset_tasks([t])
        unreal.log("IMPORT_OK " + ",".join(t.imported_object_paths))
    except Exception as e:
        unreal.log_error("IMPORT_ERR " + str(e))
    finally:
        unreal.unregister_slate_post_tick_callback(st["h"])
_g["_imp"]["h"] = unreal.register_slate_post_tick_callback(_do)
```
It runs on the **next** editor tick, so the call returns immediately — wait a few seconds (externally),
then confirm from a **separate** call with `EditorAssetLibrary.does_asset_exist(path)` and read back
`st["ran"]/result`. FBX with PBR textures imports the mesh **plus** its textures/material into the
`destination_path` (they land loose beside the `SM_` — move to subfolders if you want tidiness). Measure
the result with `sm.get_bounding_box()` before scaling to a target footprint. The same defer-to-tick
pattern applies to any heavy task-graph-flushing op driven through the MCP bridge.

### 1.16 ⭐ Material renders FLAT GREY (default fallback) on a TextureSample SamplerType↔compression mismatch
A material built via `MaterialNodeService` renders as the **flat grey default material** in the viewport
when **any** `TextureSample` node's `SamplerType` doesn't match the bound texture's compression settings —
and the **whole** material falls back, so even correctly-wired channels show grey (not just the bad one).
The trap: `MaterialService.compile_material()` still **returns `True`** on this error, so `compile==True`
is NOT proof the material is valid.

**Match sampler type to texture compression when creating samples:**

| Map | `srgb` | compression | SamplerType |
|---|---|---|---|
| BaseColor | `True` | `TC_DEFAULT` | `SAMPLERTYPE_Color` |
| Normal | `False` | `TC_NORMALMAP` | `SAMPLERTYPE_Normal` |
| Single-channel rough/metal/AO | `False` | `TC_GRAYSCALE` | `SAMPLERTYPE_LinearGrayscale` |
| Packed masks (RGBA linear, e.g. glTF ORM) | `False` | `TC_MASKS` | `SAMPLERTYPE_Masks` |

Set these on the **texture asset** (`set_editor_property("srgb", …)` / `("compression_settings", …)` +
`save_asset`) AND on the node (`MaterialNodeService.set_expression_property(mat, id, "SamplerType", …)`),
then `compile_material` + `save_asset`. Remove a stray output wire with
`MaterialNodeService.disconnect_output(mat, "EmissiveColor")`.

**Diagnostics:**
- Temporarily wire the BaseColor sample → `EmissiveColor`. If the mesh **still** doesn't change color at
  all, the material is erroring out (default fallback) — NOT a lighting/exposure issue.
- The **editor viewport does NOT redraw** on asset/material changes when realtime is off, so consecutive
  `unreal_capture_viewport` calls return a **stale, byte-identical** frame. Nudge the camera with
  `UnrealEditorSubsystem.set_level_viewport_camera_info(loc, rot)` (then wait 3–4 s) to force a fresh frame
  before capturing — otherwise you'll misread "no change" as "fix didn't work".
- Swapping a `StaticMesh`'s slot material: `static_materials[i].set_editor_property("material_interface", …)`
  does **NOT persist** (structs returned by copy; `imported_material_slot_name` is read-only so you can't
  rebuild the array either). Use `sm.set_material(index, mat)` + `save_asset` instead.

---

## 2. Tooling map (which server for what)

MCP servers come from `.mcp.json` (need the editor running + Claude launched from the project dir):

- **VibeUE** (HTTP :8088) — **primary**. `manage_skills` (load the skills above),
  `execute_python_code` (arbitrary `unreal.*` + `BlueprintService`/`WidgetService`/`ActorService`/
  `ScreenshotService`), `manage_asset` (search/find/open/save/duplicate/move/delete). Best for
  Blueprint graphs, materials, UMG, assets. (`.mcp.json` connects **directly** to the in-editor
  server on :8088; :8089 is only the optional `vibeue-proxy.py` that forwards to :8088.)
- **UnrealClaude** (HTTP :3000) — inline viewport **screenshots** (`unreal_capture_viewport`) and
  gimbal-lock-free actor **rotation/move** (`unreal_move_actor`). Use for the two things in §1.3.
- **ue-python-cli** via Bash — `ue-python.exe exec "<python>"` for PIE control, asset creation, any
  `unreal.*` when MCP isn't exposing what you need.

If VibeUE is NOT connected, its skills are unreachable — say so and fall back to UnrealClaude +
ue-python-cli + this file.

---

## 3. Verification loop (Blueprint project, no C++ test harness)

1. Edit BP (VibeUE `BlueprintService`) or assets (Python) — **load the matching VibeUE skill first**.
2. Compile the BP in-editor; check `compile_blueprint(...).success` + errors.
3. Re-read graph: `get_nodes_in_graph` + `get_connections` confirm the intended wiring exists.
4. Start PIE, then **wait externally** (PowerShell `Start-Sleep`) — never `time.sleep` inside
   `execute_python_code`; it runs on the game thread and freezes PIE, so nothing advances and the wait is wasted.
5. Read live state: `world = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_game_world()`
   → `unreal.GameplayStatics.get_player_pawn(world, 0)` / `get_player_controller(world, 0)` →
   `obj.get_editor_property("MyVar")`. **Reading** a live PIE instance works; only **writing** raises
   "cannot be edited on instances" (§1.10). ⚠️ `EditorAssetLibrary.load_asset()` can return `None` during
   PIE — do asset/CDO introspection with PIE **stopped**.
6. Screenshot: UnrealClaude `unreal_capture_viewport` for the 3D scene (wait 4–5 s after scene changes);
   for **UMG/HUD** use `ScreenshotService.capture_editor_window(png)` + `Read` the file (§1.3).
7. Read `Saved/Logs/<Project>.log` for PrintString/UE_LOG output.
8. End PIE + wait 2–3 s before the next BP edit.
