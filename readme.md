# ⚡ Interactive Dual-Edge Throttle Visualizer

An interactive educational tool and telemetry dashboard designed to visually demystify **Rate Limiting**, **Closures**, and **Dual-Edge (Leading & Trailing) Throttling** while scrolling in JavaScript.

---

## 📖 Overview

High-frequency browser events such as `scroll`, `resize`, and `mousemove` can fire over **100 times per second**. Binding unthrottled operations (such as layout queries, DOM rewrites, or API network requests) directly to these events leads to **layout thrashing, dropped frames (jank), and battery drain**.

This project provides a **live visual telemetry engine** that contrasts raw native browser input against a generic, configurable **Dual-Edge Throttle**. 

It demonstrates that throttling does **not** stop the user from physically scrolling; rather, it protects system resources by rate-limiting the execution of the callback while preserving the final stopping state.

---

## 🎯 Key Features

* **Dual-Edge Execution (Leading & Trailing):** Executes immediately on the first event tick (zero UI latency) and guarantees that the terminal state is processed when activity stops (zero lost data).
* **Live Dual-Dot Visual Rail:** 
  * 🟠 **Orange Dot (Raw):** Tracks the real-time physical scroll position at native display frequency.
  * 🔵 **Blue Dot (Throttled):** Lags behind during cooldown, proving that rate-limiting batches executions in the background.
* **Real-World State Commitment ("Saved @ nnnnpx"):** Demonstrates reading-progress persistence and ScrollSpy functionality by highlighting the exact in-viewport card upon catch-up.
* **Interactive Dynamic Slider:** Adjust the cooldown window from **100ms up to 5000ms (5.0 seconds)** in real time to observe the cooldown gate lifecycle.
* **150 Curated JS Concepts Feed:** Built-in deep scroll container populated with 150 core JavaScript concepts (V8 internals, Event Loop, Prototypes, Memory Management).
* **100% Viewport-Responsive:** Fully fluid CSS Grid architecture configured to fit `100vh` without outer page scrollbars.

---

## 🧠 Architectural Concepts

### The Mechanics of Dual-Edge Throttling

Throttling enforces a strict upper limit on how often a function can run over time. A dual-edge implementation manages state across three distinct phases:

1. **The Leading Edge (Immediate Trigger):**  
   The very first event in a sequence executes immediately. This guarantees that user interactions (like touching a scroll wheel or dragging a slider) feel instant and responsive, with zero perceived latency.
2. **The Lockout Period (Cooldown Window):**  
   Once triggered, a closure-based flag locks the function for the duration of the specified limit. Any events firing during this window are prevented from executing. Instead, the engine continuously overwrites a temporary buffer with the **latest incoming arguments**.
3. **The Trailing Edge (Catch-Up):**  
   When the timer reaches zero, the engine inspects the buffer. If new events occurred during the lockout, it invokes the target function one final time with the most recent data. This guarantees pixel-perfect accuracy once motion stops.

---

## 🔬 Telemetry & Real-World Semantics

### 1. Physical Scrolling vs. Code Throttling
A common misconception is that throttling limits physical scroll speed (often confused with *scrolljacking*). 
* **The physical scroll** runs freely on the browser's hardware-accelerated compositor thread.
* **The throttle engine** purely meters the JavaScript main-thread work. 

The **Orange vs. Blue visual track** proves this: the user moves freely (Orange), while the application state (Blue) updates in discrete, batched intervals.

### 2. The Semantic of `Saved @ nnnnpx`
When the trailing edge fires, the blue dot announces:
`💾 Saved @ 2450px (Concept #28)`

This simulates real-world production behaviors:
* **Reading Progress Persistence:** Storing reading offsets to `localStorage` or a remote API (e.g., Medium, Kindle Web). Instead of executing 500 disk/network writes during a flick of the trackpad, the application executes **one** clean write when movement ceases.
* **ScrollSpy Navigation:** The application computes the card currently intersected by the viewport and applies a neon-green highlight, demonstrating how documentation sites track active table-of-contents navigation links.

