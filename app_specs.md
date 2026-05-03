# 3D Shadow & Light Lab: App Specifications & Architecture

## Part 1: Product Vision & Specifications

**App Overview**
The "3D Shadow & Light Lab" is a purely client-side, web-based 3D scene editor and lighting simulator. It allows users to spawn geometric primitives, manipulate a physical light source to create realistic shadows, apply physically-based materials, perform complex rigid-body alignments, and use AI to generate or critique scenes. 

**Core Philosophies**
* **Zero-Build, Single File:** The entire application (HTML, CSS, JS, Shaders) exists in a single `index.html` file without a build step or bundler. 
* **Database-less Sharing:** Scenes can be fully serialized and compressed into the URL hash, allowing users to share exact states without relying on a backend database (though a simple key-value backend is used for shortlinks).
* **Literal 3D over Abstraction:** The AI generation tools are instructed to build "Lego-like" literal creations out of primitives rather than standard 3D meshes.

### Features & Functional Specs
1. **The Environment:**
   * A 15x15x15 unit translucent wireframe bounding box with visual axis ticks.
   * A dynamic spotlight with a physical bulb, casing, and a volumetric translucent light beam. Both the light source and the target can be dragged.
2. **Object Management:**
   * **Primitives:** Add Cubes, Spheres, Cones, Cylinders, and Tori.
   * **Polygons:** Add custom flat Planes.
3. **Transform Tools:**
   * **Box Select / Multi-Select:** Draw a 2D box on the screen to group objects, or hold `Ctrl/Cmd` to toggle multi-selection. Grouped items transform together.
   * **Translate / Rotate / Scale:** Standard 3D manipulation gizmos with optional grid snapping.
   * **Vertex Editing:** Double-click plane edges to add vertices, double-click vertices to delete them. Drag vertices to deform flat polygons.
4. **Advanced Snapping:**
   * **Edge Snap (Magnet):** Select an edge on object A, click an edge on object B. Object A rotates and translates to perfectly align the two edges. Double-clicking the target flips the alignment 180 degrees.
   * **Vertex Snap:** Click a vertex on Object A, click a target point in the scene. If Object A is a primitive, the whole object moves. If Object A is a custom polygon, it deforms the specific vertex to the target.
5. **Materials & Aesthetics:**
   * Custom Color Hex and Opacity sliders.
   * **Matte:** Standard rough material.
   * **Glass:** Refractive, transparent physical material.
   * **Wood:** Procedural texture generated via HTML Canvas.
   * **Mirror:** Reflects the live 3D environment.
6. **AI Magic (Gemini Integration):**
   * **Text-to-Scene:** Prompts AI to return a JSON array of primitive coordinates to build recognizable objects.
   * **Image-to-Scene:** Uses vision models to analyze an uploaded image and recreate it in 3D primitives.
   * **Critique:** Passes the serialized scene state to the AI to write a pretentious museum plaque description.
7. **Persistence:**
   * Internal Undo stack (stores up to 30 history states).
   * JSON Export/Import.
   * **Long Link:** Compresses JSON state via LZ-String into the `#scene=` hash.
   * **Short Link:** Sends state to a REST API (`nocodebackend.com`), gets a `3dlab_` GUID, and shares via `#id=` hash.

---

## Part 2: Software Architecture & Design Choices

The application uses **Three.js (r128)** for rendering, **Tailwind CSS** for UI styling, and **LZ-String** for URL compression. Below are the key architectural decisions that solve common 3D engine challenges.

### 1. Z-Fighting Prevention: The "Micro-Jitter" Method
* **The Problem:** When AI generates scenes or users snap objects together, objects often share the exact same mathematical coordinates/planes, causing the GPU depth-buffer to flicker rapidly between textures (Z-fighting).
* **The Solution:** Rather than relying on `polygonOffset` (which breaks physical occlusion ordering), a microscopic jitter (`Math.random() * 0.002 - 0.001`) is applied to the `x, y, z` coordinates of every object upon creation. This guarantees no two objects ever occupy the exact same plane, preserving true physical depth sorting.

### 2. Shadow Mapping: Eliminating "Peter Panning"
* **The Problem:** Spotlights cast shadows on zero-thickness custom planes. If the shadow map bias is too high, the shadow gets pushed entirely through the flat plane, causing light to "bleed" through 100% opaque objects.
* **The Solution:** The spotlight uses a heavily reduced depth `bias` (`-0.00005`) to prevent light bleed, combined with a `normalBias` (`0.02`) to artificially thicken surfaces during the shadow pass, eliminating shadow acne (pixelated artifacts) without detaching the shadow from the object.

