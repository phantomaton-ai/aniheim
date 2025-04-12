# Aniheim Code Plan 📋⚙️ (Revision 2)

This document outlines the file structure and interfaces for the Aniheim Rendering Engine build tool, based on `DESIGN.md` (Revision 4).

## Core Concepts

*   **Build Tool:** Node.js app (`build.js`) processing linked Markdown files.
*   **Contextual Processing:** Uses separate `necronomicon` setups for different file types (episode, scene, character, etc.).
*   **Asset Generation:** Focuses build-time generation on binary assets (MP3 voice/music). Text assets (`.abc`, `.json`, `artwork.md`) are expected to exist (generated upstream by LLM); build fails with clear errors if missing. Regeneration via manual file deletion.
*   **Runtime:** Self-contained HTML with bundled JS for playback. Expression evaluation happens at runtime.

## File Structure

```
aniheim/
├── build.js                # CLI Entry Point
├── src/
│   ├── build/              # Build-time logic
│   │   ├── episode/
│   │   │   ├── commands/
│   │   │   │   └── scene.js    # Handles /scene directive
│   │   │   └── process.js      # Processes episode.md
│   │   ├── scene/
│   │   │   ├── commands/       # Commands valid within scene.md
│   │   │   │   ├── background.js
│   │   │   │   ├── character.js
│   │   │   │   ├── music.js
│   │   │   │   ├── move.js
│   │   │   │   ├── animate.js
│   │   │   │   └── dialog.js
│   │   │   └── process.js      # Processes scene.md
│   │   ├── character/
│   │   │   ├── commands/       # Commands valid within character.md
│   │   │   │   ├── voice.js
│   │   │   │   └── artwork.js  # Links to artwork.md
│   │   │   └── process.js      # Processes character.md
│   │   ├── music/
│   │   │   ├── commands/       # Command valid within music.md
│   │   │   │   └── score.js
│   │   │   └── process.js      # Processes music.md
│   │   ├── background/
│   │   │   ├── commands/       # Command valid within background.md
│   │   │   │   └── artwork.js  # Links to artwork.md
│   │   │   └── process.js      # Processes background.md
│   │   ├── artwork/
│   │   │   ├── commands/       # Commands valid within artwork.md
│   │   │   │   ├── ellipse.js
│   │   │   │   └── polygon.js
│   │   │   ├── parse.js        # Parses artwork.md -> Artwork structure
│   │   │   └── process.js      # Processes artwork.md
│   │   ├── generate/           # Binary asset generation
│   │   │   ├── voice.js        # TTS MP3 generation
│   │   │   └── music.js        # ABC -> MP3 generation
│   │   ├── state.js          # Shared build state object definition
│   │   └── emit.js             # HTML emitter/bundler
│   ├── runtime/                # Code bundled into output HTML
│   │   ├── index.js            # Entry point
│   │   ├── player.js           # Main loop & event processor
│   │   ├── render.js           # Canvas 2D renderer
│   │   ├── audio.js            # Audio manager (Howler.js)
│   │   ├── evaluate.js         # Runtime expression evaluator
│   │   ├── state.js            # Runtime state manager (interpolation)
│   │   ├── interpolate.js      # Lerp function
│   │   └── data.js             # Injected episode data
│   └── data/                   # Static definitions (managed via source control)
│       ├── emoji_map.json
│       ├── pose_definitions.json
│       └── expression_definitions.json
├── package.json
├── PLAN.md
├── DESIGN.md
```

## Definitions (Build Time)

*   **`Cutout` (Render Time):** Represents a fully calculated shape ready for drawing.
    ```typescript
    interface Cutout {
      id?: string;
      type: 'ellipse' | 'polygon';
      color: string; // #RRGGBBAA
      x: number;     // Final canvas X
      y: number;     // Final canvas Y
      z: number;     // Layering value
      rotation: number; // Final degrees
      rx?: number;    // Final ellipse radius X
      ry?: number;    // Final ellipse radius Y
      points?: Array<[number, number]>; // Final polygon points [[x1, y1], ...]
    }
    ```
*   **`ShapeTemplate` (Build Time):** Represents a parsed shape directive from `artwork.md`.
    ```typescript
    interface ShapeTemplate {
      id?: string;
      type: 'ellipse' | 'polygon';
      attributes: { // Unevaluated expressions as strings
        x: string; y: string; z: string; rotation: string; color: string;
        rx?: string; ry?: string; // Ellipse
        pointsExpr?: string; // Polygon override variable name
      };
      defaultPoints?: Array<[number, number]>; // Polygon default points
    }
    ```
