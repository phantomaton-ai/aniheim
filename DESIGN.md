# Aniheim Rendering Engine - Design Document ✍️📐 (Revision 4)

This document details the design of the Aniheim Rendering Engine, specifying the interfaces, formats, and workflow required to fulfill the `SPECIFICATION.md`. Our goal is a **build system**, orchestrated by Node.js and leveraging `necronomicon` to execute `smarkup` directives, that transforms LLM-friendly source files (primarily Markdown) into self-contained HTML animations.

## 1. Architecture Overview

The engine operates as a command-line build tool (`node build.js <episode_file>`). It parses a primary Episode Markdown file, which primarily defines the sequence of scenes by linking to external Scene Markdown files. It recursively processes directives within these files (linking to characters, music, backgrounds), generates assets (voice audio, music audio/sheets, voice settings, artwork definitions) using an LLM-in-the-loop approach if they don't exist, and emits a single, self-contained HTML file for playback.

```mermaid
graph LR
    A[Episode.md] --> B{Build Engine};
    S[Scene.md] --> B; # Episode links to Scene
    C[Character.md] --> B; # Scene links to Character
    D[Music.md] --> B; # Scene links to Music
    E[Background.md] --> B; # Scene links to Background
    Art[Artwork.md] --> B; # Character/BG links to ArtworkDef
    F{LLM (Music/Voice/Art)} -- Generate --> G[Generated Assets (.abc, .mp3, .json, .md)];
    B -- Needs Asset? --> F;
    B -- Uses --> G;
    B --> H[Output.html];

    subgraph Source Files (.md)
        A; S; C; D; E; Art;
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

Invocation remains minimal:

```bash
node build.js <episode_markdown_file> [--output <output_html_file>]
```

*   `<episode_markdown_file>`: Path to the main `.md` file for the episode (Required).
*   `--output`: Path for the output HTML file (Defaults to `<episode_name>.html` next to the source).
*   Asset regeneration is handled manually by deleting the target generated file. Paths are relative.

## 3. Core File Formats (`.md`) 📄

Markdown is the primary format. Directives use `smarkup` default syntax: `/directive(attr:value)`. Attribute values cannot contain commas or parentheses. Files link to each other to manage complexity.

### 3.1. Episode Structure (`episode.md`)

Defines the sequence of scenes.

```markdown
# Episode 1 The Phantom Frege

This episode introduces Alice and Bob.

/scene(file: ./scene1_park_intro.md) {
    Alice meets Bob in the park. We learn they share an interest in language theory.
} scene!
/scene(file: ./scene2_frege_discussion.md) {
    Alice and Bob go to a coffee shop and discuss the work of Gottlob Frege.
} scene!
# ... more scenes
```

*   **Title:** Inferred from the top-level H1 heading.
*   `/scene`: Marks the start of a new scene and points to the Markdown file containing its detailed definition. Contains a short summary of the scene to aid in continuity and generation.

### 3.2. Scene Definition (`scene.md`)

Contains the detailed content of a single scene. Example: `scene1_park_intro.md`

```markdown
# Scene: Park Introduction (Duration ~15s)

/background(file: ../settings/a-public-park.md)

# Load Assets (IDs unique within scene)
/character(id: Alice, source: ../characters/alice.md, x: -0.2, y: 0.1, z: 0.5)
/character(id: Bob, source: ../characters/bob.md, x: 0.75, y: 0.15, z: 0.5)
/music(id: bgm, source: ../music/happy-day.md, action: play, start: 0, duration: 15)

# Animation & Dialog Timeline (start/duration in seconds)
/move(character: Alice, start: 0, duration: 5, x: 0.25, y: 0.1, z: 0.5)
/animate(character: Alice, start: 0, duration: 5) {
    🚶‍♀️🙂
} animate!
/dialog(character: Alice, start: 5, duration: 4, audio: ./audio/ep1_alice_01.mp3) {
    👋 Hi, Bob! 😊 Hows it going?
} dialog!
/animate(character: Bob, start: 5, duration: 1) {
    🤔
} animate!
/dialog(character: Bob, start: 9, duration: 4, audio: ./audio/ep1_bob_01.mp3) {
    Not bad, Alice. 🙂 How are you?
} dialog!
/animate(character: Alice, start: 9, duration: 4) {
    😊
}
```

*   Contains directives for background, characters, music, movement, animation, and dialog specific to this scene.
*   **Initial Pose:** Set by the first emoji in the first `/animate` or `/dialog` for that character within the scene.

### 3.3. Character Definition (`character.md`)

```markdown
# Alice

A cheerful programmer ghost 👻💻 with translucent blue skin and glowing circuit patterns.

/voice(settings: ./alice_voice.json) {
  A clear friendly female voice slightly synthesized or ethereal mid-range pitch Use Google TTS Wavenet-F with slight reverb
} voice!