### 3. Click Detection: Ghost Volumes & Bounding Boxes
* **The Problem:** The spotlight includes a massive translucent light beam (a ConeGeometry). Standard Three.js raycasting would hit this invisible cone first, blocking the user from clicking any objects behind it. Furthermore, grouping the light into a single bounding box resulted in a box that covered the whole room.
* **The Solution:** * The beam's click detection is entirely disabled via an empty function override: `beam.raycast = function() {};`.
  * For Box Selection, the calculation explicitly ignores the light beam and calculates the bounding box of the tiny light bulb mesh instead of the overarching Light Group.

### 4. Multi-Select Logic: The Proxy Group
* **The Problem:** Three.js `TransformControls` can only attach to a single object at a time.
* **The Solution:** When multiple items are selected, a hidden proxy `THREE.Group` (`multiSelectGroup`) is used.
  * The exact 3D center of all selected objects is calculated.
  * The proxy group is moved to that center.
  * The objects are detached from the scene and attached as children of the proxy group.
  * `TransformControls` manipulates the proxy group. 
  * Upon deselecting, the objects calculate their new world matrices and are reattached to the main scene.

### 5. 2D Box Selection Algorithm
* **The Problem:** Converting a drawn HTML `div` into a 3D selection volume.
* **The Solution:** Instead of complex Frustum math, the app iterates through every interactable object and grabs the 8 corners of its 3D bounding box. It projects those 8 corners into 2D screen space (`corner.project(camera)`). If any of the 8 corners (or the mathematical center point) falls within the `left/right/top/bottom` boundaries of the HTML `div`, the object is selected.

### 6. Custom Polygon Triangulation
* **The Problem:** Three.js cannot natively render flat planes with an arbitrary number of vertices without a complex triangulation step.
* **The Solution:** Custom planes store an array of 3D vectors (`userData.points`). A custom function `updatePolygonGeo` dynamically builds a `BufferGeometry` using a **Centroid-Fan** method. It averages all points to find the center, then draws triangles connecting `Point A -> Point B -> Centroid` all the way around the shape.

### 7. Rigid Body Alignment (Magnet Snap)
* **The Problem:** Moving and rotating a 3D object so one of its edges perfectly touches an edge on another object.
* **The Solution:**
  1. The local midpoint and directional vector of the source edge are saved.
  2. Upon clicking the target edge, the target's directional vector is calculated.
  3. A `THREE.Quaternion().setFromUnitVectors()` calculates the rotation required to align the source direction to the target direction. This is multiplied against the object's current rotation.
  4. The object is rotated.
  5. The post-rotation position of the source midpoint is calculated in world space.
  6. A translation vector is derived to move the newly rotated source midpoint directly onto the target midpoint. 

### 8. Dynamic Environment Maps (Mirrors)
* **The Problem:** Making standard materials reflect objects moving around them in real-time.
* **The Solution:** The app places a `THREE.CubeCamera` at `(0,3,0)`. Inside the main `animate()` loop:
  1. The scene background is set to a static gradient.
  2. UI elements (Transform controls, snap dots) are hidden (`visible = false`).
  3. The `CubeCamera` captures a 360 snapshot.
  4. The UI elements and backgrounds are restored.
  5. The main camera renders the scene.
  * The resulting texture is assigned to `scene.environment`, which physically-based materials (`MeshStandardMaterial`) automatically use for reflections.

### 9. State Management & Serialization
* The state is not saved by serializing the massive Three.js object graph. Instead, `getSceneStateData()` creates a lightweight dictionary containing only: Geometry type, IsPolygon, Points Array, Position, Rotation (Quaternion), Scale, Color Hex, Opacity, Transparent flag, and Material type. 
* This allows the state to remain tiny enough to be heavily compressed by LZ-String and passed safely through a URL Hash constraint.

# 3D Shadow & Light Lab: Developer Lore & Debugging History

This document preserves the "war stories" and highly specific fixes discovered during the development of the 3D Light Lab. It explains *why* certain non-obvious code exists.

## 1. Web Browser Quirks & UI Fixes

**The Iframe "Same-Origin" Trap**
* **Bug:** Share links weren't loading when hosted on sites like `gisthost.github.io`.
* **Reason:** Wrapper sites load projects inside hidden `iframes`. Browsers enforce a strict "Same-Origin Policy" that completely blocks an `iframe` from reading the address bar (`window.location.hash`) of its parent window.
* **Fix:** We added a `try/catch` block that attempts to read `window.top.location.href`. If the browser blocks it via a security exception, it safely falls back to reading its own iframe URL, which is why the "Paste URL to Import" fallback was added.

