# Pulsar Native — Cleanup & Systems Report

Date: May 20, 2026

Each top-level section below is a **site** (a location or area of the codebase).
The details under each site describe what is wrong, what the better alternative is, and what to do.
Scan the headings to get the full picture fast; open a section for the specifics.

---

## Sites Index

- [Level Editor — Inspector / Properties Panel](#level-editor--inspector--properties-panel)
- [Level Editor — Scene Database](#level-editor--scene-database)
- [Level Editor — Viewport Panel](#level-editor--viewport-panel)
- [Level Editor — World Settings](#level-editor--world-settings)
- [Level Editor — Component Hierarchy](#level-editor--component-hierarchy)
- [Level Editor — Playback Toolbar](#level-editor--playback-toolbar)
- [File Manager — Item Rendering](#file-manager--item-rendering)
- [Reflection Crate](#reflection-crate)
- [Engine FS — Path Tooling](#engine-fs--path-tooling)
- [Engine Backend — Helio Renderer](#engine-backend--helio-renderer)
- [Engine Backend — Networking](#engine-backend--networking)
- [UI Common — Asset Picker](#ui-common--asset-picker)
- [UI Crate — Menu and Input](#ui-crate--menu-and-input)
- [UI Crate — Settings](#ui-crate--settings)
- [UI Plugin Manager](#ui-plugin-manager)
- [Agent Chat Tools](#agent-chat-tools)
- [Script / Bootstrap](#script--bootstrap)

---

## Level Editor — Inspector / Properties Panel

**Files**
- [ui-crates/ui_level_editor/src/level_editor/workspace_panels.rs](ui-crates/ui_level_editor/src/level_editor/workspace_panels.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/properties_panel.rs](ui-crates/ui_level_editor/src/level_editor/ui/properties_panel.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/bound_field.rs](ui-crates/ui_level_editor/src/level_editor/ui/bound_field.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/field_bindings.rs](ui-crates/ui_level_editor/src/level_editor/ui/field_bindings.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/object_header_section.rs](ui-crates/ui_level_editor/src/level_editor/ui/object_header_section.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/transform_section.rs](ui-crates/ui_level_editor/src/level_editor/ui/transform_section.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/material_section.rs](ui-crates/ui_level_editor/src/level_editor/ui/material_section.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/object_type_fields_section.rs](ui-crates/ui_level_editor/src/level_editor/ui/object_type_fields_section.rs)

**What is here**

Two parallel property editing systems currently live in the same inspector surface.

Old system — BoundField + FieldBinding:
- Each property type (F32, String, Bool, Vec3) has its own wrapper struct that creates an input entity and subscribes to input events directly.
- Used today in ObjectHeaderSection (name, visibility, lock), TransformSection (9 position/rotation/scale axes), and MaterialSection.
- `workspace_panels.rs` explicitly marks the old manual path as `DEPRECATED: Old manual property editing (will be removed)`.
- `properties_panel.rs` has an old section block commented out with `TODO: convert to binding system`.

New system — component detail rendering:
- Properties are driven by reflection metadata via the pulsar_reflection registry.
- `ObjectTypeFieldsSection` reads `ComponentInstance` objects from `SceneDatabase`, queries their properties via `get_properties()`, and renders controls per `PropertyType`.
- Write path flows through `update_component` / `update_component_property` on `SceneDatabase`.
- This path is already active and is explicitly protected from changes in `REFACTOR_PLAN.md`.

**Decision**

Replace the BoundField path fully. Once object core, transform, and material properties are represented as metadata-backed components, the bound_field, field_bindings, and the three BoundField-based sections can be deleted.

**Priority** — High

---

## Level Editor — Scene Database

**Files**
- [ui-crates/ui_level_editor/src/level_editor/scene_database.rs](ui-crates/ui_level_editor/src/level_editor/scene_database.rs)

**What is here**

- Two sibling reorder stubs are marked `TODO: implement when SceneDb exposes sibling reordering` at lines 332–337. These are currently no-ops.
- The file is also the central write path that should be the only way component data is modified; two paths (BoundField and component detail) currently both try to write through it, which should converge to one after the inspector migration above.

**Decision**

No structural change needed. Unblock the reorder stubs when `SceneDb` gains that capability. The write-path convergence follows naturally from the inspector migration.

**Priority** — Low (reorder) / Follows inspector migration

---

## Level Editor — Viewport Panel

**Files**
- [ui-crates/ui_level_editor/src/level_editor/ui/panel.rs](ui-crates/ui_level_editor/src/level_editor/ui/panel.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/viewport/helio_viewport.rs](ui-crates/ui_level_editor/src/level_editor/ui/viewport/helio_viewport.rs)
- [ui-crates/ui_level_editor/src/level_editor/ui/viewport/mod.rs](ui-crates/ui_level_editor/src/level_editor/ui/viewport/mod.rs)

**What is here**

- A debug toggle `debug_replace_with_yellow` is wired through panel creation to the viewport. Currently hardcoded to `false`. This is development scaffolding and should be removed when the viewport is considered stable.
- Two open TODOs:
  - `TODO: modal dialog` when warning about unsaved changes on close.
  - `TODO: Frame selected object in viewport` (focus camera on selection using F key).
- The viewport module comment says it has already been refactored into focused reusable components, so no structural split is needed here.

**Decision**

Remove the yellow debug toggle. Track the modal dialog and frame-on-select TODOs as feature tasks.

**Priority** — Low (toggle cleanup immediate, features are tracked work)

---

## Level Editor — World Settings

**Files**
- [ui-crates/ui_level_editor/src/level_editor/world_settings_data.rs](ui-crates/ui_level_editor/src/level_editor/world_settings_data.rs)

**What is here**

- `TODO: Wire up to actual renderer/physics/audio systems` — world settings data changes are not yet connected to runtime systems.
- `TODO: Convert to actual RGBA if needed by renderer` — color values may need format conversion before passing to the renderer.

**Decision**

Both are feature gaps, not cleanup items. Track them as integration work when the renderer/physics/audio connection layer is being built.

Note: world settings will eventually be component-based through special `WorldComponent` types, following the same metadata-driven path as scene object properties. The current `world_settings_data.rs` data model should be treated as transitional — the canonical home for these settings will be `WorldComponent` instances in the scene database.

**Priority** — Medium (integration-driven)

---

## Level Editor — Component Hierarchy

**Files**
- [ui-crates/ui_level_editor/src/level_editor/ui/component_hierarchy.rs](ui-crates/ui_level_editor/src/level_editor/ui/component_hierarchy.rs)

**What is here**

- Component hierarchy items have `selected: false` hardcoded with `TODO: Implement selection`. Selection state is not wired into the hierarchy display.

**Decision**

Implement selection tracking when component-level selection is needed for the inspector or other UI flows.

**Priority** — Medium

---

## Level Editor — Playback Toolbar

**Files**
- [ui-crates/ui_level_editor/src/level_editor/ui/toolbar/playback_controls.rs](ui-crates/ui_level_editor/src/level_editor/ui/toolbar/playback_controls.rs)

**What is here**

- `TODO: Implement pause` — the pause button exists in the toolbar but the action is not wired.

**Decision**

Feature gap. Implement pause action as part of playback control work.

**Priority** — Medium

---

## File Manager — Item Rendering

**Files**
- [ui-crates/ui_file_manager/src/file_manager_drawer/drawer_impl/render_content.rs](ui-crates/ui_file_manager/src/file_manager_drawer/drawer_impl/render_content.rs)
- [ui-crates/ui_file_manager/src/file_manager_drawer/drawer_impl/action_handlers.rs](ui-crates/ui_file_manager/src/file_manager_drawer/drawer_impl/action_handlers.rs)
- [ui-crates/ui_file_manager/src/drawer/mod.rs](ui-crates/ui_file_manager/src/drawer/mod.rs)

**What is here**

`render_grid_item` and `render_list_item` are two nearly identical rendering paths. They share:
- Variable extraction and icon loading
- Drag data setup
- All drag/drop event handlers
- Mouse event handlers (click, right-click, drag-move)
- Icon rendering with conditional styling
- Name display and rename input
- Context menu setup

They differ only in their outer wrapper: grid uses a card with fixed width/height, list uses a row with fixed height.

Additionally, `action_handlers.rs` has four stubbed actions marked TODO:
- Asset validation not implemented
- Favorite toggling not implemented
- Hidden file toggling not implemented
- File history viewer not implemented

And `drawer/mod.rs` has a commented-out content module with a lifetime issue note.

**Better approach already in repo**

`HierarchicalTreeView` in `hierarchical_list.rs` uses a `HierarchyLayout` enum (Panel/Widget) to dispatch shared logic to mode-specific wrappers. The same pattern applies here.

**Decision**

Merge grid/list into one shared item renderer with a `FileItemLayout` enum. Implement or formally stub out the four action TODOs.

**Priority** — High (duplication), Medium (stubs)

---

## Reflection Crate

**Files**
- [crates/pulsar_reflection/src/lib.rs](crates/pulsar_reflection/src/lib.rs)
- [crates/engine_class_derive/src/lib.rs](crates/engine_class_derive/src/lib.rs)

**What is here**

Two reflection mechanisms that look similar but serve different purposes:

`PropertyMetadata` / `get_properties()`:
- Driven by the `#[property]` derive macro.
- Used for editor UI generation — "what fields does this component expose?"
- Consumed by `ObjectTypeFieldsSection` and `AddComponentDialog`.

`ScenePropsProjector` / `apply_scene_props_for_class`:
- Maps component JSON data into flat `snap.props` entries for the renderer.
- Used by `LightComponent` and `StaticMeshComponent`.
- Consumed by `SceneDatabase` during snapshot generation.

**Decision**

Keep both. They operate at different layers and neither replaces the other. Add tests around `ScenePropsProjector` registration and projection correctness since that path has no test coverage.

**Priority** — Medium (tests)

---

## Engine FS — Path Tooling

**Files**
- [crates/engine_fs/src/tooling.rs](crates/engine_fs/src/tooling.rs)

**What is here**

Path normalization and containment logic is split into two parallel branches depending on whether the path is local or cloud-scheme. The functions `normalize_local_path`, `normalize_cloud_path`, `ensure_within_root`, and `rel_display` each manually dispatch local vs cloud logic.

No unified path abstraction trait exists yet.

**Decision**

Backlog refactor. Not urgent — defer until a cloud/local path bug or feature makes this code an active work area.

**Priority** — Medium (deferred)

---

## Engine Backend — Helio Renderer

**Files**
- [crates/engine_backend/src/subsystems/render/helio_renderer/renderer.rs](crates/engine_backend/src/subsystems/render/helio_renderer/renderer.rs)

**What is here**

- 1194 lines in a single module covering multiple rendering concerns.
- Top of the file has a section marked `Legacy types (unused but referenced by UI code)` and a mid-file section `Legacy (unused)`.
- The adjacent viewport UI module was explicitly refactored into focused reusable components, establishing the preferred direction for this kind of code.

**Decision**

Remove the two marked legacy blocks if they are genuinely unused. Do staged modular extraction by concern only while implementing adjacent rendering features — avoid a big-bang split.

**Priority** — Low (legacy removal immediate if safe, structural split deferred)

---

## Engine Backend — Networking

**Files**
- [crates/engine_backend/src/subsystems/networking/p2p.rs](crates/engine_backend/src/subsystems/networking/p2p.rs)
- [crates/engine_backend/src/subsystems/game_network/client/connection.rs](crates/engine_backend/src/subsystems/game_network/client/connection.rs)
- [crates/engine_backend/src/subsystems/networking/multiuser.rs](crates/engine_backend/src/subsystems/networking/multiuser.rs)
- [crates/multiuser_server/src/sync_protocol.rs](crates/multiuser_server/src/sync_protocol.rs)

**What is here**

- `p2p.rs` has a cluster of unimplemented stubs: STUN binding, candidate exchange, P2P transport for git, and multiuser client integration — all marked TODO.
- `connection.rs` has `conn_ref: Option<()>` with a TODO to replace with the actual connection reference type.
- `multiuser.rs` and `sync_protocol.rs` have `Legacy file-based messages` sections marked for backwards compatibility.

**Decision**

P2P stubs are feature work. Track separately when P2P implementation is active. Legacy message sections should be reviewed for whether the old wire format still needs to be supported or can be dropped.

**Priority** — Low/Medium (feature-driven)

---

## UI Common — Asset Picker

**Files**
- [ui-crates/ui_common/src/asset_picker.rs](ui-crates/ui_common/src/asset_picker.rs)

**What is here**

Multiple `println!` debug statements in live runtime code paths:
- In `MeshAssetPicker::new` — logs builtin count and each asset being added.
- In `query_assets` — logs project root, extension queries, and each matched file path.

These run on every asset picker initialization and query.

**Decision**

Remove all. If tracing is needed for diagnosis, convert to `tracing::debug!` behind a compile flag — do not leave raw prints in production paths.

**Priority** — Immediate

---

## UI Crate — Menu and Input

**Files**
- [crates/ui/src/menu/popup_menu.rs](crates/ui/src/menu/popup_menu.rs)
- [crates/ui/src/input/state/layout.rs](crates/ui/src/input/state/layout.rs)
- [crates/ui/src/input/popovers/completion_menu.rs](crates/ui/src/input/popovers/completion_menu.rs)
- [crates/ui/src/input/element.rs](crates/ui/src/input/element.rs)
- [crates/ui/src/input/search.rs](crates/ui/src/input/search.rs)

**What is here**

- `popup_menu.rs` has two FIXMEs: submenu unselection on hover-out is not properly handled, and overflow-scroll prevents submenus from displaying.
- `layout.rs` has a FIXME: double-clicking a non-word character does not select the word correctly.
- `completion_menu.rs` has a FIXME: the filter input does not receive focus.
- `element.rs` has a TODO for a performance fix: line cache is not used in a hot path.
- `input/search.rs` has a FIXME: should use stream find but doesn't.

**Decision**

These are real bugs. Prioritize focus and selection FIXMEs (user-visible) over the performance TODO. Track as input system bug fixes.

**Priority** — Medium (user-visible bugs), Low (performance)

---

## UI Crate — Settings

**Files**
- [crates/ui/src/settings/mod.rs](crates/ui/src/settings/mod.rs)
- [ui-crates/ui_settings/src/settings/views/project.rs](ui-crates/ui_settings/src/settings/views/project.rs)
- [ui-crates/ui_settings/src/settings/views/advanced.rs](ui-crates/ui_settings/src/settings/views/advanced.rs)

**What is here**

- `settings/mod.rs` re-exports `EngineSettings` from a local module with a comment saying it should move to a shared crate to avoid duplication. The duplication exists today.
- `project.rs` has `TODO: Implement folder picker` — the folder path input in project settings has no browse button yet.
- `advanced.rs` has `TODO: Open extensions marketplace` — extensions button is not wired.

**Decision**

Move `EngineSettings` to the shared crate when settings are actively being refactored. Folder picker and extensions marketplace are feature gaps.

**Priority** — Low (settings move), Medium (feature gaps)

---

## UI Plugin Manager

**Files**
- [ui-crates/ui_plugin_manager/src/window.rs](ui-crates/ui_plugin_manager/src/window.rs)

**What is here**

- Plugin unload button calls `todo!()` with a message directing implementors to call `PluginManager`. This will panic at runtime if the unload button is clicked.

**Decision**

Either implement plugin unloading or disable/hide the unload button until it is implemented. A live `todo!()` panic is a user-visible crash.

**Priority** — High (crash risk)

---

## Agent Chat Tools

**Files**
- [agent-providers/agent_chat_tools/src/lib.rs](agent-providers/agent_chat_tools/src/lib.rs)

**What is here**

Three separate `OnceLock<Mutex<...>>` globals: `RUNTIME_STATE`, `SUBAGENT_STORE`, and `TASK_MANIFEST`. Each has its own locking, getter, and mutator functions with repeated `get_or_init` and lock/unwrap boilerplate.

No unified context struct exists. The current approach is verbose but stable.

**Decision**

Consolidate into a single state context struct when this module is being modified for a feature. Not urgent.

**Priority** — Low/Medium (opportunistic)

---

## Script / Bootstrap

**Files**
- [script/bootstrap](script/bootstrap)

**What is here**

- The `bootstrap` script is marked as deprecated with a redirect message. It still exists, which can confuse new contributors who find it first.

**Decision**

Delete the file, or replace its content with a single-line error pointing to the correct script.

**Priority** — Low

---

## Execution Order Summary

| When | What |
|---|---|
| Now | Remove `println!` debug output from asset picker and object type fields section |
| Now | Handle plugin manager `todo!()` panic (implement or hide the button) |
| Soon | Complete inspector migration from BoundField to component detail, then delete BoundField modules |
| Soon | Merge file manager grid/list item renderer |
| Next | Add `ScenePropsProjector` tests |
| Ongoing | Fix user-visible input/menu FIXMEs as they come up |
| When touched | engine_fs path abstraction, agent_chat_tools state consolidation, helio renderer extraction |
| Low priority | Viewport debug toggle, bootstrap script cleanup, settings module move |