/artwork(file: ./alice_artwork.md) {
  Key features translucent blue circle head simple rectangle body glowing lines for circuits Eyes are simple white ovals Allow standard expressions smile frown surprised Default pose is floating slightly
} artwork!
```

*   **Name:** Inferred from H1.
*   `/voice`: Generates/references TTS settings JSON. Body is LLM prompt.
*   `/artwork`: Points to an external file containing the artwork microformat definition. The body is an LLM prompt used to generate that file if it doesn't exist.

### 3.4. Music Definition (`music.md`)

```markdown
# Happy Day Theme

A light upbeat tune suitable for pleasant outdoor scenes.

/score(sheet: ./happy_day.abc, audio: ./happy_day.mp3) {
  Tempo 120bpm Key C-Major Instrument piano Mood cheerful simple loopable
} score!
```

*   `/score`: Defines the music asset.
    *   `sheet`: Path to the canonical music notation (ABC format preferred).
    *   `audio`: Path to the rendered audio file (MP3/OGG).
    *   **Generation:**
        1.  If `sheet` is missing, use LLM + body description to generate ABC content into the `sheet` file.
        2.  If `audio` is missing, render the `sheet` file (ABC -> MIDI -> Audio) and save to the `audio` path.

### 3.5. Background Definition (`background.md`)

```markdown
# A Public Park Setting

A simple park with grass a tree and a bench.

/artwork(file: ./park_artwork.md) {
    Layered 2D shapes Bottom layer solid green rectangle grass Middle layer brown rectangle trunk plus green circle canopy tree on the left Front layer simple brown polygon bench on the right Blue rectangle sky at top
} artwork!

```
* Uses `/artwork` directive pointing to an external file containing the artwork microformat definition. Body is LLM prompt for generation.

### 3.6. Artwork Definition (`artwork.md`)

Contains the actual artwork microformat directives. Example: `park_artwork.md`

```markdown
# Static shapes defined using Artwork Microformat
/polygon(id: grass, color: #00A000, rotation: 0, z: 1) {
    [-1 0] [1 0] [1 0.3] [-1 0.3]
} polygon!
/polygon(id: sky, color: #87CEEB, rotation: 0, z: 0.9) {
    [-1 0.3] [1 0.3] [1 1] [-1 1]
} polygon!
# ... tree bench etc ...
```
Example: `alice_artwork.md`
```markdown
# Body uses the Artwork Microformat (see Section 5)
# Defines shapes relative to the character anchor (position.x position.y)
# Can use pose.* and expression.* variables

/polygon(id: body, color: #A0A0A0AA, rotation: pose.torso.angle, z: 0.5) {
    [-0.05 -0.1] [0.05 -0.1] [0.05 0.1] [-0.05 0.1]
} polygon! # A simple rectangle body

