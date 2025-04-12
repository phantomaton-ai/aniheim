# Aniheim Code Plan 📋⚙️

This document outlines the file structure and interfaces for the Aniheim Rendering Engine build tool, based on `DESIGN.md` (Revision 4).

## Core Concepts

*   **Build Tool:** A Node.js application that reads source Markdown files and outputs a self-contained HTML animation player.
*   **Directive-Driven:** Uses `smarkup` for syntax and `necronomicon` for processing directives found in Markdown files.
*   **Asset Generation:** Generates necessary assets (audio, music sheets, config JSONs, artwork definitions) via LLM prompts and external tools if they don't exist. Regeneration is triggered by deleting the target file.
*   **Runtime:** The output HTML contains bundled JavaScript code for animation playback, rendering, and audio.

## File Structure

```
aniheim/
├── build.js                # CLI Entry Point & Orchestration
├── src/
│   ├── build/              # Build-time logic
│   │   ├── commands/       # Necronomicon command definitions
│   │   │   ├── scene.js
│   │   │   ├── character.js
│   │   │   ├── music.js
│   │   │   ├── background.js
│   │   │   ├── artwork.js
│   │   │   ├── score.js
│   │   │   ├── voice.js
│   │   │   ├── move.js
│   │   │   ├── animate.js
│   │   │   ├── dialog.js
│   │   │   └── index.js    # Exports all commands list
│   │   ├── process.js      # Necronomicon execution wrapper
│   │   ├── generate/       # Asset generation helpers
│   │   │   ├── settings.js # Voice settings JSON generation
│   │   │   ├── voice.js    # TTS MP3 generation
│   │   │   ├── score.js    # ABC generation
│   │   │   ├── music.js    # ABC -> MP3 generation
│   │   │   ├── artwork.js  # artwork.md generation
│   │   │   └── llm.js      # LLM interaction helper
│   │   ├── parse/          # Specific format parsers
│   │   │   └── artwork.js  # artwork.md microformat -> shape def list
│   │   └── emit.js         # HTML emitter/bundler
│   └── runtime/            # Code to be bundled into the output HTML
│       ├── index.js        # Entry point for runtime bundle
│       ├── player.js       # Main player loop state manager
│       ├── render.js       # Canvas 2D rendering implementation
│       ├── audio.js        # Audio management (Howler.js wrapper)
│       ├── evaluate.js     # Expression evaluator (`expr` attributes)
│       ├── state.js        # Manages character pose/expression/position state
│       └── data.js         # Placeholder for injected episode data
├── package.json
├── PLAN.md
├── DESIGN.md
├── config/                 # Static configuration data
│   ├── emoji_map.json
│   ├── pose_definitions.json
│   └── expression_definitions.json
```

/comment(user:woe) {
    Let's organize commands a little bit differently. For example, the `/scene` command is only going to get executed in the context of an episode, so I'd put that somewhere like `episodes/commands/scenes`. In practice, I think we have a few different "necronomicon wrappers" in the sense that we have a few different file types (and associated directive sets) which we'll support.

    Similarly, `config` is the wrong idea for poses/expressions. Those will be globally reused, but we'll want to iterate on them via LLM interactions in much the same way as everything else. Those should live under the `/src` hierarchy somewhere.

    I do like the split between `build` and `runtime`, so let's retain that.
}

## File Details

---

### `build.js`

*   **Purpose:** Handles CLI arguments, initializes the build process.
*   **Imports:** `path`, `fs`, `yargs`, `./src/build/process.js`
*   **Exports:** None (Executable script).
*   **Interface:**
    *   Parses CLI args (`<episode_markdown_file>`, `--output`).
    *   Determines absolute input/output paths.
    *   Calls `process(episodePath, outputPath)` to start the build.

---

### `src/build/process.js`

*   **Purpose:** Orchestrates the build using Necronomicon to process Markdown files. Reads the entry point episode file and recursively processes linked files via commands. Collects all necessary data for the final HTML emission.
*   **Imports:** `fs`, `path`, `necronomicon`, `./commands/index.js`, `./emit.js`
*   **Exports:** `async function process(episodePath, outputPath)`
*   **Interface:**
    *   `process(episodePath, outputPath)`:
        *   Initializes `necronomicon` with commands from `./commands/index.js`.
        *   Reads `episodePath` content.
        *   Executes the content using `necronomicon.execute()`. The execution context/state will be built up by the command implementations (e.g., storing scene data, character defs).
        *   After processing, calls `emit(outputPath, buildState)` with the final collected state.

---

### `src/build/commands/index.js`

*   **Purpose:** Aggregates and exports the list of command definitions for Necronomicon.
*   **Imports:** `./scene.js`, `./character.js`, `./music.js`, `./background.js`, `./artwork.js`, `./score.js`, `./voice.js`, `./move.js`, `./animate.js`, `./dialog.js`
*   **Exports:** `Array<CommandDefinition>` (Format expected by `necronomicon`)

