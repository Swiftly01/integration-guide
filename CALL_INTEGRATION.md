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
- [Phase 10 — Handling Disconnects & Missed Calls](#phase-10--handling-disconnects--missed-calls)
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

> ✅ **Rules:**
> - Only one side calls `call_initiate` per session. The server creates the `CallSession`, assigns it an ID, and notifies the callee via their personal socket room.
> - The server rejects self-calls (`calleeId === callerId`) with a `WsException`. Never send `call_initiate` with your own user ID as the callee.

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

Create the peer connection **before** emitting `call_initiate` using Google STUN as a placeholder. The server returns real ICE servers (including TURN) in the ack, which you apply via `setConfiguration()` without recreating the connection. Creating the PC early closes the race where `call_peer_joined` arrives before the ack.

```js
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
// Buffer them if callId is not yet known (ack hasn't arrived)
let pendingOutgoingIce = []
pc.onicecandidate = (ev) => {
  if (!ev.candidate) return
  if (callId) {
    callSocket.emit('call_ice_candidate', {
      callId,
      candidate:     ev.candidate.candidate,
      sdpMid:        ev.candidate.sdpMid,
      sdpMLineIndex: ev.candidate.sdpMLineIndex,
    })
  } else {
    pendingOutgoingIce.push(ev.candidate)
  }
}
```

### Step 3 — Emit `call_initiate`

```js
callSocket.emit('call_initiate', {
  calleeId: '<userB-uuid>',
  type: 'video',  // or 'voice'
}, (ack) => {
  // NEW: server returns call_unreachable if callee has no active socket
  if (ack.event === 'call_unreachable') {
    showOfflineUI('User is not online')
    closePeerConnection()
    return
  }

  // ack = { event: 'call_initiated', callId, iceServers }
  callId = ack.callId

  // Upgrade ICE servers on the existing PC (adds TURN credentials)
  pc.setConfiguration({ iceServers: ack.iceServers })

  // Flush any ICE candidates that were gathered before we had a callId
  for (const c of pendingOutgoingIce) {
    callSocket.emit('call_ice_candidate', {
      callId,
      candidate:     c.candidate,
      sdpMid:        c.sdpMid,
      sdpMLineIndex: c.sdpMLineIndex,
    })
  }
  pendingOutgoingIce = []

  // If call_peer_joined already arrived before this ack, send the offer now
  if (peerJoinedBeforeAck) {
    peerJoinedBeforeAck = false
    createAndSendOffer()
  }

  showRingingScreen()
})
```

> 📌 **ICE servers:** The server returns its configured STUN/TURN servers in the ack. Apply them with `pc.setConfiguration()` — do not recreate the `RTCPeerConnection`, as the connection has already started gathering candidates.

> ⚠️ **`call_peer_joined` race:** `call_peer_joined` can arrive on the caller's socket before the `call_initiate` ack (because the callee emits `call_join` which triggers the server to broadcast `call_peer_joined` immediately). Guard against this with a `peerJoinedBeforeAck` flag as shown above.

---

## Phase 4 — Receiving a Call (Callee)

User B receives `call_incoming` on their personal socket room the moment User A emits `call_initiate`. This fires even if User B has not joined any call room — the server delivers it directly to their connected socket.

> 📌 **Callee must be online:** The server now checks whether the callee has any active sockets before creating the call. If User B is offline, the caller receives `call_unreachable` in the ack and no `call_incoming` is ever sent.

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

// NEW: ringing timed out — server marks call MISSED after 30s with no answer
callSocket.on('call_missed', (data) => {
  // data = { callId, fullName }
  hideIncomingCallUI()
  showMissedCallNotification(data.fullName)
  cleanupCall()
})
```

> 📱 **Mobile:** If the app is backgrounded, use a push notification to wake it and show the incoming call UI. Once the socket reconnects, `call_incoming` will **not** re-fire — so the push payload should include `callId` and caller info directly. If the socket reconnects and the call is already `MISSED`, no further events arrive — use the push payload to show a missed call screen.

---

## Phase 5 — Answering a Call (Callee)

### Step 1 — Acquire media and create peer connection

Create the peer connection **before** emitting `call_join` using the ICE servers from `call_incoming`. This closes the symmetric race where `call_offer` arrives before `pcs[B]` exists.

```js
async function answerCall(callId, iceServers) {
  const stream = await navigator.mediaDevices.getUserMedia({
    audio: true,
    video: true,
  })
  localVideoEl.srcObject = stream
  localVideoEl.muted = true

  // Create PC before emitting call_join
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
let peerJoinedBeforeAck = false

callSocket.on('call_peer_joined', async ({ callId }) => {
  // Guard: if the call_initiate ack hasn't arrived yet, defer the offer
  if (!pc || !callId) {
    peerJoinedBeforeAck = true
    return
  }
  await createAndSendOffer()
})

async function createAndSendOffer() {
  const offer = await pc.createOffer()
  await pc.setLocalDescription(offer)

  callSocket.emit('call_offer', {
    callId,
    sdpType: offer.type,
    sdp:     offer.sdp,
  })
}
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

Either participant can end the call. The server broadcasts `call_ended` to everyone in the call room and posts a summary message to the linked conversation (if one exists).

> ⚠️ **Event name:** The emit event is `call_end` (underscore) and the received event is `call_ended`. Do not use `call:end` or `call:ended` — those use colons and will not be handled by the gateway.

```js
function endCall(callId) {
  callSocket.emit('call_end', { callId })
}

// Both sides receive:
callSocket.on('call_ended', ({ callId, endedBy, durationSeconds }) => {
  stopCallTimer()
  closePeerConnection()
  stopLocalMedia()
  showCallSummary({ durationSeconds })
})
```

---

## Phase 9 — Declining or Cancelling

### Callee declines

Only the callee can decline. The server enforces this — a caller emitting `call_decline` will receive a `WsException`.

```js
callSocket.emit('call_decline', { callId })

// Caller receives:
callSocket.on('call_decline', ({ callId, calleeId }) => {
  showCallDeclinedUI()
  closePeerConnection()
})
```

### Caller cancels before answer

Only the caller can cancel. The server enforces this — a callee emitting `call_cancel` will receive a `WsException`.

```js
callSocket.emit('call_cancel', { callId })

// Callee receives:
callSocket.on('call_cancelled', ({ callId, callerId }) => {
  hideIncomingCallUI()
})
```

---

## Phase 10 — Handling Disconnects & Missed Calls

### Peer disconnects during a call

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

### Callee does not answer (missed call)

If the callee does not answer within 30 seconds, the server marks the call `MISSED` and emits `call_missed` to both participants.

```js
// Both sides receive this after 30s with no answer
callSocket.on('call_missed', ({ callId, fullName }) => {
  stopCallTimer()
  closePeerConnection()
  stopLocalMedia()
  // Caller: show "No answer" UI
  // Callee: show missed call notification
  showMissedCallUI({ fullName })
})
```

### Callee is offline

If the callee has no active socket when `call_initiate` is sent, the server does not create a ringing call. Instead it returns `call_unreachable` in the ack and immediately marks the call `MISSED`.

```js
callSocket.emit('call_initiate', dto, (ack) => {
  if (ack.event === 'call_unreachable') {
    showOfflineUI('User is currently offline')
    closePeerConnection()
    return
  }
  // ... normal ringing flow
})
```

---

## Phase 11 — Cleanup

Always clean up after every call outcome — ended, declined, cancelled, missed, unreachable, or failed. Failing to stop tracks leaves the camera/mic indicator active in the browser.

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
  pendingCallId      = null
  bufferedCandidates = []
  pendingOutgoingIce = []
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
| 4a | Caller | Handle offline callee | `ack.event === 'call_unreachable'` → show offline UI |
| 5 | Callee | Receive incoming call notification | `on call_incoming` |
| 5a | Callee | Handle missed (no answer in 30s) | `on call_missed` → show missed UI |
| 6 | Callee | Get media + create peer connection | `getUserMedia()` + `new RTCPeerConnection()` |
| 7 | Callee | Join call room | `emit call_join` |
| 8 | Caller | Create and send SDP offer | `on call_peer_joined` → `emit call_offer` |
| 9 | Callee | Set remote desc + send answer | `on call_offer` → `emit call_answer` |
| 10 | Caller | Set remote answer | `on call_answer` |
| 11 | Both | Exchange ICE candidates | `emit/on call_ice_candidate` |
| 12 | Both | ICE connects — call is live | `oniceconnectionstatechange` |
| 13 | Either | End call | `emit call_end` → `on call_ended` |
| 14 | Both | Cleanup media + peer connection | `cleanupCall()` |

---

## Key Rules

| Rule | Reason |
|------|--------|
| Connect to `/call` namespace, not `/` | The gateway runs on its own namespace |
| Only the caller emits `call_initiate` | One side creates the session; callee responds |
| Never send your own user ID as `calleeId` | Server rejects self-calls with a `WsException` |
| Handle `call_unreachable` in the ack | Server skips ringing if callee is offline; no `call_incoming` is sent |
| Handle `call_missed` on both sides | Server fires this after 30s of no answer |
| Only the callee emits `call_decline` | Server enforces this; callers use `call_cancel` |
| Only the caller emits `call_cancel` | Server enforces this; callees use `call_decline` |
| Acquire media before emitting `call_initiate` | Tracks must exist before the SDP offer is created |
| Create RTCPeerConnection before emitting `call_initiate` | Closes the race where `call_peer_joined` arrives before the ack |
| Add tracks to PC before SDP negotiation | Tracks added after `createOffer()` are not included |
| Buffer ICE candidates until remote desc is set | `addIceCandidate()` fails without a remote description |
| Buffer outgoing ICE candidates until `callId` is known | Cannot emit `call_ice_candidate` without a `callId` |
| Apply server ICE servers via `setConfiguration()`, not by recreating the PC | The PC has already started gathering candidates; recreating it loses them |
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
| `call_decline` | `{ callId }` | Callee only — to decline an incoming call |
| `call_cancel` | `{ callId }` | Caller only — to cancel before answer |
| `call_end` | `{ callId }` | Either side — to end an active call |

### Events you receive (server → client)

| Event | Payload | What to do |
|-------|---------|------------|
| `call_incoming` | `{ callId, type, caller, iceServers }` | Show incoming call UI (callee) |
| `call_peer_joined` | `{ callId, userId }` | Create and send SDP offer (caller) |
| `call_offer` | `{ callId, sdpType, sdp }` | Set remote desc + send answer (callee) |
| `call_answer` | `{ callId, sdpType, sdp }` | Set remote description (caller) |
| `call_ice_candidate` | `{ callId, candidate, sdpMid, sdpMLineIndex }` | Add to `RTCPeerConnection` (both) |
| `call_ended` | `{ callId, endedBy, durationSeconds }` | Show summary, clean up (both) |
| `call_decline` | `{ callId, calleeId }` | Show declined UI, clean up (caller) |
| `call_cancelled` | `{ callId, callerId }` | Hide incoming UI, clean up (callee) |
| `call_failed` | `{ callId, reason }` | Show error, clean up (both) |
| `call_missed` | `{ callId, fullName }` | Show missed call UI, clean up (both) |
| `call_error` | `{ message }` | Auth failure or participant/role violation |

> 📌 **`call_unreachable`** is returned as an ack payload from `call_initiate` (not as a separate socket event). Check `ack.event === 'call_unreachable'` in the emit callback.

---

*Internal developer documentation*