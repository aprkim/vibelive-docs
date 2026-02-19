# VibeLive Integration Guide

**Combined reference — MiniChat API Guide v0.63 + Design Guide v2.1**

Last updated: 2026-02-18

---

## Authority Rule

When the API Guide and Design Guide conflict:

- **API Guide (Part 1)** defines system behavior and lifecycle semantics
- **Design Guide (Part 2)** defines visual presentation and interaction rules

For visual conflicts, the Design Guide wins. See [CONFLICTS.md](CONFLICTS.md) for the full list of known conflicts and their resolutions.

---

---

# Part 1 — MiniChat API Guide

**Build an anonymous video chat app with MiniChat**

Version 0.63 | February 18, 2026

> **Note:** Code examples in this section have been updated to comply with the Design Guide's visual rules (inline SVG icons instead of emoji, "Your Screen" label for local screenshare, LIVE badge on screenshare tiles). See [CONFLICTS.md](CONFLICTS.md) for details.

---

## Setup

Import and initialize MiniChat:

```html
<script type="module">
    import MiniChat from 'https://proto2.makedo.com:8883/v05/scripts/makedo-vibelive.min.js';

    MiniChat.init({
        contextId: 'YOUR_CONTEXT_ID',
        contextAuthToken: 'YOUR_CONTEXT_TOKEN'
    });

    // Wire up events...
    MiniChat.on('channelSelected', (channel) => { /* ... */ });
    MiniChat.on('remoteStreamStart', (memberId, streamType) => { /* ... */ });
</script>

<!-- onclick handlers just work — no window.xxx workaround needed -->
<button onclick="MiniChat.signup('Alex')">Join</button>
<button onclick="MiniChat.startLive()">Go Live</button>
```

> **🧪 Test Credentials**: Use `contextId: 'Kw6w6w6w6w'` and `contextAuthToken: 'qftRdeQ12ZcrKYixauWpxGiB'` for quick testing.

`MiniChat` is automatically available on `window` — all methods work directly in `onclick` handlers without any extra wiring.

---

## User Flow

Anonymous users have two paths — **create** or **join** a room:

```
signup(name) → createRoom(title) → enterByRoomCode(code) → startLive()
signup(name) → enterByRoomCode(code) → startLive()
```

### Create a Room

```javascript
await MiniChat.signup('Alex');
const room = await MiniChat.createRoom("Alex's Room");
// room.room_code is the shareable code (e.g., "X7kQ3m")
await MiniChat.enterByRoomCode(room.room_code);
// Now in PRE-LIVE — call startLive() when ready
```

### Join a Room

```javascript
await MiniChat.signup('Jordan');
await MiniChat.enterByRoomCode('X7kQ3m');
// Now in PRE-LIVE — call startLive() when ready
```

### Change Display Name on Rejoin

Pass an optional `displayName` to update your name when entering a channel:

```javascript
// Exit current room
await MiniChat.exitRoom();

// Re-enter with new display name (no page reload needed)
await MiniChat.enterByRoomCode('X7kQ3m', 'NewName');
// Other members will receive a member_config_update event with your new name
```

### Always `await` Authentication

`signup()` must complete before any other calls. It establishes the session and WebSocket connection.

---

## Member Lifecycle: PRE-LIVE → LIVE → EXIT

```
PRE-LIVE    →    LIVE    →    PRE-LIVE or EXIT
(preparing)      (streaming)   (back or gone)
```

| State | What's happening | How to enter |
|-------|-----------------|--------------|
| **PRE-LIVE** | Channel selected, no WebRTC | `enterByRoomCode()` |
| **LIVE** | WebRTC connected, sending/receiving media | `startLive()` |
| **PRE-LIVE** | WebRTC disconnected, still in channel | `stopLive()` |
| **EXIT** | Fully departed, camera released | `exitRoom()` |

- `startLive()` — Connect WebRTC, go LIVE
- `stopLive()` — Disconnect WebRTC, return to PRE-LIVE (quick rejoin possible)
- `exitRoom()` — Full teardown, release camera/mic, stay logged in (can enter a different room)
- `backToList()` — Like `exitRoom()` but intended for multi-room flows: stops media and clears the channel, returning the user to a room-selection state without logging out
- `logout()` — Full session teardown including authentication

---

## Media Controls

### Toggle vs Mute

Two different concepts — understand the difference:

| Action | What Happens | Camera Light | Others See |
|--------|--------------|--------------|------------|
| `toggleVideo()` | Start/stop capture | On/Off | Video appears/disappears |
| `toggleMuteVideo()` | Hide while capturing | Stays On | Black frame |
| `toggleAudio()` | Start/stop microphone | — | Audio appears/disappears |
| `toggleMuteAudio()` | Silence while capturing | — | Silence |

`toggleScreenshare()` starts/stops screen sharing.

### Recommended UX Pattern

**Camera button:** Use `toggleVideo()`
- Users expect the camera light to turn OFF (privacy)
- Stopping capture releases system resources

**Microphone button:** Use `toggleMuteAudio()` (after initial `toggleAudio()` to start capture)
- Users expect instant unmute (common in meetings)
- Keeps microphone warm — no permission prompt when unmuting

**Example:**
```javascript
// On room entry: start both
await MiniChat.startLive();

// User clicks mic button (during call)
MiniChat.toggleMuteAudio();  // Mute/unmute — instant, no hardware restart

// User clicks camera button (during call)
MiniChat.toggleVideo();      // Stop/start — hardware light on/off
```

Use `toggleMuteVideo()` only for specialized cases (e.g., "hide self while fixing appearance" but keep capturing).

### Screenshare

Screen sharing is typically offered in **LIVE mode only**. In both cases, follow the Element-First Rule: create and register the screenshare tile in `channelSelected` (hidden with `display: none`).

### Reading Local Media State

```javascript
MiniChat.mediaState
// → { audio: true/false, video: true/false, audioMuted: true/false, videoMuted: true/false }

MiniChat.screenState
// → { video: true/false, videoMuted: true/false }
```

### Reading Remote Media State

```javascript
const states = MiniChat.getMediaStates(memberId);
// states.cam_audio_detail: 'ON' | 'MUTED' | 'OFF'
// states.cam_video_detail: 'ON' | 'HIDDEN' | 'OFF'
// states.screen_audio_detail: 'ON' | 'MUTED' | 'OFF'
// states.screen_video_detail: 'ON' | 'HIDDEN' | 'OFF'
```

### Local vs Live Media State

There are two separate "worlds" of media state:

| Source | What it reflects | Use for |
|--------|-----------------|--------|
| `MiniChat.mediaState` | **Hardware state** — is the camera/mic capturing? | Local user's indicators |
| `MiniChat.getMediaStates(id)` | **WebRTC state** — what's being transmitted? | Remote member indicators |

