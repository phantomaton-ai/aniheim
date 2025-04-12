# Aniheim Rendering Engine - Specification 👻📄

This document specifies the capabilities and formats for the Aniheim Rendering Engine, tracing back to our nefarious `REQUIREMENTS.md`.

## 1. Goal 🎯

To provide a web-based playback engine for animated episodes defined by plain-text and JavaScript sources, primarily intended for generation by language models. The engine prioritizes simplicity and reproducibility for preview purposes.

/comment(user:woe) {
    Beautiful! The correct information is here and it's very concise. There's an image which this clarifies for me (which I think we could emphasize even further) where there's an input side (belonging to the LLM, or tuned for the LLM) and an output side (the rendered animation). The key thing here is that animation is not a "native language" for LLMs, so we need to bridge that gap and present it in a way which feels fluent to them.
}

## 2. Functional Specifications ✨

### ▶️ Episode Playback

*   **Spec:** The system provides a single-page web application (SPA) served by a Node.js backend. This page displays the selected animated episode.
*   **Trace:** `REQUIREMENTS.md: ▶️ Episode Playback`

/comment(user:woe) {
    I like that this is scoped to single episodes. We'll want series/episode navigation at some point, maybe, but we can start with a smaller scope
}

### 📖 Story Structure & Scenes

*   **Spec:** Episodes are defined using Markdown files (`.md`).
    *   A hierarchical structure will be inferred from directory layout: `series_name/episode_name/scene_N.md`.
    *   Each `scene_N.md` file defines a single scene using standard Markdown for text and custom directives (`🪄✨ directive(...)`) for control.
    *   **Directives Specification:**
        *   `🪄✨ setting(background: 'path/to/image.png' | { color: '#RRGGBB' })`: Defines the scene background (static image or solid color).
        *   `🪄✨ characters(list: [{ name: 'char1', initial_pos: [x, y], initial_pose: 'idle' }, ...])`: Declares characters present in the scene, their starting positions (simple 2D coordinates), and initial pose identifier.
        *   `🪄✨ action(character: 'char1', type: 'move', to: [x, y], duration: '2s')`: Specifies a character action (e.g., moving position).
        *   `🪄✨ action(character: 'char1', type: 'pose', name: 'surprised', duration: '0.5s')`: Specifies a character changing pose/expression.
        *   `🪄✨ dialog(character: 'char1', line: 'Frege would approve!', audio: 'path/to/line1.mp3', duration: '3s')`: Specifies a line of dialog, linking to its pre-rendered audio file and suggested display duration.
        *   `🪄✨ camera(action: 'pan', to: [x, y], duration: '2s')`: Specifies camera movement.
        *   `🪄✨ camera(action: 'zoom', level: 1.5, duration: '1s')`: Specifies camera zoom.
        *   `🪄✨ music(track: 'path/to/music.mp3', action: 'play' | 'stop' | 'fade_in' | 'fade_out')`: Controls background music playback.
    *   Events within a scene file are processed sequentially by default. Explicit timing/synchronization directives are deferred for simplicity.
*   **Trace:** `REQUIREMENTS.md: 📖 Story Structure & Scenes`

/comment(user:woe) {
    This gets a little bit into details I'd consider "design" (out-of-scope for specification). "Episodes are defined using Markdown files (`.md`)" is almost sufficient, though it's worth mentioning they may contain custom directives (without listing out the directives at this point) and links out to things like characters and music. We'll also want to be able to specify background artwork, as well as some environmental foreground elements on a per-scene basis. Finally, I'm realizing we don't have any notion of a "shot" in here (e.g. going close-up on an individual character during a scene) but we can defer that for now.
}

### 🧑‍🎨 Character Representation

*   **Spec:** Characters are defined by individual JavaScript modules (`.js`) located in a designated character library directory (e.g., `series_name/library/characters/`).
    *   Each module exports an object or function that provides:
        *   Character name (string).
        *   A mapping of pose names (e.g., 'idle', 'walking', 'surprised') to their corresponding artwork data (see 🖼️ Artwork Integration).
        *   (Optional) Default pose name.
