# Working with the native UE 5.8 MCP (+ VibeUE) — universal guide

This is a project-agnostic guide to driving an Unreal Engine 5.8 editor through the **native UE 5.8
Model-Context-Protocol server plus the VibeUE plugin**. It is all hard-won operating
knowledge — the tool surface, the bring-up, the crash recovery, and the long list of silent traps in
PCG, materials, Python, and lighting. None of it assumes a particular map, asset, or graph; apply it to
whatever project is open. Where a value is given (a property name, an enum, a compression setting) it is
the exact name the engine expects — take those literally, but take *graph/asset names* from the project
you are actually in.

---

## 0. The tool surface

The editor exposes one merged MCP tool surface: the native UE 5.8 MCP toolsets plus the tools the
**VibeUE plugin** adds on top (most importantly `execute_python_code`, arbitrary `unreal.*`). To the
agent they are just one list of tools — pick whatever fits the task. The one thing worth knowing: a
*bare* native server with no VibeUE installed has no `execute_python_code`, so if that tool is absent
you must work through the typed toolsets alone.

- **Tool discovery:** with `bEnableToolSearch=true`, `tools/list` exposes only three meta-tools
  (`list_toolsets`, `describe_toolset`, `call_tool`); every real tool is reached *through* `call_tool`
  with a `toolset_name` + `tool_name`. Don't conclude the server is empty because `tools/list` is short.

### What the toolset covers
- **PCG** — full graph CRUD and introspection: `CreateGraph`, `AddNode`, `AddSubgraphNode`,
  `Connect/DisconnectNodePins`, `UpdateNode`, `RemoveNode`, `SpawnGraphInstance`, `ExecuteGraphInstance`,
  `ListNativeNodes`, `GetNativeNodeSchema`, `GetGraphSchema`, `GetGraphStructure`, `GetNodeInfo`,
  `GetNodeDataView`, `SetGraphParams`, `Set/Get/ResetGraphInstanceParams`.
- **Editor surface (EditorToolset family)** — actor/scene/asset/material/static-mesh/object/primitive
  CRUD, the `ProgrammaticToolset` (scripted scene fills), and `EditorAppToolset` for `CaptureViewport`,
  PIE start/stop, and camera get/set. `SceneTools` covers `load_level`, `add_to_scene_from_asset`,
  `add_to_scene_from_class`, `remove_from_scene`, `find_actors` (incl. by tag).
- **`execute_python_code`** — the escape hatch for anything without a dedicated tool: setting spline
  points, building a material node graph, setting texture/light/PPV properties, GeometryScript mesh
  edits, enumerating ISM/HISM per-instance transforms. Use it directly when it fits.

---

## 1. Bring-up & confirming the surface is complete

1. Launch the editor with the MCP server enabled (`-ModelContextProtocolStartServer`, optionally
   `-ModelContextProtocolPort=N`) — however the project normally launches it.
2. **Confirm the EditorToolset family is live**, not just PCG/AI/skill toolsets: `list_toolsets` must
   show the actor/scene/asset/material/static-mesh tools and `EditorAppToolset` (CaptureViewport/PIE).
   If only PCG shows, EditorToolset is not enabled in the `.uproject` — enable it and restart. Enabling
   it unlocks ~15 additional tools, i.e. full native actor + asset CRUD.
3. **HTTP transport facts** (matter when you drive the server raw): the native MCP returns tool results
   **inline on the POST** to `/mcp` (the SSE event comes back on the same response); there is **no
   GET-SSE channel** (a `GET /mcp` returns 405). Initialization returns an `Mcp-Session-Id` header.
   Keeping a tiny raw-HTTP client around lets you stay autonomous if the in-session tool connection drops.
4. Console controls: `ModelContextProtocol.StartServer [port]`, `StopServer`, `RefreshTools`. **Adding a
   new UFUNCTION tool requires an editor restart** — Live Coding does not propagate new tools.
5. Verify the surface in one line: `list_toolsets` shows the EditorToolset family alongside PCG,
   `CaptureViewport` returns a frame, and `execute_python_code` reports engine `5.8.0`.

---

## 2. Crash recovery (uncommon on 5.8 — but know the steps)

The 5.8 build crashes rarely. If the editor *does* go down, recover and resume without waiting to be
asked:
1. Kill the `CrashReportClientEditor` dialog and the frozen `UnrealEditor` process.
2. Relaunch the editor (this re-starts the MCP server).
3. Wait until it is actually up (~60–120 s: process working set climbs and the port begins listening),
   then continue.
