# Aniheim Rendering Engine - Design Document ✍️📐

This document details the design of the Aniheim Rendering Engine, specifying the interfaces, formats, and workflow required to fulfill the `SPECIFICATION.md`. Our goal is a build system that transforms LLM-friendly source files into self-contained HTML animations.

## 1. Architecture Overview

The engine operates as a command-line build tool. It parses a primary Episode Markdown file, resolves dependencies (characters, music, voice), generates necessary assets (audio), and finally emits a single, self-contained HTML file capable of playing the animation.

```mermaid
graph LR
    A[Episode.md] --> B{Build Engine (Node.js)};
    C[CharacterLib/*.js] --> B;
    D[MusicDefs/*.abc] -- Optional --> B;
    E{LLM (Music Gen)} -- Optional --> D;
    F{LLM (TTS Gen)} -- Optional --> G[AudioCache/*.mp3];
    B --> G;
    B --> H[Output.html];

    subgraph Inputs
        A
        C
        D
    end
    subgraph External Services
        E
        F
    end
    subgraph Build Process
        B
        G
    end
    subgraph Output
        H
    end
```

## 2. Command-Line Interface (CLI) 💻

The engine is invoked via Node.js:

```bash
node build.js <episode_markdown_file> [--output <output_html_file>] [--force-regenerate] [--character-lib <path>] [--music-lib <path>] [--audio-cache <path>]
```

*   `<episode_markdown_file>`: Path to the main `.md` file for the episode (Required).
*   `--output`: Path for the output HTML file (Defaults to `<episode_name>.html` in the current directory).
*   `--force-regenerate`: Forces regeneration of cached assets (music, voice).
*   `--character-lib`: Path to the character definition directory (Defaults to `./characters`).
*   `--music-lib`: Path to the music definition directory (Defaults to `./music`).
*   `--audio-cache`: Path to store generated audio files (Defaults to `./audio_cache`).

/comment(user:woe) {
    I like This! Let's omit `--force-regenerate`, `--character-lib`, `--music-lib`, and `--audio-cache` for now. Regeneration can be manual (deleting the undesired assets); the "libs" can just be referenced via hyperlinks in the episode markdown file; and audio files can just be generated as locations specified in episodefiles as well.
}

## 3. Episode File Format (`.md`) 📄

