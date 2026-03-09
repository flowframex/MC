# Checkerboard Rendering Mod — Complete Beginner Guide

## What does this mod do?

Instead of rendering every pixel every frame, this mod renders pixels in a **checkerboard pattern**:

```
Frame 0:  Render █ , reuse □     Frame 1:  Reuse █ , render □
          □ █ □ █                           █ □ █ □
          █ □ █ □                           □ █ □ █
          □ █ □ █                           █ □ █ □
```

Each frame, only ~50% of pixels are freshly rendered. The other half are copied from the previous frame. This halves pixel shading work at the cost of slight ghosting on fast motion.

---

## Project Structure

```
checkerboard-mod/
├── build.gradle                          ← Gradle build config
├── gradle.properties                     ← Version numbers
├── settings.gradle                       ← Gradle settings
└── src/main/
    ├── java/com/example/checkerboard/
    │   ├── CheckerboardMod.java           ← Mod entry point
    │   ├── CheckerboardRenderer.java      ← Manages framebuffers & post-effect
    │   └── mixin/
    │       └── GameRendererMixin.java     ← Hooks into Minecraft's render loop
    └── resources/
        ├── fabric.mod.json               ← Mod metadata
        ├── checkerboard.mixins.json      ← Mixin config
        └── assets/checkerboard/shaders/
            ├── post/
            │   └── checkerboard.json     ← Post-effect chain definition
            └── program/
                ├── checkerboard_reconstruct.json  ← Shader program metadata
                ├── checkerboard_reconstruct.vsh   ← Vertex shader (GLSL)
                └── checkerboard_reconstruct.fsh   ← Fragment shader (GLSL) ← THE MAGIC
```

---

## How to Set Up & Build

### Prerequisites
- **Java 17 or 21** — [Download from Adoptium](https://adoptium.net/)
- **IntelliJ IDEA** (recommended) — [Download Community Edition free](https://www.jetbrains.com/idea/download/)

### Steps

1. **Download this project folder** and open it in IntelliJ IDEA
   - File → Open → select the `checkerboard-mod` folder

2. **Let Gradle sync** — IntelliJ will download all dependencies automatically (takes a few minutes first time)

3. **Generate Minecraft sources** (optional but helpful for understanding vanilla code):
   ```
   ./gradlew genSources
   ```

4. **Build the mod**:
   ```
   ./gradlew build
   ```
   The `.jar` file will appear in `build/libs/checkerboard-mod-1.0.0.jar`

5. **Test in-game**: Copy the jar to your `.minecraft/mods/` folder and launch Fabric 1.20.4

---

## How the Code Works (Plain English)

### 1. `CheckerboardMod.java` — The Entry Point
This is just the "hello world" that Fabric loads first. It logs a message and is where you'd register keybinds/config later.

### 2. `CheckerboardRenderer.java` — The Manager
This class:
- Creates a **"previous frame" framebuffer** — a copy of the last rendered frame stored in GPU memory
- Loads our **post-processing shader** (the GLSL files)
- Each frame: tells the shader which pattern to use (even or odd pixels this frame), runs the shader, then increments the frame counter

### 3. `GameRendererMixin.java` — The Hook
Mixins let you inject code into Minecraft's existing classes without modifying them.
- `@Inject` at `loadPrograms` → Initialize our renderer when Minecraft's shaders load
- `@Inject` at `render` → Run our checkerboard effect right before the frame is shown
- `@Inject` at `onResized` → Recreate our framebuffers when the window is resized

### 4. `checkerboard_reconstruct.fsh` — The Fragment Shader (The Real Magic)
```glsl
int pixelParity = (pixel.x + pixel.y) & 1;  // Is this an "even" or "odd" pixel?
int frame = int(FrameIndex) & 1;             // Which pattern are we rendering?

if (pixelParity == frame) {
    fragColor = texture(DiffuseSampler, texCoord);  // Fresh pixel from THIS frame
} else {
    fragColor = texture(PrevSampler, texCoord);     // Reused pixel from LAST frame
}
```
That's it! Two lines of actual logic. Everything else is plumbing to get data to those two lines.

---

## Known Limitations & Next Steps

### Current Limitations
- **Ghosting on fast movement** — reused pixels don't move with the camera, causing blur/trails
- **No temporal reprojection** — the fancy fix for ghosting (reprojects previous pixels using motion vectors)
- The "previous frame" buffer needs proper implementation to avoid the `copyDepthFrom` hack

### Ways to Improve This Mod
1. **Add a config screen** to toggle the effect on/off (use Cloth Config API)
2. **Add a keybind** to toggle: register in `CheckerboardMod.java` using Fabric's KeyBinding API
3. **Temporal reprojection**: Instead of reusing pixels at the same screen position, reproject them using the camera's movement delta — this eliminates ghosting
4. **Half-resolution rendering**: Instead of post-process masking, render the actual 3D scene at half resolution (harder, requires deeper rendering hooks)
5. **Motion blur blending**: Blend the checkerboard seams with a small blur pass

### Fixing the "Previous Frame" Problem
The current design has a chicken-and-egg issue: to reconstruct the frame we need the previous output, but our post-effect runs on the main framebuffer. A proper fix:
1. Create **two** extra framebuffers: `pingBuffer` and `pongBuffer`
2. Alternate which one is "current output" and which is "previous frame"
3. Blit final result to main framebuffer

---

## Useful Resources for Learning More

- [Fabric Wiki](https://fabricmc.net/wiki/tutorial:introduction) — Official Fabric modding tutorials
- [Mixin Wiki](https://github.com/SpongePowered/Mixin/wiki) — Understanding how @Inject, @Overwrite etc. work
- [Yarn Mappings](https://yarn.fabricmc.net/) — Look up Minecraft class/method names
- [GLSL Reference](https://www.khronos.org/opengl/wiki/Fragment_Shader) — Fragment shader documentation
- [Existing Minecraft Shaders](https://github.com/search?q=minecraft+post+effect+shader) — Study vanilla post-effect shaders for examples

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `ClassNotFoundException` for mixin | Check `checkerboard.mixins.json` — class name must match exactly |
| Shader doesn't load | Check the JSON files for typos; look at the game log for shader compile errors |
| Black screen | Your shader has a GLSL error — check `latest.log` for `ERROR` lines |
| No visual difference | The inject point in `GameRendererMixin` may be wrong for your MC version — try different `At` targets |
| Gradle fails to sync | Make sure Java 17 is installed and set as the project SDK in IntelliJ |
