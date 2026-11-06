# Tandem Backend

The backend provides Tandem's HTTP API, persistent chat data, authentication, and Socket.IO real-time events.

## Stack

- Node.js ES modules and Express 4
- MongoDB with Mongoose 8
- Socket.IO 4 for authenticated real-time connections
- Zod request validation, bcrypt password hashing, and Google ID-token verification
- Multer and `file-type` for profile image upload validation; Cloudinary for image storage
- dotenv for environment loading

## Prerequisites

- Node.js and npm
- A reachable MongoDB deployment, local or hosted
- Cloudinary credentials to enable avatar uploads
- Google OAuth client configuration only if Google sign-in is required

The package does not declare a Node.js engine version. Use a current Node.js release compatible with the dependencies.

## Local setup

From this directory, copy the checked-in environment example and install the locked dependencies:

```sh
cp .env.example .env
npm ci
npm run dev
```

On PowerShell, replace the copy command with `Copy-Item .env.example .env`. Start MongoDB and set `MONGODB_URI` before starting the server. The backend loads `.env` from this project directory. By default, the API and Socket.IO server listen on port `4000`.

## Environment variables

See [`.env.example`](.env.example). These settings belong to the backend; do not place backend secrets in the frontend environment.

| Variable | Purpose | Example / default |
| --- | --- | --- |
| `PORT` | HTTP and Socket.IO server port. | `4000` when unset |
| `FRONTEND_URL` | Allowed frontend origin(s), comma-separated. Configure origins only, without paths. | `http://localhost:3000` |
| `MONGODB_URI` | MongoDB connection URI; required to start. | `mongodb://127.0.0.1:27017/tandem-chat` |
| `SESSION_TTL_DAYS` | Session lifetime in days. | `7` when unset or invalid |
| `GOOGLE_CLIENT_ID` | OAuth client ID used to verify Google sign-in tokens. | `your-google-oauth-client-id` |
| `CLOUDINARY_CLOUD_NAME` | Cloudinary account name for profile images. | `your-cloud-name` |
| `CLOUDINARY_API_KEY` | Cloudinary API key. | `your-cloudinary-api-key` |
| `CLOUDINARY_API_SECRET` | Cloudinary API secret. | `your-cloudinary-api-secret` |
| `NODE_ENV` | Node environment; development mode enables additional Socket.IO diagnostics. | Set to `production` in production; not in `.env.example` |

Use real credentials only in an ignored local `.env` or your hosting platform's secret store. The placeholders above are not working credentials. The backend does not use a `JWT_SECRET`: sessions use random opaque bearer tokens, and MongoDB stores their SHA-256 hashes and expiry dates.

## Scripts

| Command | Description |
| --- | --- |
| `npm run dev` | Start the server with Nodemon. |
| `npm start` | Start the server with Node.js. |
| `npm test` | Run the Node.js test suite with `node --test`. |

There is no backend build script; the application runs directly from `src/index.js`.

## Project structure

```text
src/
  app.js                 Express middleware, health check, and route registration
  index.js               HTTP server, Socket.IO authentication/events, startup
  config/                Environment and allowed-origin configuration
  controllers/           Authentication, user, and conversation handlers
  db/                    MongoDB connection
  middleware/            Authentication, upload, and error handling
  models/                User, session, conversation, and message models
  routes/                HTTP route definitions
  schemas/               Zod request validation
  utils/                 Session auth, Cloudinary, presence, and API helpers
test/                    Node.js tests
```

## HTTP API

All endpoints use the `/api` prefix. Successful responses use the API's response envelope. `GET /api/auth/me`, all `/api/users` routes, and all `/api/conversations` routes require `Authorization: Bearer <token>`. Registration, login, Google login, logout, and the health check do not use the `requireAuth` middleware; logout revokes the bearer token when one is supplied.