---

### `src/build/commands/scene.js` (and other command files)

*   **Purpose:** Defines a single command for Necronomicon (e.g., `/scene`). Handles attribute validation and execution logic during build time. Execution often involves reading linked files, triggering asset generation, or updating the build state.
*   **Imports:** `fs`, `path`, `necronomicon` (potentially, if recursive processing needed), `../generate/*`, `../parse/*` (as needed).
*   **Exports:** `CommandDefinition` object { name, description, validate, execute, example }.
*   **Interface (`execute` function):**
    *   `async execute(attributes, body, context)`:
        *   `attributes`: Object of directive attributes.
        *   `body`: String content of directive body (if any).
        *   `context`: Shared build state object (passed by `necronomicon`). Allows commands to read/write data like parsed definitions, event lists, etc.
        *   **Logic:** Validates attributes. Reads linked files (`attributes.file`, `attributes.source`). Calls generation functions (e.g., `generate.voice()`) if target assets (`attributes.audio`, `attributes.settings`) don't exist, passing LLM prompts from `body`. Parses content (e.g., `parse.artwork()`). Updates the `context` object with parsed data or adds events to a timeline. May recursively call `necronomicon.execute()` on linked file content.
        *   **Returns:** (Typically `undefined` or status message). Result inclusion controlled by Necronomicon options.

---

### `src/build/generate/llm.js`

*   **Purpose:** Abstract interaction with the configured LLM service (e.g., Gemini via `phantomaton-gemini`). Handles prompt formatting, API calls, and potentially retries/validation loops (a la `povgoblin`).
*   **Imports:** `phantomaton-gemini` (or other LLM lib), configuration loader.
*   **Exports:** `async function generate(prompt, validationFn?)`
*   **Interface:**
    *   `generate(prompt, validationFn?)`: Sends `prompt` to LLM. If `validationFn` provided, retries until `validationFn(result)` returns true or max attempts reached.
    *   **Returns:** `Promise<string>` (Generated text content).

/comment(user:woe) {
    Let's just omit this for now. There are some cases in here where the `povgoblin` approach might be useful, but for the most part episodes will be LLM-generated from the beginning, so the LLM can fill in those blanks.
}

---

### `src/build/generate/voice.js`, `settings.js`, `score.js`, `music.js`, `artwork.js`

*   **Purpose:** Specific asset generation logic. Checks file existence, calls `generate.llm()` with appropriate prompts (from directive bodies), uses external tools (TTS API, `timidity++`), and saves results to target file paths specified in directive attributes.
*   **Imports:** `fs`, `path`, `./llm.js`, SDKs (e.g., `@google-cloud/text-to-speech`), `child_process` (for `timidity++`), `abcjs`.
*   **Exports:** Functions like `async ensureVoiceAudio(filePath, text, settingsPath)`, `async ensureVoiceSettings(filePath, prompt)`, `async ensureScoreSheet(filePath, prompt)`, `async ensureScoreAudio(filePath, sheetPath)`, `async ensureArtworkDef(filePath, prompt)`.
*   **Interface:** Functions take target file paths and necessary inputs (text, prompts, source paths). They are idempotent (check existence first). Return `Promise<void>`.

