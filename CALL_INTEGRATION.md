# Call Integration Guide
> Web & Mobile Implementation — WebRTC + Socket.IO (`/call` namespace)

---

## Table of Contents

- [Overview](#overview)
- [Phase 1 — Authentication](#phase-1--authentication)
- [Phase 2 — Who Initiates a Call](#phase-2--who-initiates-a-call)
- [Phase 3 — Initiating a Call (Caller)](#phase-3--initiating-a-call-caller)
- [Phase 4 — Receiving a Call (Callee)](#phase-4--receiving-a-call-callee)
- [Phase 5 — Answering a Call (Callee)](#phase-5--answering-a-call-callee)
- [Phase 6 — WebRTC Negotiation (Automatic)](#phase-6--webrtc-negotiation-automatic)
- [Phase 7 — In-Call Controls](#phase-7--in-call-controls)
- [Phase 8 — Ending a Call](#phase-8--ending-a-call)
- [Phase 9 — Declining or Cancelling](#phase-9--declining-or-cancelling)
- [Phase 10 — Handling Disconnects](#phase-10--handling-disconnects)
- [Phase 11 — Cleanup](#phase-11--cleanup)
- [Complete Flow Summary](#complete-flow-summary)
- [Key Rules](#key-rules)
- [Socket Event Reference](#socket-event-reference)

---

## Overview

The call system uses two interfaces:

- **WebSocket (Socket.IO / `/call` namespace)** — all signalling: initiating, joining, SDP offer/answer exchange, ICE candidates, and call termination.
- **REST API** — used indirectly. The server creates and persists call sessions. The client never calls call REST endpoints directly during a live call.

> ⚠️ **Namespace:** The call gateway runs on the `/call` Socket.IO namespace, not the root `/`. Connect to `SERVER_URL/call` — not `SERVER_URL`.

---

## Phase 1 — Authentication

The `/call` namespace uses the same JWT as the chat namespace. Connect with the token in the socket `auth` object. The server verifies it and attaches the user to the socket before any events are processed.

> 💡 **Two sockets:** Run two separate socket connections — one for chat (`/`) and one for calls (`/call`). They share the same JWT but are independent namespaces. Connect both at login.

### Connect to the call namespace

```js
const callSocket = io(SERVER_URL + '/call', {
  auth: { token: '<jwt>' },
  transports: ['websocket'],
  reconnection: true,
})

callSocket.on('connect', () => {
  console.log('Call socket connected:', callSocket.id)
})

callSocket.on('call_error', (data) => {
  // Fired on auth failure or participant violation
  console.error(data.message)
})
```

---

## Phase 2 — Who Initiates a Call

The user who taps the call button is the **caller**. Only the caller emits `call_initiate`. The callee never calls this event — they receive `call_incoming` and respond with `call_join` (answer) or `call_decline`.

**Typical trigger points:**
- User A opens User B's profile and taps the video or audio call button.
- User A is inside a chat conversation and taps the call icon.
- The app collects the callee's user ID from the profile or conversation participants.

> ✅ **Rule:** Only one side calls `call_initiate` per session. The server creates the `CallSession`, assigns it an ID, and notifies the callee via their personal socket room.

---

## Phase 3 — Initiating a Call (Caller)

### Step 1 — Acquire local media

Request camera and microphone access **before** emitting `call_initiate`. This avoids a race condition where the SDP offer is requested before tracks are available.

```js
const stream = await navigator.mediaDevices.getUserMedia({
  audio: true,
  video: callType === 'video',  // false for audio-only calls
})

// Attach local preview to a <video> element
localVideoEl.srcObject = stream
localVideoEl.muted = true  // always mute local preview
```

### Step 2 — Create RTCPeerConnection

```js
// ICE servers come back in the call_initiate ack (step 3)
// Create with Google STUN as placeholder for now
const pc = new RTCPeerConnection({
  iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
})

// Add local tracks so they are included in the SDP offer
stream.getTracks().forEach(track => pc.addTrack(track, stream))

// Handle remote stream — set up before negotiation starts
pc.ontrack = (ev) => {
  remoteVideoEl.srcObject = ev.streams[0]
}

// Forward ICE candidates to the server
pc.onicecandidate = (ev) => {
  if (ev.candidate) {
    callSocket.emit('call_ice_candidate', {
      callId,
      candidate: ev.candidate.candidate,
      sdpMid: ev.candidate.sdpMid,
      sdpMLineIndex: ev.candidate.sdpMLineIndex,
    })
  }
}
```

### Step 3 — Emit `call_initiate`

```js
callSocket.emit('call_initiate', {
  calleeId: '<userB-uuid>',
  type: 'video',  // or 'audio'
}, (ack) => {
  // ack = { event: 'call_initiated', callId, iceServers }
  callId = ack.callId

  // Recreate PC with real ICE servers from server
  pc.close()
  pc = createPeerConnection(ack.iceServers, stream)

  // Show ringing UI
  showRingingScreen()
})
```

> 📌 **ICE servers:** The server returns its configured STUN/TURN servers in the ack. Always use these rather than hardcoding your own. Recreate the `RTCPeerConnection` with them before SDP negotiation begins.

---

## Phase 4 — Receiving a Call (Callee)

User B receives `call_incoming` on their personal socket room the moment User A emits `call_initiate`. This fires even if User B has not joined any call room — the server delivers it directly to their connected socket.

### Listen for incoming calls globally (set up at login)

```js
callSocket.on('call_incoming', (data) => {
  // data = { callId, type, caller: { id, name }, iceServers }

  // Store for when user answers
  pendingCallId     = data.callId
  pendingIceServers = data.iceServers

  // Show incoming call screen / notification
  showIncomingCallUI({
    callerName: data.caller.name,
    callType:   data.type,
    onAnswer:   () => answerCall(data.callId, data.iceServers),
    onDecline:  () => declineCall(data.callId),
  })
})
```

> 📱 **Mobile:** If the app is backgrounded, use a push notification to wake it and show the incoming call UI. Once the socket reconnects, `call_incoming` will **not** re-fire — so the push payload should include `callId` and caller info directly.

---

## Phase 5 — Answering a Call (Callee)

### Step 1 — Acquire media and create peer connection

```js
async function answerCall(callId, iceServers) {
  const stream = await navigator.mediaDevices.getUserMedia({
    audio: true,
    video: true,
  })
  localVideoEl.srcObject = stream
  localVideoEl.muted = true

  pc = new RTCPeerConnection({ iceServers })
  stream.getTracks().forEach(track => pc.addTrack(track, stream))

  pc.ontrack = (ev) => { remoteVideoEl.srcObject = ev.streams[0] }

  pc.onicecandidate = (ev) => {
    if (ev.candidate) {
      callSocket.emit('call_ice_candidate', {
        callId,
        candidate:     ev.candidate.candidate,
        sdpMid:        ev.candidate.sdpMid,
        sdpMLineIndex: ev.candidate.sdpMLineIndex,
      })
    }
  }
```

### Step 2 — Emit `call_join`

```js
  // Joining the call room triggers the server to notify the caller.
  // The caller then creates and sends the SDP offer.
  callSocket.emit('call_join', { callId }, (ack) => {
    console.log('Joined call room:', ack)
  })
}
```

> 🔁 **Trigger chain:** `call_join` → server emits `call_peer_joined` to caller → caller creates SDP offer → caller emits `call_offer` → callee receives `call_offer` and creates answer. This chain is automatic once `call_join` fires.

---

## Phase 6 — WebRTC Negotiation (Automatic)

After `call_join` fires, the SDP offer/answer exchange and ICE negotiation happen automatically. Wire up these listeners once and they handle themselves.

### Caller — listen for `call_peer_joined` and create offer

```js
callSocket.on('call_peer_joined', async ({ callId }) => {
  const offer = await pc.createOffer()
  await pc.setLocalDescription(offer)

  callSocket.emit('call_offer', {
    callId,
    sdpType: offer.type,
    sdp:     offer.sdp,
  })
})
```

### Callee — receive offer and send answer

```js
callSocket.on('call_offer', async ({ callId, sdpType, sdp }) => {
  await pc.setRemoteDescription({ type: sdpType, sdp })

  // Drain any ICE candidates that arrived before remote desc was set
  for (const c of bufferedCandidates) await pc.addIceCandidate(c)
  bufferedCandidates = []

  const answer = await pc.createAnswer()
  await pc.setLocalDescription(answer)

  callSocket.emit('call_answer', {
    callId,
    sdpType: answer.type,
    sdp:     answer.sdp,
  })
})
```

### Caller — receive answer

```js
callSocket.on('call_answer', async ({ sdpType, sdp }) => {
  await pc.setRemoteDescription({ type: sdpType, sdp })

  for (const c of bufferedCandidates) await pc.addIceCandidate(c)
  bufferedCandidates = []
})
```

### Both sides — receive ICE candidates

```js
callSocket.on('call_ice_candidate', async ({ candidate, sdpMid, sdpMLineIndex }) => {
  const c = { candidate, sdpMid, sdpMLineIndex }

  if (pc.remoteDescription) {
    await pc.addIceCandidate(c)
  } else {
    // Buffer until remote description is set
    bufferedCandidates.push(c)
  }
})
```

### Both sides — detect connection established

```js
pc.oniceconnectionstatechange = () => {
  if (pc.iceConnectionState === 'connected' ||
      pc.iceConnectionState === 'completed') {
    // Call is live — start call timer, show in-call UI
    showActiveCallUI()
    startCallTimer()
  }
  if (pc.iceConnectionState === 'failed') {
    handleCallFailed('ICE connection failed')
  }
}
```

---

## Phase 7 — In-Call Controls

All media controls operate on the local `MediaStream` tracks directly. No socket events are needed for mute/camera toggle.

### Mute / unmute microphone

```js
function toggleMic(stream) {
  stream.getAudioTracks().forEach(t => t.enabled = !t.enabled)
}
```

### Camera on / off

```js
function toggleCamera(stream) {
  stream.getVideoTracks().forEach(t => t.enabled = !t.enabled)
}
```

### Speaker / earpiece (mobile only)

```js
// React Native / Capacitor
InCallManager.setSpeakerphoneOn(true)   // speaker
InCallManager.setSpeakerphoneOn(false)  // earpiece
```

### Switch front / rear camera (mobile)

```js
const newStream = await navigator.mediaDevices.getUserMedia({
  video: { facingMode: currentCamera === 'user' ? 'environment' : 'user' }
})

// Replace the video track in the peer connection without renegotiating
const sender = pc.getSenders().find(s => s.track?.kind === 'video')
await sender.replaceTrack(newStream.getVideoTracks()[0])
```

---

## Phase 8 — Ending a Call

Either participant can end the call. The server broadcasts `call:ended` to everyone in the call room and posts a summary message to the linked conversation (if one exists).

```js
function endCall(callId) {
  callSocket.emit('call:end', { callId })
}

// Both sides receive:
callSocket.on('call:ended', ({ callId, endedBy, durationSeconds }) => {
  stopCallTimer()
  closePeerConnection()
  stopLocalMedia()
  showCallSummary({ durationSeconds })
})
```

---

## Phase 9 — Declining or Cancelling

### Callee declines

```js
callSocket.emit('call_decline', { callId })

// Caller receives:
callSocket.on('call_decline', ({ callId, calleeId }) => {
  showCallDeclinedUI()
  closePeerConnection()
})
```

### Caller cancels before answer

```js
callSocket.emit('call_cancel', { callId })

// Callee receives:
callSocket.on('call_cancelled', ({ callId }) => {
  hideIncomingCallUI()
})
```

---

## Phase 10 — Handling Disconnects

If a participant's socket drops during an active or ringing call, the server automatically marks the call as `FAILED` and notifies the remaining participant.

```js
// Both sides should listen for this
callSocket.on('call_failed', ({ callId, reason }) => {
  stopCallTimer()
  closePeerConnection()
  stopLocalMedia()
  showErrorUI(`Call ended: ${reason}`)
})

// Also handle the socket disconnect itself
callSocket.on('disconnect', (reason) => {
  if (activeCallId) {
    // Socket dropped mid-call — attempt reconnect
    // The server will fire call_failed to the other side
    attemptReconnect()
  }
})
```

> ⚠️ **Reconnect strategy:** If the socket reconnects quickly, `call_failed` has already fired on the server. Do not attempt to resume — show the user a "Call ended" screen and offer to call back.

---

## Phase 11 — Cleanup

Always clean up after every call outcome — ended, declined, cancelled, or failed. Failing to stop tracks leaves the camera/mic indicator active in the browser.

```js
function cleanupCall() {
  // Stop all local media tracks
  localStream?.getTracks().forEach(t => t.stop())
  localStream = null

  // Close peer connection
  pc?.close()
  pc = null

  // Clear video elements
  localVideoEl.srcObject  = null
  remoteVideoEl.srcObject = null

  // Reset state
  activeCallId       = null
  bufferedCandidates = []
}
```

---

## Complete Flow Summary

| Step | Who | Action | Method |
|------|-----|--------|--------|
| 1 | Both | Connect call socket at login | `io(URL + '/call', { auth })` |
| 2 | Caller | Get camera/mic access | `getUserMedia()` |
| 3 | Caller | Create RTCPeerConnection + add tracks | `new RTCPeerConnection()` |
| 4 | Caller | Initiate call | `emit call_initiate` |
| 5 | Callee | Receive incoming call notification | `on call_incoming` |
| 6 | Callee | Get media + create peer connection | `getUserMedia()` + `new RTCPeerConnection()` |
| 7 | Callee | Join call room | `emit call_join` |
| 8 | Caller | Create and send SDP offer | `on call_peer_joined` → `emit call_offer` |
| 9 | Callee | Set remote desc + send answer | `on call_offer` → `emit call_answer` |
| 10 | Caller | Set remote answer | `on call_answer` |
| 11 | Both | Exchange ICE candidates | `emit/on call_ice_candidate` |
| 12 | Both | ICE connects — call is live | `oniceconnectionstatechange` |
| 13 | Either | End call | `emit call:end` → `on call:ended` |
| 14 | Both | Cleanup media + peer connection | `cleanupCall()` |

---

## Key Rules

| Rule | Reason |
|------|--------|
| Connect to `/call` namespace, not `/` | The gateway runs on its own namespace |
| Only the caller emits `call_initiate` | One side creates the session; callee responds |
| Acquire media before emitting `call_initiate` | Tracks must exist before the SDP offer is created |
| Add tracks to PC before SDP negotiation | Tracks added after `createOffer()` are not included |
| Buffer ICE candidates until remote desc is set | `addIceCandidate()` fails without a remote description |
| Recreate PC with server-provided ICE servers | Server may return TURN credentials; always use them |
| Always clean up after every call outcome | Avoids camera/mic indicator staying on in browser |
| Do not try to resume after `call_failed` | Server has already torn down the session; start a new call |

---

## Socket Event Reference

### Events you emit (client → server)

| Event | Payload | Who sends it |
|-------|---------|--------------|
| `call_initiate` | `{ calleeId, type }` | Caller only |
| `call_join` | `{ callId }` | Callee on answer |
| `call_offer` | `{ callId, sdpType, sdp }` | Caller after `call_peer_joined` |
| `call_answer` | `{ callId, sdpType, sdp }` | Callee after receiving offer |
| `call_ice_candidate` | `{ callId, candidate, sdpMid, sdpMLineIndex }` | Both sides |
| `call_decline` | `{ callId }` | Callee to decline |
| `call_cancel` | `{ callId }` | Caller to cancel before answer |
| `call:end` | `{ callId }` | Either side to end active call |

### Events you receive (server → client)

| Event | Payload | What to do |
|-------|---------|------------|
| `call_incoming` | `{ callId, type, caller, iceServers }` | Show incoming call UI (callee) |
| `call_peer_joined` | `{ callId, userId }` | Create and send SDP offer (caller) |
| `call_offer` | `{ callId, sdpType, sdp }` | Set remote desc + send answer (callee) |
| `call_answer` | `{ callId, sdpType, sdp }` | Set remote description (caller) |
| `call_ice_candidate` | `{ callId, candidate, sdpMid, sdpMLineIndex }` | Add to `RTCPeerConnection` (both) |
| `call:ended` | `{ callId, endedBy, durationSeconds }` | Show summary, clean up (both) |
| `call_decline` | `{ callId, calleeId }` | Show declined UI, clean up (caller) |
| `call_cancelled` | `{ callId }` | Hide incoming UI (callee) |
| `call_failed` | `{ callId, reason }` | Show error, clean up (both) |
| `call_error` | `{ message }` | Auth failure or participant violation |

---

*Internal developer documentation*
