# Component Method Call Compilation Support

## Overview

The component method calling infrastructure has been fully implemented at the macro, reflection, and UI levels, but runtime compilation and execution support is still needed. Component method nodes appear in the blueprint editor palette and can be added to graphs, but they cannot yet be compiled and executed.

## Current State

### ✅ Completed Infrastructure

1. **Reflection System** - All metadata types and registry support for component methods
   - `MethodMetadata`, `MethodParameter`, `MethodReturnType`, `MethodType` types defined
   - `ComponentMethodRegistration` for inventory-based auto-discovery
   - `EngineClassRegistry::get_methods()` and `get_method()` for lookup
   - Location: `Pulsar-Native/crates/pulsar_reflection/`

2. **Macro System** - Automatic code generation for exposing methods
   - `#[component_methods]` attribute macro on impl blocks
   - `#[method(type: MethodType::Fn/Pure)]` on individual methods
   - Auto-generated property getter/setter methods from `#[property]` fields
   - Generated caller closures with proper downcasting and type conversion
   - Location: `Pulsar-Native/crates/engine_class_derive/src/lib.rs`

3. **Blueprint Editor UI** - Node generation and palette integration
   - `NodeDefinitions::generate_component_nodes()` - Creates bound method nodes per component instance
   - `property_type_to_data_type()` - Converts reflection types to blueprint types
   - Palette integration in `NodePaletteView` - Dynamically adds component nodes when prefab loaded
   - Node ID format: `method_{class_name}_{method_name}_{instance_index}`
   - Location: `Plugin_Blueprints/src/core/definitions.rs` and `Plugin_Blueprints/src/ui_components/palette_view.rs`

### ❌ Missing: Runtime Compilation & Execution

Component method nodes can be placed in graphs but cannot be compiled or executed because:

1. **PBGC doesn't recognize component method nodes** - They're not registered in `pulsar_std` like regular blueprint nodes
2. **No dispatch functions** - Component methods don't have `__bp_dispatch_*` symbols registered in the global function table
3. **No component instance access** - Blueprints don't have a mechanism to access component instances at runtime
4. **Bytecode VM doesn't support component calls** - The VM has no instructions for calling component methods via reflection

## Technical Challenge

The existing blueprint compilation system expects all nodes to be statically registered in `pulsar_std` via the `#[blueprint]` macro. Component method nodes are dynamically generated per-prefab, creating a fundamental mismatch:

- **Static nodes**: `add`, `subtract`, `print` - registered at compile-time with fixed dispatch functions
- **Dynamic component nodes**: `method_PhysicsComponent_apply_impulse_0` - unique per blueprint, bound to specific component instances

## Required Implementation

### Phase 1: PBGC Compiler Extension

**Goal**: Recognize and handle component method nodes during compilation

**Location**: `PBGC/src/compiler.rs` and related files

**What needs to be done**:

1. **Detect component method nodes** during compilation
   ```rust
   // In graph compilation phase
   if node.node_type.starts_with("method_") {
       // Extract: class_name, method_name, instance_index
       let parts: Vec<&str> = node.node_type.strip_prefix("method_").unwrap().split('_').collect();
       let instance_index = parts.last().unwrap().parse::<usize>().unwrap();
       let method_name = parts[parts.len() - 2];
       let class_name = parts[0..parts.len() - 2].join("_");

       // Handle as component method call instead of regular node
       compile_component_method_call(class_name, method_name, instance_index, node);
   }
   ```

2. **Generate special bytecode** for component method calls
   - New `Instruction::CallComponentMethod` variant
   - Store component instance index, class name, and method name
   - Marshal arguments from blueprint arena to `PropertyValue` vector
   - Call via `MethodMetadata.caller` closure from registry
   - Marshal return value back to arena

3. **Add component metadata to compilation context**
   ```rust
   pub struct CompilationContext {
       // ... existing fields
       component_instances: Vec<ComponentInstanceInfo>,  // NEW
   }

   pub struct ComponentInstanceInfo {
       index: usize,
       class_name: String,
       serialized_data: serde_json::Value,  // From prefab.json
   }
   ```

**Files to modify**:
- `PBGC/src/compiler.rs` - Main compilation logic
- `PBGC/src/instruction.rs` - Add `CallComponentMethod` instruction
- `PBGC/src/metadata.rs` - Extend metadata provider to include component info
- `PBGC/src/graph.rs` - Add component instance context to GraphDescription

### Phase 2: Bytecode VM Extension

**Goal**: Execute component method call instructions

**Location**: `PBGC/src/vm.rs` and `pulsar_bp_executor`

**What needs to be done**:

1. **Extend VM state** to hold component instances
   ```rust
   pub struct VMState {
       // ... existing fields
       component_instances: Vec<Box<dyn EngineClass>>,  // NEW
   }
   ```