In **PRE-LIVE**, the local camera can be on (for preview) but nothing is transmitted. This is correct — the local indicator reflects what the user cares about: *"Is my camera on?"*

In **LIVE**, the two states naturally align. No special synchronization logic is needed.

**Always use `MiniChat.mediaState` for the local user's indicators** — never `MiniChat.getMediaStates(MiniChat.memberId)`, which reflects WebRTC state and won't be meaningful in PRE-LIVE.

---

## The Element-First Rule

> **For local streams: register video elements before streams arrive, not in response to them.**

When MiniChat creates a media stream, it immediately attaches to whatever `<video>` element you've registered. No element registered = stream silently lost.

| Media Type | When to Register | Why |
|---|---|---|
| **Local camera** | At tile creation time | `toggleVideo()` attaches immediately |
| **Local screenshare** | At tile creation time (hidden) | `toggleScreenshare()` attaches immediately |
| **Remote camera** | Inside `remoteStreamStart` handler | Stream is arriving *right now* |
| **Remote screenshare** | Inside `remoteStreamStart` handler | Stream is arriving *right now* |

> **Note:** The "before streams arrive" requirement applies only to **local** streams. For remote streams, `remoteStreamStart` is both the signal and the correct registration moment — there is nothing to pre-create.

```javascript
// ✅ Register both local elements at tile creation
MiniChat.setLocalCamera(cameraVideoEl);
MiniChat.setLocalScreen(screenVideoEl);

// ❌ DON'T create elements in localMediaChange — too late!
```

**Why this matters for screenshare:** When you call `toggleScreenshare()`, the stream is produced **instantly**. If you wait until `localMediaChange` fires to create the tile and call `setLocalScreen()`, the stream has already been produced with no element to attach to — result: blank screen. The element must exist and be registered **before** the user clicks the screenshare button.

---

## Building Video Tiles

Each video stream gets its own **independent tile** identified by `tile-{memberId}-{streamType}`. This gives maximum layout flexibility — screenshare tiles can span multiple grid columns, expand on click, or move to a featured area.

### createVideoTile() Pattern

```javascript
function createVideoTile(memberId, name, streamType, isLocal) {
    const tileId = `tile-${memberId}-${streamType}`;
    if (document.getElementById(tileId)) return; // Guard duplicates

    const tile = document.createElement('div');
    tile.className = 'video-tile';
    tile.id = tileId;
    tile.dataset.memberId = memberId;
    tile.dataset.streamType = streamType;

    // Video container
    const videoContainer = document.createElement('div');
    videoContainer.className = 'video-container';

    const placeholder = document.createElement('div');
    placeholder.className = 'video-placeholder';
    if (streamType === 'camera') {
        placeholder.innerHTML = `<span>${isLocal ? `${name} (You)` : name}</span>`;
    } else {
        placeholder.innerHTML = `<span>${isLocal ? 'Your' : name + "'s"} Screen</span>`;
    }

    const video = document.createElement('video');
    video.autoplay = true;
    video.playsInline = true;   // Required for iOS
    video.muted = (isLocal || streamType === 'screenshare');  // Prevent echo

    videoContainer.appendChild(placeholder);
    videoContainer.appendChild(video);

    // Info bar
    const memberInfo = document.createElement('div');
    memberInfo.className = 'member-info';
    if (streamType === 'camera') {
        memberInfo.innerHTML = `
            <span class="member-name">${isLocal ? `${name} (You)` : name}</span>
            <div class="member-indicators">
                <span class="status-badge"></span>
                <span class="indicator cam-video" title="Camera"><!-- SVG icon --></span>
                <span class="indicator cam-audio" title="Microphone"><!-- SVG icon --></span>
            </div>
        `;
    } else {
        memberInfo.innerHTML = `
            <span class="member-name">${isLocal ? 'Your' : name + "'s"} Screen</span>
            <div class="member-indicators">
                <span class="status-badge">LIVE</span>
            </div>
        `;
    }

    tile.appendChild(videoContainer);
    tile.appendChild(memberInfo);
    // Route screenshare tiles to a separate container if one exists
    const targetGrid = streamType === 'screenshare'
        ? (document.getElementById('screenshareGrid') || document.getElementById('videoGrid'))
        : document.getElementById('videoGrid');
    targetGrid.appendChild(tile);

    // Register local elements immediately (Element-First Rule)
    if (isLocal) {
        if (streamType === 'camera') {
            MiniChat.setLocalCamera(video);
        } else {
            MiniChat.setLocalScreen(video);
        }
    }
}
```

**Key points:**
- Each tile is independent — camera and screenshare are separate grid items
- `data-member-id` and `data-stream-type` attributes enable CSS targeting
- CSS example: `.video-tile[data-stream-type="screenshare"] { grid-column: span 2; }`
- `playsInline` is required for iOS
- `muted = true` on local video prevents audio feedback
- **Screenshare placement:** `createVideoTile()` routes screenshare tiles to `#screenshareGrid` if it exists, falling back to `#videoGrid`. Add `<div id="screenshareGrid"></div>` above `videoGrid` in your HTML to place screenshare tiles in a separate area — no code changes needed.
- **Icons:** Use inline SVG icons for camera and microphone indicators — no emoji (see Design Guide §Core Design Philosophy and §Media Indicators)
- **Screenshare labels:** Local user sees "Your Screen"; remote users see "Name's Screen" (see Design Guide §Screen Share Tile Semantics)
- **Screenshare LIVE badge:** Screenshare tiles include a LIVE badge matching the styling of camera tiles (see Design Guide §Screen Share Tile Semantics)

---

## Moving Video Tiles (Preview Areas, Featured Views)

When implementing preview modes or featured speaker layouts, **always move the existing DOM element** — never remove and recreate.

### ✅ The Safe Way: appendChild()

```javascript
// Move to preview area (PRE-LIVE)
const tile = document.getElementById(`tile-${MiniChat.memberId}-camera`);
document.getElementById('previewArea').appendChild(tile);

// Move back to main grid (LIVE)
document.getElementById('videoGrid').appendChild(tile);
```

**Why this works:** `appendChild()` **moves** the element. The `<video>` element's `srcObject` persists. Element registration remains valid. No re-registration needed.

### ❌ Mistakes That Break Streams

```javascript
// ❌ Remove then recreate — stream lost
oldTile.remove();
createVideoTile(...);

// ❌ Clone the element — new element has no stream
const clone = original.cloneNode(true);

// ❌ Replace innerHTML — destroys the original element
container.innerHTML = tile.outerHTML;
```

---

## Event Handlers

### Registering Events

Use `MiniChat.on(event, callback)`:

