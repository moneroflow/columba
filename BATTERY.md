# Battery Optimization Research: Signal & SimpleX vs. Columba

## The Fundamental Conflict
Signal and SimpleX solve a problem that Columba has in an entirely different way. 

### 1. The "Client-Server" Model (Signal)
Signal relies on **FCM (Firebase Cloud Messaging)**. 
*   **How it works:** The app does *not* maintain a persistent connection to the server. Instead, it sleeps completely. When a message arrives, Google's servers send a "poke" (push notification) to the device.
*   **Battery Impact:** Extremely low. The radio only wakes up when Google tells it to.
*   **Columba Contrast:** Columba is a **Mesh/P2P network**. There is no central Google server to "poke" the device. Columba must actively listen for peers via BLE/TCP, which is why it requires a Foreground Service and WakeLocks.

### 2. The "Zero-Server" Model (SimpleX)
SimpleX is closer to Columba's philosophy but uses "SMP Servers" as relays.
*   **How it works:** SimpleX uses a similar push-notification-based wake-up system as Signal to avoid polling.
*   **Battery Impact:** Low, because it offloads the "waiting" part to a relay server.
*   **Columba Contrast:** Columba is designed for environments where those relay servers might not exist (off-grid, local-only mesh).

## Analysis: How Columba can adapt these "Pro" strategies

Since Columba cannot use a central "wake-up" server without compromising its mesh/local-first nature, we must implement **Client-Side Intelligent Sleeping**.

### Strategy A: The "Heartbeat Taper" (Implemented in Phase 2)
*   **Concept:** Instead of a fixed "on" or "off" state, use a gradient.
*   **Implementation:** 
    *   Active $\rightarrow$ Idle (30s) $\rightarrow$ Deep Idle (5m).
    *   This mimics the "low power" state of a server while still being discoverable.

### Strategy B: "Context-Aware" Scanning
*   **Concept:** Only scan when the device is likely to find someone.
*   **Potential implementation:** 
    *   Use the **Android Activity Recognition API**. If the accelerometer detects the phone hasn't moved in 10 minutes, it's likely on a table—stop all BLE scans and enter a "Hibernation" mode.
    *   Resume scanning only when the device is picked up.

### Strategy C: "Adaptive WakeLocks"
*   **Concept:** Instead of one long lock, use "Windowed" locks.
*   **Implementation:**
    *   Schedules a `WorkManager` task to wake up every 15 minutes, check for messages, and go back to sleep immediately.
    *   This prevents the CPU from staying awake for 10 hours straight.

## Conclusion for Columba's Roadmap
Columba can never match Signal's battery life because it does the "hard work" of being the server and the client at the same time. However, by implementing **Tapered Back-off** and **Context-Aware Scanning**, we can move from "Battery Drainer" to "Efficient Background Service."

**Verdict:** The current "Deep Idle" fix is the most impactful first step. The next "Pro" move would be integrating the Activity Recognition API to stop scanning when the phone is stationary.