2. **Implement `CallComponentMethod` instruction execution**
   ```rust
   Instruction::CallComponentMethod {
       component_index,
       method_name,
       arg_offsets,
       ret_offset
   } => {
       // 1. Get component instance
       let component = &mut vm_state.component_instances[component_index];

       // 2. Look up method metadata
       let class_name = component.class_name();
       let method = pulsar_reflection::REGISTRY
           .get_method(class_name, method_name)
           .expect("Method not found");

       // 3. Marshal arguments from arena to PropertyValue
       let args: Vec<PropertyValue> = arg_offsets.iter().map(|offset| {
           // Read from arena at offset, convert to PropertyValue based on param type
           read_property_value_from_arena(arena, *offset, &method.params[i].param_type)
       }).collect();

       // 4. Call method via reflection
       let result = (method.caller)(component.as_mut(), args);

       // 5. Write return value to arena
       if let Some(ret) = result {
           write_property_value_to_arena(arena, ret_offset, ret);
       }
   }
   ```

3. **Add arena marshalling helpers**
   ```rust
   fn read_property_value_from_arena(
       arena: &[u8],
       offset: usize,
       prop_type: &PropertyType
   ) -> PropertyValue {
       match prop_type {
           PropertyType::F32 { .. } => {
               let value = unsafe { *(arena.as_ptr().add(offset) as *const f32) };
               PropertyValue::F32(value)
           },
           PropertyType::Vec3 => {
               let value = unsafe { *(arena.as_ptr().add(offset) as *const [f32; 3]) };
               PropertyValue::Vec3(value)
           },
           // ... other types
       }
   }

   fn write_property_value_to_arena(
       arena: &mut [u8],
       offset: usize,
       value: PropertyValue
   ) {
       match value {
           PropertyValue::F32(v) => unsafe {
               *(arena.as_mut_ptr().add(offset) as *mut f32) = v;
           },
           // ... other types
       }
   }
   ```

**Files to modify**:
- `PBGC/src/vm.rs` - VM execution loop
- `PBGC/src/instruction.rs` - Add new instruction type
- `pulsar_bp_executor/src/lib.rs` - Extend `BpExecutor` to accept component instances

### Phase 3: Blueprint Editor Integration

**Goal**: Pass component instances to blueprint execution

**Location**: `Plugin_Blueprints/src/features/compilation/compiler.rs`

**What needs to be done**:

1. **Extract component instances from prefab** before compilation
   ```rust
   impl BlueprintEditorPanel {
       pub fn compile_to_bytecode(&self) -> Result<Vec<pbgc::BpProgram>, String> {
           // Existing variable extraction
           let variables: HashMap<String, String> = self.class_variables
               .iter()
               .map(|v| (v.name.clone(), v.var_type.clone()))
               .collect();

           // NEW: Extract component instances
           let component_instances = self.prefab_asset.as_ref()
               .map(|prefab| extract_component_instances(prefab))
               .unwrap_or_default();

           let graph = self.build_graphy_description()?;

           // Pass components to compiler
           pbgc::compile_graph_to_bytecode_with_variables_and_components(
               &graph,
               variables,
               component_instances  // NEW
           ).map_err(|e| format!("Bytecode compilation failed: {}", e))
       }
   }

   fn extract_component_instances(prefab: &PrefabAsset) -> Vec<Box<dyn EngineClass>> {
       prefab.components.iter().map(|comp_instance| {
           // Deserialize component data and create instance
           let instance = pulsar_reflection::REGISTRY
               .create_instance(&comp_instance.class_name)
               .expect("Component class not registered");

           // Hydrate instance from serialized data
           hydrate_component_from_json(instance, &comp_instance.data);

           instance
       }).collect()
   }
   ```

2. **Pass component instances to VM execution**
   ```rust
   pub fn execute_bytecode_programs(
       &self,
       programs: Vec<pbgc::BpProgram>,
   ) -> Result<(), String> {
       // ... existing setup

       // NEW: Extract and pass component instances
       let component_instances = self.prefab_asset.as_ref()
           .map(|prefab| extract_component_instances(prefab))
           .unwrap_or_default();

       state.runtime
           .as_mut()
           .ok_or_else(|| "Bytecode runtime failed to initialize".to_string())?
           .execute_for_class_with_components(
               class_key,
               programs,
               component_instances  // NEW
           )
   }
   ```

**Files to modify**:
- `Plugin_Blueprints/src/features/compilation/compiler.rs` - Add component extraction and passing
- `Plugin_Blueprints/src/features/prefabs/mod.rs` - Add component deserialization helpers

### Phase 4: Rust Code Generation (Alternative Path)

**Goal**: Generate Rust code that calls component methods

**Location**: `PBGC/src/codegen.rs` (if using Rust backend instead of bytecode)

