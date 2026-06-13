# Real-Time Integration Guides

This folder contains the integration documentation for the **chat** and **call** real-time systems. Both run over Socket.IO alongside the existing REST API and are intended for the web and mobile client teams.

Each guide is provided in two formats:

- **`.md`** — for quick reference inside the codebase (renders natively on GitHub and in VS Code)
- **`.docx`** — for sharing with non-technical stakeholders or pasting into design docs

---

## 📄 Documents

| Guide            | Markdown                                       | Word                                                           |
| ---------------- | ---------------------------------------------- | -------------------------------------------------------------- |
| Chat integration | [`CHAT_INTEGRATION.md`](./CHAT_INTEGRATION.md) | [`chat-integration-guide.docx`](./chat-integration-guide.docx) |
| Call integration | [`CALL_INTEGRATION.md`](./CALL_INTEGRATION.md) | [`call-integration-guide.docx`](./call-integration-guide.docx) |

---

## 💬 Chat Integration — Summary

Covers the messaging system: `ChatGateway` (root `/` namespace) + REST `ChatController` / `MessagesController`.

**Key points:**

- One socket connection at login (root namespace), kept alive for the whole session.
- **Conversation creation is one-sided** — the user who initiates contact calls `POST /api/v1/conversations` once. The other participant discovers it via their conversation list or the `new_message` event.
- Message history is loaded via REST (`GET /conversations/:id/messages`); the socket never replays history.
- Entering a chat screen requires `join_conversation` to receive typing indicators and read receipts.
- Sending is socket-first (`send_message`), with REST `POST /messages` as an offline fallback.
- **In-app notifications work without push** — every connected user is automatically in a personal room (`user:<id>`), and the server pushes `new_message` there directly so banners/popups work even outside the conversation room.

➡️ Full details, code samples, and event tables: [`CHAT_INTEGRATION.md`](./CHAT_INTEGRATION.md)

---

## 📞 Call Integration — Summary

Covers the WebRTC signalling system: `CallGateway` (`/call` namespace).

**Key points:**

- **Separate socket connection** to the `/call` namespace — connect both chat and call sockets at login, same JWT.
- **Calls are entirely socket-driven** — there is no HTTP step to start a call. The caller emits `call_initiate`; the server creates the `CallSession` and returns the `callId` directly in the ack.
- The callee receives `call_incoming` via their personal room the moment `call_initiate` fires — no need to have joined any room first.
- WebRTC negotiation (offer/answer/ICE) is automatic once the callee emits `call_join` — wire up the listeners once and the chain runs itself.
- ICE candidates that arrive before the remote description is set must be buffered and drained afterward.
- Always use the **server-provided ICE/TURN servers** returned in the ack/`call_incoming` payload.
- Every call outcome (`call:ended`, `call_decline`, `call_cancelled`, `call_failed`) must trigger full cleanup — stop tracks, close the peer connection, clear video elements.

➡️ Full details, code samples, and event tables: [`CALL_INTEGRATION.md`](./CALL_INTEGRATION.md)

---

## 🧭 Quick Decision Guide

| Question                     | Chat                                              | Call                                                   |
| ---------------------------- | ------------------------------------------------- | ------------------------------------------------------ |
| Namespace                    | `/`                                               | `/call`                                                |
| Who starts it                | Either user, conventionally the initiator         | Caller only (`call_initiate`)                          |
| How is the ID obtained       | `POST /api/v1/conversations` response             | Ack of `call_initiate` (or `call_incoming` for callee) |
| HTTP involved?               | Yes — conversation creation, history, read counts | No — fully socket-driven                               |
| First event to fire on entry | `join_conversation`                               | `call_join` (callee only)                              |

---

## 🛠 For New Team Members

1. Read the **summary sections above** first to understand the high-level flow.
2. Open the relevant `.md` guide for full code samples — copy-paste ready for both web (React/Vue) and mobile (React Native).
3. Both guides assume the JWT-based auth flow described in [`CHAT_INTEGRATION.md` → Phase 1](./CHAT_INTEGRATION.md#phase-1--authentication).

---

_Internal developer documentation_