```javascript
// Connection state (local)
MiniChat.on('localJoined', () => { });       // You went LIVE
MiniChat.on('localLeft', () => { });         // You returned to PRE-LIVE

// Remote members
MiniChat.on('remoteJoined', (memberId) => { });
MiniChat.on('remoteLeft', (memberId) => { });
MiniChat.on('memberUpdate', (memberId) => { });   // Status changed (includes self!)

// Streams
MiniChat.on('remoteStreamStart', (memberId, streamType) => { });  // 'camera' or 'screenshare'
MiniChat.on('remoteStreamEnd', (memberId, streamType) => { });
MiniChat.on('remoteMediaChange', (memberId, streamType) => { });  // Mute/unmute

// Local media
MiniChat.on('localMediaChange', () => { });

// Channel
MiniChat.on('channelSelected', (channel) => { });

// Errors
MiniChat.on('error', (context, error) => { });
```

Events can be chained:
```javascript
MiniChat.on('localJoined', handleJoined)
        .on('localLeft', handleLeft)
        .on('error', handleError);
```

### Handling Remote Streams

When a remote member starts streaming, create their tile and attach the stream:

```javascript
MiniChat.on('remoteStreamStart', (memberId, streamType) => {
    if (!MiniChat.isLive) return;  // Privacy-aware: only show when you're live

    const m = MiniChat.getMember(memberId);
    const tileId = `tile-${memberId}-${streamType}`;

    // Create tile on demand if needed (streams can arrive before join events!)
    let tile = document.getElementById(tileId);
    if (!tile) {
        createVideoTile(memberId, m?.displayName || 'Unknown', streamType, false);
        tile = document.getElementById(tileId);
    }
    if (!tile) return;

    const placeholder = tile.querySelector('.video-placeholder');
    const video = tile.querySelector('video');

    if (placeholder) placeholder.classList.add('hidden');
    if (video) {
        video.classList.add('visible');
        if (streamType === 'camera') {
            MiniChat.setRemoteCamera(memberId, video);
        } else {
            MiniChat.setRemoteScreen(memberId, video);
        }
    }

    updateMemberIndicators(memberId, streamType);
});

MiniChat.on('remoteStreamEnd', (memberId, streamType) => {
    const tileId = `tile-${memberId}-${streamType}`;
    const tile = document.getElementById(tileId);
    if (!tile) return;

    if (streamType === 'screenshare') {
        // Screenshare: remove tile entirely
        tile.remove();
    } else {
        // Camera: show placeholder (audio may still be active)
        const video = tile.querySelector('video');
        const placeholder = tile.querySelector('.video-placeholder');
        if (video) video.classList.remove('visible');
        if (placeholder) placeholder.classList.remove('hidden');
    }

    updateMemberIndicators(memberId, streamType);
});
```

> **⚠️ Always create tiles on demand in `remoteStreamStart`.**
> When you call `startLive()`, WebRTC begins immediately. If others are already streaming, `remoteStreamStart` can fire **before** `remoteJoined`. If the tile doesn't exist, create it right there. Since `createVideoTile()` guards against duplicates, this is always safe.

### Video Placeholder vs Stream Presence

A stream can exist with only an audio track (video stopped). Use `remoteMediaChange` to check the actual video state:

```javascript
MiniChat.on('remoteMediaChange', (memberId, streamType) => {
    const states = MiniChat.getMediaStates(memberId);
    const tileId = `tile-${memberId}-${streamType}`;
    const tile = document.getElementById(tileId);
    if (!tile) return;

    const video = tile.querySelector('video');
    const placeholder = tile.querySelector('.video-placeholder');

    // Show placeholder when video is not ON (handles 'OFF' and 'HIDDEN')
    let showPlaceholder = false;
    if (streamType === 'camera') {
        showPlaceholder = states.cam_video_detail !== 'ON';
    } else {
        showPlaceholder = states.screen_video_detail !== 'ON';
    }

    if (showPlaceholder) {
        video?.classList.remove('visible');
        placeholder?.classList.remove('hidden');
    } else {
        video?.classList.add('visible');
        placeholder?.classList.add('hidden');
    }

    updateMemberIndicators(memberId, streamType);
});
```

### Handling Local Media Changes

MiniChat attaches streams but **never controls visibility**. Use `localMediaChange` to show/hide your own video:

```javascript
MiniChat.on('localMediaChange', () => {
    const s = MiniChat.mediaState;
    const screen = MiniChat.screenState;

    // Camera tile
    const camTile = document.getElementById(`tile-${MiniChat.memberId}-camera`);
    if (camTile) {
        const placeholder = camTile.querySelector('.video-placeholder');
        const video = camTile.querySelector('video');
        if (s.video) {
            placeholder?.classList.add('hidden');
            video?.classList.add('visible');
        } else {
            placeholder?.classList.remove('hidden');
            video?.classList.remove('visible');
        }
    }

    // Screenshare tile (hide tile entirely when inactive)
    const screenTile = document.getElementById(`tile-${MiniChat.memberId}-screenshare`);
    if (screenTile) {
        if (screen.video) {
            screenTile.style.display = 'flex';
            screenTile.querySelector('.video-placeholder')?.classList.add('hidden');
            screenTile.querySelector('video')?.classList.add('visible');
        } else {
            screenTile.style.display = 'none';
        }
    }

    updateMemberIndicators(MiniChat.memberId);
});
```

**Important:** Local video is attached **directly** when you call `toggleVideo()`, not via WebRTC events. `remoteStreamStart` only fires for **remote** members.

### Handling Member Left

Check whether they truly exited or just returned to PRE-LIVE:

```javascript
MiniChat.on('remoteLeft', (memberId) => {
    const m = MiniChat.getMember(memberId);
    if (m?.displayStatus === 'INACTIVE') {
        // Remove tiles — they've left
        document.getElementById(`tile-${memberId}-camera`)?.remove();
        document.getElementById(`tile-${memberId}-screenshare`)?.remove();
    }
    // If PRE-LIVE, keep tiles — they may rejoin
});
```

---

## Privacy-Aware Visibility

Recommended pattern: only show remote member tiles when the local user is LIVE. This gives users a private "green room" in PRE-LIVE for adjusting camera/mic.

```javascript
MiniChat.on('channelSelected', async (channel) => {
    // PRE-LIVE: show only your own tiles
    const members = await MiniChat.getMembers();
    const self = members.find(m => m.id === MiniChat.memberId);
    if (self) {
        createVideoTile(self.id, self.displayName, 'camera', true);
        createVideoTile(self.id, self.displayName, 'screenshare', true);
        // screenshare tile starts hidden (Element-First Rule)
    }
});

MiniChat.on('localJoined', async () => {
    // Going LIVE: reveal remote members
    const members = await MiniChat.getMembers();
    members.forEach(m => {
        if (m.id === MiniChat.memberId) return;
        if (m.displayStatus === 'ACTIVE' || m.displayStatus === 'PRE-LIVE') {
            createVideoTile(m.id, m.displayName, 'camera', false);
        }
    });
});

MiniChat.on('localLeft', () => {
    // Back to PRE-LIVE: remove remote tiles for privacy
    document.querySelectorAll('.video-tile').forEach(tile => {
        if (tile.dataset.memberId !== MiniChat.memberId) {
            tile.remove();
        }
    });
});

MiniChat.on('remoteJoined', (memberId) => {
    if (!MiniChat.isLive) return;  // Don't show if we're not live
    const m = MiniChat.getMember(memberId);
    createVideoTile(memberId, m?.displayName || 'Unknown', 'camera', false);
});
```