---

## 📊 Event Lifecycle Timeline (Example at 1000ms Limit)

| Elapsed Time | User Interaction | Engine Phase | Callback Action | Cooldown Gate | Visual Output |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **0ms** | Scroll starts (Tick 1) | **Leading Edge** | Executes immediately | Flips to `LOCKED` | Orange & Blue move |
| **250ms** | Continuous scroll (Tick 2) | Locked Window | Blocked; caches args | `LOCKED` | Orange moves; Blue frozen |
| **500ms** | Continuous scroll (Tick 3) | Locked Window | Blocked; overwrites args | `LOCKED` | Orange moves; Blue frozen |
| **999ms** | User stops moving (Tick 4) | Locked Window | Blocked; overwrites args | `LOCKED` | Final arguments buffered |
| **1000ms** | *No user input* | **Trailing Edge** | Executes with Tick 4 data | Re-arms / Releases | Blue catches up; `💾 Saved!` |
| **1050ms** | *No user input* | Rest State | Idle | Flips to `READY` | Both dots synced |

---

### 🧩 The Simplified Dual-Edge Throttle Implementation

A generic, production-ready implementation supporting both **Leading** and **Trailing** edges, preserving execution context (`this`), and accepting either a static delay or a dynamic limit getter:

```javascript
/**
 * Generic Dual-Edge Throttle (Leading + Trailing)
 *
 * @param {Function} Fx - The target callback function to rate-limit.
 * @param {number|Function} limit - Cooldown in milliseconds (or a getter function for dynamic limits).
 * @returns {Function} - The throttled wrapper function.
 */

function throttle(Fx, limit) {
  let wait = false;       // Gatekeeper flag persisting in closure scope
  let lastArgs = null;     // Buffers the most recent arguments during lockout
  let lastThis = null;     // Preserves execution context ('this')

  // Resolves delay whether passed as a static number or a dynamic getter
  const getDelay = () => (typeof limit === 'function' ? limit() : limit);

  function timeoutHandler() {
    if (lastArgs) {
      // --- TRAILING EDGE ---
      // Events occurred during lockout: execute with the final recorded state
      Fx.apply(lastThis, lastArgs);
      lastArgs = null;
      lastThis = null;

      // Re-arm cooldown to enforce spacing before unlocking
      setTimeout(timeoutHandler, getDelay());
    } else {
      // Cooldown expired with no pending activity: unlock the gate
      wait = false;
    }
  }

  return function (...args) {
    if (!wait) {
      // --- LEADING EDGE ---
      // Gate is open: fire immediately on initial interaction (zero latency)
      Fx.apply(this, args);
      wait = true;

      // Start the cooldown timer
      setTimeout(timeoutHandler, getDelay());
    } else {
      // --- COOLDOWN ACTIVE ---
      // Gate is locked: cache the latest arguments to ensure terminal accuracy
      lastArgs = args;
      lastThis = this;
    }
  };
}
```
#### Usage Example:
```javascript
// 1. Define your heavy task (DOM updates, calculations, network requests)
function onScrollHandler(event) {
  console.log("Processing scroll at offset:", window.scrollY);
}

// 2. Wrap with a 1000ms dual-edge throttle
const throttledScroll = throttle(onScrollHandler, 1000);

// 3. Attach directly to high-frequency events
window.addEventListener("scroll", throttledScroll);
```
## 🚀 Getting Started

1. Clone or download this repository.
2. Open `index.html` directly in any modern desktop web browser (Chrome, Firefox, Safari, Edge).
3. Adjust the **Throttle Limit Slider** to `2000ms` or `5000ms`.
4. Scroll rapidly inside the dedicated container to watch the trailing-edge catch-up mechanism in action.

---

## 📝 License

Distributed under the MIT License. Feel free to use, modify, and distribute for educational or commercial purposes.