/ellipse(id: head, color: #0000FFAA, x: 0, y: 0.15, rx: 0.05, ry: 0.05, rotation: pose.head.tilt, z: 0.6)

# Eyes - positions adjusted by expression
/ellipse(id: eyeL, color: white, x: -0.02, y: 0.17 + expression.eyes.offsetY, rx: 0.01 * expression.eyes.open, ry: 0.01 * expression.eyes.open, rotation: 0, z: 0.7)
/ellipse(id: eyeR, color: white, x: 0.02, y: 0.17 + expression.eyes.offsetY, rx: 0.01 * expression.eyes.open, ry: 0.01 * expression.eyes.open, rotation: 0, z: 0.7)

# Mouth - shape points potentially defined by expression parameters
/polygon(id: mouth, pointsExpr: expression.mouth.shape, color: white, x: 0, y: 0.13, rotation: 0, z: 0.7) {
    # Default points (neutral) if expression.mouth.shape is null/undefined? Or rely on expression defs.
    [-0.03 0] [0.03 0]
} polygon!
```


## 4. Pose & Expression Library (Emoji Microformat) 😀🚶‍♀️

Emoji map to named parameter sets applied to `pose.*` and `expression.*` variables in the artwork microformat.

*   **Mapping Files:**
    1.  `emoji_map.json`: `{"😀": "happy", "🚶‍♀️": "walk", "🤔": "thinking_pose", ...}`
    2.  `pose_definitions.json`: Defines parameter sets for poses.
    3.  `expression_definitions.json`: Defines parameter sets for expressions.
*   **Parameter Definition (Initial Minimal Set):**
    *   Uses `.` notation. Ranges are guides; exact interpretation depends on artwork definition.
    *   **Pose Parameters:**
        *   `pose.head.tilt`: (-1 right .. 0 center .. 1 left)
        *   `pose.torso.bend`: (-1 back .. 0 straight .. 1 forward)
        *   `pose.left.arm.lift`: (0 down .. 1 horizontal) - Shoulder lift
        *   `pose.left.arm.bend`: (0 straight .. 1 fully bent) - Elbow bend
        *   `pose.right.arm.lift`: (0 down .. 1 horizontal)
        *   `pose.right.arm.bend`: (0 straight .. 1 fully bent)
        *   *(Shoulder forward/back, leg movements deferred)*
    *   **Expression Parameters:**
        *   `expression.mouth.open`: (0 closed .. 1 wide open)
        *   `expression.mouth.smile`: (-1 frown .. 0 neutral .. 1 wide smile) - Affects corner elevation/shape
        *   `expression.eyes.open`: (0 closed .. 1 wide open) - Affects eye shape radius/height
        *   `expression.eyebrows.lift`: (-1 furrowed down .. 0 neutral .. 1 raised high) - Affects eyebrow vertical position/rotation
        *   `expression.mouth.shape`: (Optional: Key referencing point list for complex shapes, e.g., `'o_shape'`. Default: null)
*   **Timeline:** Parameters are linearly interpolated between states defined by emoji sequences over the directive's duration.

## 5. Artwork Representation & Rendering 🖼️

Defined declaratively in `artwork.md` files using `smarkup`-like directives.

*   **Coordinate System:**
    *   Normalized units based on **viewport height**. `1.0` = viewport height.
    *   `y`: `0.0` bottom, `1.0` top.
    *   `x`: `0.0` center. (`-aspectRatio/2` to `+aspectRatio/2`).
    *   `z`: `0.0` closest, `1.0` furthest.
    *   Shape directive coordinates (`x`, `y`) are **relative** to the character's anchor (`position.x`, `position.y`).
*   **Artwork Microformat Directives:**
    *   `/ellipse(id: string, x: expr, y: expr, z: expr, rx: expr, ry: expr, color: hex, rotation: expr)`
    *   `/polygon(id: string, x: expr, y: expr, z: expr, color: hex, rotation: expr, ?pointsExpr: var) { [x1 y1] [x2 y2] ... } polygon!` (Body defines default points, relative to shape x,y. `pointsExpr` can override with variable from expression defs).
*   **Attribute Expressions (`expr`):** Simple arithmetic: constants, `position.*`, `pose.*`, `expression.*`, `+`, `-`, `*`, `/`. Evaluated at runtime.
*   **Runtime Rendering:** Player evaluates expressions based on current state, passes calculated shape properties (final coords, color, rotation, points) to the rendering engine (e.g., Canvas 2D).

## 6. Music Rendering (ABC) 🎶

*   **Format:** ABC Notation (`.abc` files). Text-based, good for LLMs, standard tools exist.
*   **Generation:** Uses LLM prompt (from `/score` body) if `.abc` file is missing.
*   **Rendering:** Uses `abcjs` + `timidity++` (or similar SoundFont synth) triggered by the build process if the target `.mp3` is missing.

## 7. Voiceover Generation 🗣️

*   **Trigger:** `/dialog` directive specifies an `audio` file path.
*   **Workflow:**
    1. Build checks if `audio` file exists.
    2. If not:
        *   Load character's `/voice` settings JSON (generate JSON via LLM from description in `character.md` if missing).
        *   Clean dialog text (remove emoji).
        *   Call configured voice service API (e.g., Google TTS) with text and settings.
        *   Save output to the `audio` path.
*   **Requires:** Pre-configured API keys/credentials for the chosen voice service.

## 8. Build Process Flow (Revised) ⚙️➡️📄

1.  **Start**: `node build.js episode.md`
2.  **Parse Episode**: Parse `episode.md` using `smarkup` + `necronomicon`. Execute `/scene` directives.
3.  **Parse Scene**: For each scene file linked by `/scene`:
    *   Parse the `scene.md` file.
    *   Execute `/background`, `/character`, `/music` directives:
        *   Recursively parse linked `background.md`, `character.md`, `music.md`.
        *   Execute contained `/voice`, `/score`, `/artwork` directives.
            *   `/voice`/`/score`: Generate/validate assets (`.json`, `.abc`, `.mp3`) using LLM if files missing.
            *   `/artwork`: Generate/validate linked `artwork.md` file using LLM if missing. Parse the `artwork.md` microformat into a structured **shape definition list**. Store definition associated with character/background ID.
    *   Collate `/move`, `/animate`, `/dialog`, `/music (play/stop)` into a timed event list for the scene. Resolve `/dialog[audio]` paths (trigger TTS gen if needed). Extract emoji cues.
4.  **Code Generation**: Assemble data for runtime player:
    *   Character/Background definitions (incl. parsed shape definitions).
    *   Music definitions (path to audio).
    *   Timed event lists for all scenes.
    *   Pose/Expression definitions (from JSONs).
    *   Emoji map (from JSON).
5.  **HTML Emission**: Bundle player runtime JS, renderer JS, CSS, and episode data.

## 9. Output HTML Structure <html>

*   Single HTML file with CSS, canvas/container, bundled JS (player core, renderer, episode data), minimal UI controls (Play/Pause).

## 10. Technology Choices (Initial)

*   Build: Node.js, `smarkup`, `necronomicon`.
*   Music: ABC, `abcjs`, `timidity++`.
*   Voice: Cloud TTS SDKs.
*   LLM: `phantomaton-gemini` or similar.
*   Audio (Runtime): `howler.js`.
*   Renderer (Runtime): Canvas 2D via swappable module.
*   Expression Evaluation (Runtime): Simple math expression parser (e.g., `mathjs` subset or custom).