> **Design choice: who gets a tile on `localJoined`?**
>
> The example above creates tiles for both `'ACTIVE'` and `'PRE-LIVE'` members when you go LIVE. This is the recommended default — you see everyone already in the channel, including those still preparing.
>
> You can restrict to `'ACTIVE'` only if your app only wants to show members who are actively streaming:
> ```javascript
> // Variation: streamers only
> if (m.displayStatus === 'ACTIVE') {
>     createVideoTile(m.id, m.displayName, 'camera', false);
> }
> ```
> The tradeoff:
> - **`'ACTIVE' || 'PRE-LIVE'`** — everyone present gets a tile immediately; good for small groups and social apps
> - **`'ACTIVE'` only** — tiles appear only when streaming starts; better for larger rooms or broadcast-style apps where PRE-LIVE presence is invisible by design
>
> Note: `remoteJoined` fires when a member goes LIVE (not when they enter PRE-LIVE), so it always represents an `'ACTIVE'` member — no filtering needed there.

```javascript

MiniChat.on('remoteStreamStart', (memberId, streamType) => {
    if (!MiniChat.isLive) return;  // Don't show if we're not live
    // ... attach stream (see Handling Remote Streams above)
});
```

---

## Member Info

```javascript
const member = MiniChat.getMember(memberId);
member.displayName     // "Alex"
member.displayStatus   // 'ACTIVE', 'PRE-LIVE', or 'INACTIVE'
member.hasCamera       // boolean
member.hasScreenshare  // boolean

// Fetch all current channel members from server
const members = await MiniChat.getMembers();
// → [{ id, displayName, displayStatus, ... }]
```

Use `MiniChat.memberId` for your own member ID. Use `MiniChat.roomCode` for the shareable room code.

`memberUpdate` fires for ALL members including yourself — useful for updating status badges.

---

## Complete Minimal Example

A working app in under 50 lines of JavaScript:

```html
<!DOCTYPE html>
<html>
<head>
    <title>MiniChat Demo</title>
    <style>
        .video-grid { display: flex; flex-wrap: wrap; gap: 10px; }
        .video-tile { position: relative; width: 320px; height: 240px; background: #222; }
        .video-tile video { width: 100%; height: 100%; object-fit: cover; }
        .video-placeholder { position: absolute; inset: 0; display: flex;
            align-items: center; justify-content: center; color: white; font-size: 1.2em; }
        .hidden { display: none; }
        .visible { display: block; }
    </style>
</head>
<body>
    <h2>MiniChat</h2>

    <div id="entry">
        <input id="name" placeholder="Your name">
        <button onclick="doCreate()">Create Room</button>
        <input id="code" placeholder="Room code">
        <button onclick="doJoin()">Join Room</button>
    </div>

    <div id="controls" style="display: none;">
        <span id="roomInfo"></span>
        <button onclick="MiniChat.startLive()">Go Live</button>
        <button onclick="MiniChat.stopLive()">Stop Live</button>
        <button onclick="MiniChat.toggleVideo()">Toggle Camera</button>
        <button onclick="MiniChat.toggleMuteAudio()">Mute/Unmute Mic</button>
    </div>

    <div id="videoGrid" class="video-grid"></div>

    <script type="module">
        import MiniChat from 'https://proto2.makedo.com:8883/v05/scripts/makedo-vibelive.min.js';

        MiniChat.init({ contextId: 'Kw6w6w6w6w', contextAuthToken: 'qftRdeQ12ZcrKYixauWpxGiB' });

        // --- Entry functions ---

        window.doCreate = async () => {
            await MiniChat.signup(document.getElementById('name').value || 'Guest');
            const room = await MiniChat.createRoom('My Room');
            await MiniChat.enterByRoomCode(room.room_code);
        };

        window.doJoin = async () => {
            await MiniChat.signup(document.getElementById('name').value || 'Guest');
            await MiniChat.enterByRoomCode(document.getElementById('code').value);
        };

        // --- Events ---

        MiniChat.on('channelSelected', async (channel) => {
            document.getElementById('entry').style.display = 'none';
            document.getElementById('controls').style.display = 'block';
            document.getElementById('roomInfo').textContent = `Room: ${MiniChat.roomCode}`;

            // Create local tile (Element-First Rule)
            const self = (await MiniChat.getMembers()).find(m => m.id === MiniChat.memberId);
            if (self) {
                createVideoTile(self.id, self.displayName, 'camera', true);
                createVideoTile(self.id, self.displayName, 'screenshare', true);
            }

            // Start camera + mic for preview
            await MiniChat.toggleVideo();
            await MiniChat.toggleAudio();
        });

        MiniChat.on('localJoined', async () => {
            const members = await MiniChat.getMembers();
            members.forEach(m => {
                if (m.id !== MiniChat.memberId &&
                    (m.displayStatus === 'ACTIVE' || m.displayStatus === 'PRE-LIVE')) {
                    createVideoTile(m.id, m.displayName, 'camera', false);
                }
            });
        });

        MiniChat.on('remoteJoined', (id) => {
            if (!MiniChat.isLive) return;
            const m = MiniChat.getMember(id);
            createVideoTile(id, m?.displayName || 'Unknown', 'camera', false);
        });

        MiniChat.on('remoteStreamStart', (id, type) => {
            if (!MiniChat.isLive) return;
            const m = MiniChat.getMember(id);
            let tile = document.getElementById(`tile-${id}-${type}`);
            if (!tile) {
                createVideoTile(id, m?.displayName || 'Unknown', type, false);
                tile = document.getElementById(`tile-${id}-${type}`);
            }
            const video = tile?.querySelector('video');
            if (video) {
                video.classList.add('visible');
                tile.querySelector('.video-placeholder')?.classList.add('hidden');
                if (type === 'camera') MiniChat.setRemoteCamera(id, video);
                else MiniChat.setRemoteScreen(id, video);
            }
        });

        MiniChat.on('remoteStreamEnd', (id, type) => {
            const tile = document.getElementById(`tile-${id}-${type}`);
            if (!tile) return;
            if (type === 'screenshare') { tile.remove(); return; }
            tile.querySelector('video')?.classList.remove('visible');
            tile.querySelector('.video-placeholder')?.classList.remove('hidden');
        });

        MiniChat.on('localMediaChange', () => {
            const s = MiniChat.mediaState;
            const camTile = document.getElementById(`tile-${MiniChat.memberId}-camera`);
            if (camTile) {
                const v = camTile.querySelector('video');
                const p = camTile.querySelector('.video-placeholder');
                if (s.video) { p?.classList.add('hidden'); v?.classList.add('visible'); }
                else { p?.classList.remove('hidden'); v?.classList.remove('visible'); }
            }
            const screenTile = document.getElementById(`tile-${MiniChat.memberId}-screenshare`);
            if (screenTile) screenTile.style.display = MiniChat.screenState.video ? 'block' : 'none';
        });

        MiniChat.on('remoteLeft', (id) => {
            const m = MiniChat.getMember(id);
            if (m?.displayStatus === 'INACTIVE') {
                document.getElementById(`tile-${id}-camera`)?.remove();
                document.getElementById(`tile-${id}-screenshare`)?.remove();
            }
            // If PRE-LIVE, keep tiles — they stopped streaming but haven't left
        });

        MiniChat.on('error', (ctx, err) => console.error(`[${ctx}]`, err.message));

        // --- Tile creation ---

        function createVideoTile(memberId, name, streamType, isLocal) {
            const tileId = `tile-${memberId}-${streamType}`;
            if (document.getElementById(tileId)) return;

            const tile = document.createElement('div');
            tile.className = 'video-tile';
            tile.id = tileId;
            tile.dataset.memberId = memberId;
            tile.dataset.streamType = streamType;
            if (streamType === 'screenshare' && isLocal) tile.style.display = 'none';

            tile.innerHTML = `
                <div class="video-placeholder"><span>${streamType === 'camera'
                    ? (isLocal ? `${name} (You)` : name)
                    : (isLocal ? 'Your' : name + "'s") + ' Screen'}</span></div>
                <video autoplay playsinline ${isLocal || streamType === 'screenshare' ? 'muted' : ''}></video>
            `;

            // Screenshare tiles go to screenshareGrid if present, otherwise videoGrid
            const targetGrid = streamType === 'screenshare'
                ? (document.getElementById('screenshareGrid') || document.getElementById('videoGrid'))
                : document.getElementById('videoGrid');
            targetGrid.appendChild(tile);

            if (isLocal) {
                const video = tile.querySelector('video');
                if (streamType === 'camera') MiniChat.setLocalCamera(video);
                else MiniChat.setLocalScreen(video);
            }
        }
    </script>
</body>
</html>
```