*   **Trace:** `REQUIREMENTS.md: 🧑‍🎨 Character Representation`

/comment(user:woe) {
    Good, but I think we could remove the sub-bullets about what modules export (we can save these for design). On the other hand, I think we should add some detail and specify that we have a common pose and expression library which we can reuse (and augment and extend) across different hominid characters.
}

### 🖼️ Artwork Integration

*   **Spec:** Character artwork is represented using a simple JSON structure describing layered 2D shapes, suitable for LLM generation and rendering via the HTML Canvas 2D API. This fulfills the "paper cutout" aesthetic.
    *   **Format:** An array of shape objects, e.g.:
        ```json
        [
          { "type": "circle", "color": "#FF0000", "radius": 50, "x": 0, "y": 0, "z": 0 },
          { "type": "rect", "color": "#0000FF", "width": 80, "height": 30, "x": -40, "y": 50, "z": 1 },
          { "type": "polygon", "color": "#00FF00", "points": [[0,0], [-10, 10], [10, 10]], "x": 0, "y": -60, "z": 2 }
        ]
        ```
    *   Coordinates (`x`, `y`) are relative to the character's anchor point. `z` determines layering (higher z = on top).
    *   Supported `type` values: `circle`, `rect`, `polygon`. Other types may be added later.
    *   This JSON data will be embedded within or referenced by the character's JS module for each pose.
*   **Trace:** `REQUIREMENTS.md: 🖼️ Artwork Integration`

/comment(user:woe) {
    I like where this is going, but again, let's save the format described in the sub-bullets for design. Instead, just give a general summary. I'm thinking we'll also want `.js` files instead of `.json` so that we can pass things in like `character.render(pose, expression)` although we might get back an array like what you showed. (Again, we don't want to specify design details like `character.render` yet, just details like "`.js` files with a suitable API" which will guide us when we design that API.) Finally, I don't think we should commit to Canvas 2D just yet; WebGL may be better, SVG may be better.
}

### 🕺 Animation Logic

*   **Spec:** The web playback engine interprets scene directives sequentially.
    *   It updates character positions (`x`, `y`) on the 2D canvas based on `action(type: 'move')`.
    *   It changes character appearance by swapping the rendered shape array based on `action(type: 'pose')`. Simple transitions (e.g., cross-fades) are deferred.
    *   It manipulates the canvas view transform to simulate camera `pan` and `zoom`.
*   **Trace:** `REQUIREMENTS.md: 🕺 Animation Logic`

/comment(user:woe) {
    Good, but let's prune design details like `action(type: 'pose')` and focus on specifying what we need to do (update character position, expression, pose). I like the idea of deferring transitions; I'd initially imagined we'd want things like interpolated poses, but let's defer that too. Similarly, let's defer camera controls and just display scenes in fixed frames for now.
}

### 🎶 Music Integration

*   **Spec:** Music cues reference pre-rendered audio files (e.g., MP3, OGG, WAV).
    *   A simple text-based notation format (e.g., simplified MML) can be used *externally* to generate these audio files, but the engine itself only consumes the audio files referenced via `music()` directives.
    *   The engine uses HTML5 `<audio>` elements for playback, synchronizing start/stop with scene events.
*   **Trace:** `REQUIREMENTS.md: 🎶 Music Integration`

/comment(user:woe) {
    This is the right idea; Music Macro Language is a great thought, in particular. However, let's not make this external; we should just have MML files that our LLM animator is responsible for generating.
}

### 💬 Dialog Display

*   **Spec:** Dialog lines specified via `dialog()` directives are displayed as simple, non-overlapping subtitles at the bottom of the playback area. Styling is minimal initially.
*   **Trace:** `REQUIREMENTS.md: 💬 Dialog Display`

/comment(user:woe) {
    Great, but "via directives" would suffice.
}

### 🗣️ Voice Performance

