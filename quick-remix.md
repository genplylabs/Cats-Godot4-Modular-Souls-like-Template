# quick-remix.md — cats-souls (vibe coding cheat sheet)

One-file knowledge base so ANY agent can remix this project with the godot-vibe MCP.
Project root: `/src/remix-hub/godot/cats-souls` · Godot 4.6 · 3D · entry scene `res://demo_level/world_castle.tscn`.

---

## 1. What this project is

Third-person cat action demo: one castle level, controllable cat player (movement,
melee, items, interactables), an enemy with a health bar, a UI HUD (health, item slot,
death card, touch controls), and a background music track. No menus, no level flow —
`world_castle.tscn` is the whole game.

Live/derived copies exist under `/src/projects/<user>/godot/workspace/cats-souls`.
**Remix the copy the MCP tool reports, never the hub source directly.**

## 2. Hard rules (violating these = failed remix)

1. **Scene is the source of truth.** Model/texture/sound slots are discovered by
   reading the `.tscn` files. There is no slot database to edit and no registry to keep
   in sync.
2. **Never full-rebuild.** `project_rebuild(name, full=false)` only (incremental packs).
   `full=true` runs `--import` and takes minutes — only on explicit user order.
3. **Never fall back to a full build silently.** If an incremental rebuild cannot fix a
   problem, STOP and report the reason instead of proceeding.
4. **Never author `res://` paths yourself.** Paths come back from tool output
   (`asset_refs`, `library_use`, `model_report`, `bind_material`).
5. **No guessing.** No blind `edit_lines`/`write_project_file` edits to swap assets —
   repeated blind fixes are the documented failure mode. Diagnose first.
6. **One path convention:** paths in MCP arguments are project-relative
   (`game/...`), never `workspace/cats-souls/...` and never absolute.
7. **No `library_add`.** Asset registration is the admin's job; agent tools are
   `library_list` and `library_use` only.
8. **Proof required.** Never claim a swap worked without `capture_preview` evidence
   (see §7).

## 3. Where every asset lives (project-relative paths)

| Role | Location | Notes |
|---|---|---|
| 3D character models (glb) | `assets/characters/*.glb` | `mannyquin.glb`, `minnyquinn.glb`, `minnyquinn_01.glb` … pets under `assets/pets/` |
| Weapon models | `assets/templateweapons.glb` | Mesh referenced by `player/equipment_system/equipment/*.tscn` |
| UI icon textures | `assets/icons/*.png`, `assets/tex/` | item icons, panel borders |
| Level music | `audio/Bone_In_The_Walls.ogg` | played by `Music` AudioStreamPlayer in the level |
| SFX | `audio/SoundFX/*.ogg` | plus `audio/Music/` |
| Audio buses | `audio/default_bus_layout.tres` | |
| Gridmap meshes | `assets/gridmap_assets/` | used by `demo_level/gridmap` |
| HUD / touch UI | `ui/*.tscn`, `ui/touchscreen/` | `health_bar`, `death_card`, `item_slot`, `contol_card`, `touch_control` |
| Enemy scene | `enemy/enemy_base_root_motion.tscn` + `enemy/enemy.gd` | has its own `health_bar.gd` |
| Player scene | `player/player_charbody3d.tscn` + `player/player_charbody3D.gd` | model node, animation tree, equipment |
| Equipment scenes | `player/equipment_system/equipment/{sword,Ax}.tscn` | |
| Items | `player/item_system/items/{potion,firebomb}.tscn` + `.tres` | icon textures inside `.tres` |
| Animation libs | `player/animation_libraries/` | `MannyAnimations.zip` pack in project root |
| Main level | `demo_level/world_castle.tscn` | instances player, camera, HUD, interactables |
| Interactables | `interactable objects/*` | `spawn_site`, `torch`, … |

Everything is wired by `ext_resource` inside `.tscn` files — grep a `.tscn` for a path
to find every consumer of an asset.

## 4. How to swap a MODEL (player / enemy / weapon)

The tool resolves the slot from the scene itself — do not pass `target`.

