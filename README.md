# Wetin Sup

A mobile-first, real-time messaging app built as a two-person portfolio project.
Wetin Sup brings global chat, discoverable groups, and private conversations
together with live presence, typing indicators, image sharing, and read receipts.

> **Live demo:** [Add your deployed Vercel URL]
>
> **Demo account:** [Add a safe demo account, or instructions for creating one]

## Screenshots and demo

Add genuine screenshots and a short demonstration GIF after deploying and
testing the app:

| Chat list | Live conversation | Profile and image sharing |
|---|---|---|
| ![Wetin Sup chat list](./docs/images/chat-list.png) | ![Wetin Sup conversation](./docs/images/conversation.png) | ![Wetin Sup profile](./docs/images/profile.png) |

**Demo GIF:** [Add `docs/images/wetin-sup-demo.gif`](./docs/images/wetin-sup-demo.gif)

The image paths above are portfolio placeholders. Add your own screenshots/GIF
to `docs/images/` before publishing the README.

## Features

- Phone-number and password signup/sign-in with bcrypt password hashes and
  signed JWT authentication.
- Authenticated real-time Socket.IO connections.
- Persistent global chat, public/discoverable groups, and private one-to-one
  conversations.
- Membership-protected rooms and message history.
- Online/offline presence, typing indicators, and live message delivery.
- Persistent read receipts and unread message badges.
- Emoji picker, optional notification sound, and Cloudinary image messages.
- Editable profile with a Cloudinary avatar.
- Responsive, mobile-first dark interface.
- Security basics: Helmet headers, CORS allowlist, request limits, auth/upload
  rate limits, input validation, and no credentials in the client bundle.

## Tech stack

| Area | Technologies |
|---|---|
| Frontend | React 18, Vite, CSS |
| Backend | Node.js 20+, Express, Socket.IO |
| Database | MongoDB Atlas, Mongoose |
| Authentication | JWT, bcryptjs |
| Image uploads | Cloudinary signed uploads |
| Hosting | Vercel (client), Render (API), MongoDB Atlas |

## Project structure

```text
.
├── client/
│   ├── src/
│   │   ├── components/
│   │   ├── context/
│   │   ├── lib/
│   │   └── screens/
│   ├── public/
│   │   ├── wetin-sup-icon.png
│   │   └── wetin-sup-logo.png
│   └── vite.config.js
├── server/
│   └── src/
│       ├── config/
│       ├── controllers/
│       ├── middleware/
│       ├── models/
│       ├── routes/
│       ├── services/
│       └── socket/
├── API.md
├── render.yaml
└── vercel.json
```

## Run locally

### Prerequisites

- Node.js 20 or newer and npm.
- MongoDB locally, or a MongoDB Atlas database.
- Cloudinary account and a signed upload preset for image/profile uploads
  (optional while developing other features).

### Install and configure

```powershell
Copy-Item .env.example .env
npm.cmd install
```

Edit `.env`:

1. Set `MONGODB_URI` to your local or Atlas connection string.
2. Set `JWT_SECRET` to a random secret of at least 32 characters. Generate one
   with:

   ```powershell
   node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"
   ```

3. Optional image uploads require all four `CLOUDINARY_*` variables. Create a
   **signed** preset restricted to the `wetin-sup` folder, JPG/PNG/WebP formats,
   and a 5 MB maximum. Never put the Cloudinary API secret in Vite/client
   variables.

Start the API and client together:

```powershell
npm.cmd run dev
```

Open `http://localhost:5173`. Check `http://localhost:5000/api/health` for the
API/database status. If PowerShell blocks `npm.ps1`, use the `npm.cmd` commands
shown here. The client API URL defaults to `http://localhost:5000/api`.

## Test the main flows

1. Sign up with an E.164 phone number (for example `+14155550123`), a password
   with at least 8 characters, a full name, and a nickname.
2. Open a second browser/private window and create a second account.
3. Send global messages; confirm they appear in both windows and persist after
   refresh.
4. Create a public group, invite the second account, and test group chat,
   member presence, discovery, join, and leave.
5. From **People**, start a one-to-one conversation. Confirm only the two
   participants can access it.
6. Type in one window and verify the other sees the typing indicator. Check
   online status as each window connects/disconnects.
