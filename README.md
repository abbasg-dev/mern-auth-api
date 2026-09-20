# mern-auth-api

An Express and MongoDB (Mongoose) REST API for accounts: email-verified sign-up, sign-in, password reset, Google and Facebook sign-in, and two role levels (`subscriber`, `admin`). It is the server for `mern-auth-client`.

- The data model, constraints and query plans are in [`DATA_MODEL.md`](./DATA_MODEL.md).
- This file covers the endpoints, how to run it, and what I would revisit at ten times the traffic.

## Stack

Node.js (ES modules), Express 4, Mongoose 5, `express-validator`, `jsonwebtoken` and `express-jwt`, SendGrid for email, `google-auth-library` for Google tokens, `node-fetch` for the Facebook Graph API. `package.json` pins `engines.node` to `14.15.4`; newer Node versions have not been tested here.

```
server.js              app setup: morgan, body-parser, cors, mounts routes under /api
config/db.js           mongoose.connect(process.env.DATABASE)
models/user.js         the only model
routes/authRoutes.js   public auth endpoints
routes/userRoutes.js   endpoints that need a signed-in user
controllers/auth.js    signup, activation, signin, reset, social login, requireSignin, adminMiddleware
controllers/user.js    read and update profile
validators/            express-validator rules; a middleware turns the first failure into a 422
Procfile               web: node server.js   (Heroku-style hosts)
```

## Running it locally

### 1. Prerequisites