4. If the in-session MCP tools dropped, drive the server over **raw HTTP** rather than waiting for a
   manual reconnect.

Two safety notes:
- A `CrashReportClientEditor` process that shares the editor's exact StartTime is the normal
  out-of-process crash **monitor** launched at boot — **not** evidence of a crash. Check the log and
  `Saved/Crashes` timestamps before concluding anything actually crashed.
- **Never delete `Binaries/` or `Intermediate/`** — it triggers a long missing-modules rebuild.

---

## 3. Skills-first protocol — MANDATORY before the first action in any domain

Loading skills is **your job** and it is a gate, not optional setup. Nothing injects them automatically.
Skipping it is how you spend an hour re-deriving a documented trap.

- **Two libraries — run both listings once per session.** The native `AgentSkillToolset`
  (`ListSkills` / `GetSkills`) holds the PCG / ShapeGrammar recipes; the Python
  `manage_skills(action="list")` holds the Blueprint / UMG / material / lighting recipes
  (e.g. `default_outdoor_lighting`). They are separate.
- **Load the matching skill before the FIRST node/asset in that domain**, then follow its real
  signatures. Native: `GetSkills([...])`. Python: `manage_skills(action="load", ...)`.
- **Take signatures from the authoritative place, never the prose examples:** native →
  `GetNativeNodeSchema` / `GetGraphSchema`; Python → the skill response's `vibeue_apis` block or
  `discover_python_class(...)`. If a name is not in `ListNativeNodes`, a schema call, or a loaded
  skill's API block, **it does not exist** — do not invent it.
- **Python `manage_skills(load)` with several skills at once can exceed the output cap** and get dumped
  to a file — load Python skills **one at a time**.

### ShapeGrammar — the precise truth (don't over- or under-claim it)
The native **grammar NODES** (`Subdivide Spline`, `Subdivide Segment`, `Select Grammar`,
`Spline to Segment`, `Get Segment`, `Attribute Partition`) **exist and work**, and are the right tool
for road / facade / path-along-spline work — prefer them over hand-rolling. *Separately*, the
`/PCGPrimitives/` **subgraph examples** (`Assign_ShapeGrammarDefinition`, the `SGD_*` / `RULE_*` data
assets, blockout meshes) may be **unmounted** in a given project. So: **check `ListNativeNodes` / asset
existence first.** If the grammar nodes are present, use them; if you specifically need the
`/PCGPrimitives/` subgraph system and it is unmounted, hand-roll with `Spawn Spline Mesh` + a tagged
spline instead. Don't say "ShapeGrammar is unusable" as a blanket — it conflates the two.

---

## 4. `execute_python_code` — the escape hatch and its edit-time traps

- **It swallows output on exception.** Start scripts with `import warnings; warnings.simplefilter("ignore")`
  and `print` after every create/modify. There is no auto-rollback — the log is your only undo trail.
- **World-lifecycle ops hard-crash at EDIT time.** `LevelEditorSubsystem.new_level(...)` run directly
  inside `execute_python_code` asserts (`TickTaskManager`) then access-violates, because it re-enters the
  level manager from inside a tick/TaskGraph task. **Fix:** build in the already-open level, or defer the
  op onto a one-shot `register_slate_post_tick_callback` (runs top-of-stack, like File→New Level). The
  same defer-to-tick applies to any heavy task-graph-flushing op (FBX/texture/GLB import, GeometryScript
  mesh build).
- **…but during PIE these are FIXED on 5.8.** What was a hard crash on 5.7 no longer crashes on 5.8:
  `open_level` from editor Python *during PIE* performs a clean world travel, and
  `spawn_actor_from_class` during PIE returns `None` (safe no-op). So distinguish: **edit-time
  world-lifecycle = defer-to-tick; PIE-time world-lifecycle = safe.**
- **`set_actor_transform` resets unset fields to identity** — always pass full location + rotation +
  scale; use `look_at` afterwards for facing (it preserves scale).
