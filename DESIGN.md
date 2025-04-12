# Aniheim Rendering Engine - Design Document ✍️📐 (Revision 3)

This document details the design of the Aniheim Rendering Engine, specifying the interfaces, formats, and workflow required to fulfill the `SPECIFICATION.md`. Our goal is a **build system**, orchestrated by Node.js and leveraging `necronomicon` to execute `smarkup` directives, that transforms LLM-friendly source files (primarily Markdown) into self-contained HTML animations.

## 1. Architecture Overview

The engine operates as a command-line build tool (`node build.js <episode_file>`). It parses a primary Episode Markdown file, recursively processes directives that link to external Markdown files (characters, music, backgrounds), generates assets (voice audio, music audio/sheets, voice settings, potentially artwork render functions) using an LLM-in-the-loop approach if they don't exist, and emits a single, self-contained HTML file for playback.

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

Invocation remains minimal:

```bash
node build.js <episode_markdown_file> [--output <output_html_file>]
```

*   `<episode_markdown_file>`: Path to the main `.md` file for the episode (Required).
*   `--output`: Path for the output HTML file (Defaults to `<episode_name>.html` next to the source).
*   Asset regeneration is handled manually by deleting the target generated file. Paths are relative.

## 3. Core File Formats (`.md`) 📄

Markdown is the primary format. Directives use `smarkup` default syntax: `/directive(attr:value)`. Note `smarkup` limitation: attribute values cannot contain commas or parentheses.

### 3.1. Episode Structure (`episode.md`)

```markdown
# Episode 1 The Phantom Frege

This is narrative text in Markdown.

/scene(duration: 30, background: ../settings/a-public-park.md)

    # Load Assets for this Scene (IDs must be unique within the scene)
    /character(id: Alice, source: ../characters/alice.md, initialX: -0.2, initialY: 0.1, initialZ: 0.5)
    /character(id: Bob, source: ../characters/bob.md, initialX: 0.75, initialY: 0.15, initialZ: 0.5)
    /music(id: bgm, source: ../music/happy-day.md, action: play, start: 0, volume: 0.8)

    # Animation & Dialog Timeline (start/duration in seconds)
    /move(character: Alice, start: 0, duration: 5, endX: 0.25, endY: 0.1, endZ: 0.5)
    /animate(character: Alice, start: 0, duration: 5) { 🚶‍♀️🙂 } # Walk in neutral. Initial pose/expression set here.
    /dialog(character: Alice, start: 5, duration: 4, audio: ./audio/ep1_alice_01.mp3) { 👋 Hi Bob! 😊 Hows it going } # Wave then smile. Note: Text for TTS cannot contain markup/emoji.
    /animate(character: Bob, start: 5, duration: 1) { 🤔 } # Bob turns head? (Pose)
    /dialog(character: Bob, start: 9, duration: 4, audio: ./audio/ep1_bob_01.mp3) { Not bad Alice 🙂 How are you }
    /animate(character: Alice, start: 9, duration: 4) { 😊 } # Keep smiling

    /music(id: bgm, action: stop, start: 29)

/scene! # Optional end marker

# More scenes can follow...
```

/comment(user:woe) {
    Getting close, but nested directives (e.g. all those things in a scene) aren't supported by Smarkup. We'll have to get away with using a single-line `/scene` to establish a new scene boundary.
    
    Also, it's a nit, but `x` `y` and `z` for coordinate names please. The "initial" and "end" aspects can be inferred from context and/or explained in documentation.
}

*   **Title:** Inferred from the top-level H1 heading.
*   **Coordinates:** `initialX`, `initialY`, `initialZ`, `endX`, `endY`, `endZ`. System defined in Section 5.
*   **Initial Pose:** Not an attribute; set by the first emoji in the first `/animate` or `/dialog` for that character.

### 3.2. Character Definition (`character.md`)

