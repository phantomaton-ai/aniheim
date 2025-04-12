# Aniheim Rendering Engine - Design Document ✍️📐 (Revision 2)

This document details the design of the Aniheim Rendering Engine, specifying the interfaces, formats, and workflow required to fulfill the `SPECIFICATION.md`. Our goal is a **build system**, orchestrated by Node.js and leveraging `necronomicon` to execute `smarkup` directives, that transforms LLM-friendly source files (primarily Markdown) into self-contained HTML animations.

## 1. Architecture Overview

The engine operates as a command-line build tool (`node build.js <episode_file>`). It parses a primary Episode Markdown file, recursively processes directives that link to external Markdown files (characters, music, backgrounds), generates assets (voice audio, music audio/sheets, voice settings) using an LLM-in-the-loop approach if they don't exist, and emits a single, self-contained HTML file for playback.

```mermaid
graph LR
    A[Episode.md] --> B{Build Engine (Node.js + Necronomicon)};
    C[Character.md] --> B;
    D[Music.md] --> B;
    E[Background.md] --> B;
    F{LLM (Music/Voice/Art)} -- Generate --> G[Generated Assets (.abc, .mp3, .json, .js)];
    B -- Needs Asset? --> F;
    B -- Uses --> G;
    B --> H[Output.html];

    subgraph Source Files (.md)
        A; C; D; E;
    end
    subgraph External Services
        F;
    end
    subgraph Build Process
        B;
    end
    subgraph Generated/Cached Assets
        G;
    end
    subgraph Output
        H;
    end
```

## 2. Command-Line Interface (CLI) 💻

Invocation is simplified:

```bash
node build.js <episode_markdown_file> [--output <output_html_file>]
```

*   `<episode_markdown_file>`: Path to the main `.md` file for the episode (Required).
*   `--output`: Path for the output HTML file (Defaults to `<episode_name>.html` next to the source).
*   Asset regeneration is handled manually by deleting the target generated file (e.g., deleting `./audio/alice_line_1.mp3` triggers regeneration on the next build). Asset library paths are implicitly relative to the file containing the directive.

## 3. Core File Formats (`.md`) 📄

Markdown is the primary format for episodes, characters, music definitions, and backgrounds, enabling reusability and LLM familiarity. Directives follow the `smarkup` default syntax: `/directive(attr:value) { body } directive!`.

### 3.1. Episode Structure (`episode.md`)

Contains the overall narrative flow and scene composition.

*   **Top-Level:** May contain metadata directives (e.g., `/title(Episode 1: The Phantom Frege)`).
*   **Scene Definition:** Episodes are composed of sequential `/scene` directives.
    ```markdown
    /scene(duration: 30, background: '../settings/a-public-park.md')

    # Scene Content - directives are processed based on timing attributes

    # Load Assets for this Scene
    /character(id: Alice, source: '../characters/alice.md', initialPos: [-0.2, 0.1, 0.5], initialPose: 'idle')
    /character(id: Bob, source: '../characters/bob.md', initialPos: [0.75, 0.15, 0.5], initialPose: 'idle')
    /music(id: bgm, source: '../music/happy-day.md', action: 'play', start: 0, volume: 0.8)

    # Animation & Dialog Timeline
    /move(character: Alice, start: 0, duration: 5, endPos: [0.25, 0.1, 0.5])
    /animate(character: Alice, start: 0, duration: 5) { 🚶‍♀️🙂 } # Walk in, look neutral
    /dialog(character: Alice, start: 5, duration: 4, audio: './audio/ep1_alice_01.mp3') { 👋 Hi Bob! 😊 How's it going? } # Wave, then smile. Audio generation triggered if file missing.
    /animate(character: Bob, start: 5, duration: 1) { 🤔 } # Bob turns head? (Pose)
    /dialog(character: Bob, start: 9, duration: 4, audio: './audio/ep1_bob_01.mp3') { Not bad, Alice! 🙂 How are you? }
    /animate(character: Alice, start: 9, duration: 4) { 😊 } # Keep smiling while Bob talks

    # Stop Music at end of scene (optional)
    /music(id: bgm, action: 'stop', start: 29)

    /scene! # End of scene block (optional marker, parsing stops at next /scene or EOF)
    ```
