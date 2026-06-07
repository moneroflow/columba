# Bluetooth Audio Routing Research for Columba VoIP

## Objective
Enable seamless Bluetooth headset support (mic and audio) for the Columba voice calling system, similar to the implementation found in professional VoIP apps like Signal.

## Technical Overview: Android Bluetooth Audio
In Android, Bluetooth audio for calls is handled differently than "media" audio (like Spotify). VoIP apps must explicitly manage the routing to ensure the headset's microphone is used instead of the phone's bottom mic.

### 1. The Two Main Bluetooth Profiles
*   **A2DP (Advanced Audio Distribution Profile):** Used for high-quality music. It is **one-way** (Phone $\rightarrow$ Headset). Using this for calls results in the user hearing the other person in the headset but their own voice coming from the phone's microphone.
*   **HFP/HSP (Hands-Free / Headset Profile):** Used for calls. It is **two-way** (Bi-directional). This triggers the **SCO (Synchronous Connection Oriented)** channel, which is lower quality but allows the headset microphone to function.

### 2. Implementation Strategy (The "Signal" Approach)
To implement this, Columba will need to manage the `AudioManager` and `BluetoothAdapter` in the following sequence:

#### A. Audio Mode Switching
When a call starts, the app must switch the system audio mode:
```kotlin
audioManager.mode = AudioManager.MODE_IN_COMMUNICATION
```
This tells Android that the app is now a "communication" app, which prioritizes voice routing over media.

#### B. Routing to Bluetooth SCO
To force the audio to the headset mic and speaker:
1.  Check if a Bluetooth headset is connected.
2.  Start the SCO connection:
    ```kotlin
    audioManager.startBluetoothSco()
    audioManager.isBluetoothScoOn = true
    ```
3.  **The Challenge:** `startBluetoothSco()` is asynchronous. The app must listen for a `ACTION_SCO_AUDIO_STATE_UPDATED` broadcast to confirm the headset is actually ready before starting the audio stream.

#### C. Handling the "Call Screen" UI
Professional apps (like Signal) implement a "Bluetooth/Speaker" toggle button. 
*   **Toggled ON:** Explicitly calls `startBluetoothSco()`.
*   **Toggled OFF:** Calls `stopBluetoothSco()` and sets the mode back to `MODE_NORMAL` or routes to the earpiece.

### 3. Potential Pitfalls for Columba
*   **Permissions:** Requires `BLUETOOTH_CONNECT` (Android 12+) and `MODIFY_AUDIO_SETTINGS`.
*   **Latency:** Since Columba uses a mesh network (Reticulum), we must ensure the Bluetooth SCO overhead doesn't introduce jitter into the already constrained bandwidth of the mesh.
*   **Device Variability:** Some headsets handle the transition from A2DP to SCO automatically; others require the app to be explicit.

## Summary for Development
To fix "Bluetooth audio not working in calls," we need to:
1. Set `AudioManager.MODE_IN_COMMUNICATION`.
2. Programmatically trigger `startBluetoothSco()`.
3. Implement a listener for the SCO state change to verify the routing.
4. Add a UI toggle on the call screen to let the user manually switch between "Phone" and "Bluetooth."
