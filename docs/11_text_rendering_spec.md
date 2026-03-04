# CrystalOS: "Carved Inside" Text Rendering Specifications

## 1. Overview
As part of CrystalOS Phase 6, this document outlines the unique "carved inside" text rendering paradigm. In CrystalOS, text does not merely sit on top of the screen; it appears physically *carved into* the display. This embossed, inset visual style is a core element of the CrystalOS minimalist design language, providing a physical, tactile feel to the purely digital interface.

To achieve our "Boot in 3 seconds, Respond in milliseconds" promise, this rendering engine is designed for extreme efficiency, prioritizing pre-baked lighting models and custom bitmap formats over expensive real-time computational shading.

## 2. The "Carved Inside" Visual Model
The "carved" effect creates an illusion of depth by applying specific lighting cues:
*   **Inner Shadow (Top/Left):** Simulates a directional light source casting a shadow inside the recessed glyph.
*   **Highlight (Bottom/Right):** A subtle, bright edge that catches the light on the lower lip of the carving.
*   **Alpha Blending:** Subpixel blending ensures smooth edges, preventing jagged artifacts that would break the physical illusion.

### 2.1 Pre-baked Lighting
To keep the OS lightweight and maintain rendering times under 10ms, these lighting models are **pre-baked** into the font assets. The graphics pipeline does not compute inner shadows or highlights dynamically per frame. Instead, the font format inherently stores the depth and shading information.

## 3. Custom Font Format: `.crfont` (CrystalFont)
CrystalOS utilizes a custom, perfected, and unique font format designed specifically for the "carved inside" aesthetic and extreme performance.

### 3.1 Primary Representation: Bitmaps
The primary representation for `.crfont` is **bitmap-based**.
*   **Why Bitmaps?** Bitmaps are incredibly fast to blit to the screen. Since the complex shading (shadows, highlights) is already baked into the pixel data, the Display Agent (Agent 1) only needs to perform a simple alpha-blended copy operation.
*   **Pre-rendered Sizes:** The system will ship with pre-rendered bitmap sets for common UI sizes (e.g., 10pt, 12pt, 16pt, 24pt, 36pt).

### 3.2 Secondary Representation: Vectors (Rare Cases)
While bitmaps cover 99% of UI text, `.crfont` includes a minimal vector fallback (similar to Signed Distance Fields) for rare edge cases:
*   Smooth scaling for massive, non-standard typography (e.g., gigantic lock screen clocks or specialized design tools).
*   Dynamic zooming scenarios where pre-rendered bitmaps would become pixelated.
*   *Note:* Using the vector fallback incurs a slight performance penalty as shading must be approximated in real-time, hence its use is restricted to rare cases.

## 4. Performance Targets
*   **Rendering Latency:** The total time to process a text string request from an application and composite it onto the screen must be between **6-10 ms**.
*   **Memory Footprint:** The pre-baked bitmap sets are optimized and compressed (via LZ4/ZSTD in CrystalFS) to ensure the memory footprint remains minimal, aligning with the 500KB microkernel philosophy.

## 5. Architectural Integration (Agent 1 - Display)
The text rendering pipeline is tightly integrated with the 7-Agent Architecture, specifically relying on **Agent 1 (Display)**.

### 5.1 IPC Message Flow
When an application (User-Space) needs to render text, it does not draw pixels itself. It communicates with Agent 1 via the lock-free IPC queues.

1.  **Request `(App -> Kernel -> Agent 1)`:**
    *   The App sends a `RenderTextMsg` containing:
        *   String payload (UTF-8).
        *   Font ID (`.crfont` reference).
        *   Size/Weight index (mapping to the pre-baked bitmap set).
        *   Screen coordinates (X, Y).
        *   Target color (the pre-baked shadows adapt their opacity over this base color).
2.  **Processing `(Agent 1)`:**
    *   Agent 1 retrieves the pre-baked glyph bitmaps from the font cache (populated by Agent 3 - Memory/Storage).
    *   It performs a rapid, SIMD-accelerated alpha-blend blitting operation onto the backbuffer composition layer.
3.  **Completion `(Agent 1 -> App)`:**
    *   Agent 1 sends a `RenderCompleteMsg` (if synchronous blocking is requested) or fires an async event on the system event bus.

### 5.2 Strict Boundaries
To maintain security and stability, User-Space applications cannot inject custom shaders or bypass the `.crfont` pipeline. The "carved inside" look is uniformly enforced at the system level by Agent 1.