| Method and path | Function |
| --- | --- |
| `GET /api/health` | Health response (`status: ok`). |
| `POST /api/auth/register` | Create an email/password account and return a session token. |
| `POST /api/auth/login` | Authenticate an email/password account and return a session token. |
| `POST /api/auth/google` | Verify a Google credential and return a session token; requires `GOOGLE_CLIENT_ID`. |
| `POST /api/auth/logout` | Revoke the bearer session token, if supplied. |
| `GET /api/auth/me` | Return the authenticated user's profile. |
| `GET /api/users?q=...&limit=...` | Search other users by name or email (`limit` defaults to 10 and is capped at 20). |
| `PATCH /api/users/name` | Update the authenticated user's display name. |
| `POST /api/users/avatar` | Upload a multipart field named `avatar`. |
| `DELETE /api/users/avatar` | Remove the uploaded avatar. |
| `GET /api/conversations` | List the user's one-to-one conversations, last messages, and unread counts. |
| `POST /api/conversations` | Find or create a conversation with `{ "userId": "..." }`. |
| `GET /api/conversations/:conversationId/messages` | Read conversation history; supports `limit` and `before` pagination parameters. |
| `POST /api/conversations/:conversationId/read` | Clear the current user's unread count for that conversation. |

## Authentication and Socket.IO

Registration and login return a random opaque token with an expiry. The frontend stores it and sends it as a bearer token for HTTP requests; Socket.IO sends it in `handshake.auth.token`. The backend persists only a hash of the token in MongoDB. Protected HTTP routes and Socket.IO connections verify the session against MongoDB.

Socket.IO is served by the same HTTP server and accepts the configured `FRONTEND_URL` origins. A connection joins its private user room and receives `presence:snapshot` with currently online user IDs. Other connected clients receive `presence:changed` when a user's online status changes.

| Event | Direction | Behavior |
| --- | --- | --- |
| `conversation:join` | Client to server | Join a conversation room after the server confirms membership; acknowledges `{ ok: true }` or an error. |
| `conversation:leave` | Client to server | Leave the conversation room. |
| `message:send` | Client to server | Send `{ conversationId, text }`; the server verifies membership, stores the message, updates unread counts, and acknowledges the result. |
| `message:new` | Server to room | Broadcast a persisted message to joined conversation members. |
| `conversation:updated` | Server to each participant's user room | Notify clients of the latest message and that participant's unread count. |
| `typing:update` | Client to server | Send `{ conversationId, isTyping }`; the server verifies membership before relaying. |
| `typing:changed` | Server to conversation room | Relay typing status with the conversation and user IDs. |
| `presence:snapshot` | Server to newly connected client | Provide the current set of online user IDs. |
| `presence:changed` | Server to clients | Announce a user becoming online or offline. |

## MongoDB and profile images

`MONGODB_URI` is required. The default example points to a local database named `tandem-chat`; use a managed MongoDB URI and appropriate network access controls when deployed. Users, sessions, conversations, and messages are stored in MongoDB.

Avatar upload uses Multer memory storage and accepts one file in the `avatar` multipart field. The backend limits files to 5 MB and validates JPEG, PNG, or WebP content before sending it to Cloudinary's `tandem/avatars` folder. All three Cloudinary variables are required to upload an image; without them, upload returns a configuration error. Removing an uploaded image updates the profile to the user's Google avatar when available, otherwise an empty avatar; deleting the old Cloudinary asset is best-effort cleanup.

## Production deployment

No Docker, hosting-provider, or deployment configuration is included. Deploy the backend as a long-running Node.js service that supports HTTP and Socket.IO/WebSocket connections, and run `npm start`. Configure `PORT` from the platform, a production MongoDB URI, an explicit `FRONTEND_URL`, and the needed Google/Cloudinary settings in the platform's environment or secret store. Set `NODE_ENV=production` in the hosting environment.

For the deployed frontend `https://tandem-real-time-chat-application-f.vercel.app`, set this backend environment variable:

```dotenv
FRONTEND_URL=https://tandem-real-time-chat-application-f.vercel.app
```

The deployed backend origin is `https://tendem-backend.onrender.com`. Configure the frontend build with `NEXT_PUBLIC_API_URL=https://tendem-backend.onrender.com/api` and `NEXT_PUBLIC_SOCKET_URL=https://tendem-backend.onrender.com`. The backend uses `FRONTEND_URL` as the allowed origin for both Express CORS and Socket.IO. If it is unset, the code falls back to `http://localhost:3000`, so browser requests and Socket.IO handshakes from the deployed frontend will not be allowed. Use HTTPS, keep the database private, and ensure any reverse proxy forwards Socket.IO upgrade requests. Provider-specific domain, TLS, database, secret, scaling, and proxy steps depend on the hosting platform and are not defined in this repository.

## Related Repository

- **Frontend:** [Tandem Frontend](https://github.com/aleeza-waris/tandem-real-time-chat-application-frontend)