---

## API Quick Reference

### Methods

| Method | Description |
|--------|-------------|
| `init({ contextId, contextAuthToken, serverUrl? })` | Initialize MiniChat (required first) |
| `signup(name)` | Create anonymous guest session |
| `login(email, password)` | Login with existing account |
| `logout()` | Logout and cleanup |
| `createRoom(title)` | Create room, returns `{ id, room_code, title }` |
| `enterByRoomCode(code, displayName?)` | Enter room → PRE-LIVE (displayName optional, updates member name) |
| `startLive()` | Connect WebRTC → LIVE |
| `stopLive()` | Disconnect WebRTC → PRE-LIVE |
| `exitRoom()` | Full teardown, release camera/mic — stay logged in |
| `backToList()` | Stop media and clear channel — return to room selection without logout |
| `toggleAudio()` | Start/stop mic hardware |
| `toggleVideo()` | Start/stop camera hardware |
| `toggleScreenshare()` | Start/stop screen share |
| `toggleMuteAudio()` | Mute/unmute mic (keeps capturing) |
| `toggleMuteVideo()` | Hide/show video (keeps capturing) |
| `toggleMuteScreenshare()` | Hide/show screenshare (keeps capturing) |
| `getMember(memberId)` | Get member info |
| `getMemberIds()` | Get all member IDs |
| `getMembers()` | Fetch all channel members from server |
| `getMediaStates(memberId)` | Get `cam_audio_detail`, `cam_video_detail`, etc. |
| `getStream(memberId, streamType)` | Get raw MediaStream |
| `setLocalCamera(videoEl)` | Register local camera element |
| `setLocalScreen(videoEl)` | Register local screenshare element |
| `setRemoteCamera(id, videoEl)` | Register remote camera element |
| `setRemoteScreen(id, videoEl)` | Register remote screenshare element |
| `clearLocalCamera()` | Unregister local camera element |
| `clearLocalScreen()` | Unregister local screenshare element |
| `clearRemoteCamera(id)` | Unregister remote camera element |
| `clearRemoteScreen(id)` | Unregister remote screenshare element |
| `on(event, callback)` | Register event handler (chainable) |

### Getters

| Property | Type | Description |
|----------|------|-------------|
| `isLoggedIn` | boolean | Authenticated? |
| `isLive` | boolean | WebRTC connected (LIVE)? |
| `user` | Object | Current user |
| `memberId` | string | Your member ID |
| `roomCode` | string | Shareable room code |
| `channel` | Object | Current channel: `{ id, room_code, title, description, memberCount }` |
| `mediaState` | Object | `{ audio, video, audioMuted, videoMuted }` |
| `screenState` | Object | `{ video, videoMuted }` |
| `hasMedia` | boolean | Any local media active? |
| `core` | Object | Underlying MiniChatCore (escape hatch) |

### Events

| Event | Callback signature | Description |
|-------|-------------------|-------------|
| `login` | `(user)` | Logged in |
| `loginError` | `(error)` | Login failed |
| `logout` | `()` | Logged out |
| `channelSelected` | `(channel)` | Entered a room (PRE-LIVE). `channel`: `{ id, room_code, title, description, memberCount }` |
| `localJoined` | `()` | You went LIVE |
| `localLeft` | `()` | You returned to PRE-LIVE |
| `remoteJoined` | `(memberId)` | Remote member went LIVE |
| `remoteLeft` | `(memberId)` | Remote member stopped LIVE |
| `memberUpdate` | `(memberId)` | Member status changed (includes self) |
| `remoteStreamStart` | `(memberId, streamType)` | Remote stream arrived |
| `remoteStreamEnd` | `(memberId, streamType)` | Remote stream ended |
| `remoteMediaChange` | `(memberId, streamType)` | Remote mute/unmute |
| `localMediaChange` | `()` | Local media state changed |
| `error` | `(context, error)` | Error occurred |

---

## Common Mistakes

1. **Not calling `init()` first** — Every method throws `"Call MiniChat.init() first"` without it.

2. **Not awaiting `signup()`** — Session and WebSocket won't be ready for subsequent calls.