*   **Structure**: One `.md` file per episode, containing the full narrative script and animation directives. Hyperlinks or simple path references point to external character/music definitions.
*   **Directives**: Uses `🪄✨ directive(...) { body } directive⚡️` syntax.
    *   `🪄✨ setting(background: 'path/image.png' | '#RRGGBB', foreground?: 'path/overlay.png')`
        *   Specifies scene background (image path or hex color) and optional static foreground elements.
        *   **Trace:** `SPEC: 📖 Story Structure & Scenes`
    *   `🪄✨ characters(list: [{ name: 'CharName', module: './path/to/CharName.js', pos: [x, y], pose: 'idle' }, ...])`
        *   Declares characters used in the episode, linking to their JS module and setting initial state. `pos` is likely %-based coords `[50, 50]` = center. `pose` references a key in the character's pose map.
        *   **Trace:** `SPEC: 📖 Story Structure & Scenes`, `SPEC: 🧑‍🎨 Character Representation`
    *   `🪄✨ action(character: 'CharName', type: 'move', to: [x, y], duration: '2s')`
        *   Moves a character. Duration suggests speed.
        *   **Trace:** `SPEC: 🕺 Animation Logic`
    *   `🪄✨ action(character: 'CharName', type: 'pose', name: 'thinking' | 'surprised', duration: '0.5s')`
        *   Changes character pose/expression instantly or over a short duration (implementation deferred, initially instant swap). Uses pose keys from the character module.
        *   **Trace:** `SPEC: 🕺 Animation Logic`
    *   `🪄✨ dialog(character: 'CharName', voice: './audio_cache/charname_line_01.mp3') { Hello there! 👋 I'm feeling happy! 😀 } dialog⚡️`
        *   Defines a line of dialog spoken by a character.
        *   `voice`: Path for the cached TTS audio file. If the file doesn't exist, the engine generates it using the text body and character-specific TTS settings. The path is relative to the `--audio-cache` directory.
        *   **Microformat (Emoji)**: Emoji within the body control pose/expression *during* playback of this line.
            *   `👋`: Maps to a predefined 'wave' pose/action.
            *   `😀`: Maps to a predefined 'happy' expression (overrides base pose's expression).
            *   Mapping defined globally or per-character (TBD). Parser extracts emoji; TTS receives cleaned text.
        *   **Trace:** `SPEC: 💬 Dialog Display`, `SPEC: 🗣️ Voice Performance`
    *   `🪄✨ music(sheet: './music/theme.abc', audio: './audio_cache/theme.mp3', action: 'play' | 'stop' | 'set_volume(0.5)') { A spooky yet jaunty theme 👻 } music⚡️`
        *   Controls background music.
        *   `sheet`: Path to the music definition file (ABC notation preferred). Relative to `--music-lib`.
        *   `audio`: Path for the cached rendered audio file. Relative to `--audio-cache`.
        *   `action`: Controls playback state.
        *   **LLM Generation & Caching**:
            *   If `sheet` path doesn't exist AND body has description: Call LLM to generate ABC notation based on description, save to `sheet` path.
            *   If `audio` path doesn't exist: Render `sheet` file (ABC -> MIDI -> MP3 using e.g., `abcjs` + `timidity++`/SoundFont wrapper), save to `audio` path.
            *   Existence of files skips generation unless `--force-regenerate`.
        *   **Trace:** `SPEC: 🎶 Music Integration`
    *   `🪄✨ // This is a comment directive //`
        *   Allows comments within the directive flow.

/comment(user:woe) {
    I'm having a bit of trouble navigating the content in this section, which makes me think we may want to break it down some; perhaps subsections instead of bullets? Or maybe we can just trim some? Or move stuff out elsewhere?

    Instead of `setting`, let's call that a `background` directive. We don't want an image file there; we'll want a set of paper cutouts, ultimately. But, we want the episode files to be simply-readable.

    Maybe a scene should look something like:

    /scene(duration:30,setting:../settings/a-public-park.md)
    /music(start:0,duration:30,source:../music/happy-day.md)
    /character(name:Alice,source:../characters/alice.md,x:-0.25,y:0.1,z:0.5)
    /character(name:Bob,source:../characters/bob.md,x:0.75,y:0.15,z:0.5)
    /move(start:0,duration:10,x:0.25,y:0.1,x:0.5,character:Alice)
    /animate(start:0,duration:10,character:Alice) {
        🚶‍♀️🙂😄
    }
    /dialog(start:10,duration:10,character:Alice) {
        👋 Hi Bob! 😊 How's it going? 🤔
    }
    /dialog(start:20,duration:10,character:Bob) {
        Not bad, Alice! 🧍 How are you?
    }

    In this scene, Alice walks in from off-screen and greets Bob. Miscellaneous thoughts in here:

    * Music and scene files can just be externalized. We'll want to reuse these a lot! Even from episode to episode, just like characters. (This is a good opportunity to simplify here.)
    * Characters aren't just their artwork. I think standardizing on Markdown and branching out to more specific formats like JS only when needed might work here.
    * We'll need a description of scenespace somewhere. I'm assuming x=0 is left, x=1 is right, y=0 is bottom, y=1 is top, z=0 is front, z=1 is off in the distance (and z=0.5 is standard for character placement in a scene). I'm a little nervous that a numeric scene-space will be hard for an LLM to manage, so maybe someday we'll want a microformat where we can say "/move(character:Bob,to:near[Alice]) or something.
    * I'm realizing we'll need something like an `animate` directive for times when we want a character to change expressions or pose when they're not talking.
    * Can't currently animate non-characters with this format, but I guess that's fine.
}

## 4. Character Module Interface (`.js`) 🧑‍🎨

Located via `--character-lib`. Each file defines one character.

*   **Exports**: A module should export an object with at least:
    *   `name`: (String) Character's display name.
    *   `ttsVoiceSettings`: (Object) Parameters for the TTS engine (e.g., `{ voiceId: 'en-US-Wavenet-F', pitch: -2 }`). Used when generating audio for `dialog` directives.
    *   `poses`: (Object) A map where keys are pose names (e.g., `'idle'`, `'walking'`, `'surprised'`) and values are artwork data (see below) or functions returning artwork data. Can potentially reference/extend a base pose library.
    *   `getArtworkData(poseName, expressionName)`: (Function) Takes pose and expression names (expression might modify base pose data), returns artwork data structure. Allows dynamic artwork generation if needed.
*   **Trace:** `SPEC: 🧑‍🎨 Character Representation`, `SPEC: 🗣️ Voice Performance`

/comment(user:woe) {
    Let's actually do characters as Markdown files too, it'll make it easier to simply describe them. We also don't want poses to be character-specific; let's have a pose library we can reuse across characters (maybe a separate section and file type?)

    Voice is good; let's just have a `/voice(settings=voice.json) { Semantic description of a voice }` directive that we use to define that. Again, this can be rendered using the if-file-exists approach; when it doesn't exist, we'll ask an LLM or something to generate voice settings in that file.

    Similarly, we should have an `/artwork(renderer=./artwork.js) { Semantic description of character's appearance }`.

    The name doesn't need its own directive; maybe we infer that from the top-level section title?
}

## 5. Artwork Representation & Rendering 🖼️

*   **Data Format**: The `getArtworkData` function (or `poses` map values) returns an array of shape objects, as specified previously:
    ```javascript
    // Example return value for getArtworkData('idle', 'neutral')
    [
      { type: 'circle', color: '#FACE00', radius: 30, x: 0, y: -40, z: 1 }, // Head
      { type: 'rect', color: '#A0A0A0', width: 10, height: 50, x: 0, y: 0, z: 0 }, // Body
      // ... etc ...
    ]
    ```
    *   Supported `type`: `circle`, `rect`, `polygon`. `color` can include alpha (e.g., `#RRGGBBAA`). `x`, `y`, `z` for positioning and layering. Coordinates are relative to the character's anchor point on screen.
*   **Renderer Interface**: The player embedded in the output HTML will expect a rendering module conforming to an interface like:
    *   `init(canvasElement)`: Initialize the rendering context.
    *   `clear()`: Clear the canvas.
    *   `drawShapes(shapes, baseX, baseY, scale)`: Draw an array of shape objects at a specific screen position and scale.
    *   Implementations for Canvas 2D, WebGL, SVG can be developed. Initial implementation: Canvas 2D.
*   **Trace:** `SPEC: 🖼️ Artwork Integration`, `SPEC: 🛠️ Technology Stack (Rendering Interface)`

/comment(user:woe) {
    Hm, I think we'll want more shapes here, as well as the ability to support anchored shape rotation (which will make animation easier). Wouldn't hurt to give shapes an `id` as well for stuff like keyframing. I'm thinking we'll want ovals/ellipses, and maybe even things like clipping shapes (e.g. to use a half-circle for a character's foot). On the other hand, we could omit `rect` since we have `polygon`.

    Importantly, this is going to need to be a little rendering language which looks reasonably-fluent to LLMs: They'll need to be able to work out where eyes go, where noses go, and so on. Starting with something simple and easy-to-render makes sense, but we'll want to push back and design a more LLM-oriented interface to produce that easy-to-render format at some point.
}

## 6. Music Integration (ABC Notation) 🎶

*   **Format**: ABC Notation (`.abc` files). Text-based, good for LLMs, standard tools exist. Example:
    ```abc
    X: 1
    T: Spooky Theme
    M: 4/4
    L: 1/8
    K: Am
    "Am"A2 E2 G2 E2 | "G"B2 G2 d2 G2 | "Am"c4 B4 | A8 |]
    ```
*   **Rendering**: Use Node.js libraries to orchestrate:
    1.  `abcjs`: Parse ABC to MIDI data structure or potentially directly to WAV (if library supports it).
    2.  MIDI-to-Audio Renderer: Use a tool like `timidity++` (via child process or Node wrapper) with a suitable SoundFont (e.g., `FluidR3_GM`) to convert MIDI data/files to MP3/OGG/WAV.
*   **Trace:** `SPEC: 🎶 Music Integration`

/comment(user:woe) {
    Superb! One note: Let's assume that music starts off as Markdown files which contain a `score` directive like:

    /score(sheet:something.abc,audio:something.mp3) {
        A haunting song titled Uncanny Melody, played on the cello.
    }

    These live in separate Markdown files so they can be reused like Characters.
}

## 7. TTS Integration 🗣️

*   **Workflow**:
    1.  Parser identifies `dialog` directive.
    2.  Extracts emoji for animation cues, cleans text for TTS.
    3.  Checks if `voice` file exists in `--audio-cache`.
    4.  If not found or `--force-regenerate`:
        *   Retrieve `ttsVoiceSettings` from character module.
        *   Call configured TTS API (e.g., Google Cloud TTS, AWS Polly via SDKs) with cleaned text and settings.
        *   Save returned audio to the specified `voice` path.
*   **Configuration**: TTS provider credentials and API choice configured globally for the build tool (e.g., via environment variables or a config file).
*   **Trace:** `SPEC: 🗣️ Voice Performance`

/comment(user:woe) {
    nit: Let's call this Voice or Voiceover integration instead of TTS.

    Instead of using an audio cache, I think we should have an `audio` attribute on `dialog` directives. In the future, maybe we can add some smart defaults when this is absent, but let's just require it for now.
}

## 8. Build Process Flow ⚙️➡️📄

1.  **Initialization**: Parse CLI arguments, identify input/output paths, libraries.
2.  **Episode Parsing**: Read the main `.md` file. Parse directives sequentially.
3.  **Asset Resolution & Generation (Iterative)**:
    *   For `characters` directive: Load specified JS modules. Validate exports.
    *   For `dialog` directive: Check/generate TTS audio, cache path. Store text, character, audio path, and extracted emoji cues.
    *   For `music` directive: Check/generate ABC notation (via LLM if needed), check/render audio, cache paths. Store action and audio path.
    *   Store other directives (setting, actions) in an ordered event list.
4.  **Code Generation**: Generate JavaScript code for the embedded player:
    *   Include character data (or references).
    *   Include the ordered event list (actions, dialog cues + audio paths, music cues + audio paths, setting changes).
    *   Include animation logic (how to interpret events and update state).
    *   Include the chosen rendering module code (e.g., Canvas 2D renderer).
5.  **HTML Emission**: Create the final `.html` file:
    *   Basic HTML structure.
    *   CSS for styling (player UI, subtitles).
    *   Embedded player JS (generated in step 4).
    *   Necessary runtime libraries (e.g., `howler.js` for audio management if needed).

/comment(user:woe) {
    Great overview, really helped clarify the complexity. (Note that this may need updating to reflect revisions elsewhere based on other comments)
}

## 9. Output HTML Structure <html>

*   A single HTML file.
*   `<style>` block for CSS.
*   `<canvas>` (or other) element for rendering.
*   `<script>` block containing:
    *   The renderer implementation (Canvas/WebGL/SVG).
    *   The core player logic (event loop, state management).
    *   The episode-specific data (event list, character info, audio paths).
    *   Minimal UI controls (Play/Pause).

## 10. Technology Choices (Initial)

*   **Markdown Parser**: `marked` or `markdown-it` with plugin support for custom directives.
*   **Music Rendering**: `abcjs` (for ABC->MIDI/intermediate format), Node wrapper around `timidity++` or similar synth + SoundFont for MIDI->Audio.
*   **TTS**: Node SDKs for cloud providers (Google TTS, AWS Polly).
*   **Audio Playback (Runtime)**: HTML5 `<audio>` or a library like `howler.js` for better control.
*   **Renderer (Runtime)**: HTML Canvas 2D API.

/comment(user:woe) {
    Excellent. 
}