- **Two name forms for the same bool flag — use the right one for the channel you're on.** When setting
  a bool UPROPERTY through **Python `set_editor_property`**, the leading `b` is stripped to snake_case
  (`bSetDensity` → `set_density`, `bUnbounded` → `unbounded`, `bUseConstantThreshold` →
  `use_constant_threshold`, `bScaleMeshToBounds` → `scale_to_bounds`); the original `b*` name throws
  "Failed to find property" there. But as a **`jsonParams` key or schema field name** the original
  `b`-prefixed form is the correct one (`bAlwaysRequeryActors`, `bScaleMeshToBounds`, …). So the same
  flag legitimately appears `b`-prefixed in JSON and de-`b`'d in Python — that's not a contradiction.
- **Other 5.8 binding gaps:** `unreal.Vector.size()` → AttributeError (compute manually); some
  `hasattr` checks on components return False (use `set_editor_property`); `BlueprintService`/component
  property setters can silently fail for `FName` props — verify the write.

---

## 5. PCG param-setting reality — read before any graph work

This is the single biggest source of wasted time. `AddNode` / `UpdateNode` take `jsonParams` typed as a
**stringified, NESTED dict** — never a raw object, never dotted keys. (`{"actorSelector":{"actorSelection":"ByTag"}}`,
never `"actorSelector.actorSelection"`.) Both wrong forms return `true` and silently apply nothing.

- **On UE 5.8, native `UpdateNode` reliably sets ordinary value params** — filter operators/thresholds,
  `Distance` settings, attribute-maths ops, range thresholds, and it can wire an attribute *selector* to
  a pin — when you pass `jsonParams` as a **JSON STRING** with the full struct fields and a
  (can-be-empty) `nodeTitle`. `UpdateNode` also invalidates the compiled-subgraph cache, so a plain
  `ExecuteGraphInstance` afterwards picks up the change. Prefer `UpdateNode(JSON string)` and **verify
  with `GetNodeInfo`** (read `paramOverrides`) before trusting it.
- **Some things still silently no-op through `jsonParams` and require `execute_python_code`
  `node.get_settings().set_editor_property(...)`:**
  - the **Static Mesh Spawner mesh selector** — an instanced subobject `SetObjectProperties` cannot
    reach in any JSON form (see §6 for the two valid outputs);
  - the **`PCGAttributePropertyInputSelector` literal name** — `UpdateNode` can wire the selector, but
    its literal name resists every setter (`set_attribute_name` / `import_text` report ok yet it stays
    `@Last` / `$Density`); rely on **`@Last`** and put the producing node immediately upstream of its
    consumer instead. The sibling *output* selector (e.g. `Distance.output_attribute`) DOES accept
    `set_attribute_name`; for a point property use `set_point_property(unreal.PCGPointProperties.DENSITY)`;
  - several **spline/struct props** (e.g. `Get Spline Data`'s `actorSelector`, `Spline Sampler`,
    `Create Spline`, and `Spawn Spline Mesh` descriptors) that don't always stick. To edit a struct,
    **copy it out, mutate, set it back.**
- **`AddNode` jsonParams are ignored more often than `UpdateNode`'s** — a common reliable pattern is
  `AddNode` (bare) → `UpdateNode(JSON string)` → verify → fall back to `set_editor_property` for the
  hold-outs.
- **`SetGraphParams` ZEROES every externalized param unless ALL FIVE fields are passed** per item:
  `name`, `type` (e.g. `"Double"` / `"Vector"`), `description`, `containerType` (`"None"` / `"Array"`),
  `defaultValueJson` (itself a JSON string, e.g. `"35000.0"` or `"{\"x\":1500,\"y\":1500,\"z\":200}"`).
  A zeroed Double on a `Distance` node makes every clearance filter pass; a zeroed Vector degenerates a
  grid to 0 points. Verify via `user_parameters.export_text()`.
- **Asset-array graph params** (weighted mesh pools fed as overrides) must be typed **`SoftObjectPath` +
  `Array`** (items read as `/Script/CoreUObject.Object`); plain `Object` / `SoftObject` give
  `PropertyBagMissingObject` and silently reject every override.
