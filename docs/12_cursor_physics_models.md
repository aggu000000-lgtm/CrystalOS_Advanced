# Phase 7: Cursor Physics Mathematical Models

## Introduction
CrystalOS aims to provide a deeply satisfying, organic, and fluid user experience. The cursor is the primary point of interaction between the user and the OS. Instead of a static, infinitely rigid pointer, the CrystalOS cursor operates under a specialized physics model. This model incorporates inertia, momentum, friction, terminal velocity, and spring-damper dynamics to create a cursor that feels alive, weighty yet responsive, and incredibly satisfying to use without ever feeling sluggish or irritating.

## Core Physics Principles
- **Inertia & Momentum:** The cursor has a slight sense of mass. It doesn't just teleport; it accelerates smoothly. This prevents jitter from minute hand tremors while maintaining high responsiveness.
- **Friction (Kinematic Damping):** To prevent the cursor from feeling like it's sliding on ice, dynamic friction is applied. The friction scales with velocity to ensure the cursor stops precisely where the user intends, eliminating any "floaty" or irritating lingering movement.
- **Terminal Velocity:** The cursor has a maximum speed limit to maintain visual coherence and prevent the user from "losing" the cursor across large multi-monitor setups.
- **Spring-Damper System:** When interacting with UI elements (like magnetic buttons or window edges), the cursor is influenced by a critically damped spring model. This pulls the cursor gently toward targets (a satisfying snap) without oscillating or overshooting (an irritating wobble).

## Mathematical Formulas

### 1. Velocity and Acceleration (Inertia & Momentum)
The basic movement updates every frame (\(\Delta t\)) based on the raw input from the mouse/trackpad.

$$ \vec{a} = \frac{\vec{F}_{input}}{m} - \mu \cdot \vec{v} $$
$$ \vec{v}_{new} = \vec{v}_{old} + \vec{a} \cdot \Delta t $$
$$ \vec{p}_{new} = \vec{p}_{old} + \vec{v}_{new} \cdot \Delta t $$

Where:
- \(\vec{F}_{input}\) is the raw delta input from the hardware.
- \(m\) is the virtual mass of the cursor (kept very low to ensure snappiness).
- \(\mu\) is the friction coefficient.
- \(\vec{v}\) is the current velocity.
- \(\vec{p}\) is the position.

### 2. Terminal Velocity Clamping
To prevent the cursor from moving too fast, the velocity magnitude is clamped.

$$ |\vec{v}_{new}| = \min(|\vec{v}_{new}|, V_{max}) $$

Where \(V_{max}\) is the terminal velocity.

### 3. Spring-Damper System (Magnetic UI Snapping)
When the cursor enters the gravity well of a UI element, a spring force pulls it toward the center of the element.

$$ \vec{F}_{spring} = -k (\vec{p} - \vec{p}_{target}) - c \cdot \vec{v} $$

To make it satisfying and not irritating, we use **critical damping** so it snaps perfectly without bouncing.

$$ c = 2 \sqrt{m \cdot k} $$

Where:
- \(k\) is the spring constant (stiffness of the magnetic pull).
- \(c\) is the damping coefficient.
- \(\vec{p}_{target}\) is the center of the UI element.

## Integration with the UI and Input Agent

### Interaction with the Input Agent
CrystalOS uses a 7-Agent Architecture. The **Input Agent** operates entirely lock-free, reading raw hardware interrupts (mouse movements, trackpad gestures) and writing them to a ring buffer.

1. **Raw Input:** The Input Agent polls or receives interrupts from HID devices and pushes raw \((\Delta x, \Delta y)\) values to a lock-free queue.
2. **Physics Processing:** The **Display Agent** (or a dedicated UI sub-system) reads this queue at the refresh rate of the monitor (e.g., 120Hz or 240Hz). It applies the physics models described above.
3. **Low Latency:** Because the Input Agent does not block on the physics calculations, input latency remains incredibly low (often sub-millisecond). The physics engine smooths out the raw data, ensuring the on-screen cursor is perfectly synced with the display's vsync.

### UI Interaction
- **Hover States & Gravity Wells:** When the cursor approaches a button, the UI communicates a bounding box and a "gravity" strength to the physics engine. The spring-damper system gently guides the cursor to the center.
- **Organic Resistance:** When dragging items or pulling past the edge of a scrollable area, the friction coefficient (\(\mu\)) dynamically increases, simulating a rubber-band effect or weight resistance, providing satisfying tactile feedback without requiring haptic hardware.

This combination of low-latency lock-free input processing and highly tuned, critically damped physics creates a cursor that feels like an extension of the user's hand—effortless, precise, and uniquely satisfying.