*   **Scene Space:** Coordinates (`initialPos`, `endPos`, etc.) use a normalized system:
    *   `x`: -1 (left edge) to +1 (right edge)
    *   `y`: 0 (bottom edge) to 1 (top edge) - Maybe adjust based on aspect ratio? Let's start with 0-1.
    *   `z`: 0 (closest to camera) to 1 (furthest). Default character plane might be 0.5.
    *   *(Design Note: LLM interaction with numeric coords needs monitoring. Future: symbolic placement like `pos: near(Bob)`)*
*   **Timing:** `start` and `duration` attributes (in seconds) control the timeline within a scene.

/comment(user:woe) {
    Great! Continually improving here. Note that attributes supported by Smarkup are still pretty janky; they can't contain commas or parentheses, for example, due to the weak parsing. To work around that:

    * Split up arrays (no `initialPos` because of the commas in the value; `x`, `y`, and `z`)
    * Prefer to lean on the semantics of Markup when convenient
        * For example, we could infer the title from the top-level section name (so that we don't break when we try to give an episode a name with a comma in it)
        * This also reduces the number of directives we need to invent
    * Remove any quotes from examples; assume things like filenames are comma/paren-free

    Additionally, I like `initialPose`, but it can be made redundant by just including some emoji at the start of a dialog body. Let's prune it.
}

### 3.2. Character Definition (`character.md`)

Defines a character's appearance, voice, and potentially default behaviors.

```markdown
# Alice

A cheerful programmer ghost 👻💻 with translucent blue skin and glowing circuit patterns.

/voice(settings: './alice_voice.json') {
  A clear, friendly female voice, slightly synthesized or ethereal, mid-range pitch. Use Google TTS Wavenet-F with slight reverb.
} voice!

/artwork(renderer: './alice_renderer.js') {
  Render as layered 2D shapes. Key features: translucent blue circle head, simple rectangle body, glowing lines for circuits. Eyes are simple white ovals. Allow standard expressions (smile, frown, surprised). Default pose is floating slightly.
} artwork!

# Optional: Default pose/expression
/default(pose: 'idle', expression: 'neutral')
```
/comment(user:woe) {
    nit: "Render as layered 2D shapes" doesn't need to be specified, since we'll always be rendering that way.
}

*   **Name:** Inferred from the H1 heading (`# Alice`).
*   `/voice`: Specifies voice characteristics.
    *   `settings`: Path to a JSON file containing specific TTS parameters (e.g., `{ "provider": "google", "voiceId": "en-US-Wavenet-F", "pitch": 0, "effects": ["reverb_small"] }`).
    *   **Generation:** If `settings` file doesn't exist, the build tool uses the body description (via an LLM helper, like `povgoblin`) to generate the JSON content.
*   `/artwork`: Specifies visual appearance.
    *   `renderer`: Path to a JS module responsible for translating pose/expression requests into the standard shape array format (see Section 5).
    *   **Generation:** If `renderer` file doesn't exist, an LLM helper *could* attempt to generate a basic renderer based on the body description, outputting simple shapes. (Complexity: High - maybe start with requiring this file manually).
*   **Trace:** `SPEC: 🧑‍🎨 Character Representation`, `SPEC: 🗣️ Voice Performance`, `SPEC: 🖼️ Artwork Integration`

### 3.3. Music Definition (`music.md`)

Defines a reusable piece of music.

```markdown
# Happy Day Theme

A light, upbeat tune suitable for pleasant outdoor scenes.

/score(sheet: './happy_day.abc', audio: './happy_day.mp3') {
  Tempo: 120bpm
  Key: C Major
  Instrumentation: Piano main melody, light strings accompaniment.
  Mood: Cheerful, simple, slightly whimsical.
  Structure: AABA, approx 30 seconds loopable.
} score!
```

*   `/score`: Defines the music asset.
    *   `sheet`: Path to the canonical music notation (ABC format preferred).
    *   `audio`: Path to the rendered audio file (MP3/OGG).
    *   **Generation:**
        1.  If `sheet` is missing, use LLM + body description to generate ABC content into the `sheet` file.
        2.  If `audio` is missing, render the `sheet` file (ABC -> MIDI -> Audio) and save to the `audio` path.
*   **Trace:** `SPEC: 🎶 Music Integration`

### 3.4. Background Definition (`background.md`)

Defines a reusable scene background using artwork directives.

```markdown
# A Public Park Setting

A simple park with grass, a tree, and a bench.

/artwork(renderer: './park_renderer.js') {
  Layered 2D shapes. Bottom layer: solid green rectangle (grass). Middle layer: brown rectangle trunk + green circle canopy (tree) on the left. Front layer: simple brown polygon bench on the right. Blue rectangle sky at top.
} artwork!
```

*   `/artwork`: Similar to characters, defines the visual elements. Static backgrounds might use simpler renderers or directly embed shape data.
*   **Trace:** `SPEC: 📖 Story Structure & Scenes`

## 4. Pose & Expression Library (Emoji Microformat) 😀🚶‍♀️

Poses and expressions are primarily controlled via standard Unicode emoji within `/dialog` and `/animate` directive bodies.

*   **Mapping:** A global configuration (e.g., `pose_map.json`) maps specific emoji to:
    *   **Pose Keys:** e.g., `🚶‍♀️` -> `'walk'`, `🧍` -> `'idle'`, `👋` -> `'wave'`. These keys are passed to the character's artwork renderer.
    *   **Expression Keys:** e.g., `🙂` -> `'neutral'`, `😀` -> `'happy'`, `😢` -> `'sad'`, `🤔` -> `'thinking'`. These keys are also passed to the renderer.
*   **Interpretation:**
    *   The parser extracts emoji sequences.
    *   The runtime player applies the corresponding pose/expression change when the emoji is encountered in the timeline associated with the directive's duration.
    *   Multiple emoji in a sequence imply a sequence of changes within the directive's duration. `/animate(duration: 2) { 🤔 V }` -> Thinking for 1s, Peace sign for 1s.
*   **Extensibility:** Character artwork renderers can implement custom mappings or handle standard keys appropriately. The global map defines engine defaults.
*   **Trace:** `SPEC: 🕺 Animation Logic`

/comment(user:woe) {
    Let's try to design our interface for representing individual poses and expressions here. Both will be sets of well-defined parameters for controlling the rendering of a character; poses will control the relative orientation of limbs, and expressions will control the relative size, position, and orientation of facial elements. Let's go ahead and document all of our parameters here, as well as the coordinate system we'll work with.
}

## 5. Artwork Representation & Rendering 🖼️

*   **Data Format (Runtime):** Character/Background `/artwork` renderers (`.js` modules) must ultimately provide data as an array of shape objects when called by the player.
    ```javascript
    // Example output from alice_renderer.js for getArtworkData('idle', 'happy')
    [
      // Shapes sorted by Z-index by renderer? Or player? Player responsibility.
      { id: 'body', type: 'polygon', points: [[-5,-25],[5,-25],[5,25],[-5,25]], color: '#A0A0A0AA', x: 0, y: 0, z: 0.5, rotation: 0 }, // Body - relative coords
      { id: 'head', type: 'ellipse', color: '#0000FFAA', rx: 15, ry: 15, x: 0, y: 35, z: 0.51 }, // Head
      { id: 'eyeL', type: 'circle', color: '#FFFFFF', radius: 3, x: -5, y: 40, z: 0.52 }, // Left Eye (relative to head's center?) - Coordinate system needs refinement. Let's assume relative to character anchor for now.
      { id: 'eyeR', type: 'circle', color: '#FFFFFF', radius: 3, x: 5, y: 40, z: 0.52 }, // Right Eye
      { id: 'mouth', type: 'polygon', points: [[-5, 30], [0, 32], [5, 30]], fill: 'none', stroke: '#FFFFFF', lineWidth: 1, x: 0, y: 0, z: 0.52 } // Simple smile line
      // Clipping shapes TBD. Rotation anchor implicit (shape's x,y)?
    ]
    ```
    *   **Shapes:** `circle`, `ellipse`, `polygon`.
    *   **Properties:** `id` (optional string), `type`, `color` (CSS format, includes alpha), `x`, `y`, `z` (relative position), `rotation` (degrees, optional), shape-specific props (`radius`, `rx`, `ry`, `points`), drawing style (`fill`, `stroke`, `lineWidth`, optional).
*   **LLM Bridge:** The *body* of the `/artwork` directive provides the semantic description. The associated `renderer.js` file is the **crucial bridge**, responsible for interpreting this description (or specific pose/expression keys) and generating the above shape array. Simple renderers might ignore the description and just use pose keys. Complex ones might use the description to modify base shapes.
*   **Runtime Renderer Interface:** The player uses a swappable rendering module (Canvas2D, WebGL, SVG) with an interface like:
    *   `init(containerElement)`
    *   `renderFrame(shapeArrays)`: Takes a list of shape arrays (one per character/background element) and draws them. Handles layering (z-index), positioning (baseX/Y from directives), and scaling.
    *   Initial implementation: Canvas 2D.
*   **Trace:** `SPEC: 🖼️ Artwork Integration`, `SPEC: 🛠️ Technology Stack (Rendering Interface)`

/comment(user:woe) {
    Looking at the output format, I'm realizing this looks a lot like a list of directives:

    /polygon(id:body,color:#A0A0A0AA,...etc) {
        [-5,-25],[5,-25],[5,25],[-5,25]
    } polygon!

    Given that we just want to change the position of these based on pose and expression, maybe we should instead gravitate toward microformats with some basic arithmetic support, e.g.:

    /circle(id:foo,x:position.x+4,radius:0.2,rotation:pose.shoulder.left.rotation)

    We should also define our coordinate space. Let's assume that 1.0 is equivalent to the height of the display in pixels.
}

## 6. Music Rendering (ABC) 🎶

*   **Format:** ABC Notation (`.abc`).
*   **Generation:** Uses LLM prompt (from `/score` body) if `.abc` file is missing.
*   **Rendering:** Uses `abcjs` + `timidity++` (or similar SoundFont synth) triggered by the build process if the target `.mp3` is missing.
*   **Trace:** `SPEC: 🎶 Music Integration`

## 7. Voiceover Generation 🗣️

*   **Trigger:** `/dialog` directive specifies an `audio` file path.
*   **Workflow:**
    1. Build checks if `audio` file exists.
    2. If not:
        *   Load character's `/voice` settings JSON (generate JSON via LLM from description if missing).
        *   Clean dialog text (remove emoji).
        *   Call configured voice service API (e.g., Google TTS) with text and settings.
        *   Save output to the `audio` path.
*   **Requires:** Pre-configured API keys/credentials for the chosen voice service.
*   **Trace:** `SPEC: 🗣️ Voice Performance`

## 8. Build Process Flow (Simplified) ⚙️➡️📄

1.  **Start**: `node build.js episode.md`
2.  **Parse Entry Point**: Use `smarkup` + `necronomicon` to parse `episode.md`.
3.  **Directive Execution (Build Time)**: `necronomicon` executes directives:
    *   `/scene`: Defines scope for timeline processing.
    *   `/character`, `/music`, `/background`: Recursively parse linked `.md` files, caching definitions. Execute contained `/voice`, `/artwork`, `/score` directives to ensure assets (JSON settings, renderers, ABC, MP3) are generated *if missing*, potentially calling LLM helpers. Store paths to required assets.
    *   `/move`, `/animate`, `/dialog`, `/music (play/stop)`: Collate these into a timed event list associated with the current scene. For `/dialog`, ensure the `audio` path is resolved (triggering generation if needed). Extract emoji cues from `/dialog`, `/animate`.
4.  **Code Generation**: Assemble data for the runtime player:
    *   Character definitions (paths to renderers, voice settings).
    *   Background definitions.
    *   Music definitions (path to audio).
    *   The timed event list for all scenes.
    *   Global pose/expression emoji map.
5.  **HTML Emission**: Bundle player runtime JS, renderer JS (e.g., Canvas 2D), CSS, and the generated episode data into a single `.html` file.

## 9. Output HTML Structure <html>

*   Single HTML file.
*   CSS for player UI, subtitles.
*   `<canvas>` (or other) element.
*   Bundled JavaScript:
    *   Player Core (event loop, state management, audio handling via Howler.js?).
    *   Renderer Implementation (Canvas 2D initially).
    *   Episode Data (event list, asset paths/data).
    *   Minimal UI controls (Play/Pause, maybe timeline scrub).

## 10. Technology Choices (Initial)

*   **Build Orchestration**: Node.js.
*   **Directive Parsing/Execution**: `smarkup`, `necronomicon`.
*   **Music Notation**: ABC (`.abc`).
*   **Music Rendering**: `abcjs` + `timidity++`/SoundFont wrapper.
*   **Voice Generation**: Cloud TTS SDKs (Google, AWS, etc.).
*   **LLM Integration**: Use libraries like `phantomaton-gemini` (seen in `povgoblin`) for supervised generation loops.
*   **Audio Playback (Runtime)**: `howler.js` recommended for better control over audio synchronization.
*   **Renderer (Runtime)**: HTML Canvas 2D API initially, via swappable module.

This revised design embraces modularity via Markdown files, simplifies configuration, clarifies the LLM-driven generation process for assets, and introduces emoji for intuitive animation control. Ready for the next stage of conjuration, Dr. Woe! 🪄✨👻