- After any node edit the **first `ExecuteGraphInstance` may race the recompile** ("Failed to call
  Execute on instance") — just execute a second time.

---

## 6. Cross-cutting PCG node gotchas (universal)

- **`Get Spline Data` caches.** `bAlwaysRequeryActors` defaults false → it reads STALE points after a
  spline edit. Set it `true` for any live / auto-regen workflow. It matches **actor** tags (not
  component tags); the literal tag string must be exact.
- **`Spawn Spline Mesh`:** the mesh ref *is* settable (`splineMeshDescriptor.staticMesh`), but **width
  comes from the spline POINT scale**, not `startScale`/`endScale` (silently ignored). Set
  `bScaleMeshToBounds=FALSE` — `true` collapses the mesh to zero on a non-landscape spline ("convex hull
  zero particles", invisible). **One node meshes only the FIRST input spline** → give each spline its
  own `Get Spline Data` + `Spawn Spline Mesh` pair, or wrap the per-spline work in a subgraph invoked
  once per spline.
- **Volume-scale ×N bug (Spawn Spline Mesh only).** Its `SplineMeshComponent`s are children of the
  PCGVolume, so the volume's `scale3d` MULTIPLIES generated world positions (a (150,150,…) volume bakes
  the result ~150× away). **Keep such volumes at scale (1,1,1).** Scatter / Spawn-Actor / HISM output is
  world-space and unaffected.
- **`Create Points Grid`:** `coordinateSpace` = `World` | `OriginalComponent` | `LocalComponent`
  (`"Local"` is rejected). **Z-degeneracy:** points-per-axis = `floor(2*gridExtents/cellSize)`; if
  `gridExtents.z < cellSize.z/2` the grid silently produces **zero** points. `World` centers at origin;
  `LocalComponent` follows the PCGVolume.
- **THE SUBGRAPH BIG RULE:** bounds/actor/landscape-dependent nodes return **EMPTY inside a subgraph** —
  `Surface Sampler`, `Get Landscape Data`, `Get Actor Data`(Self), and unbounded `Spline Sampler` all
  silently produce zero data (they work standalone). Inside a subgraph use `Create Points Grid`(World) +
  offsets, and feed every `Spline Sampler` a non-empty **Bounding Shape** (an `Extents Modifier`-inflated
  point cloud). Override pins DO work in subgraphs (their labels are the raw property names).
- **Distance exclusion direction.** `Distance` with `bSetDensity=true` sets density =
  `clamp(dist/maximumDistance)` → far points → 1, near → 0. To KEEP points away from a corridor, chain
  `Distance` → `Filter`/`Density Filter` ≥ threshold. The target must be **points** (sample the spline
  first). When you must preserve the density a spawner reads, use `set_density=false` +
  `output_to_attribute=true` and filter on that named attribute (or `@Last`). Difference/Create-Surface
  is for closed AREAS; Distance is best for linear clearance zones (any spline-driven corridor).
- **Static Mesh Spawner output — two valid routes** (the selector subobject is JSON-blocked):
  (a) **discrete actors** — `Add Attribute`(`Mesh`, `SoftObjectPath`) → `Spawn Actor`
  (`StaticMeshActor`, `option="NoMerging"`, override `StaticMeshComponent.StaticMesh`;
  `CollapseActors` spawns invisible empties); (b) **real HISM** — Static Mesh Spawner +
  `PCGMeshSelectorByAttribute` reading a `Mesh` attribute, with the selector's `AttributeName` set via
  `execute_python_code`. For a **weighted pool**, mutate `mesh_entries` **in place**
  (`mesh_selector_parameters` is read-only); the node `seed` folds into the weighted pick (set different
  seeds on mirrored chains to desync them).
- **`GetNodeDataView` is a two-pass dance.** The first call only ENABLES inspection and returns
  "produced no data"; `ExecuteGraphInstance` again, then read. Per-element JSON is keyed `element_N`
  (count them; there is no total field).
- **Auto-regen via subgraph merge.** A second spline-driven PCG volume will NOT auto-regenerate on
  viewport spline edits even with identical settings (enlarging its box does not help). The working fix:
  merge the work into the reliably-tracked graph with `AddSubgraphNode` so one component drives both.
  `ListAvailableSubgraphs` may return `[]` yet `AddSubgraphNode` still works and executes; terminal
  spawners need no output wiring. Prefer the **editor-path `PCGComponent.generate(force=True)`** over
  native `ExecuteGraphInstance` (which under-produces and skips editor change-tracking), and
  `flush_pcg_cache()` after a child-graph edit so the parent drops its stale subgraph snapshot.
- **BP-spline points revert.** A BP-created `USplineComponent`'s programmatic point edits revert to the
  BP default on reconstruction while `bSplineHasBeenEdited == False`; setting tags/properties can trigger
  that reconstruct. Fix once: after setting points, set `bSplineHasBeenEdited=True` and save — points
  then survive reload, and a viewport drag flips the flag automatically. **Never re-exec a spline-driven
  volume after points may have reverted** (with `bAlwaysRequeryActors=true` it reads the empty default
  and bakes EMPTY over your output — and the `GetNodeDataView` dance itself re-executes). After the final
  correct build, just SAVE.

---

## 7. Materials

- **Prefer the materials skill** (VibeUE `MaterialNodeService` / `MaterialService`) for building node
  graphs; raw `MaterialEditingLibrary` also works. Either way, load the material skill first (§3) — don't
  hand-write node graphs without having pulled the real node-type strings and pin names.
- **⭐ The flat-grey SamplerType trap.** If any `TextureSample`'s `SamplerType` mismatches its texture's
  compression, the WHOLE material silently falls back to flat grey — **and `compile` returns `True`, so a
  successful compile is NOT proof.** Set on each **texture asset**: BaseColor `srgb=True` / `TC_DEFAULT` /
  `SAMPLERTYPE_Color`; Normal `srgb=False` / `TC_NORMALMAP` / `SAMPLERTYPE_Normal`; single-channel
  rough/AO/height `srgb=False` / `TC_GRAYSCALE` / `SAMPLERTYPE_LinearGrayscale`; packed RGBA masks
  `srgb=False` / `TC_MASKS` / `SAMPLERTYPE_Masks`. Diagnostic: wire the BaseColor sample to
  EmissiveColor — if it's still grey, the material is *erroring* (silent fallback), not a
  lighting/exposure problem.
- **World-aligned tiling.** Feed `WorldPosition` (it has a ready `XY` output) × a `TileScale` scalar into
  each `TextureSample`'s **`Coordinates`** pin (the UV pin is named "Coordinates", not "UVs"). This keeps
  real-world texel size constant across differently-scaled meshes/blocks with no seams — the right choice
  for ground/road/large surfaces. Use `TextureCoordinate` (mesh UV) only when you want per-mesh UV.
- **Anti-tiling on large surfaces.** World-aligned tiling shows two artifacts: a distant uniform-brightness
  lattice and near-field repetition. Two layered fixes (no single button): a large-scale **macro
  variation** multiply (kills the aerial lattice) plus **break the sample periodicity** for mid/near. The
  engine ships the functions — `Texture_Bombing`(+`_POM`) in `Engine_MaterialFunctions01/Texturing`, and
  `TextureVariation`/`_RotateUV`/`_RotateNormals` in `Engine_MaterialFunctions03/Texturing`.
- **Importing non-FBX textures/meshes.** Native import is FBX/OBJ-only; bring in textures/GLB via Python
  Interchange / `AssetImportTask`, and **defer the import to a post-tick callback** (it is a heavy
  task-graph op that crashes if run synchronously through the MCP). Interchange nests a GLB under a
  source-filename subfolder (`…/<name>/StaticMeshes/<name>`).
- **Forcing a redraw.** With realtime off, the viewport won't redraw on an asset change — nudge the
  camera (`set_level_viewport_camera_info`) and wait a few seconds before capturing.
- **RVT masking pattern (one system masking another).** Have material A write a constant into a Runtime
  Virtual Texture, and material B sample that RVT to drive density/blend (e.g. a road footprint masking
  landscape grass). The **`material_type` must match** across the RVT asset, the writer's
  `RuntimeVirtualTextureOutput`, and the reader's `RuntimeVirtualTextureSample`, and an
  `RuntimeVirtualTextureVolume` actor must cover the region or nothing is written.

---

## 8. Lighting gotchas

- **"Brown mush" / blown-white sky = LOCKED auto-exposure.** A PostProcessVolume with
  `auto_exposure_min_brightness == max == 1.0` cannot adapt — a dark scene stays dark while a bright sky
  clips to white (a grazing near-horizon sun and a heavy warm color grade compound it). The generic fix
  is to UNLOCK exposure (`HISTOGRAM`, `min < max`) instead of pinning it.
- **5.8 gotcha:** `fog_inscattering_color` is no longer a direct component property on
  `ExponentialHeightFog` (moved into a struct) — setting it fails; skip it.

---

## 9. Heavy-GPU operations & the headless `-nullrhi` commandlet

Assigning a `LandscapeMaterial` (or any op that recompiles large GPU resources) causes a VRAM spike. In a
heavy scene near the VRAM ceiling it OOMs (`Out of video memory … RHISubmissionThread (D3D12)`) and
crashes both the live `set_editor_property` and `ObjectTools.set_properties` — in-editor cvars don't free
resident VRAM. The only reliable late fix is **no GPU at all**: close the live editor, then run the op
headless via `UnrealEditor-Cmd … -run=pythonscript -script=<no-space path> -nullrhi -unattended`. Five
gotchas: (1) **quote the `.uproject`** path if it contains a space, and keep the `.py` in a no-space dir;
(2) `AssetRegistry.scan_paths_synchronous(["/Game"], …)` + `wait_for_completion()` **first** or
`load_asset` returns `None` in a commandlet; (3) `unreal.Landscape` has no `post_edit_change()`;
(4) `save_asset(level, only_if_is_dirty=False)` force-saves; (5) `print`/Display don't reach stdout (only
Error) and the **exit code is 1 even on success** — trust a result file you write, not the exit code.

