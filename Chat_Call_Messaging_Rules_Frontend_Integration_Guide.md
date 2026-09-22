# Chat & Call Messaging Rules Integration Guide

This guide explains how the frontend should handle chat and voice/video-call restrictions returned by the backend.

## 1. What changed

Messaging and calls are allowed only when the required relationship exists—for example, a valid appointment or accepted pharmacy order—and only within the configured messaging window.

Previously, blocked requests returned a generic:

```json
{ "message": "Forbidden" }
```

They now return a structured error with:

- `statusCode`
- `code`
- `message`
- Optional context such as `expiresAt`, `windowDays`, `providerRole`, or `cause`

**Always branch on `code`, never on `message`.**

---

## 2. Error shape

All chat/call restrictions use this base shape:

```json
{
  "statusCode": 403,
  "code": "CHAT_WINDOW_EXPIRED",
  "message": "Messaging with this doctor closed on 14 Sep 2026, 2 days after your last appointment.",
  "expiresAt": "2026-09-14T10:32:00.000Z",
  "windowDays": 2,
  "providerRole": "doctor"
}
```

Required:

```text
statusCode
code
message
```

Optional:

```text
expiresAt
windowDays
providerRole
cause
```

---

## 3. Error codes

| Code                               | Meaning                                            | Frontend behavior                                                                   |
| ---------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `CHAT_NO_APPOINTMENT`              | No appointment exists between the users.           | Show booking CTA.                                                                   |
| `CHAT_WINDOW_EXPIRED`              | The messaging window has expired.                  | Disable composer and show `message`; offer follow-up booking.                       |
| `CHAT_NO_ACCEPTED_ORDER`           | Pharmacy order has not been accepted.              | Tell user messaging becomes available after acceptance; link to order.              |
| `CHAT_PAIR_NOT_ALLOWED`            | These user roles cannot communicate.               | Show generic "You can't message this user."                                         |
| `CHAT_CONVERSATION_TYPE_FORBIDDEN` | The requested conversation type cannot be created. | Log the error and show a generic message.                                           |
| `CHAT_NOT_PARTICIPANT`             | User is not part of the conversation.              | Return to conversation list.                                                        |
| `CALL_NOT_ALLOWED`                 | Call was blocked because of a chat restriction.    | Use `cause` to determine the actual reason and show the corresponding call message. |

---

## 4. REST errors

REST returns the error directly:

```js
const res = await fetch(`${API_BASE}/messages`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${token}`,
  },
  body: JSON.stringify({
    conversationId,
    content,
    type: "text",
  }),
});

if (!res.ok) {
  const error = await res.json();
  showChatError(error);
  return;
}
```

---

## 5. Chat WebSocket errors

Socket.IO errors are returned through the `emit` acknowledgement:

```js
socket.emit(
  "send_message",
  { conversationId, content, type: "text" },
  (ack) => {
    if (ack?.success === false) {
      showChatError(ack.error);
      return;
    }

    // Success: ack contains the created message.
  },
);
```

The error is nested:

```js
{
  success: false,
  error: {
    code: "CHAT_WINDOW_EXPIRED",
    message: "...",
    ...
  }
}
```

Also register the `exception` event as a fallback:

```js
socket.on("exception", (error) => {
  showChatError(error);
});
```

The `exception` payload is the flat error object.

---

## 6. Call WebSocket errors

Calls use the same acknowledgement pattern:

```js
socket.emit("call_initiate", { calleeId, type: "video" }, (ack) => {
  if (ack?.success === false) {
    showCallError(ack.error);
    return;
  }

  // Success: continue with WebRTC setup.
});
```

Call restrictions look like:

```json
{
  "code": "CALL_NOT_ALLOWED",
  "cause": "CHAT_WINDOW_EXPIRED",
  "message": "...",
  "expiresAt": "..."
}
```

---

## 7. Check eligibility before typing

`GET /conversations/:id` includes a `messaging` object:

```json
{
  "messaging": {
    "canSend": true,
    "expiresAt": "2026-09-23T10:32:00.000Z",
    "code": null,
    "message": null
  }
}
```

Handle it as follows:

### `canSend: true`

Enable the composer.

If `expiresAt` exists, the messaging window has a deadline and a countdown may be shown.

### `canSend: false`

Disable the composer and use the returned `code` and `message` to display the appropriate state.

---