**What needs to be done**:

1. **Generate component accessor code**
   ```rust
   // For each component in prefab, generate accessor
   let physics_component_0: &mut PhysicsComponent = get_component_mut(0);
   ```

2. **Generate method call code**
   ```rust
   // For component method node
   physics_component_0.apply_impulse([force_x, force_y, force_z]);
   ```

3. **Handle component state persistence**
   - Components need to persist between blueprint executions
   - Generate code to serialize/deserialize component state

**Files to modify**:
- `PBGC/src/codegen.rs` - Rust code generation
- Blueprint Rust backend integration

## Implementation Order

1. **Start with bytecode VM path** (simpler, more self-contained)
   - Phase 2: Bytecode VM Extension
   - Phase 1: PBGC Compiler Extension
   - Phase 3: Blueprint Editor Integration

2. **Then add Rust codegen path** (if needed)
   - Phase 4: Rust Code Generation

## Testing Requirements

### Unit Tests

1. **PBGC Compiler Tests**
   - Test component method node detection
   - Test bytecode generation for component calls
   - Test marshalling of various parameter types

2. **VM Execution Tests**
   - Test `CallComponentMethod` instruction
   - Test arena marshalling (PropertyValue ↔ arena bytes)
   - Test component instance access

3. **Integration Tests**
   - Create test component with various method signatures
   - Compile graph with component method calls
   - Execute and verify methods are called with correct arguments

### End-to-End Test

1. Create `TestPhysicsComponent` with methods:
   ```rust
   #[derive(EngineClass, Default, Clone)]
   pub struct TestPhysicsComponent {
       #[property]
       pub velocity: [f32; 3],
   }

   #[component_methods]
   impl TestPhysicsComponent {
       #[method(type: MethodType::Fn)]
       pub fn apply_force(&mut self, force: [f32; 3]) {
           self.velocity[0] += force[0];
           self.velocity[1] += force[1];
           self.velocity[2] += force[2];
       }

       #[method(type: MethodType::Pure)]
       pub fn get_speed(&self) -> f32 {
           (self.velocity[0].powi(2) +
            self.velocity[1].powi(2) +
            self.velocity[2].powi(2)).sqrt()
       }
   }
   ```

2. Create blueprint with:
   - `TestPhysicsComponent` in prefab
   - Graph that calls `apply_force` with constant vector
   - Graph that calls `get_speed` and prints result

3. Compile and execute:
   - Verify `apply_force` modifies velocity
   - Verify `get_speed` returns correct value
   - Verify state persists between executions

## Technical Considerations

### Memory Safety

- Component instances must remain alive for entire blueprint execution
- Use `Box<dyn EngineClass>` to maintain proper ownership
- Ensure arena pointers don't outlive the arena

### Type Safety

- Validate method exists before compilation
- Validate argument types match method signature
- Handle type conversion errors gracefully

### Performance

- Component instance lookup should be O(1) via index
- Method metadata lookup cached in registry
- Minimal overhead vs direct method calls

### Error Handling

- Clear error messages when:
  - Component not found in prefab
  - Method not found on component
  - Type conversion fails
  - Downcast fails

## Success Criteria

- [ ] Component method nodes compile without errors
- [ ] Bytecode VM can execute component method calls
- [ ] Methods are called with correct arguments
- [ ] Return values are properly marshalled back to graph
- [ ] Component state persists between blueprint executions
- [ ] Multiple component instances can be called independently
- [ ] Property getter/setter methods work correctly
- [ ] Custom methods with various signatures work correctly

## Related Files Summary

### Pulsar-Native (Reflection & Macros)
- ✅ `crates/pulsar_reflection/src/lib.rs` - Method metadata types
- ✅ `crates/pulsar_reflection/src/registry.rs` - Registry lookup
- ✅ `crates/engine_class_derive/src/lib.rs` - Macro implementation

### PBGC (Compiler)
- ❌ `src/compiler.rs` - Detect and compile component method nodes
- ❌ `src/instruction.rs` - Add `CallComponentMethod` instruction
- ❌ `src/vm.rs` - Execute component method instructions
- ❌ `src/metadata.rs` - Add component instance context

### Plugin_Blueprints (Editor)
- ✅ `src/core/definitions.rs` - Node generation (complete)
- ✅ `src/ui_components/palette_view.rs` - Palette integration (complete)
- ❌ `src/features/compilation/compiler.rs` - Pass component instances to execution

### pulsar_bp_executor
- ❌ `src/lib.rs` - Accept and manage component instances

## Notes

- The reflection and UI infrastructure is production-ready
- All component method metadata is available at compile-time
- The main challenge is bridging the dynamic nature of component nodes with the static blueprint compilation system
- Consider using a "component node adapter" pattern that wraps dynamic component calls in a static interface