GeometryScript mesh edits (scaling/re-pivoting baked meshes) are the same class of heavy op: defer to a
post-tick callback, process **≤2 meshes per tick** with a **busy re-entry guard** (the synchronous
`copy_mesh_to_static_mesh` build re-enters the post-tick callback → multi-bake corruption + crash), and
`print` after each. `git checkout HEAD -- <assets>` restores a clean baseline if a bake corrupts them.

---

## 10. VibeUE evil cases still present on 5.8

- **`WidgetService.bind_event` is a silent NO-OP** (returns `True`, creates zero nodes). Use
  `add_delegate_bind_node` + `add_get_variable_node` + `add_create_delegate_node` instead. Param gotcha:
  `target_class="Button"` works (real GUID), `"UButton"` returns empty/FAILS despite the docstring.
- **VibeUE can corrupt mid-session** (SystemError on every call) — finish the task **native-only**
  (`UpdateNode` + `ExecuteGraphInstance`).
- See §4 for the Python-binding gaps that also bite through VibeUE.

---

## 11. VibeUE on 5.8 — install & licensing

VibeUE ships a **5.8-compatible build** — install it as a normal plugin (drop it under the project's
`Plugins/` folder, enable it in the `.uproject`, restart the editor). **No source porting is needed on
5.8**: you install the 5.8 build, you do not compile the old 5.7 plugin. The license key goes in
`Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini` and is validated online at server start —
so the editor needs network access the first time VibeUE starts up.