*   **Spec:** The `dialog()` directive must include a path to a pre-generated audio file for the voice line.
    *   The engine plays this audio file synchronized with the display of the subtitle.
    *   Ensuring distinct voices per character is the responsibility of the *content creation process* (i.e., providing appropriately named/organized audio files), not the engine itself. The engine simply plays the specified file.
*   **Trace:** `REQUIREMENTS.md: 🗣️ Voice Performance`

/comment(user:woe) {
    Rather than saying the file is pre-generated, the system should be able to call out to an API and generate audio files for lines of dialog as-needed. (This adds some specificity over our more general requirement.)
}

### 📁 Input Source

*   **Spec:** The Node.js backend serves content based on a simple directory structure convention provided at runtime (e.g., `./path/to/my_series`). It expects subdirectories like `episode_1/`, `episode_2/`, `library/characters/`, `library/audio/` etc. containing the `.md`, `.js`, `.json` (if artwork external), and audio files.
*   **Trace:** `REQUIREMENTS.md: 📁 Input Source`

/comment(user:woe) {
    Also great; could prune out the "It expects..." part as we'll define the directory structure at design-time.
}

## 3. Non-Functional Specifications 👍

### <0xF Frege> Reproducibility

*   **Spec:** The engine must be deterministic. Given identical input files and directory structure, playback must be visually and audibly identical across sessions (barring browser/OS-level audio/rendering variations). No reliance on external state or randomness within the core engine.

/comment(user:woe) {
    We should loosen this just a little bit. Some of our generative content (esp. voice) may not be strictly deterministic with respect to the source files. However, it should be generated with consistent parameters. Finally, we should employ a strategy where we snapshot (e.g. commit to version control) generated content and don't regenerate it unless explicitly ask. That allows the repository to get to a state wherein it is reliably deterministic, while allowing us to bring in less-predictable tools in well-defined ways.
}

### 🧩 Modularity

*   **Spec:** Code will be organized into distinct modules for:
    *   Input Parsing (Markdown, JS modules, JSON artwork).
    *   Scene State Management.
    *   Canvas Rendering Logic.
    *   Audio Playback Control.
    *   Web Server Logic (Node.js/Express).

/comment(user:woe) {
    Excellent. May need a few more bullets here for artwork, characters, poses, etc.
}

### 💨 Performance (Web Preview)

*   **Spec:** Target playback > 30 FPS in modern desktop browsers for scenes with typical complexity (e.g., <10 characters, <100 total shapes, basic animations). No complex physics or collision detection.

/comment(user:woe) {
    Let's target 60 FPS. Should be achievable.
}

### 🛠️ Technology Stack

*   **Spec:**
    *   Backend: Node.js with Express.js (or similar minimal framework).
    *   Frontend: HTML5, CSS3, Vanilla JavaScript (initially, potential for simple framework later if needed).
    *   Rendering: HTML Canvas 2D API.
    *   No external database required for core playback.

/comment(user:woe) {
    Excellent! I'm a little bit less sold on Canvas 2D for rendering, though; I think that works well as a starting point, but let's architect it such that we can easily swap out WebGL or SVG implementations for performance testing (can you capture that here somewhere?)
}

### 📄 Plain-Text Driven

*   **Spec:** Core structural and descriptive elements rely on `.md`, `.js`, `.json`, and potentially a simple `.txt` for music notation (though generation is external). Binary assets (audio, background images) are referenced by path.

/comment(user:woe) {
    Golden! ✨
}

## 4. Deferred Specifications (Future Phantoms 👻)

*   **💾 File Rendering:** No CLI or video file output mechanism.
*   **📐 High-Resolution Output:** Canvas resolution tied to browser window; no specific high-res support.
*   **🤸 Advanced Animation:** No skeletal animation, tweening libraries, particle effects, or physics. Shape swapping and linear position changes only.
*   **📦 Asset Database/Management:** Simple filesystem conventions only.
*   **🤝 Real-time Collaboration:** Single-user playback focus.