7. Send an image and update the profile avatar (requires Cloudinary config).
8. Send a message while the recipient is elsewhere, verify the unread badge and
   optional sound, then open the conversation and check the read receipt.
9. Verify `GET /api/v1/users/me` returns `401` without a bearer token.

See [API.md](./API.md) for REST routes, socket events, data shapes, and limits.

## Deploy

The repository includes [render.yaml](./render.yaml) for the API and
[vercel.json](./vercel.json) for the React SPA. Deploying requires you to create
the provider projects and enter your own environment values; no live URLs or
credentials are committed to this repository.

### 1. Create MongoDB Atlas database

1. Create an Atlas cluster and database user with a strong unique password.
2. Add the connection URI to Render as `MONGODB_URI`; encode special characters
   in the URI password as required by MongoDB.
3. Restrict Atlas network access to the Render service's outbound IP addresses.
   If stable addresses are needed, configure a Render static outbound IP first.
   Avoid opening the database to all IPs for a production deployment.

### 2. Deploy the API to Render

1. Create a Render Blueprint from this repository and review `render.yaml`.
2. Set `CLIENT_URL` to the exact HTTPS Vercel production origin, with no path or
   trailing slash (for example `https://your-project.vercel.app`). Add additional
   trusted origins as a comma-separated list when necessary.
3. Add Atlas credentials. Render generates `JWT_SECRET`; keep it private.
4. For uploads, configure the four Cloudinary values and the signed preset
   described above. Leave all four unset to disable uploads.
5. Deploy and wait for `GET /api/health` to report database status `connected`.
6. Copy the Render service URL (for example `https://your-api.onrender.com`).

The Render health check is `/api/health`. Its response indicates MongoDB
connectivity; it does not expose database credentials or private data.

### 3. Deploy the client to Vercel

1. Import the repository as a Vercel project using the repository root.
   `vercel.json` installs workspace dependencies, builds the client, serves
   `client/dist`, and routes SPA paths to the React entry point.
2. In Vercel **Project Settings → Environment Variables**, set
   `VITE_API_URL` to the Render API base URL with `/api` appended, for example
   `https://your-api.onrender.com/api`.
3. Redeploy after setting or changing `VITE_API_URL`; Vite embeds it at build
   time. Production builds intentionally fail when this variable is missing or
   points to a non-HTTPS, non-local host.
4. Confirm Render `CLIENT_URL` exactly matches the deployed Vercel origin.
   Update `CLIENT_URL` and redeploy Render if the Vercel domain changes.

### Production environment variables

| Variable | Where | Required | Notes |
|---|---|---:|---|
| `NODE_ENV` | Render | Yes | Set to `production` |
| `PORT` | Render | Provider-managed | Render supplies the listening port |
| `MONGODB_URI` | Render | Yes | Atlas connection URI; keep secret |
| `JWT_SECRET` | Render | Yes | At least 32 random characters; keep secret |
| `CLIENT_URL` | Render | Yes | Exact HTTPS frontend origin(s), comma-separated |
| `VITE_API_URL` | Vercel build environment | Yes | HTTPS API URL ending with `/api`; public URL, not a secret |
| `CLOUDINARY_CLOUD_NAME` | Render | Optional | Set all Cloudinary values together |
| `CLOUDINARY_API_KEY` | Render | Optional | Server-side signed upload config |
| `CLOUDINARY_API_SECRET` | Render | Optional | Secret; never add as a `VITE_` variable |
| `CLOUDINARY_UPLOAD_PRESET` | Render | Optional | Signed, restricted preset |

## Team workflow

- `main` contains reviewed, deployable releases; `dev` is the integration branch.
- Use focused branches such as `feat/profile-uploads` or `fix/unread-counts`.
- Pull requests target `dev`, include test evidence, and receive a teammate
  review. Merge `dev` to `main` after the phase is integrated and verified.

## Current limitations

- Presence uses in-memory state and is intended for one backend instance.
  Multi-instance deployment needs a shared Socket.IO adapter and presence store.
- The prototype stores the JWT in browser local storage. Before handling
  sensitive production traffic, assess HttpOnly cookie sessions, token refresh
  and revocation, stronger operational monitoring, and abuse controls.
- No live demo, screenshots, or demo GIF are included until you deploy and
  capture the real app.