3. **Forgetting `startLive()`** — `enterByRoomCode()` only puts you in PRE-LIVE. You must call `startLive()` to connect WebRTC and start sending/receiving media.

   PRE-LIVE is intentional, not just a waiting room. In PRE-LIVE, the local camera and mic work normally — you can call `toggleVideo()` and `toggleAudio()` and the user will see their own preview. Nothing is transmitted to others yet. This is the "green room": users adjust their setup privately before going live.

   Common mistakes stemming from misunderstanding PRE-LIVE:
   - Calling `startLive()` immediately inside `channelSelected` — this skips the green room and connects WebRTC before the user is ready.
   - Expecting remote tiles to appear in PRE-LIVE — remote streams don't arrive until you call `startLive()`.
   - Being surprised that `toggleVideo()` works in PRE-LIVE but nobody else can see you — correct behaviour, by design.

4. **Creating video elements too late (Element-First Rule)** — Register elements *before* streams arrive. If you create a `<video>` element in `localMediaChange`, the stream is already lost.

   **⚠️ CRITICAL for screenshare:** The screenshare `<video>` element must exist and be registered with `setLocalScreen()` BEFORE calling `toggleScreenshare()`. Create both camera and screenshare tiles in `channelSelected`, with the screenshare tile hidden (`display: none`).

5. **Not creating tiles on demand in `remoteStreamStart`** — Stream events can arrive before `remoteJoined`. Always check if the tile exists and create it if needed.

6. **Missing `playsInline` on video elements** — Required for iOS. Videos won't autoplay inline without it.

7. **Not muting your own `<video>` element** — Always set `muted = true` on local video elements to prevent audio feedback/echo.

8. **Not checking status in `remoteLeft`** — A member with `displayStatus === 'PRE-LIVE'` hasn't left; they may rejoin. Only remove tiles for `'INACTIVE'` members.

9. **Using `getMediaStates()` for local indicators** — Use `MiniChat.mediaState` for the local user. `getMediaStates(MiniChat.memberId)` reflects WebRTC state and is meaningless in PRE-LIVE.

10. **Confusing toggle vs mute** — Camera: use `toggleVideo()` (hardware on/off). Microphone: use `toggleMuteAudio()` after initial startup (instant mute/unmute).

11. **Removing and recreating tiles to move them** — Use `appendChild()` to move the existing element. Never `.remove()` then recreate — the stream attachment is lost.

12. **Ignoring self status changes in `memberUpdate`** — While you shouldn't create tiles for yourself, you MUST update your UI controls (Start/Stop Live buttons, status text) when `memberId === MiniChat.memberId`. Check `member.displayStatus` and update buttons BEFORE returning:
   ```javascript
   MiniChat.on('memberUpdate', (memberId) => {
       const member = MiniChat.getMember(memberId);
       if (memberId === MiniChat.memberId) {
           // Update YOUR controls based on status
           if (member.displayStatus === 'ACTIVE') {
               startBtn.disabled = true;
               stopBtn.disabled = false;
           } else if (member.displayStatus === 'PRE-LIVE') {
               startBtn.disabled = false;
               stopBtn.disabled = true;
           }
           return;  // NOW skip tile operations
       }
       // Handle remote member tiles...
   });
   ```

---

*MiniChat API v1.0 — Facade over MiniChatCore. For advanced use, access `MiniChat.core` for the full MiniChatCore API.*

---

---

# Part 2 — Design Guide

Version: 2.1
Last updated: 2026-02-18

---

## Authority & Scope

This document defines **visual, interaction, and layout rules** for MiniChatCore-based apps.

### Authority Rule (Important)

When behavior or lifecycle rules conflict:

- **MINI_GUIDE defines system behavior and lifecycle semantics**
- **DESIGN_GUIDE defines visual presentation and interaction rules**

DESIGN_GUIDE must not override MiniChatCore lifecycle behavior.

---

## Lifecycle Model (Authoritative)

This guide follows the MiniChatCore lifecycle exactly:

```
PRE-LIVE → LIVE → PRE-LIVE / EXIT
```

### State Definitions

- **PRE-LIVE**
  - User is inside the room context
  - Camera and mic preview may be active
  - WebRTC is NOT connected
  - No media is transmitted

- **LIVE**
  - WebRTC is connected
  - Media is actively transmitted

- **EXIT**
  - User has fully left the room
  - Camera and mic are released
  - Tiles must be destroyed

> PRE-LIVE is NOT a lobby and NOT a partial join.

---

## Core Design Philosophy (from v2.0)

1. **Good defaults beat configuration** — if the user does nothing, the UI should still feel right.
2. **Visual fairness** — participants are equal unless explicitly designed otherwise.
3. **Space-aware layouts** — no cramped or floating tiles.
4. **Mobile-first, desktop-enhanced** — vertical clarity first, spatial balance later.
5. **Neutral first, accent second** — accents communicate meaning, not decoration.
6. **No emoji** — use inline SVG icons exclusively. Emoji are platform-inconsistent and not styleable.

---

## Color Theme & Tokens (Default: VibeLive Teal)

Default theme used unless overridden by product-specific theming.

- Accent: `#0EA5A4`
- Radius: `16px`
- Max content width: `980px`
- Font stack: `ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial`

### Canonical CSS Variables

```css
:root {
  --radius: 16px;
  --max: 980px;
  --sans: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;

  --bg: #f7f9f8;
  --card: #ffffff;
  --soft: #f0f4f3;
  --border: #e2e8f0;
  --borderSoft: #dbe2ea;

  --text: #1e293b;
  --muted: #64748b;
  --muted2: #9ca3af;

  --accent: #0EA5A4;
  --accentHover: #0d9488;
  --accentSoft: rgba(14,165,164,.10);
  --accentBorder: rgba(14,165,164,.35);

  --liveBg: #e8f4ed;
  --liveBorder: #a6c5a3;
  --liveText: #324252;

  --disabledBg: #f3f4f6;
  --disabledBorder: #e5e7eb;
  --disabledText: #6b7280;

  --dangerBg: #FEF2F2;
  --dangerBorder: #fca5a5;
  --dangerText: #b91c1c;

  --overlay: rgba(30, 41, 59, 0.5);
}
```

---

## Entry Flow

### Entry Screen (Required)

Use **two separated cards**:

#### Card Structure

```
┌─────────────────────────────┐
│  Name Card                  │
│  ┌───────────────────────┐  │
│  │ Your name             │  │
│  │ [___________________] │  │
│  └───────────────────────┘  │
└─────────────────────────────┘

┌─────────────────────────────┐
│  Action Card                │
│                             │
│  [ + Start a Room ] (primary)│
│                             │
│  ────────── OR ──────────   │
│                             │
│  Join with room code        │
│  [_________] [ Join ]       │
│                             │
└─────────────────────────────┘
```

- **Name card**: Label + text input (max 24 chars), **autofocus on load** — typing your name is the default CTA
- **Action card**: Primary CTA, OR divider, then room code input + Join button side by side
- Both actions are **always visible** — no hiding/showing based on code input
- **Join row layout**: Room code input takes **75%**, Join button takes **25%** (`flex: 0 0 25%`)

#### Disabled Button Styling