- Node.js 14 (or try a newer LTS)
- A MongoDB you can reach: local `mongod`, Docker (`docker run -p 27017:27017 mongo:4.4`), or an Atlas cluster
- For the email flows: a SendGrid API key and a verified sender address
- For social login: a Google OAuth client ID (Facebook needs none on the server; the client sends the user's access token)

### 2. Install

```bash
npm install
npm install lodash     # controllers/auth.js imports lodash, but it is missing from package.json
```

The `lodash` line is needed today. Better: add it to `dependencies` and commit the change.

### 3. Configure

Create `.env` in the project root (it is git-ignored):

```
PORT=5000
DATABASE=mongodb://localhost:27017/mern-auth
CLIENT_URL=https://localhost:3000

JWT_SECRET=<a long random string>
JWT_ACCOUNT_ACTIVATION=<another long random string>
JWT_RESET_PASSWORD=<a third long random string>

SENDGRID_API_KEY=<your key>
EMAIL_FROM=<verified sender address>

GOOGLE_CLIENT_ID=<your Google OAuth client id>
```

What each one does:

| Variable | Used for |
|---|---|
| `PORT` | Listening port (default 5000) |
| `DATABASE` | MongoDB connection string. If it is wrong the process exits with code 1. |
| `CLIENT_URL` | Base of the links inside emails. Must equal the front end's real origin. |
| `JWT_SECRET` | Signs sign-in tokens (7 days) |
| `JWT_ACCOUNT_ACTIVATION` | Signs activation-link tokens (10 minutes) |
| `JWT_RESET_PASSWORD` | Signs reset-link tokens (10 minutes) |
| `SENDGRID_API_KEY`, `EMAIL_FROM` | Activation and reset emails |
| `GOOGLE_CLIENT_ID` | Audience check when verifying Google ID tokens |

> **A trap to know about before your first sign-in works.** `controllers/auth.js` *signs* sign-in tokens with `JWT_SECRET` but `requireSignin` *verifies* them against a string written directly in the source. Tokens only verify if `JWT_SECRET` in your `.env` is character-for-character equal to that literal. Until that is fixed (change `requireSignin` to use `process.env.JWT_SECRET`), every protected endpoint returns 401 if the two differ. The literal is in a public repository, so do not reuse it anywhere real.

### 4. Start

```bash
npm run dev     # nodemon, restarts on change
npm start       # plain node
```

You should see `API is running on port 5000` and `MongoDB Connected`.

### 5. Check it is alive

```bash
curl -i -X POST localhost:5000/api/signup -H 'Content-Type: application/json' -d '{"name":"","email":"nope","password":"1"}'
```

Expected: `422` with `{"error":"Name is required"}`. That proves routing, JSON parsing and validation without touching MongoDB or SendGrid.

### 6. Getting a user without SendGrid

Sign-up needs email, so for local work create a user directly. Save as `seed-user.mjs` in the project root:

```js
import dotenv from 'dotenv'
dotenv.config()
import mongoose from 'mongoose'
import User from './models/user.js'

await mongoose.connect(process.env.DATABASE, {
  useNewUrlParser: true, useUnifiedTopology: true, useCreateIndex: true
})
await new User({
  name: 'Local Admin',
  email: 'admin@example.com',
  password: 'secret123',   // the virtual setter salts and hashes this
  role: 'admin'
}).save()
await mongoose.disconnect()
```

Run `node seed-user.mjs`, then sign in:

```bash
curl -s -X POST localhost:5000/api/signin -H 'Content-Type: application/json' \
  -d '{"email":"admin@example.com","password":"secret123"}'
```

You get back `{ token, user }`. Send the token as `Authorization: Bearer <token>` on the protected routes. (I have not run this script; it uses only the model and connection code that already exists in the repo.)

## Endpoints

All paths are under `/api`. Bodies are JSON. Error bodies are `{ "error": "<message>" }` unless stated. Validation failures (422) report only the **first** failing rule.

"Auth" means a `Authorization: Bearer <token>` header from `/signin`. A missing or invalid token makes `express-jwt` throw, and because the app has no error-handling middleware, Express answers **401 with an HTML error page**, not JSON.

### Public

| Method & path | Body | Status codes |
|---|---|---|
| `POST /signup` | `name`, `email`, `password` (min 6) | **200** activation email sent. **200** *also* returned when SendGrid fails (body is `{message: <SendGrid error>}`). **400** `Email is taken`. **422** validation. |
| `POST /account-activation` | `token` from the emailed link | **200** user created. **200** `Something went wrong` when no token is sent. **401** `Expire link. Signup again` (bad or expired token). **401** `Error saving user in database` (includes a duplicate email). |
| `POST /signin` | `email`, `password` (min 6) | **200** `{ token, user:{_id,name,email,role} }`. **400** `User with that email does not exist` or `Email and password do not match`. **422** validation. |
| `PUT /forgot-password` | `email` | **200** reset email sent (message text wrongly says "activate your account"). **200** also when SendGrid fails. **400** no such user, or DB error. **422** validation. |
| `PUT /reset-password` | `resetPasswordLink` (token), `newPassword` (min 6) | **200** password changed. **400** `Expired link`, or no user holds that token, or save failed. **422** validation. **No response at all** if `resetPasswordLink` is omitted: the handler has no `else`, so the request hangs until the client times out. |
| `POST /google-login` | `idToken` | **200** `{ token, user }` (creates the user on first login). **400** email not verified, or user save failed. A rejected token throws inside a promise with no `.catch`, so the request **hangs**. |
| `POST /facebook-login` | `userID`, `accessToken` | **200** `{ token, user }`. **400** user save failed. **200** with `{error}` in the body when the Graph API call fails. A Facebook account with no email is not handled. |

### Authenticated

| Method & path | Body | Status codes |
|---|---|---|
| `GET /user/:id` | none | **200** the user (minus `hashed_password` and `salt`). **400** `User not found`, which also covers a malformed id. **401** no or bad token. Any signed-in user can read **any** user's document. |
| `PUT /user/update` | `name` (required), `password` (optional, min 6) | **200** updated user. **400** `User not found`, `Name is required`, `Password should be min 6 characters long`, or `User update failed`. **401** no or bad token. It always updates the user in the token, never one named in the body. |
| `PUT /admin/update` | same as above | Same as `/user/update`, plus **400** `Admin resource. Access denied.` for a non-admin (403 would be the correct code). It updates the *caller's* own record, so it grants no extra power. |

## Decisions I would revisit at ten times the traffic

Ordered by how soon each would hurt.

1. **Email is sent inside the request.** `signup` and `forgot-password` wait on SendGrid before answering, and report SendGrid failures as 200. At 10x, SendGrid latency or rate limits become the API's latency and its silent failures. Move sending to a queue and return 202.
2. **The reset-token lookup has no index.** `resetPassword` runs `findOne({ resetPasswordLink })`, which scans the whole collection. It is the one query whose cost grows with the number of users (details and plan in `DATA_MODEL.md`).
3. **Password hashing is HMAC-SHA1 with a salt built from `Math.random()`.** It is cheap, which is exactly the problem for security. The fix (bcrypt or argon2) is deliberately expensive in CPU, so at 10x, sign-in becomes CPU-bound on a single Node thread. That needs rate limiting and either worker processes or a separate auth service. The hash also stores no algorithm marker, so migrating means adding one.
4. **Sign-in tokens live 7 days and cannot be revoked.** There is no store to check. Compromised accounts and role changes take up to a week to take effect. At scale I would add short-lived access tokens with a revocable refresh token.
5. **`adminMiddleware` does a database read on every admin request,** while other protected routes trust the token alone. It is fine at this size but it is the first per-request DB hit to appear in a profile.
6. **One Node process, default Mongoose pool, `morgan('dev')` logging to stdout, no rate limiting, CORS open to every origin.** None of it is a problem at low traffic. At 10x, abuse of `/signin`, `/signup` and `/forgot-password` (each of which can send email or burn CPU) is the first real load.
7. **Social login is find-then-insert.** Two simultaneous first-time logins for one email race; the unique index makes one fail with a 400 "signup failed" instead of falling back to sign-in.
8. **Schema indexes are built automatically at startup** (`useCreateIndex`, Mongoose's default `autoIndex`). On a large collection, turn that off and create indexes deliberately.

## Known defects (not scale problems)

- `lodash` is imported but not declared in `package.json`.
- The `resetPasswordLink` field in the schema looks wrongly defined, so reset tokens may never be persisted. See `DATA_MODEL.md` and test password reset end to end before relying on it.
- `requireSignin` uses a hardcoded secret (see the trap above).
- `GET /user/:id` has no ownership check, and its response can include `resetPasswordLink`.
- The sign-up token carries the plaintext password inside the emailed link (a JWT is signed, not encrypted).
- Social-login accounts get the password `email + JWT_SECRET`, which is predictable to anyone who knows the secret and could be used at `/signin`.
- No tests, no error-handling middleware, callback-style Mongoose calls inside `asyncHandler` so thrown errors are not caught.
