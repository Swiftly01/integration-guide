# Chat Integration Guide
> Web & Mobile Implementation — REST + WebSocket

---

## Table of Contents

- [Overview](#overview)
- [Phase 1 — Authentication](#phase-1--authentication)
- [Phase 2 — Creating a Conversation](#phase-2--creating-a-conversation)
- [Phase 3 — Opening a Chat Screen (HTTP)](#phase-3--opening-a-chat-screen-http)
- [Phase 4 — Socket Connection](#phase-4--socket-connection)
- [Phase 5 — Entering a Chat Screen](#phase-5--entering-a-chat-screen)
- [Phase 6 — Sending Messages](#phase-6--sending-messages)
- [Phase 7 — Uploading Files & Media](#phase-7--uploading-files--media)
- [Phase 8 — Typing Indicators](#phase-8--typing-indicators)
- [Phase 9 — Read Receipts](#phase-9--read-receipts)
- [Phase 10 — In-App Notifications (No Push Required)](#phase-10--in-app-notifications-no-push-required)
- [Complete Flow Summary](#complete-flow-summary)
- [Key Rules](#key-rules)
- [Socket Event Reference](#socket-event-reference)

---

## Overview

The backend exposes two interfaces:

- **REST API (HTTP)** — for creating conversations, loading message history, and managing participants.
- **WebSocket (Socket.IO)** — for real-time messaging, typing indicators, read receipts, and online status.

Both require a valid JWT on every request.

---

## Phase 1 — Authentication

Every request — both HTTP and WebSocket — requires a valid JWT. After login, store the token securely and attach it to all subsequent calls.

### Login

```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "..."
}
```

**Response:**
```json
{
  "access_token": "<jwt>",
  "user": { "id": "...", "email": "..." }
}
```

### Token Storage

| Platform | Recommended storage |
|----------|-------------------|
| iOS | Keychain Services |
| Android | EncryptedSharedPreferences |
| Web | Memory (state) + httpOnly cookie for refresh token |

### Attaching the Token

**HTTP requests:**
```http
Authorization: Bearer <jwt>
```

**WebSocket handshake:**
```js
const socket = io(SERVER_URL, {
  auth: { token: '<jwt>' },
  transports: ['websocket'],
})
```

---

## Phase 2 — Creating a Conversation

Only **one user** needs to create the conversation. The convention is: **the user who initiates contact makes this call**. Think of it like WhatsApp — when you tap "Message" on someone's profile, your client creates the conversation. The other user does not call this endpoint.

> **Note:** A conversation only needs to be created once. After that, both users reuse the same conversation ID for all subsequent messages.

### Create Conversation

```http
POST /api/v1/conversations
Authorization: Bearer <userA-token>
Content-Type: application/json

{
  "participantIds": ["<userB-id>"]
}
```

**Response:**
```json
{
  "id": "<conversation-uuid>",
  "participants": [...],
  "createdAt": "..."
}
```

After creating the conversation, the app should:

1. Store the returned `id` locally.
2. Navigate the initiating user to the chat screen.
3. User B will discover the conversation via their conversation list or the real-time `new_message` socket event (covered in [Phase 10](#phase-10--in-app-notifications-no-push-required)).

### Group Chats

Pass multiple IDs in `participantIds`:

```json
{
  "participantIds": ["<userB-id>", "<userC-id>", "<userD-id>"]
}
```

### Loading Existing Conversations

On app launch or when the user opens their inbox:

```http
GET /api/v1/conversations?page=1&limit=20
Authorization: Bearer <token>
```

Returns all conversations the authenticated user is a participant of.

---

## Phase 3 — Opening a Chat Screen (HTTP)

Before rendering the chat UI, load existing message history via HTTP. **The socket does not replay history.**

### Load Message History

```http
GET /api/v1/conversations/:conversationId/messages?page=1&limit=20
Authorization: Bearer <token>
```

- Render messages oldest-first (reverse the array if needed).
- Implement infinite scroll by incrementing the `page` param.

### Load Unread Count

```http
GET /api/v1/conversations/:conversationId/messages/unread-count
Authorization: Bearer <token>
```

**Response:**
```json
{ "conversationId": "...", "unreadCount": 4 }
```

---

## Phase 4 — Socket Connection

> ⚠️ **Important:** Connect the socket **once at app launch / login** — not per screen. Keep it alive for the entire session so messages and status events are never missed.

### Connect Once at Login

```js
const socket = io(SERVER_URL, {
  auth: { token: '<jwt>' },
  transports: ['websocket'],
  reconnection: true,
  reconnectionAttempts: 5,
})

socket.on('connect', () => {
  console.log('Connected:', socket.id)
})

socket.on('disconnect', (reason) => {
  console.log('Disconnected:', reason)
})
```

### Global Listeners (set up once, not per screen)

```js
// New message — drives in-app notification banners
socket.on('new_message', (message) => {
  updateConversationBadge(message.conversationId)
  if (currentScreen !== message.conversationId) {
    showInAppBanner(message) // popup / toast
  }
})

// Online / offline status
socket.on('user_status_changed', ({ userId, status }) => {
  updateUserPresence(userId, status)
})
```

---

## Phase 5 — Entering a Chat Screen

Every time the user opens a specific conversation, join its socket room. This subscribes the client to all broadcast events for that conversation (messages, typing, read receipts).

### On Screen Mount

```js
// 1. Join the conversation room
socket.emit('join_conversation', { conversationId })

// 2. Mark all messages as read
socket.emit('message_read', { conversationId })
//    → other participants receive a 'read_receipt' event
```

### On Screen Unmount / Navigation Away

```js
socket.emit('leave_conversation', { conversationId })
// Stops receiving room broadcasts for this conversation
```

> **Why this matters:** Without joining the room, the client will not receive typing indicators or read receipts, even though it still receives `new_message` via the personal room (see [Phase 10](#phase-10--in-app-notifications-no-push-required)).

---

## Phase 6 — Sending Messages

Prefer the WebSocket path. The REST endpoint exists as a fallback for when the socket is unavailable (bad network, backgrounded app).

### Via WebSocket (primary)

```js
socket.emit('send_message', {
  conversationId: '<id>',
  content: 'Hey!',
  type: 'text',   // required — ValidationPipe rejects without it
})
```

The server saves the message and broadcasts `new_message` to everyone in the conversation room — including the sender. **Always render messages from the `new_message` event, not from a local optimistic state.**

### Via REST (fallback)

```http
POST /api/v1/messages
Authorization: Bearer <token>
Content-Type: application/json

{
  "conversationId": "<id>",
  "content": "Hey!",
  "type": "text"
}
```

### Receiving Messages

```js
socket.on('new_message', (message) => {
  // message contains: id, conversationId, senderId, content, type, createdAt
  appendMessageToUI(message)

  if (message.senderId !== currentUserId) {
    socket.emit('message_read', { conversationId: message.conversationId })
  }
})
```

---

## Phase 7 — Uploading Files & Media

Files (images, audio, or any other attachment) never travel over the socket. Upload the raw file over REST first, get back a hosted `url`, then send **that URL** as the message content through the normal messaging flow (Phase 6).

### Upload File(s)

```http
POST /upload
Authorization: Bearer <token>
Content-Type: multipart/form-data

files: <binary>            # field name "files" — up to 10 files, 10MB max each
folder: "chat_images"      # optional — inferred from mimetype if omitted
resourceType: "auto"       # optional — "auto" | "image" | "video" | "raw"
```

**Response:**
```json
{
  "files": [
    {
      "url": "https://res.cloudinary.com/.../chat_images/abc123.jpg",
      "publicId": "chat_images/abc123",
      "format": "jpg",
      "size": 204800
    }
  ]
}
```

> Uploads are hosted on Cloudinary. Nothing about the response needs to be transformed — the `url` field is exactly what gets sent as message `content`.

### Delete a File

```http
DELETE /upload?publicId=chat_images/abc123&resourceType=image
Authorization: Bearer <token>
```

Returns `204 No Content`. Use this to clean up a file if the user cancels before sending, or removes an attachment.

### Sending the Uploaded File as a Message

Once you have the `url`, treat it exactly like Phase 6 — just swap the `content` for the URL and set `type` to match what was uploaded instead of `"text"`.

```js
async function sendFileMessage(file, conversationId) {
  // 1. Upload the file over REST
  const formData = new FormData()
  formData.append('files', file)

  const res = await fetch(`${API_URL}/upload`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
    body: formData,
  })
  const { files } = await res.json()
  const { url } = files[0]

  // 2. Send the URL as message content, same as any other message
  socket.emit('send_message', {
    conversationId,
    content: url,  
    type: 'image',    // or 'audio' / 'file' depending on the mimetype
  })
}
```

Nothing else changes downstream: the recipient still receives it through the standard `new_message` event (Phase 6). `message.content` will just be a URL instead of plain text — the UI decides how to render it (image, audio player, file/download link) based on `message.type`.

### Deciding Which `type` to Send

| Uploaded mimetype | `type` to send |
|---|---|
| `image/*` | `"image"` |
| `audio/*` | `"audio"` |
| anything else | `"file"` |

---

## Phase 8 — Typing Indicators

Fire `typing_start` when the user begins typing. Fire `typing_stop` when they pause for ~2 seconds or submit the message. Debounce to avoid flooding the server.

### Emitting

```js
let typingTimer = null
let isTyping = false

function onInputChange() {
  if (!isTyping) {
    isTyping = true
    socket.emit('typing_start', { conversationId })
  }

  clearTimeout(typingTimer)
  typingTimer = setTimeout(() => {
    isTyping = false
    socket.emit('typing_stop', { conversationId })
  }, 2000)
}

// Also call when the user sends a message
function onSend() {
  clearTimeout(typingTimer)
  isTyping = false
  socket.emit('typing_stop', { conversationId })
}
```

### Receiving

```js
socket.on('user_typing', ({ userId, userName, conversationId }) => {
  showTypingIndicator(`${userName} is typing...`)
})

socket.on('user_stopped_typing', ({ userId }) => {
  hideTypingIndicator()
})
```

---

## Phase 9 — Read Receipts

```js
// Emit when the user opens a conversation or scrolls to the bottom
socket.emit('message_read', { conversationId })

// Other participants receive:
socket.on('read_receipt', ({ conversationId, userId, readAt }) => {
  markMessagesAsRead(conversationId, userId, readAt)
  // Show double-tick / "Seen" indicator in UI
})
```

---

## Phase 10 — In-App Notifications (No Push Required)

When a user is logged in and the socket is connected, they are automatically subscribed to their **personal room** on the server (`user:<userId>`). The server notifies this room whenever someone sends them a message — even if they have not joined the conversation room yet.

> **How it works:** The server calls `notifyUser(userId, 'new_message', payload)` after every `send_message` event. This targets the recipient's personal room directly, so they receive the event without needing to have joined the conversation room first.

### Banner / Popup Pattern

```js
// Set up globally at login — not inside any screen component
socket.on('new_message', (message) => {
  const isCurrentConversation =
    getCurrentConversationId() === message.conversationId

  if (!isCurrentConversation) {
    showBanner({
      title: message.senderName,
      body: message.content,
      avatarUrl: message.senderAvatar,
      onTap: () => navigate(`/chat/${message.conversationId}`),
    })
  }

  // Always update the conversation list badge
  incrementUnreadBadge(message.conversationId)
})
```

### Coverage

| Scenario | Personal room covers? | Needs push notification? |
|----------|-----------------------|--------------------------|
| App open, foreground | ✅ Yes | No |
| App open, backgrounded | ⚠️ Depends on OS socket keep-alive | Recommended |
| App closed / killed | ❌ No | Yes (FCM / APNs) |

---

## Complete Flow Summary

| Step | Who | Action | Method |
|------|-----|--------|--------|
| 1 | Both | Log in, receive JWT tokens | `POST /auth/login` |
| 2 | Both | Connect socket at app launch | Socket.IO connect with auth token |
| 3 | User A | Create conversation with User B | `POST /api/v1/conversations` |
| 4 | User A | Open chat screen, load history | `GET /conversations/:id/messages` |
| 5 | User A | Join conversation room | `socket.emit('join_conversation')` |
| 6 | User B | Receive new message banner (personal room) | `socket.on('new_message')` |
| 7 | User B | Open chat, join room | `socket.emit('join_conversation')` |
| 8 | User B | Mark messages as read | `socket.emit('message_read')` |
| 9 | User A | See read receipt | `socket.on('read_receipt')` |
| 10 | Either | Leave the screen | `socket.emit('leave_conversation')` |

---

## Key Rules

| Rule | Reason |
|------|--------|
| Socket connects once at login, not per screen | Avoids missed messages during navigation |
| `join_conversation` on every screen entry | Required to receive room broadcasts (typing, receipts) |
| `leave_conversation` on every screen exit | Prevents stale listeners accumulating |
| Render messages from `new_message` event, not emit callback | Server is source of truth; event includes saved `id` and `createdAt` |
| `GET /messages` on screen open | Socket never replays history |
| `message_read` on screen entry and scroll to bottom | Keeps unread counts accurate for all participants |
| Only the conversation creator calls `POST /conversations` | Other participants discover it via conversation list or `new_message` |
| `type` field is required in `send_message` | `ValidationPipe` rejects the DTO without it; use `"text"` for standard messages |
| Upload files via `POST /upload` first, then send the returned `url` as message `content` | Files live on Cloudinary; the DB only ever stores the URL string, never the binary |

---

## Socket Event Reference

### Events You Emit (client → server)

| Event | Payload | When to fire |
|-------|---------|--------------|
| `join_conversation` | `{ conversationId }` | On chat screen mount |
| `leave_conversation` | `{ conversationId }` | On chat screen unmount |
| `send_message` | `{ conversationId, content, type }` | On message submit |
| `typing_start` | `{ conversationId }` | On first keystroke |
| `typing_stop` | `{ conversationId }` | 2s after last keystroke or on send |
| `message_read` | `{ conversationId }` | On screen open or scroll to bottom |

### Events You Receive (server → client)

| Event | Payload | What to do |
|-------|---------|------------|
| `new_message` | Full message object | Append to chat UI or show banner |
| `user_typing` | `{ userId, userName, conversationId }` | Show typing indicator |
| `user_stopped_typing` | `{ userId, conversationId }` | Hide typing indicator |
| `read_receipt` | `{ conversationId, userId, readAt }` | Show seen / double tick |
| `user_status_changed` | `{ userId, status }` | Update online/offline indicator |
| `error` | `{ message }` | Handle auth failure or participant error |

---

*Internal developer documentation*