- Disabled buttons appear at **full opacity** — no dimming, no grey overrides
- Only `cursor: not-allowed` is applied to indicate the disabled state
- Buttons retain their full color identity (accent for primary, soft for secondary) at all times
- This ensures the visual hierarchy and color cues are clear even before the user has typed a name

#### Name Validation Hint

- When a user clicks a disabled button without entering a name, show a **hint message** below the name input: "Please enter your name first"
- The hint uses `--dangerText` color and fades in/out
- The name input is auto-focused so the user can immediately start typing
- The hint disappears after 3 seconds or when the user starts typing

#### Button Loading Behavior

- **"Start a Room"** → shows spinner + "Creating..." while connecting (indicates room creation in progress)
- **"Join"** → label stays as **"Join"** while connecting (button is disabled only, no text change)

#### Dynamic Button Priority

- **No room code entered** → "Start a Room" = primary (accent color), "Join" = secondary
- **Room code entered** → "Join" = primary (accent color), "Start a Room" = secondary
- This guides users naturally toward the most relevant action

#### URL Deep Linking (`?code=`)

- Auto-fill room code from URL parameter
- Triggers dynamic priority swap → "Join" becomes primary

#### Shareable Invite Link

- Copy/share actions must produce a full URL with `?code=<roomCode>`, not the raw code alone
- This ensures recipients land directly into the join flow via deep linking

---

## Pre-Live Setup Screen (PRE-LIVE)

Purpose:
- Camera preview
- Mic / camera toggles
- Readiness confirmation

Rules:
- **Same topbar as live screen** — app name (left), room code + copy-code + copy-link buttons (right)
- Content below topbar is **vertically and horizontally centered**
- Camera preview uses **~70% of available space** — balanced size that avoids feeling cramped or overwhelming (e.g. use `min(70%, calc((100vh - offset) * 16/9 * 0.7))`)
- The content wrapper must use `align-self: stretch` so percentage-based widths resolve against the full viewport width
- Controls below preview
- **Go Live / Enter Room** is primary CTA, placed in an action row below controls
- **Back button** sits alongside "Go Live" in the same row — secondary styling (soft background, border), navigates back to the entry screen, exits the channel, and releases camera/mic preview
- No other participants visible
- Camera off → initials placeholder (stable tile size)
- **No drag/resize handles** — tile interaction controls (drag handle, resize handle) are hidden in pre-live; they only appear on live room tiles

---

## Video Tile Rules (Unified)

### Tile Anatomy (Camera Tiles)

Each **camera tile** contains:
- Video OR initials placeholder
- Name label (bottom-left) — local user shows **"Name (You)"** (e.g. "April (You)"), remote users show their display name
- Camera + mic indicators (**always visible**)
- **LIVE status badge** — shown on all tiles (local and remote) when the participant is live. Uses `--liveBg` / `--liveText` styling. Badge is hidden when the participant is not live.

Tiles never contain controls.