---

## 12. Verification discipline — the user is the visual oracle (GLOBAL rule)

"Done means verified": after each build step, re-read / compile / enumerate before moving on. But split
verification by kind:

- **Programmatic checks are yours to run and trust:** `GetNodeInfo` (`paramOverrides` stuck?),
  `GetNodeDataView` (non-zero `element_N`?), `user_parameters.export_text()` (param defaults non-zero?),
  enumerating components/instances (counts, positions, Z≈0, min-distances). Do these yourself.
- **Visual verdicts are the USER's, always.** For ANY judgment about how the viewport *looks* — is the
  geometry right, what material/color is showing, is there z-fighting or flicker, does the road follow
  the spline — do **not** conclude from your own reading of a `CaptureViewport`/screenshot. Capture the
  frame, then **ask the user and let them decide.** Don't treat your own screenshot reading as the
  verification gate; it is input for the user's call.

---

## 13. A typical PCG workflow shape (principles, not a fixed recipe)

Most procedural-environment builds follow the same arc, and the reliable structure is the same regardless
of theme: a base ground material assigned to the landscape; spline-driven features (roads/rivers/walls)
read **by tag** through `Get Spline Data` (never by hard pointer) so consumers stay decoupled;
scatter/buildings authored as **subgraphs merged into the spline graph** so they auto-regenerate when the
spline moves; clearance/exclusion via `Distance` → `Filter` against sampled spline points; and any
"mask one system by another" need (e.g. grass off the road) solved with the RVT pattern (§7). Externalize
the tuning knobs (clearance radii, cell sizes, densities) as graph params surfaced to the volume actor so
they're editable without touching nodes. Build each layer, verify it (§12), then move on.

---

## 14. Blueprint & level-edit hard-won lessons (quiz-wall / ocean session)

These are reflections from a real failure where quiz-wall "doors" disappeared and I burned a lot of
effort on the wrong diagnosis. Read them before touching any Blueprint actor in a level.

