# Support

## 1. User Support

Users can:
- Create a support ticket
- View their tickets
- Open a ticket and see its messages
- Reply to a ticket
- Attach an image/file to a reply
- See ticket status and priority

### Ticket statuses

| Status | UI meaning |
|---|---|
| `open` | Waiting for support |
| `in_progress` | Support has responded |
| `resolved` | Support marked the issue as solved |
| `closed` | Ticket is closed |

### Ticket priorities

| Priority | SLA |
|---|---|
| `urgent` | 30 min |
| `high` | 1 hour |
| `medium` | 4 hours |
| `low` | 24 hours |

The frontend does **not** need to calculate the SLA. The API provides `slaDueAt` and `isSlaBreached`.

---

# 2. User APIs

## Create ticket

`POST /support/tickets`

Request:

```json
{
  "subject": "Provider Did Not Join Consultation",
  "description": "My consultation started but the provider never joined.",
  "category": "consultation",
  "priority": "high"
}
```

`category` and `priority` are optional.

Defaults:
- category → `other`
- priority → `medium`

---

## Get my tickets

`GET /support/tickets?page=1&limit=20&status=open`

Returns a paginated list of the user's tickets.

---

## Get ticket

`GET /support/tickets/:id`

Returns:
- Ticket information
- Recent messages
- Current status
- Priority
- Assigned admin
- SLA information

---

## Get messages

`GET /support/tickets/:id/messages?page=1&limit=20`

Returns paginated messages.

---

## Reply to ticket

`POST /support/tickets/:id/messages`

Text:

```json
{
  "content": "I still need help with this."
}
```

Attachment:

```json
{
  "mediaUrl": "https://...",
  "fileName": "screenshot.png"
}
```

At least one of `content` or `mediaUrl` is required.

---

# 3. Important User Behaviors

### When support replies

The ticket automatically changes:

`open → in_progress`

The frontend does not need to call a separate status endpoint.

### When a ticket is resolved/closed

If the user replies again, the ticket automatically reopens:

`resolved/closed → open`

A new SLA period starts.

### Real-time messages

Users receive new support replies through the existing socket/chat infrastructure.

Listen for:

`new_message`

No separate support socket connection is required for the user.

---

# 4. Admin Support Dashboard

Admin users need:

### Dashboard

`GET /admin/support/summary`

Provides counts such as:

```json
{
  "total": 4,
  "open": 2,
  "inProgress": 1,
  "resolved": 1,
  "closed": 0,
  "slaBreached": 1
}
```

### Ticket queue

`GET /admin/support/tickets`

Supports filters:

- status
- priority
- category
- assigned admin
- search
- date range

Example:

`GET /admin/support/tickets?status=open&priority=high&search=consultation`

The API handles the default sorting.

### Ticket detail

`GET /admin/support/tickets/:id`

Admin can view any ticket.

### Reply

`POST /support/tickets/:id/messages`

Same endpoint used by users.

### Assign ticket

`PATCH /admin/support/tickets/:id/assign`

```json
{
  "adminId": "..."
}
```

To unassign:

```json
{}
```

### Change status

`PATCH /admin/support/tickets/:id/status`

```json
{
  "status": "resolved",
  "resolutionNote": "Issue resolved."
}
```

`resolutionNote` is required when resolving.

### Change priority

`PATCH /admin/support/tickets/:id/priority`

```json
{
  "priority": "urgent"
}
```

---

# 5. Admin Real-Time Updates

Admin dashboard joins the support queue once:

```js
socket.emit('join_support_queue');
```

Listen for:

### `ticket_created`

A new ticket was created.

### `ticket_updated`

A ticket changed because of:
- New reply
- Status change
- Priority change
- Assignment change

The admin dashboard should update without requiring a manual refresh.

---

# 6. Ticket Object

Frontend/mobile mainly needs these fields:

```json
{
  "id": "...",
  "ticketNumber": "T-3061",
  "subject": "...",
  "requesterName": "...",
  "category": "consultation",
  "priority": "high",
  "status": "open",
  "assignedAdminId": "...",
  "assignedAdminName": "...",
  "lastMessagePreview": "...",
  "lastMessageAt": "...",
  "isSlaBreached": false,
  "slaDueAt": "...",
  "createdAt": "...",
  "resolutionNote": null
}
```

Message:

```json
{
  "id": "...",
  "conversationId": "...",
  "senderId": "...",
  "content": "...",
  "type": "text",
  "status": "sent",
  "createdAt": "..."
}
```

---

# 7. Screens Required

## User / Mobile

1. **Support**
   - List tickets
   - Status/priority
   - Last message
   - Create ticket button

2. **Create Ticket**
   - Subject
   - Description
   - Category
   - Priority

3. **Ticket Details**
   - Ticket information
   - Message thread
   - Reply box
   - Attachment
   - Status

## Admin / Web

1. **Support Dashboard**
   - Ticket counts
   - SLA breached count

2. **Ticket Queue**
   - Ticket list
   - Search
   - Filters
   - Priority/status indicators

3. **Ticket Details**
   - Message thread
   - Reply
   - Assign/reassign
   - Change status
   - Change priority
   - Resolution note

---