*   **`Artwork` (Build Time):** The result of parsing an `artwork.md` file.
    ```typescript
    interface Artwork {
      shapes: Array<ShapeTemplate>;
    }
    ```

## File Details

---

### `build.js`

*   **Purpose:** CLI Entry Point.
*   **Imports:** `path`, `fs`, `yargs`, `./src/build/episode/process.js`
*   **Interface:** Parses args, determines paths, calls `episode.process(episodePath, outputPath)`.

---

### `src/build/<context>/process.js` (e.g., `episode/process.js`, `scene/process.js`)

*   **Purpose:** Processes a specific type of Markdown file (`episode.md`, `scene.md`, etc.).
*   **Imports:** `fs`, `path`, `necronomicon`, `./commands/*`, `../state.js`, `../<other_contexts>/process.js` (for recursion).
*   **Exports:** `async function process(filePath, buildState)`
*   **Interface:**
    *   `process(filePath, buildState)`:
        *   Reads `filePath`.
        *   Initializes `necronomicon` with commands specific to this context (imported from `./commands/`).
        *   Executes content using `necronomicon.execute(content, buildState)`. Commands update the shared `buildState`.
        *   Reports clear errors (with line numbers if possible via Necronomicon context) if required files (like `.abc`, `.json`, linked `.md`) are missing.

---

### `src/build/<context>/commands/*.js` (e.g., `episode/commands/scene.js`, `scene/commands/character.js`, `artwork/commands/ellipse.js`)

*   **Purpose:** Defines Necronomicon commands valid within a specific file context.
*   **Imports:** `fs`, `path`, `../state.js`, `../../generate/*`, `../../<other_contexts>/process.js`, `../../artwork/parse.js`.
*   **Exports:** `CommandDefinition` object { name, description, validate, execute, example }.
*   **Interface (`execute`):**
    *   `async execute(attributes, body, context)`:
        *   `context`: Shared build state object (`instanceof BuildState`).
        *   **Logic:** Validates attributes. Triggers recursive processing by calling `process()` for linked files (`attributes.file`, `attributes.source`). For commands defining assets (`/voice`, `/score`, `/artwork` linker), checks existence of target files (`attributes.settings`, `attributes.sheet`, `attributes.audio`, `attributes.file` for artwork). If *binary* target (`.mp3`) missing, calls appropriate `generate` function. If *text* target (`.json`, `.abc`, `artwork.md`) missing, throws informative error. For `/artwork` in `character.md`/`background.md`, it ensures the linked `artwork.md` is processed via `artwork.process()` which uses `artwork.parse()` and stores the resulting `Artwork` structure in the `buildState`. For `/ellipse`/`/polygon` within `artwork.md`, it adds a `ShapeTemplate` to the current artwork being parsed (held in `buildState`). Scene commands (`/move`, `/animate`, `/dialog`) add events to the scene timeline in `buildState`.

---

### `src/build/generate/voice.js`

*   **Purpose:** Generates voice MP3 using TTS if file is missing.
*   **Imports:** `fs`, `path`, TTS SDK, `../state.js` (to get settings path).
*   **Exports:** `async function ensureAudio(dialogData, buildState)`
*   **Interface:** Takes dialog info (text, target audio path, character ID) and `buildState`. Reads voice settings JSON from path stored in `buildState` for the character. Calls TTS API. Saves MP3. Throws if settings JSON missing.

---

### `src/build/generate/music.js`

*   **Purpose:** Generates music MP3 from ABC file if MP3 is missing.
*   **Imports:** `fs`, `path`, `child_process` (for `timidity++`), `abcjs`.
*   **Exports:** `async function ensureAudio(scoreData)`
*   **Interface:** Takes score info (sheet path, target audio path). Checks sheet exists (throws if not). Runs ABC -> MIDI -> MP3 pipeline. Saves MP3.

---

### `src/build/artwork/parse.js`

*   **Purpose:** Parses `artwork.md` content into an `Artwork` structure.
*   **Imports:** `fs`, `path`, `./process.js` (to run necronomicon with artwork commands).
*   **Exports:** `async function parse(artworkMdPath)`
*   **Interface:**
    *   `parse(artworkMdPath)`: Reads the file. Creates a temporary state for the artwork parser. Calls `artwork.process(artworkMdPath, tempState)` which uses Necronomicon and `/ellipse`/`/polygon` commands to populate `tempState.shapes`.
    *   **Returns:** `Promise<Artwork>` (containing the `shapes: Array<ShapeTemplate>`).