1. **When a BP actor's components/visuals ALL vanish, suspect a COMPILE ERROR first — not "SCS
   corruption".** Symptom seen: every `BP_QuizWall` instance dropped to a lone `BillboardComponent`
   (the whole door component tree gone) on every fresh load/spawn; the doors had only been surviving in
   the previous session's *memory*. Real cause: the Blueprint was in **`BS_ERROR`**. A `Call Function`
   node (`AddScore`, calling another BP) still referenced a **renamed/removed parameter pin**
   ("Passed Wall"). A BP that fails to compile never builds its SCS into the generated class / CDO, so
   **every** component fails to construct on every instance. Diagnose FAST:
   `bp.get_editor_property("status")` (`== BlueprintStatus.BS_ERROR`?), the bool return of
   `compile_blueprint` (False), and the compiler log in `Saved/Logs/*.log`
   (`LogBlueprint: Error: [AssetLog] … [コンパイラ] …` — it names the node and pin). Fix: refresh the
   offending node so its pins re-sync to the current signature (`RefreshNode`, or right-click →
   "Refresh Nodes"). **Whenever you rename/remove a BP function's parameter, every CALLER node keeps a
   stale pin and must be refreshed**, or the caller BP won't compile.

2. **`gather_subobject_data_for_blueprint` returns each component TWICE — that is NORMAL.** A healthy
   2-component test BP gathers ~6 handles (CDO + root + 2×2). Do NOT read this as "duplicate SCS nodes".
   Before treating any count/shape as a bug, **measure a known-healthy reference** (a throwaway fresh BP)
   and compare. Pin down the root cause (compiler log) BEFORE any destructive fix.

3. **Heavy BP structural surgery resets placed instance transforms.** Deleting + re-adding components,
   swapping BP versions, and repeated recompiles reset all 18 manually-placed wall instances to (0,0,0)
   (they stacked at the origin → looked like a single wall). If a BP has hand-placed instances in a
   level, AVOID structural edits to it; if unavoidable, **record every instance's transform first** and
   re-verify positions after the recompile.

4. **Verify the ACTUAL game state visually before saving/committing — not just the thing you fixed.** I
   confirmed the doors were back, then saved the level and pushed — while every wall was stacked at the
   origin (only caught when the user pointed it out, and the bad state was already pushed). Before
   `save_current_level` + commit/push: enumerate actor counts AND positions, and capture a WIDE shot of
   the real layout. "Done" includes the surrounding state, not only the symptom you targeted.

5. **Deleting an asset triggers BP-actor reconstruction across the whole level.** Deleting an unused
   material via the content browser reconstructed the level's BP actors, which re-ran the broken compile
   and wiped the in-memory doors. Don't delete assets while the level holds a BP that is in an
   unstable / uncompiled state — fix the BP first, then clean up.

6. **Don't run destructive "fixes" on a misdiagnosis.** I rebuilt the entire component tree (delete-all +
   re-add) chasing a phantom "duplication" bug; it was wasted and risky because the real fault was a
   one-node compile error. Cheap diagnosis (status + compiler log + a healthy reference) first;
   destructive surgery only after the cause is proven.

7. **Ocean: a stylized opaque plane beats the Water plugin for a thin elevated causeway.** Surrounding a
   narrow raised deck with `WaterBodyOcean` shows a view-dependent disc from above (near tessellated
   water shows the bottom through translucency; the far mesh reflects sky) that reads as a "hole", and
   seafloor-depth tuning does NOT stably remove it. Its default Gerstner waves are huge
   (`get_max_wave_height` ≈ 508 uu) and wash over low decks. For a uniform, predictable stylized sea,
   use a **single large flat StaticMesh plane + an opaque stylized water material** (two panning tiling
   normals + Fresnel-blended base color + low roughness; set the material **`two_sided`** so it never
   disappears when the camera dips below the surface). No depth/transparency artifacts, identical from
   every angle. Reserve the Water plugin for cases that genuinely need shoreline/waves and suit it.

8. **Back up an asset before risky edits; restoring from git needs the file unlocked.**
   `git checkout HEAD -- <asset>` fails with "unable to unlink" while the editor holds the lock — close
   the editor first — and the editor only picks up the on-disk version after a restart (loaded packages
   are cached). Also note the only committed version may already contain the bug, so keep a scratch copy
   of the *working* asset before destructive edits.