```
1. asset_refs(project="cats-souls", query="<human name>")
   → returns families + `next.example` (the exact call to make)
2. library_use(project="cats-souls", ref="<asset>",
               intent="scene", overwrite=true, wait=true)
   → one call; `target` stays empty; tool sniffs GLB vs FBX and rewrites the .tscn
3. capture_preview(project="cats-souls", seconds=0, fps=1)   → proof
```

Guaranteed rejections (read the error, do not retry blindly):
- `exists | fix: overwrite` → re-issue with `overwrite=true`
- `not allowed | fix: one of {model,mesh}` → wrong target; use `asset_refs` first
- `asset is audio/texture — model slot takes FBX/GLB only` → you picked the wrong ref

Response fields that prove the write: `repointed`, `res_path`, `uid`,
`artifact_changed: true`. If `artifact_changed` is false the scene was not touched —
do not report success.

## 5. How to swap a TEXTURE / SOUND (non-model assets)

```
1. asset_refs(project="cats-souls", query="<name>")
   → family rows: target res path, consumption counts, swap example
2. upload_asset / import_staged_asset if the new file is not in the library yet
3. library_use(project="cats-souls", ref="<asset>",
               intent="scene", overwrite=true, wait=true)
4. capture_preview(...) → proof
```

Same contract as §4; non-model slots accept any resource kind the family allows.

## 6. Scene edits (only when a swap is not enough)

```
read_scene_tree(project="cats-souls", path="demo_level/world_castle.tscn")
add_node / modify_node / delete_node     → target/parent in PROJECT-RELATIVE path
edit_lines(path="...", find=..., replace=...)   → pin-exact, single-line edits
write_project_file(rel_path="cats-souls/...", content=...)  → whole file; rel_path MUST start with `cats-souls/`
check_script(path="...")  → syntax check after any .gd edit
project_rebuild(project="cats-souls", full=false)  → ONLY incremental
```

## 7. Verification loop (mandatory after any change)

```
capture_preview(project="cats-souls", seconds=0, fps=1)   → base64 JPEG + full_url
get_runtime_errors(project="cats-souls")                  → script/runtime errors
get_editor_url()                                          → open the editor
```

- `seconds=0` returns a still frame; it needs a live polling tab for that user.
- For a model/texture swap, capture AFTER `wait=true` returned.
- 10-swap acceptance rule (platform-wide): each swap's capture must differ from the
  previous one. If an issue appears, restart numbering from swap 1 and fix first.
- `scene_tree`, `list_project`, `read_scene_tree`, `capture_preview`, `model_report`
  are READ-ONLY lane — safe anytime.

## 8. The knowledge base (read these when stuck)

| Topic | File |
|---|---|
| Solved startup/swap cases, do-not-repeat list | `/src/godot-vibe/solved-case.md` |
| One-call `library_use` contract | `/src/godot-vibe/docs/papertown-template-verify-plan.md` (tool table) |
| Fast 3-call swap flow | `/src/godot-vibe/.kilo/plans/1791852111283-asset-refs-fast-swap-plan.md` |
| Tool signatures (always read before trusting a doc) | `/src/godot-vibe/server/server.py` |
| Agent mistake log | `/src/godot-vibe/help-this-fuskin-glm-5.3-idocity.md` |
| Asset slot contract / address rule | `/src/godot-vibe/.kilo/plans/1791888174000-godot-3d-asset-tools-plan.md` |

## 9. Known pitfalls for THIS project

- `player/player_charbody3d.tscn` (lowercase `d`) is the real scene file — do not
  guess case variants.
- The enemy scene (`enemy/enemy_base_root_motion.tscn`) carries its own
  `health_bar.gd` reference; swapping the enemy model must keep the skeleton/root
  motion import settings from `assets/characters/*.glb.import`.
- `templateweapons.glb` is referenced from the equipment `.tscn` files, not from a
  script — a model swap rewrites those scenes.
- The level's `Music` node plays `audio/Bone_In_The_Walls.ogg` by ext_resource id;
  sound swaps must target that node, not an autoload.