```markdown
# Alice

A cheerful programmer ghost 👻💻 with translucent blue skin and glowing circuit patterns.

/voice(settings: ./alice_voice.json) {
  A clear friendly female voice slightly synthesized or ethereal mid-range pitch Use Google TTS Wavenet-F with slight reverb
} voice!

/artwork() {
  # Body uses the Artwork Microformat (see Section 5)
  # Defines shapes relative to the character anchor (position.x position.y)
  # Can use pose.* and expression.* variables

  /polygon(id: body, color: #A0A0A0AA, z: 0) {
    [-0.05 -0.1] [0.05 -0.1] [0.05 0.1] [-0.05 0.1]
  } polygon! # A simple rectangle body

  /ellipse(id: head, color: #0000FFAA, x: 0, y: 0.15, rx: 0.05, ry: 0.05, z: 0.1)

  # Eyes - positions adjusted by expression.eyeOffset.y perhaps?
  /circle(id: eyeL, color: white, x: -0.02, y: 0.17 + expression.eyeOffsetY, radius: 0.01, z: 0.2)
  /circle(id: eyeR, color: white, x: 0.02, y: 0.17 + expression.eyeOffsetY, radius: 0.01, z: 0.2)

  # Mouth - shape points defined by expression.mouthShape
  /polygon(id: mouth, points: expression.mouthShape, fill: none, stroke: white, lineWidth: 0.002, z: 0.2)

} artwork!

```

