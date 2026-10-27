# Wetin Sup API and Socket Contract

Successful REST responses use `{ "data": ... }`; errors use
`{ "error": { "code": "...", "message": "..." } }`. Protected requests require
`Authorization: Bearer <JWT>`.

## Base URLs

- Health check: `GET /api/health`
- Application API: `/api/v1`
- Socket.IO: same origin/host as the server, authenticated with
  `auth: { token: "<JWT>" }`

## REST endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/auth/signup` | No | Create an account with `phone`, `password`, `fullName`, `nickname` |
| POST | `/api/v1/auth/login` | No | Login with `phone` and `password` |
| GET | `/api/v1/users/me` | Yes | Get the signed-in user |
| PATCH | `/api/v1/users/me` | Yes | Update `fullName`, `nickname`, `about`, and Cloudinary `avatarUrl` |
| GET | `/api/v1/users?search=` | Yes | Search contacts by nickname or full name; returns up to 100 and includes online status |
| GET | `/api/v1/messages` | Yes | Get the latest 50 global messages |
| GET | `/api/v1/rooms` | Yes | List groups the user belongs to |
| GET | `/api/v1/rooms?discover=true` | Yes | List public groups the user has not joined |
| POST | `/api/v1/rooms` | Yes | Create `{ name, memberIds?, isPublic? }`; creator is automatically a member |
| GET | `/api/v1/rooms/:roomId/members` | Yes | List group members and their online status |
| POST | `/api/v1/rooms/:roomId/members/me` | Yes | Join a discoverable public group |
| DELETE | `/api/v1/rooms/:roomId/members/me` | Yes | Leave a group; an owner leaving transfers ownership to a remaining member |
| GET | `/api/v1/rooms/:roomId/messages` | Yes, member | Get the latest 50 group messages |
| GET | `/api/v1/conversations` | Yes | List direct conversations |
| POST | `/api/v1/conversations` | Yes | Find or create a direct conversation with `{ recipientId }` |
| GET | `/api/v1/conversations/:conversationId/messages` | Yes, member | Get the latest 50 direct messages |
| POST | `/api/v1/uploads/signature` | Yes | Get a signed Cloudinary upload request using the configured preset (JPG/PNG/WebP, 5 MB max) |
| GET | `/api/v1/unread` | Yes | Get persistent unread counts by `room:<id>` and `conversation:<id>` |

Sign-up/login phone numbers must use international E.164 format (for example
`+14155550123`). Passwords must contain at least 8 characters and no more than
72 UTF-8 bytes. JWTs expire after 7 days. Password hashes are never returned.
Auth requests are rate limited separately to 10 attempts per IP per 15 minutes.

## Data shapes

- User/contact: `{ id, phone, fullName, nickname, avatarUrl, about, status }`
- Group: `{ id, name, type: "group", isPublic, memberIds, createdBy, createdAt }`
- Conversation: `{ id, memberIds, participant, createdAt, updatedAt }`
- Message: `{ id, roomId?, conversationId?, text, media?, createdAt, sender: { id, nickname, fullName, avatarUrl }, readBy }`
- Auth response: `{ data: { user, token } }`
- Message history response: `{ data: { messages } }`

Messages may contain up to 2,000 characters of text and one JPG/PNG/WebP image
(5 MB max) uploaded to the configured Wetin Sup Cloudinary account. Message
history contains at most the latest 50 messages in chronological order. Group
and direct message membership is checked by both REST and Socket.IO handlers.
Only signed uploads are allowed; configure the preset as signed, restrict it to
the `wetin-sup` folder and JPG/PNG/WebP, and set its max file size to 5 MB.
Never put the Cloudinary API secret in client code. Signature requests are
limited to 20 per IP per 15 minutes.

## Socket events

| Direction | Event | Payload | Purpose |
|---|---|---|---|
| Client -> Server | `room:join` | `{ roomId: "global" \| "<groupId>" }` | Join global chat or a group the user belongs to |
| Server -> Client | `room:joined` | `{ roomId, recentMessages, members }` | Send message history and group members |
| Client -> Server | `room:leave` | `{ roomId }` | Leave a global/group socket room |
| Client -> Server | `conversation:join` | `{ conversationId }` | Join a direct conversation the user belongs to |
| Server -> Client | `conversation:joined` | `{ conversationId, recentMessages }` | Send direct message history |
| Client -> Server | `conversation:leave` | `{ conversationId }` | Leave a direct conversation socket room |
| Client -> Server | `message:send` | `{ clientId?, roomId?, conversationId?, text }` | Persist and send text to one authorized target |
| Server -> Client | `message:new` | `{ clientId?, message }` | Broadcast a persisted message to authorized target members |
| Server -> Client | `message:inbox` | `{ message }` | Notify an online participant even when they are not viewing that chat |
| Server -> Client | `message:error` | `{ clientId?, code, message }` | Explain a rejected join/send/history operation |
| Client -> Server | `message:read` | `{ messageId, roomId? \| conversationId? }` | Persist a read receipt for an incoming message |
| Server -> Client | `message:read:update` | `{ messageId, userId, readBy }` | Broadcast updated readers to members in the chat |
| Client -> Server | `typing:start` | `{ roomId? \| conversationId? }` | Notify other members that the user is typing |
| Client -> Server | `typing:stop` | `{ roomId? \| conversationId? }` | Clear the typing indicator |
| Server -> Client | `typing:update` | `{ targetId, userId, isTyping }` | Update the target's typing indicator |
| Server -> Client | `presence:update` | `{ userId, status: "online" \| "offline" }` | Broadcast connection presence |

The server validates socket authentication, target membership, message content,
and per-socket send interval. Typing indicators expire after five seconds if a
client disconnects or fails to send `typing:stop`. Presence is tracked per
connected user across multiple sockets in this single-server process; use a
shared adapter/store if the app later runs multiple backend instances.

## Planned for later phases

Typing/presence persistence, search history, push notifications, and account
verification are not implemented yet. Notification sound is client-side and
unread counts are derived from persisted message read receipts.