> **Screen share tiles** are an exception — they show a name label and LIVE badge, but no mic/camera indicators. See [Screen Share Tile Semantics](#screen-share-tile-semantics).

### Initials Placeholder

When camera is off, show initials inside the placeholder:
- **Single name** (e.g. "April") → first letter only → **A**
- **Two or more names** (e.g. "April Kim") → first letter of first + last name → **AK**
- Empty/missing name → **?**

### Tile Sizing (Non-Negotiable)

- Size must be independent of camera state
- Use `::before` spacer (`padding-top: 56.25%`)
- Video and placeholder are absolute overlays
- **Hide browser-native video controls** — Chrome and Safari show play/pause overlays on `<video>` elements on hover. Suppress with `::-webkit-media-controls` pseudo-elements set to `display: none !important`

### Media Indicators

- Default state: cam OFF, mic OFF
- Icons required (no color-only meaning)
- **No emoji characters** — all icons must be inline SVG or icon font. Emoji rendering is inconsistent across platforms and does not meet design quality standards.

Indicator visibility must be **immediately obvious** at any tile size:
- ON → `--accent` icon color + accent-tinted background pill (`rgba(accent, .25)`)
- OFF → `--dangerText` icon color + danger-tinted background pill (`rgba(danger, .25)`)
- **On video tiles** (inside `.member-info`): indicators use **higher opacity backgrounds** (`rgba(accent, .7)` / `rgba(danger, .7)`) with white icon color to ensure visibility against any video content
- Do NOT rely on opacity alone to distinguish states — opacity differences are too subtle on dark backgrounds and at small sizes

---

## Tile Lifecycle Rules

- Create tile on room entry or member visibility
- **Remote tiles appear when the participant is LIVE or PRE-LIVE** — both active and pre-live members are visible to others
- **Do NOT remove camera tiles** on LIVE → PRE-LIVE — they may rejoin
- **Screenshare tiles are ephemeral** — remove entirely when sharing stops (`remoteStreamEnd` for screenshare)
- Remove camera tiles only when displayStatus = **INACTIVE** or explicit exit

---

## Screen Share (Presentation Mode)

### Screen Share Priority

- Multiple participants may share their screen simultaneously
- All active screen shares are displayed **side by side** in the screenshare area
- On mobile (narrow viewports), multiple screenshares **stack vertically**
- Screen shares always occupy the primary visual surface above camera tiles

### Screen Share Layout Rules

- Each screen share creates a **separate tile** inside a shared `#screenshareGrid` container
- Camera tiles always remain visible in the strip below
- Screen share tiles divide the available area equally (flexbox `flex: 1`)
- Each screen share must preserve its native aspect ratio
- Letterboxing is preferred over cropping
- Screen share tiles must never be visually smaller than participant camera tiles

### Participant Camera Strip

- Participant camera tiles are placed in a horizontal strip
- Default position: bottom
- Tiles are equal size
- Strip must not obscure critical screen content
- Strip may scroll horizontally if space is limited

### Screen Share Tile Semantics

- Screen share tiles:
  - Show a **name label** identifying whose screen is being shared (e.g. "April's Screen", "Your Screen")
  - Show **LIVE badge** (same styling as camera tiles)
  - Do NOT show mic/camera indicators
- Screen share is content with ownership attribution

### Screen Share Transitions

- Entering or exiting screen share must use a smooth layout transition
- Avoid sudden jumps or full reflow
- Participant tile positions should remain stable where possible

Known SDK limitation: local self screenshare preview may appear black.

---

## Tile Layout Decision Model (Authoritative)

Tile layout selection must follow these steps **in order**.
Do not jump directly from tile count to grid.

### Step 1: Determine Context

Inputs:
- Visible tile count (exclude hidden / `display: none` tiles)
- Viewport width, height, and aspect ratio
- Screen type: desktop or mobile

### Step 2: Preserve Aspect Ratio (Hard Rule)

- All tiles must preserve **16:9**
- Cropping is allowed only as a last resort
- Letterboxing is preferred over distortion
- **Distortion is never allowed**

### Step 3: Choose Row-Based Layout First

Before using multi-row grids, attempt:
- Single-row layouts for 1–3 participants (desktop)
- Balanced row splits for odd counts (e.g. 3+2, 4+3)

### Step 4: Center the Group

- The entire tile group must be **visually centered**
- Empty space must be distributed symmetrically
- No top-left anchoring

### Step 5: Expand to Grid Only When Necessary

Use multi-row grids only when:
- A single row would reduce tiles below a comfortable size
- Or the viewport aspect ratio makes rows impractical

Tiles must always attempt to maximize usable screen space **without violating aspect ratio or visual balance**.

A layout that fills more space but feels cramped or uneven is considered incorrect.

### Space-Maximizing Principle

Single-participant and pre-live tiles must **fill the available viewport**, not use small fixed widths. Size the tile based on viewport height to maintain 16:9 without overflow:

```css
/* Example: solo tile fills available space */
width: min(100%, calc((100vh - chrome_offset) * 16 / 9));
```

Where `chrome_offset` accounts for topbar, controls bar, and padding. This ensures the tile is as large as possible regardless of screen size, and adapts correctly when the window resizes.

### Preferred Desktop Row Patterns

| Tile Count | Preferred Layout |
|-----------:|------------------|
| 1 | Single tile, centered |
| 2 | 1 row x 2 |
| 3 | 1 row x 3 (fallback: 2 + 1 centered) |
| 4 | 2 x 2 |
| 5 | 3 + 2 (centered) |
| 6 | 3 x 2 |
| 7 | 4 + 3 |
| 8 | 4 x 2 |

Notes:
- Row splits must be centered as a group
- Avoid single orphan tiles on their own row

### Three-Participant Layout Rule

Default behavior (desktop):
- Use a single-row layout (1 × 3)
- All tiles equal size
- Group centered

Fallback behavior:
- A 2 + 1 layout is permitted only when a single-row layout would reduce tiles below minimum readable size
- Fallback must preserve visual balance and must not imply hierarchy
- The single tile must be horizontally centered beneath the top row

### Mobile (<=640px)

- 2–3: vertical stack
- 4+: 2-column grid
- Remove column spanning

### Tile Layout Anti-Patterns (Do Not Implement)

- Jumping layout when a participant toggles camera
- Reordering tiles based on audio level
- Host tiles being larger by default
- Left-aligned grids with empty trailing space
- Shrinking all tiles to fit a new participant instantly

When implementing tile layout logic, **prioritize visual balance over mathematical simplicity**.

---

## Topbar (Consistent Across Screens)

PRE-LIVE and LIVE screens must share an **identical topbar**:

- **Left**: App name (e.g. "TinyRoom")
- **Right**: Room code (monospace, accent-colored pill) + copy-code button + copy-link button

This gives users persistent access to sharing tools and maintains visual continuity across state transitions. The topbar sits at the top edge, outside the centered content area.

### Copy Button Feedback

- When a copy button (copy-code or copy-link) is clicked, a **"Copied!" tooltip** appears directly below the button
- The tooltip fades in, stays for 1.5 seconds, then fades out
- Styled with inverted colors (`--text` background, `--bg` text) for contrast against the topbar
- No toast notification — feedback is localized to the button itself

---

## Controls Placement

Primary controls:
- Mic
- Camera
- Leave room

Secondary:
- Screen share
- Settings

Placement:
- Desktop: bottom center bar
- Mobile: floating bottom bar

Leave button must be visually separated (danger affordance).

### Leave Room Behavior

When the user leaves a room:
- Return to the **entry screen** (create/join), NOT the pre-live screen
- All video tiles must be destroyed
- Camera and mic preview must be released
- Room state is fully reset — the user can start or join a new room immediately

---

## Theme Mode: Light / Dark (User Choice)

Users must be able to choose between **Light mode** and **Dark mode**. The choice applies globally across entry, pre-live, and live room states.

### Principles

- **Dark mode is the default**
- Light mode is available as an equal, first-class option
- Mode switching must not affect layout, sizing, or behavior
- Only colors, shadows, and contrast change — **structure stays identical**

### Implementation Rules

- Use CSS variables for all colors (no hard-coded values)
- Theme switch is implemented by toggling a root attribute or class:
  - `data-theme="dark"` (default)
  - `data-theme="light"`
- All components must derive colors from semantic tokens (bg, card, text, accent, etc.)

### Dark Mode Token Guidance

Dark mode should:
- Reduce eye strain
- Preserve hierarchy and contrast
- Avoid pure black backgrounds

Recommended adjustments (example):

```css
[data-theme="dark"] {
  --bg: #0f172a;          /* deep slate */
  --card: #111827;        /* card surface */
  --soft: #1f2937;        /* subtle fill */
  --border: #273244;
  --borderSoft: #334155;

  --text: #e5e7eb;
  --muted: #9ca3af;
  --muted2: #6b7280;

  /* accent remains the same hue */
  --accent: #0EA5A4;
  --accentHover: #14b8a6;
  --accentSoft: rgba(14,165,164,.18);
  --accentBorder: rgba(14,165,164,.45);

  --liveBg: rgba(16,185,129,.15);
  --liveBorder: rgba(16,185,129,.4);
  --liveText: #a7f3d0;

  --dangerBg: rgba(239,68,68,.15);
  --dangerBorder: rgba(239,68,68,.4);
  --dangerText: #fca5a5;

  --overlay: rgba(0,0,0,0.6);
}
```

### UX Rules

- Theme choice must persist (localStorage or user preference)
- Theme toggle is placed in the **bottom-right corner** (fixed position, `z-index: 100`) — visible but unobtrusive across all screens
- Switching themes should be instant (no reload)
- Respect system preference on first visit (`prefers-color-scheme`)
- HTML root element must start with `data-theme="dark"` (dark is the default; JS upgrades to light if system preference or saved choice indicates light)

---

## Motion & Accessibility

Motion:
- Join / leave: soft scale + fade
- Layout reflow: smooth transitions
- Active speaker: subtle emphasis only

Accessibility:
- Never rely on color alone
- Respect reduced-motion
- Readable labels at small sizes
- Avoid flashing

---

## Summary

- MINI_GUIDE controls behavior
- DESIGN_GUIDE controls UI
- PRE-LIVE ≠ EXIT
- Camera tiles persist across LIVE ⇄ PRE-LIVE; screenshare tiles removed on stream end
- Remote members visible in both LIVE and PRE-LIVE states
- Screen share tiles show name label and LIVE badge, but no mic/camera indicators
- Media indicators always visible
- Screen share = separate tile
- Calm, human-first design

---

End of DESIGN_GUIDE_v2.1

---

---

*VibeLive Integration Guide — MiniChat API v0.63 + Design Guide v2.1 | Last updated: 2026-02-18*