---

### `src/build/state.js`

*   **Purpose:** Defines and holds the shared state accumulated during the build process.
*   **Imports:** None.
*   **Exports:** `class BuildState { constructor(); addScene(); addCharacterDef(); addBackgroundDef(); addMusicDef(); addEvent(); ... }`
*   **Interface:** A class instance passed around as the `context` in `necronomicon.execute`. Contains methods to store parsed definitions (characters, backgrounds, music, artwork structures), scene timelines (lists of events like move, animate, dialog), etc.

---

### `src/build/emit.js`

*   **Purpose:** Generates the final HTML output.
*   **Imports:** `fs`, `path`, `esbuild`.
*   **Exports:** `async function emit(outputPath, state)`
*   **Interface:** Bundles `src/runtime` code. Injects `state` (JSON-stringified, containing definitions and event timelines) into `runtime/data.js`. Writes final HTML.

---

### `src/runtime/index.js`

*   **Purpose:** Runtime entry point.
*   **Imports:** `./player.js`, `./render.js`, `./audio.js`, `./data.js`, `../data/*.json`.
*   **Interface:** Reads injected `data`. Reads static JSONs. Initializes and starts the `Player`.

---

### `src/runtime/player.js`

*   **Purpose:** Main animation loop, event processing, state delegation.
*   **Imports:** `./state.js`, `./audio.js`, `./render.js`, `./evaluate.js`.
*   **Exports:** `class Player { constructor(config); play(); pause(); }`
*   **Interface:**
    *   `constructor({ events, characters, backgrounds, music, poseMap, exprMap, audio, render })`: Stores data. Instantiates `RuntimeState`.
    *   `play()`/`pause()`: Control loop.
    *   `update(time)`: (Loop callback) Updates `RuntimeState` based on events active at `time`. Gets current character states (position, pose params, expression params) from `RuntimeState`. For each visible character/background, gets its `Artwork` structure. Evaluates each `ShapeTemplate`'s attributes using `evaluate()` with the current state params to get final `Cutout` properties. Calls `render.draw()` with the list of `Cutout` objects. Manages audio playback via `audio` manager.

---

### `src/runtime/render.js`

*   **Purpose:** Canvas 2D drawing implementation.
*   **Imports:** None.
*   **Exports:** `class Renderer { constructor(canvas); draw(cutouts); }`
*   **Interface:** `draw(cutouts)` takes an array of fully calculated `Cutout` objects and draws them. Handles Z-sorting.

---

### `src/runtime/audio.js`

*   **Purpose:** `howler.js` wrapper.
*   **Imports:** `howler`.
*   **Exports:** `class AudioManager { load(id, path); play(id, ?volume); stop(id); }`

---

### `src/runtime/evaluate.js`

*   **Purpose:** Evaluates arithmetic expressions from `ShapeTemplate` attributes at runtime.
*   **Imports:** Custom lightweight parser/evaluator.
*   **Exports:** `function evaluate(expression, context)`
*   **Interface:** `evaluate(expression, context)` takes string expression and context object (`{ position: {...}, pose: {...}, expression: {...} }`), returns `number`.

---

### `src/runtime/state.js`

*   **Purpose:** Manages current interpolated state (position, pose params, expression params) for characters based on timeline events.
*   **Imports:** `./interpolate.js`, `../data/*.json` (for definitions).
*   **Exports:** `class RuntimeState { constructor(chars, backgrounds, poseDefs, exprDefs, emojiMap); update(time, activeEvents); getCurrentParams(id); }`
*   **Interface:**
    *   `constructor(...)`: Stores definitions. Initializes character states.
    *   `update(time, activeEvents)`: Calculates current interpolated `position`, `pose` parameters, `expression` parameters for all characters based on `activeEvents` determined by `Player`.
    *   `getCurrentParams(id)`: Returns the current `{ position: {...}, pose: {...}, expression: {...} }` object for the given character ID.

---

### `src/runtime/interpolate.js`

*   **Purpose:** Linear interpolation.
*   **Imports:** None.
*   **Exports:** `function lerp(a, b, t)`

---

This revised plan reflects the contextual command processing, simplifies generation, focuses artwork definition, and clarifies the runtime evaluation flow. It seems ready, Dr. Woe! 🚀👻