/comment(user:woe) {
    Similar to the above, I think we just need `voice.js` and `score.js` (to produce voice and music mp3's) at this point. Everything else is text-based, so we can rely on the LLM that generates episodes to generate those details.

    That said, we should surface clear error messages with line numbers when any of these text-based assets are missing, so that the author LLM knows to fill those in reactively.
}

---

### `src/build/parse/artwork.js`

*   **Purpose:** Parses the artwork microformat directives within an `artwork.md` file into a structured list of shape definitions.
*   **Imports:** `smarkup`.
*   **Exports:** `function parse(artworkMdContent)`
*   **Interface:**
    *   `parse(artworkMdContent)`: Uses `smarkup` to parse the content.
    *   **Returns:** `Array<ShapeDefinition>` (e.g., `{ id: 'head', type: 'ellipse', attributes: { x: '0', y: '0.15', color: '#0000FFAA', ... } }`). Attribute values are kept as strings containing expressions.

/comment(user:woe) {
    This might need to live in something like `build/formats/artwork/parse.js` so that we can have `build/artwork/commands/ellipse.js` and `build/artwork/commands/polygon.js` and so on.

    Also, let's fully-specify `ShapeDefinition`. In keeping with our single-word names, I'd call it a `Cutout` instead.

    From an OOP perspective, it would be great if `parse(artworkMd)` returned an `Artwork` object. Then, `artwork.render(location, pose, expression)` could return `Array<Cutout>`. (This allows us to move expression evaluation to build-time.)
}

---

### `src/build/emit.js`

*   **Purpose:** Takes the final build state (collected definitions, event lists, config data) and generates the output HTML file, bundling necessary runtime JavaScript.
*   **Imports:** `fs`, `path`, `esbuild` (or other bundler).
*   **Exports:** `async function emit(outputPath, state)`
*   **Interface:**
    *   `emit(outputPath, state)`:
        *   Loads runtime JS source files (`src/runtime/*`).
        *   Injects the episode-specific `state` data (JSON stringified) into `src/runtime/data.js` (or similar mechanism).
        *   Bundles runtime JS into a single script using `esbuild`.
        *   Creates HTML structure referencing the bundled script and necessary CSS.
        *   Writes the final HTML to `outputPath`.

---

### `src/runtime/index.js`

*   **Purpose:** Entry point for the bundled JavaScript in the output HTML. Initializes the player.
*   **Imports:** `./player.js`, `./render.js`, `./audio.js`, `./data.js` (holds injected state), `../config/*.json`.
*   **Exports:** None.
*   **Interface:**
    *   Reads episode `data` (definitions, events).
    *   Reads config JSONs (emoji map, pose/expression defs).
    *   Initializes `render`, `audio`.
    *   Creates and starts `player` instance, passing necessary data and modules.

---

### `src/runtime/player.js`

*   **Purpose:** Manages the main animation loop (using `requestAnimationFrame`), updates time, processes events from the timeline, manages overall playback state (play/pause).
*   **Imports:** `./state.js`, `./audio.js`, `./render.js`.
*   **Exports:** `class Player { constructor(config); play(); pause(); }`
*   **Interface:**
    *   `constructor({ events, characters, backgrounds, music, poseMap, exprMap, audio, render, state })`: Stores references.
    *   `play()`: Starts the animation loop.
    *   `pause()`: Stops the animation loop.
    *   `update(time)`: (Called by loop) Checks `events` timeline, triggers actions (calls `state.move`, `state.animate`, `audio.play`, etc.) based on current `time`. Calls `state.update()` and `render.draw(state.getShapes())`.

---

### `src/runtime/render.js`

*   **Purpose:** Implements the Canvas 2D rendering logic. Draws shapes based on calculated properties.
*   **Imports:** None.
*   **Exports:** `class Renderer { constructor(canvas); draw(shapes); }`
*   **Interface:**
    *   `constructor(canvas)`: Gets 2D context.
    *   `draw(shapes)`: Clears canvas. Iterates through `shapes` array (pre-sorted by Z? Or sort here?), draws each ellipse/polygon using calculated `x, y, color, rotation, points, rx, ry`. Applies transformations based on scene coordinates.

---

### `src/runtime/audio.js`

*   **Purpose:** Wrapper around `howler.js` to manage loading and playback of voice and music audio tracks.
*   **Imports:** `howler`.
*   **Exports:** `class AudioManager { load(id, path); play(id, ?volume); stop(id); }`
*   **Interface:** Provides methods to load audio files (referenced by IDs used in directives) and control playback.

---

### `src/runtime/evaluate.js`

*   **Purpose:** Parses and evaluates the simple arithmetic expressions found in artwork attribute values.
*   **Imports:** `mathjs` (or custom parser).
*   **Exports:** `function evaluate(expression, context)`
*   **Interface:**
    *   `evaluate(expression, context)`: Parses `expression` string. Uses variables from `context` (e.g., `context = { position: {...}, pose: {...}, expression: {...} }`).
    *   **Returns:** `number` (Result of the expression).

/comment(user:woe) {
    Let's settle on using a custom parser here.

    Also, does this really need to be evaluated at run-time? Can't we do all the math at build-time and then just let the player play it? See notes on an `Artwork` class above.
}

---

### `src/runtime/state.js`

*   **Purpose:** Manages the current state of all characters (position, current pose parameters, current expression parameters). Calculates interpolated values based on active `/move` and `/animate` directives. Provides the final shape definitions for rendering.
*   **Imports:** `./evaluate.js`, `./interpolate.js`.
*   **Exports:** `class StateManager { constructor(chars, backgrounds, poseDefs, exprDefs); move(...); animate(...); update(deltaTime); getShapes(); }`
*   **Interface:**
    *   `constructor(...)`: Stores initial character/background definitions and pose/expression parameter definitions.
    *   `move(charId, start, duration, endPos)`: Records a movement action.
    *   `animate(charId, start, duration, sequence)`: Records pose/expression changes.
    *   `update(deltaTime)`: Updates current time. Calculates interpolated positions, pose parameters, and expression parameters for all characters based on active actions and `deltaTime`.
    *   `getShapes()`: For each character/background, evaluates its artwork shape definitions using `evaluate()` with the current interpolated state (`position`, `pose`, `expression`). Returns the final list of calculated shapes ready for rendering.

---

### `src/runtime/interpolate.js`

*   **Purpose:** Provides simple linear interpolation functions.
*   **Imports:** None.
*   **Exports:** `function lerp(a, b, t)`
*   **Interface:** Standard linear interpolation.

---

This plan provides a clear structure for implementation, mapping directly to the components and interactions defined in `DESIGN.md`. Let the coding commence! 💻👻