/comment(user:woe) {
    Again, nested directives are unsupported. We can simulate that by doing:

    /artwork(file:artwork.md) {
        A pig wearing lipstick.
    } artwork!

    ...and then having the "nested" directives interpreted within that file.

    This is also a little convenient insofar as it minimizes the number of directives we need to introduce in any given context. We can also support LLM-driven-generation-as-needed pretty easily with this approach.

    (You know, let's consider handling `/scene` the same way in the next update. A briefer scene summary should be sufficient to give a sense of the fully story of an episode; the contents can include specific dialog, but we can omit things like music at the episode-level, balancing completeness and conciseness a little better.)
}

*   **Name:** Inferred from H1.
*   `/voice`: Generates/references TTS settings JSON. Body is LLM prompt.
*   `/artwork`: No attributes. Body contains the character's visual definition using the **Artwork Microformat** (Section 5). This defines the base shapes and how they react to `pose` and `expression` variables. No external JS renderer needed by default.

### 3.3. Music Definition (`music.md`)

```markdown
# Happy Day Theme

A light upbeat tune suitable for pleasant outdoor scenes.

/score(sheet: ./happy_day.abc, audio: ./happy_day.mp3) {
  Tempo 120bpm Key C-Major Instrument piano Mood cheerful simple loopable
} score!
```

*   Generates/references ABC sheet and rendered audio. Body is LLM prompt.

/comment(user:woe) {
    100% fabulous! I think this is a good section to refer back to for other content generation types. Music generation has a lot of complexity to it, but we've made it so simple here.
}

### 3.4. Background Definition (`background.md`)

```markdown
# A Public Park Setting

A simple park with grass a tree and a bench.

/artwork() {
  # Static shapes defined using Artwork Microformat
  /polygon(id: grass, color: #00A000, z: 1) { [-1 0] [1 0] [1 0.3] [-1 0.3] } polygon!
  /polygon(id: sky, color: #87CEEB, z: 0.9) { [-1 0.3] [1 0.3] [1 1] [-1 1] } polygon!
  # ... tree bench etc ...
} artwork!
```
* Uses the same `/artwork` directive and microformat as characters, but typically without pose/expression variables.

/comment(user:woe) {
    Love it! Excellent use of microformats for the polygon points. One note is that bodies can't appear on the same line in Smarkup, so the example should be more like:

    /polygon(id: grass, color: #00A000, z: 1) {
        [-1 0] [1 0] [1 0.3] [-1 0.3]
    } polygon!
}

## 4. Pose & Expression Library (Emoji Microformat) 😀🚶‍♀️

Emoji in `/dialog` and `/animate` bodies map to named poses/expressions, which translate to parameter sets used by the `/artwork` microformat.

*   **Mapping:**
    1.  `emoji_map.json`: `{"😀": "happy", "🚶‍♀️": "walk", "🤔": "thinking_pose", ...}`
    2.  `pose_definitions.json`: `{"walk": { "leftLegAngle": 45, "rightLegAngle": -45, ... }, "idle": { ... }}`
    3.  `expression_definitions.json`: `{"happy": { "mouthShape": "smile", "eyeOffsetY": 0.005, ... }, "neutral": { ... }}`
    *   *(These JSON files define the standard library. Characters could potentially override/extend)*
*   **Interpretation:**
    *   Parser extracts emoji -> Looks up name in `emoji_map.json`.
    *   Runtime fetches parameter set from `pose/expression_definitions.json` based on name.
    *   These parameters populate the `pose.*` and `expression.*` variables available in the `/artwork` microformat.
*   **Parameter Examples (Conceptual - need refinement):**
    *   **Pose:** `pose.torso.angle`, `pose.head.tilt`, `pose.leftArm.shoulder.angle`, `pose.leftArm.elbow.angle`, `pose.rightLeg.knee.angle`, etc. (Angles in degrees).
    *   **Expression:** `expression.mouthShape` (key for predefined polygon points like `'smile'`, `'frown'`, `'o_shape'`), `expression.eyeShape` (key like `'normal'`, `'wide'`, `'squint'`), `expression.eyebrowLeft.angle`, `expression.eyebrowRight.angle`, `expression.eyeOffsetY`.
*   **Timeline:** Changes occur linearly interpolated over the duration specified by the number of emoji vs the directive duration. `/animate(duration: 2) { 🤔 V }` -> Apply "thinking_pose" params over 1s, then "peace_sign" params over the next 1s.

/comment(user:woe) {
    This is heading in the right direction. We want a fully-documented pose and expression system, though, like:

    pose:
        left:
            ankle:
                bend: -1...1 # -1 = all the way back, 0 = straight, 1 = all the way forward
            calf:
                twist: -1...1 # -1 = twisted max left, 1 = twisted max right
            knee:
                bend: 0...1  # 0 = straight, 0.5 = 90-degree bend, 1.0 = fully bent
            thigh:
                twist: -1...1 # -1 = twisted max left, 1 = twisted max right
        right:
            ...
    expression:
        left:
            eyebrow:
                elevation: 0...1 # Lifted brow
                rotation: -1...1 # -1 rotated outward, 0 flat, 1 rotated inward
                width: 0...1     # 0 = narrow, e.g. scrunched, 1 = full width
            lip:
                corner:
                    elevation: -1...1  # -1 frown, 0 neutral, 1 smiling
                    roundness: 0...1   # 0 = open-mouthed, 1 = smiling/frowning
        right:
            ...
        mouth:
            height: 0...1 # 0=closed, 1=wide open
            width: 0...1 # 0=pursed lips, 1=wide smile/frown/etc
        nose:
            height: 0...1        # 0 = fully scrunched, 1 = regular

    We'll want to strike a challenging balance between providing enough specificity to capture a wide range of poses, without overwhelming ourselves with too many parameters. The key things that matter in the example above:

    * Single-word variable names are preferred; `foo.bar` instead of `fooBar` so that we can group `foo` things easily
    * Use a simple numbering system that applies directly to the joint being controlled. We don't want to be thinking in angles all the time; we want to be thinking in terms of the way an arm is positioned as it waves.
    * Avoid enums like `mouthShape`; let's minimize the amount of prior knowledge implementations will need to show up with.
    * Document the semantics of each parameter in plain, physical terms. Make it easy for the LLM to understand the body and/or face it is positioning.

    I'm realizing there is a fair amount of latent complexity in this space. Let's find a good, initial, minimal scope to work with here, then consider refactoring out into its own package later.
}

## 5. Artwork Representation & Rendering 🖼️

Artwork is defined **declaratively** within `/artwork` bodies using a `smarkup`-like microformat.

*   **Coordinate System:**
    *   Normalized units based on **viewport height**. `1.0` = viewport height in pixels.
    *   `y`: `0.0` is bottom edge, `1.0` is top edge.
    *   `x`: `0.0` is center. Range depends on aspect ratio (e.g., `-aspectRatio/2` to `+aspectRatio/2`).
    *   `z`: `0.0` is closest, `1.0` is furthest. Used for layering.
    *   Coordinates within shape directives (`/circle`, `/polygon`) are **relative** to the character's anchor point (`position.x`, `position.y`).
*   **Artwork Microformat Directives:**
    *   `/circle(id: string, x: expr, y: expr, z: expr, radius: expr, color: hex, ?rotation: expr, ?fill: hex|none, ?stroke: hex, ?lineWidth: expr)`
    *   `/ellipse(id: string, x: expr, y: expr, z: expr, rx: expr, ry: expr, color: hex, ?rotation: expr, ?fill: hex|none, ?stroke: hex, ?lineWidth: expr)`
    *   `/polygon(id: string, x: expr, y: expr, z: expr, color: hex, ?rotation: expr, ?fill: hex|none, ?stroke: hex, ?lineWidth: expr) { [x1 y1] [x2 y2] ... } polygon!` (Points are relative to shape's x,y)
    *   *(Maybe `/image(id: string, src: path, x: expr, y: expr, z: expr, width: expr, height: expr, ?rotation: expr)` ? Deferred.)*
*   **Attribute Expressions (`expr`):** Allow simple arithmetic using:
    *   Constants (e.g., `0.1`).
    *   `position.x`, `position.y`, `position.z` (Character's current position).
    *   `pose.<param>` (e.g., `pose.leftArm.shoulder.angle`).
    *   `expression.<param>` (e.g., `expression.mouthShape`, `expression.eyeOffsetY`).
    *   Basic operators: `+`, `-`, `*`, `/` (evaluated at runtime).
*   **Runtime Rendering:**
    1.  Build process parses the `/artwork` microformat into a structured definition (list of shapes with expression-based attributes).
    2.  Runtime player evaluates the attribute expressions for each shape based on the character's current `position`, `pose`, and `expression` state for the frame.
    3.  The chosen rendering engine (Canvas2D, etc.) receives the fully calculated shape properties (positions, colors, rotations) for drawing.
*   **Runtime Renderer Interface:**
    *   `init(containerElement)`
    *   `renderFrame(calculatedShapes)`: Takes list of objects, each with final `id`, `type`, `color`, `x`, `y`, `z`, `rotation`, `points`, `radius`, etc. after expression evaluation.

/comment(user:woe) {
    Drop `/circle` (redundant to ellipse) and the  `fill`, `stroke`, and `lineWidth` attributes; we'll just draw solid shapes for now.

    `rotation` should be required for consistency.

    No need for the runtime renderer interface at this level, let's omit that for now.
}

## 6. Music Rendering (ABC) 🎶

*   Unchanged: Uses LLM for ABC generation from `/score` body if needed, renders ABC to audio via `abcjs`/synth if needed.

/comment(user:woe) {
    Please restore content to this section. I understand that it's unchanged, but we've effectively deleted it by saying "unchanged"
}

## 7. Voiceover Generation 🗣️

*   Unchanged: Uses LLM for voice settings generation from `/voice` body if needed, calls TTS service if `/dialog` audio file is missing.

/comment(user:woe) {
    Please restore content to this section (as above)
}

## 8. Build Process Flow (Revised) ⚙️➡️📄

1.  **Start**: `node build.js episode.md`
2.  **Parse Entry Point**: Use `smarkup` + `necronomicon`.
3.  **Directive Execution (Build Time)**:
    *   `/scene`: Define scope.
    *   `/character`, `/music`, `/background`: Recursively parse linked `.md`.
        *   Execute contained `/voice`, `/score` to generate/validate assets (`.json`, `.abc`, `.mp3`) using LLM helpers if files are missing.
        *   Parse contained `/artwork` microformat into a structured **shape definition list** for the character/background. Store this definition.
    *   `/move`, `/animate`, `/dialog`, `/music (play/stop)`: Collate into timed event list. Resolve `/dialog[audio]` paths (trigger TTS gen if needed). Extract emoji cues.
4.  **Code Generation**: Assemble data for runtime player:
    *   Character/Background definitions (incl. parsed shape definitions).
    *   Music definitions (path to audio).
    *   Timed event list.
    *   Pose/Expression definitions (from JSONs).
    *   Emoji map (from JSON).
5.  **HTML Emission**: Bundle player runtime JS, renderer JS, CSS, and episode data.

## 9. Output HTML Structure <html>

*   Unchanged: Single file with CSS, canvas/container, bundled JS (player core, renderer, episode data), minimal UI.

## 10. Technology Choices (Initial)

*   Build: Node.js, `smarkup`, `necronomicon`.
*   Music: ABC, `abcjs`, `timidity++`.
*   Voice: Cloud TTS SDKs.
*   LLM: `phantomaton-gemini` or similar.
*   Audio (Runtime): `howler.js`.
*   Renderer (Runtime): Canvas 2D via swappable module.
*   **Expression Evaluation (Runtime):** A simple math expression parser library (e.g., `mathjs` subset or custom).
