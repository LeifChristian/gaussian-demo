# 3D Marker Overlay Demos

Custom demos built on top of [GaussianSplats3D](https://github.com/mkkellogg/GaussianSplats3D). These interactive demos showcase 3D spatial pinpointing and marker placement systems, exploring real-time 3D visualization, marker placement, distance measurement, and scene rendering using Three.js.

## Overview

These demos demonstrate two approaches to 3D marker placement:

1. **Gaussian Splat Overlay** - Overlays interactive markers on pre-trained 3D Gaussian Splat scenes (photorealistic reconstructions from images)
2. **Dynamic Room Environment** - A procedurally generated 3D room with object placement, manipulation, and persistent scene management

Both demos share a common marker system featuring pulsating markers, dashed connection lines, distance measurement, and precise coordinate positioning.

## Gaussian Splatting Integration

### What is Gaussian Splatting?

3D Gaussian Splatting is a technique for generating photorealistic 3D scenes from 2D images. Unlike traditional mesh-based 3D models, Gaussian Splatting represents scenes as millions of 3D Gaussian ellipsoids that can be rendered in real-time. Each splat stores position, color, opacity, rotation, and scale information.

### Integration Architecture

The demos use the `DropInViewer` class from GaussianSplats3D, which wraps the core `Viewer` and allows it to be integrated into a custom Three.js scene like any other renderable object:

```javascript
import * as GaussianSplats3D from '@mkkellogg/gaussian-splats-3d';
import * as THREE from 'three';

const scene = new THREE.Scene();
const viewer = new GaussianSplats3D.DropInViewer({
  'gpuAcceleratedSort': true
});

viewer.addSplatScene('path/to/scene.ksplat');
scene.add(viewer);
```

**Key Technical Details:**

- **Rendering Pipeline**: The Gaussian Splat renderer uses a custom WebGL shader pipeline that sorts splats by depth on each frame (CPU-based with optional GPU acceleration)
- **SharedArrayBuffer**: Requires CORS headers (`Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Embedder-Policy: require-corp`) for efficient worker communication
- **Coordinate System**: Splat scenes use their own coordinate space; markers are positioned in world space relative to the Three.js scene
- **Performance**: Splat sorting is the bottleneck; scenes with >8M splats may require tuning `splatSortDistanceMapPrecision` or disabling `integerBasedSort`

## Demo 1: Gaussian Splat Marker Overlay

**File:** `demo/marker-overlay.html`  
**URL:** `http://127.0.0.1:8080/marker-overlay.html`

Overlays 3D markers on top of Gaussian Splat scenes (pre-trained 3D reconstructions from photographs).

**Requires:** Downloaded `.ksplat` scene data (see Setup below).

### Features

- **Gaussian Splat scene rendering**
  - Uses `DropInViewer` to render `.ksplat` scenes inside a Three.js scene
  - Scene selector: Garden, Bonsai, Stump, Truck
  - Each scene has tuned camera positions and marker defaults

- **Marker system**
  - Two markers (A and B) with pulsating glow animation
  - Click-to-place projects a ray at configurable depth (since splat scenes have no traditional geometry to raycast against)
  - Adjustable place depth slider
  - Manual X/Y/Z coordinate inputs for precise positioning
  - Color pickers for each marker
  - Dashed connection line between markers with distance readout in meters
  - Configurable dash size, gap size, marker size, and pulse speed

- **Camera controls**
  - Orbit, pan, and zoom via mouse (OrbitControls)
  - Drag detection prevents accidental marker placement while orbiting

### Technical Implementation

**Marker Placement:**
Since Gaussian Splat scenes don't expose traditional mesh geometry for raycasting, markers are placed by projecting a ray from the camera through the mouse position at a user-defined depth:

```javascript
const raycaster = new THREE.Raycaster();
raycaster.setFromCamera(mouse, camera);
const depth = parseFloat(depthInput.value);
const position = raycaster.ray.origin.clone()
  .add(raycaster.ray.direction.multiplyScalar(depth));
```

**Rendering Integration:**
The `DropInViewer` renders into the same WebGL context as the Three.js scene, allowing markers and lines to be composited on top of the splat rendering. The renderer's depth buffer is shared, ensuring proper occlusion.

**Scene-Specific Configuration:**
Each scene has pre-configured camera positions and marker defaults stored in a configuration object, allowing quick switching between optimized views.

**Tech:** GaussianSplats3D DropInViewer, Three.js, ES modules via import maps.

---

## Demo 2: Dynamic Room with Object Placement

**File:** `demo/room-markers.html`  
**URL:** `http://127.0.0.1:8080/room-markers.html`

A pure Three.js 3D room environment with dynamic configuration, object placement, and spatial marker pinpointing. No Gaussian Splat data required.

### Features

- **Dynamic room generation**
  - Rectangular or rounded (cylindrical) room shapes
  - Width, depth, and height sliders (2-16m range) that rebuild the room in real time
  - White walls with soft shadow lighting, baseboards, and a visible ceiling light fixture

- **Object placement palette**
  - Six shape types: Cube, Sphere, Cylinder, Cone, Pyramid, Human (mannequin)
  - Click a shape, then click any surface in the room to place it
  - Configurable color and size before placement
  - Stays in placement mode for rapid multi-placement (Escape to exit)

- **Placed objects management**
  - List of all placed objects with color swatch and type label
  - Per-object delete button to remove from scene

- **Object manipulation**
  - Select objects by clicking in 3D view or from list
  - Drag-to-move along room surfaces (disables orbit controls during drag)
  - Precise XYZ coordinate editing
  - Scale adjustment (0.1x - 4.0x)
  - Rotation around X, Y, Z axes (-180° to 180°)

- **Marker system**
  - Same two-marker + dashed line system as the overlay demo
  - Click-to-place on any room surface or placed object (raycasts against geometry)
  - Manual X/Y/Z coordinate inputs for precise positioning
  - Color pickers for each marker
  - Dashed connection line between markers with distance readout in meters
  - Configurable dash size, gap size, marker size, and pulse speed

- **Scene management**
  - Save/load scenes to `localStorage`
  - Create, rename, and delete scenes
  - Full scene serialization (room config, objects, markers, camera state)
  - Collapsible left-side panel with scene list

- **Camera controls**
  - Orbit, pan, and zoom via mouse (OrbitControls)
  - Drag detection prevents accidental marker placement while orbiting
  - Preset views: 3/4 angle, Front, Side, Top, Reset
  - Adjustable FOV and zoom sliders

**Tech:** Three.js (WebGLRenderer, MeshStandardMaterial, PCFSoftShadowMap, ACESFilmicToneMapping), GaussianSplats3D OrbitControls.

### Technical Implementation

**Dynamic Geometry Generation:**
The room is rebuilt entirely when dimensions change. All geometry is disposed and recreated to prevent memory leaks:

```javascript
function rebuildRoom() {
  // Dispose existing geometry
  roomGroup.traverse((child) => {
    if (child.geometry) child.geometry.dispose();
    if (child.material) child.material.dispose();
  });
  scene.remove(roomGroup);
  
  // Rebuild geometry based on current dimensions
  const w = parseFloat(roomIn.w.value);
  const d = parseFloat(roomIn.d.value);
  const h = parseFloat(roomIn.h.value);
  // ... create new meshes
}
```

**Raycast-Based Interaction:**
All click interactions use Three.js raycasting against a maintained array of raycast targets (room surfaces and placed objects):

```javascript
const raycaster = new THREE.Raycaster();
raycaster.setFromCamera(mouse, camera);
const intersects = raycaster.intersectObjects(raycastTargets, true);
if (intersects.length > 0) {
  const point = intersects[0].point;
  // Place marker or move object to point
}
```

**Object Selection and Dragging:**
When an object is selected, orbit controls are disabled and mouse clicks move the object along room surfaces. The drag system uses a threshold to distinguish between camera orbits and intentional clicks:

```javascript
let dragStart = null;
let isDragging = false;

function onMouseDown(event) {
  dragStart = { x: event.clientX, y: event.clientY };
}

function onClick(event) {
  if (isDragging) return; // Ignore if user was dragging
  // Handle placement/selection
}
```

**Scene Serialization:**
Scenes are serialized to JSON-compatible objects for `localStorage` persistence:

```javascript
function serializeScene() {
  return {
    room: { shape, w, d, h },
    objects: placedObjects.map(o => ({
      shape, size, color,
      position: [x, y, z],
      scale: scaleMultiplier,
      rotation: [rotX, rotY, rotZ]
    })),
    markers: { a: {...}, b: {...} },
    line: { show, color, dash, gap, ... },
    camera: { position, target, fov }
  };
}
```

**State Management:**
- `placedObjects` array tracks all placed items with references to Three.js meshes
- Raycast targets array is maintained separately for efficient intersection testing
- Selected object state is tracked separately to manage UI updates and interaction modes
- Room geometry and lighting are fully rebuilt when dimensions change; raycast target arrays are cleaned up to prevent stale references
- Placed objects are tracked in a state array with proper cleanup on deletion (scene removal + raycast target removal)
- The marker pulsation animation runs via `Math.sin(elapsed * speed)` on scale and opacity, computed each frame without allocations

---

## Setup and Running

### Prerequisites

- Node.js (v16+)
- npm

### Installation

```bash
npm install
npm run build
```

The build process copies demo files and dependencies (including Three.js library) into `build/demo/`.

### Demo Data (Gaussian Splat Overlay Only)

The marker overlay demo requires pre-trained Gaussian Splat scene data:

```bash
mkdir -p build/demo/assets/data
wget -O /tmp/gaussian_splat_data.zip https://projects.markkellogg.org/downloads/gaussian_splat_data.zip
unzip /tmp/gaussian_splat_data.zip -d build/demo/assets/data/
```

This provides `garden.ksplat`, `bonsai.ksplat`, `stump.ksplat`, and `truck.ksplat` scene files.

### Running the Server

```bash
npm run demo
```

Server starts at `http://127.0.0.1:8080` with required CORS headers for `SharedArrayBuffer` support used by the Gaussian Splat sort worker.

**Note:** The server (`util/server.js`) sets the following headers:
- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Embedder-Policy: require-corp`

These are required for the Gaussian Splat sorting worker to use shared memory efficiently.

## Architecture

### File Structure

```
demo/
  marker-overlay.html    -- Gaussian Splat scene + marker overlay
  room-markers.html      -- Dynamic room + object placement + markers
  dropin.html            -- Original GaussianSplats3D drop-in demo (unmodified)
  garden.html            -- Original garden demo (unmodified)
  ...                    -- Other original demos (unmodified)
```

### Module System

Both demos are single-file HTML with inline `<script type="module">` blocks. Three.js and GaussianSplats3D are loaded via import maps pointing to `./lib/`. No build step is needed for the demos themselves; they run directly from the served `build/demo/` directory.

The demos use ES modules with import maps:

```html
<script type="importmap">
{
  "imports": {
    "three": "./lib/three.module.js",
    "@mkkellogg/gaussian-splats-3d": "./lib/gaussian-splats-3d.module.js"
  }
}
</script>
```

### Rendering Pipeline

1. **Initialization**: Create Three.js scene, camera, renderer, and controls
2. **Scene Setup**: Add Gaussian Splat viewer (overlay demo) or generate room geometry (room demo)
3. **Interaction Setup**: Attach mouse/keyboard event listeners
4. **Render Loop**: `requestAnimationFrame` calls renderer, updates marker animations, handles object manipulation
5. **State Persistence**: Serialize scene state to `localStorage` on save (room demo)

### Performance Considerations

- **Geometry Disposal**: All geometry and materials are properly disposed when objects are removed or rooms are rebuilt
- **Raycast Optimization**: Maintain separate arrays of raycast targets to avoid testing against non-interactive objects
- **Animation Efficiency**: Marker pulsation uses `Math.sin()` calculations without object allocation in the render loop
- **Memory Management**: `localStorage` has ~5-10MB limits; scene serialization uses compact JSON representation

## Technical Stack

- **Three.js**: WebGL rendering, geometry, materials, raycasting, controls
- **GaussianSplats3D**: 3D Gaussian Splat rendering via `DropInViewer`
- **ES Modules**: Modern JavaScript module system with import maps
- **localStorage**: Browser API for persistent scene storage
- **WebGL**: Hardware-accelerated 3D graphics rendering

## Browser Compatibility

- **Chrome/Edge**: Full support (recommended)
- **Firefox**: Full support
- **Safari**: Requires iOS 16.4+ for `SharedArrayBuffer` support (required for Gaussian Splat overlay)
- **Mobile**: Limited performance on mobile devices due to CPU-based splat sorting

## Future Enhancements

Potential improvements and extensions:

- GPU-accelerated splat sorting for better performance
- Export scene configurations to JSON files
- Import/export placed objects as 3D models (GLTF/OBJ)
- Multi-marker support (more than 2 markers)
- Marker path/trajectory visualization
- Real-time collaboration via WebSocket
- VR/AR support using WebXR
- Advanced lighting and shadow options for room demo
- Texture mapping for room surfaces and objects
