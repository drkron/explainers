
## `windowAudio` option for `getDisplayMedia()`

### Introduction

The `getDisplayMedia()` API allows web applications to capture a user's display, offering choices to share a full monitor, a specific window, or a single browser tab. When `audio: true` is set in the `DisplayMediaStreamOptions`, the user agent presents options for audio capture, which can include tab, system, or application audio.

This explainer proposes the addition of a new `windowAudio` option to `DisplayMediaStreamOptions`. This option provides a hint to the user agent about the application's preference for audio sources (specifically `window` or `system` audio, or `exclude` audio entirely) when the user selects a **window** as the display surface.

### Background and Motivation

Currently, when a user selects a window for screen sharing via `getDisplayMedia()`, the user agent decides what audio options to present. While the core principle of screen sharing is that **the user always decides**, the user is often busy and benefits from choices with reasonable defaults.

Web applications, however, frequently possess richer context about the user's intent and can offer a valuable hint to the user agent.

Consider these scenarios:

1.  **Capturing a Video Player:** If a user is capturing a video player application, they likely intend to share only the audio from that specific video clip. In this case, offering **application (window) audio** would be the most intuitive default.
2.  **Recording a Lecture with Background Music:** A user might be recording themselves giving a lecture, displaying sheet music in a PDF viewer, while simultaneously playing background music through a separate music player application. Here, the user likely intends to capture all sounds originating from their computer, implying a preference for **system audio**.

Existing functionality addresses preferences for **monitor** surfaces with `SystemAudioPreferenceEnum` (allowing `include` or `exclude` system audio). However, there's a gap for **window** surfaces, where the application may prefer specific audio behavior (either window-specific or system-wide audio) that differs from the user agent's default.

The `windowAudio` option aims to bridge this gap, enabling applications to express their audio sharing preference specifically for window captures, thereby improving the user experience by offering more relevant audio choices by default.

### Proposed Solution

We propose adding a new option, `windowAudio`, to the `DisplayMediaStreamOptions` dictionary. A developer would use it like this:

```javascript
// A simple example preferring window-specific audio
const stream = await navigator.mediaDevices.getDisplayMedia({
  video: true,
  audio: true,
  windowAudio: "window" // New option
});
```

This option accepts a `WindowAudioPreferenceEnum` with the following possible values: `system`, `window`, or `exclude`.

Formally, the new members would be defined in WebIDL as:
```webidl
dictionary DisplayMediaStreamOptions {
  boolean video = false;
  boolean audio = false;
  // ... other options
  WindowAudioPreferenceEnum windowAudio; // New option
};

enum WindowAudioPreferenceEnum {
  "system",
  "window",
  "exclude"
};
```

**`WindowAudioPreferenceEnum` Values:**

* **`"system"`**: The application prefers that options to share **system audio** be offered to the user when a window display surface is selected.
* **`"window"`**: The application prefers that options to share **audio specifically from the selected window** be offered to the user when a window display surface is selected.
* **`"exclude"`**: The application prefers that options to share audio **not be offered** to the user at all when a window display surface is selected, even if `audio: true` is also set. (In this case, `audio: true` would essentially be ignored for window captures).

**Important Note:** As with other `getDisplayMedia()` options, the user agent **MAY ignore this hint**. The user retains ultimate control over what is shared, and the user agent's UI will reflect the available choices. This option merely informs the user agent's initial presentation or default selection of audio sources.

### Use Cases and Examples

**1. Preferring Window-Specific Audio (e.g., video player sharing):**

```javascript
// Application wants to share a video player window,
// and prefers to only capture audio from that specific window.
navigator.mediaDevices.getDisplayMedia({
  video: true,
  audio: true,
  windowAudio: "window"
})
.then(stream => {
  // Use the captured stream
})
.catch(error => {
  console.error("Error capturing media:", error);
});
```
*User Agent Behavior Hint:* When the user is prompted to select a window, the UI might default to selecting "Share window audio" if available, or prominently display that option.

**2. Preferring System Audio (e.g., lecture with background music):**

```javascript
// Application wants to capture a presentation window,
// but also wants to ensure all system audio (like background music) is captured.
navigator.mediaDevices.getDisplayMedia({
  video: true,
  audio: true,
  windowAudio: "system"
})
.then(stream => {
  // Use the captured stream
})
.catch(error => {
  console.error("Error capturing media:", error);
});
```
*User Agent Behavior Hint:* When the user is prompted to select a window, the UI might default to selecting "Share system audio" or prominently display that option.

**3. Explicitly Excluding Audio for Window Captures:**

```javascript
// Application only needs video when a window is selected,
// and wants to avoid presenting audio options for windows.
navigator.mediaDevices.getDisplayMedia({
  video: true,
  audio: true, // Audio might be desired for other surface types (e.g., monitor)
  windowAudio: "exclude"
})
.then(stream => {
  // Use the captured stream
})
.catch(error => {
  console.error("Error capturing media:", error);
});
```
*User Agent Behavior Hint:* If the user selects a window, the audio sharing option would not be presented, even though `audio: true` was set. The audio would only be captured if a tab or monitor was selected (and if `SystemAudioPreferenceEnum` allowed for monitor audio).

### Privacy and Security Considerations

* **User Control:** This feature *does not* grant the application more control over what is captured. It only provides a *hint* to the user agent's UI. The user's consent UI remains the ultimate arbiter of what audio source is shared.
* **Transparency:** The user agent is expected to clearly communicate to the user which audio source (if any) is being shared, regardless of the hint provided.
* **No New Information Disclosure:** The `windowAudio` option does not expose any new information about the user's system or activity to the web application beyond what is already implicitly handled by `getDisplayMedia()`.

### Alternatives Considered

* **Extending `SystemAudioPreferenceEnum`:** We considered extending the existing `systemAudio` option. However, this would overload its original purpose (which is for monitor captures) and create a less intuitive API. For instance, one possible implementation might look like this:
```javascript
navigator.mediaDevices.getDisplayMedia({
  video: true,
  audio: true,
  systemAudio: "include_preferring_window"
});
```
This approach muddies the water, making the option's name less descriptive and its behavior harder to reason about. Creating a separate `windowAudio` option provides a much clearer and more explicit API for developers.
* **No Explicit Hint:** The alternative is to leave the behavior entirely up to the user agent.
```javascript
navigator.mediaDevices.getDisplayMedia({
  video: true,
  audio: true
});
```
While user agents do their best to provide reasonable defaults, this would miss opportunities for web applications to improve the user experience by leveraging their greater context about the user's intent.

### Interoperability

This feature aligns with the existing pattern of `getDisplayMedia()` options, where applications provide hints to user agents that maintain ultimate user control. This approach should be readily adoptable by different browser engines.