**URL Decompression Corruption (`+` vs ` `)**
* **Bug:** Complex LZ-String compressed URLs would randomly fail to unpack.
* **Reason:** Some web browsers aggressively strip out the `+` character in URL hashes and replace them with empty spaces ` `, which instantly corrupts Base64/LZ-String data.
* **Fix:** The loader includes a mandatory cleanup step: `compressed.replace(/ /g, '+')` before attempting to decompress.

**iOS Auto-Zooming Inputs**
* **Bug:** Tapping the opacity slider or color picker on an iPhone would cause the whole screen to awkwardly zoom in, ruining the 3D app experience.
* **Reason:** Apple iOS Safari automatically zooms into any `<input>` element if the text size is deemed too small.
* **Fix:** `maximum-scale=1.0` was added to the `<meta name="viewport">` tag.

**Keyboard Shortcut Hijacking**
* **Bug:** The `Delete` key wouldn't work to delete objects if the user had just clicked a tool button (like "Move" or "Rotate").
* **Reason:** A safety check was added to ignore the Delete key if the user was typing in an `<input>`. However, the tool modes are built using `<input type="radio">` buttons under the hood. 
* **Fix:** The event listener had to be strictly narrowed to only ignore `active.tagName === 'TEXTAREA' || active.type === 'text'`.

## 2. Three.js Engine Quirks

**Opacity Snapping (The 70% Cutoff)**
* **Bug:** Lowering the opacity slider wouldn't fade the object; it would remain solid until ~70%, then instantly snap out of existence.
* **Reason:** In WebGL/Three.js, an object is either in the "Opaque Render Pass" or the "Transparent Render Pass". Lowering the opacity float value doesn't automatically move the object to the transparent pass.
* **Fix:** The UI slider logic explicitly forces `object.material.transparent = true` and `object.material.depthWrite = false` the moment opacity drops below 1.0 (or immediately if the material is Glass).

**The "Necromancer" Delete Bug**
* **Bug:** Deleting a group of objects removed them from the screen, but interacting with the scene caused them to respawn as un-selectable ghosts.
* **Reason:** The `Delete` function removed the objects from the 3D scene array. However, the subsequent step called `clearMultiSelect()`, which looked at what was in the selection memory and gracefully *re-added* those items back into the 3D scene to ungroup them.
* **Fix:** The selection must be explicitly cleared/detached (`selectObject(null)`) *before* the `scene.remove(obj)` array deletion executes.

**Peter Panning vs. Shadow Acne**
* **Bug:** The spotlight was bleeding directly through the 100% opaque red plane, lighting objects behind it.
* **Reason:** To prevent "Shadow Acne" (glitchy black pixels on surfaces), shadows use a `bias` to push the shadow slightly backward. Because our custom Planes have exactly *zero thickness*, pushing the shadow backward pushed it completely out the other side of the object.
* **Fix:** The `shadow.bias` was reduced to a microscopic `-0.00005`, and `shadow.normalBias = 0.02` was added to fake surface thickness during the shadow map calculation.

## 3. AI & REST API Integrations

**LLM Markdown Wrapping**
* **Bug:** The Gemini AI would occasionally crash the scene generator by returning invalid JSON.
* **Reason:** Even when instructed to return raw JSON, Large Language Models will frequently format the output by wrapping it in markdown code blocks (e.g., ` ```json \n [...] \n ``` `), which breaks `JSON.parse()`.
* **Fix:** A regex cleanup step `responseText.replace(/```json\n?/gi, '').replace(/```/g, '').trim()` strips formatting before parsing.

**NoCodeBackend REST Structure**
* The external database relies on specific payload formats. To preserve the structure in case of a server migration:
* **Headers:** Requires `Authorization: Bearer <TOKEN>`
* **Create/Write (`POST /create/keyval`):** Requires JSON body:
  `{ keyname: "3dlab_xxx", jval: "{...}", createdon: "YYYY-MM-DD HH:MM:SS", createdby: "UserAgent string" }`
* **Search/Read (`POST /search/keyval`):** Requires JSON body:
  `{ keyname: "3dlab_xxx" }` 
  *(Note: Search is used instead of direct `GET` to support non-integer alphanumeric custom GUIDs).*
