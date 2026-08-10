# Patch Note: `call_initiated_broadcast` — Call Initiation Fallback

> **For both web and mobile.**
>
> This is a small addition to the existing call implementation. Do not change the normal `call_initiate` flow.

## What changed?

The server now sends a new event:

`call_initiated_broadcast`

It is sent to the caller when the call was successfully created.

This fixes a case where the `call_initiate` acknowledgement is lost because the caller's socket reconnects.

### New event

| Event | Data |
|---|---|
| `call_initiated_broadcast` | `{ callId, iceServers }` |

The event is sent **only to the caller**.

---

## What you need to do

### 1. Listen for the event when the user logs in

Register this alongside your existing `call_incoming` listener.

```js
callSocket.on('call_initiated_broadcast', (data) => {
  // Only handle this for the current outgoing call.
  if (!isCaller || callPhase !== 'calling' || callId) {
    return
  }

  clearInitiateAckTimeout()
  finalizeOutgoingCallSetup(data.callId, data.iceServers)
})
```

**Important:** Do not put this listener inside `startCall()`.

It must be registered when the socket is set up/login happens, because the event may arrive after a socket reconnect.

---

### 2. Put the "call created" setup in one function

Create one function that handles everything that currently happens after a successful `call_initiate` ack.

```js
let outgoingSetupDone = false

function finalizeOutgoingCallSetup(newCallId, iceServers) {
  if (outgoingSetupDone) return

  outgoingSetupDone = true

  callId = newCallId
  pc.setConfiguration({ iceServers })

  for (const c of pendingOutgoingIce) {
    callSocket.emit('call_ice_candidate', {
      callId,
      candidate: c.candidate,
      sdpMid: c.sdpMid,
      sdpMLineIndex: c.sdpMLineIndex,
    })
  }

  pendingOutgoingIce = []

  if (peerJoinedBeforeAck) {
    peerJoinedBeforeAck = false
    createAndSendOffer()
  }

  showRingingScreen()
}
```

Use your existing variable/function names if they are different.

---

### 3. Change the existing ack callback

The existing `call_initiate` ack should use the same function:

```js
callSocket.emit('call_initiate', { calleeId, type }, (ack) => {
  if (ack.event === 'call_unreachable') {
    showOfflineUI('User is not online')
    closePeerConnection()
    return
  }

  clearInitiateAckTimeout()
  finalizeOutgoingCallSetup(ack.callId, ack.iceServers)
})
```

Do **not** duplicate the setup logic in the ack and broadcast handlers.

Both should call:

`finalizeOutgoingCallSetup(...)`

---

### 4. Reset the guard for every new outgoing call

When starting a new outgoing call, reset:

```js
outgoingSetupDone = false
```

Put this in the same place where you already reset `callId`, `pendingOutgoingIce`, etc.

---

## The important rule

There are now **two ways** the caller can learn that the call was created:

1. Normal `call_initiate` ack
2. `call_initiated_broadcast` fallback

Both must use the same `finalizeOutgoingCallSetup()` function.

The function must only run once.

That's what this does:

```js
if (outgoingSetupDone) return
```

### That's it

You do **not** need to change the normal call flow.

You only need to:

- Add the `call_initiated_broadcast` listener.
- Move the existing successful-ack setup into `finalizeOutgoingCallSetup()`.
- Make both the ack and broadcast call that function.
- Reset `outgoingSetupDone` when starting a new outgoing call.

**Apply the same change to web and mobile.**