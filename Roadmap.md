For this hackathon, I would build it as a **Next.js full-stack application with Neon PostgreSQL**, keeping the starter repository as the base and extending its existing APIs rather than rewriting the project.

## Recommended Architecture

```text
Next.js App
│
├── App Router / Pages
│
├── API Routes
│   ├── /api/auth/register
│   ├── /api/auth/login
│   ├── /api/auth/logout
│   ├── /api/auth/me
│   │
│   ├── /api/movies
│   └── /api/bookings
│       ├── GET
│       └── POST
│
├── Auth Middleware
│   └── Verify JWT / session
│
├── Prisma ORM
│
└── Neon PostgreSQL
```

The important design principle is:

**Existing starter endpoints → understand them → preserve them → add auth → protect booking flow → connect bookings to users.**

## Phase 1 — Study the Starter Repository

Before writing code, clone the repository and inspect:

```bash
git clone https://github.com/chaicodehq/book-my-ticket.git
cd book-my-ticket
npm install
```

Then identify:

```text
app/
pages/
api/
lib/
db/
models/
routes/
middleware/
```

The exact structure may differ, so don't immediately impose a new architecture.

Understand these things first:

1. Existing movie endpoints
2. Existing booking endpoint
3. Request/response formats
4. Existing database layer
5. Existing validation
6. Existing error handling
7. Existing environment variables

Your first milestone should be:

> **Run the starter application successfully without modifying functionality.**

---

# Phase 2 — Decide Your Stack

For your implementation, I recommend:

```text
Frontend / Backend → Next.js
Database            → Neon PostgreSQL
ORM                 → Prisma
Authentication      → JWT + HttpOnly Cookie
Password hashing    → bcrypt
Validation          → Zod
API testing         → Postman / Thunder Client
Deployment          → Vercel
Database hosting    → Neon
```

Why this stack?

* **Next.js** gives you frontend + backend in one project.
* **Neon** gives you hosted PostgreSQL.
* **Prisma** makes relational booking logic much easier.
* **JWT + HttpOnly Cookie** gives you a straightforward authentication layer.
* **bcrypt** prevents storing plaintext passwords.
* **Zod** keeps API input validation clean.

---

# Phase 3 — Database Design

You mainly need three core entities:

```text
User
Movie
Booking
```

A useful relationship is:

```text
User
 │
 └──────< Booking >────── Movie
```

### User

```text
User
──────
id
name
email
passwordHash
createdAt
updatedAt
```

### Movie

Since movie data is mocked:

```text
Movie
──────
id
title
description
duration
createdAt
```

You can alternatively keep movies in mock/static data if the starter project already does that.

### Booking

```text
Booking
────────
id
userId
movieId
seatNumber
createdAt
```

The critical constraint is:

```text
UNIQUE(movieId, seatNumber)
```

That database constraint is extremely important because it prevents two users from booking the same seat even if two requests arrive almost simultaneously.

Conceptually:

```text
Movie 1
 ├── A1 → User 1
 ├── A2 → User 2
 ├── A3 → User 3
 └── A4 → available
```

---

# Phase 4 — Set Up Neon

Create a Neon PostgreSQL database and get the connection string.

Your `.env` would look roughly like:

```env
DATABASE_URL="postgresql://..."
JWT_SECRET="your-super-secret-key"
```

Do **not** commit `.env`.

Create:

```text
.env.example
```

with:

```env
DATABASE_URL=
JWT_SECRET=
```

---

# Phase 5 — Prisma Setup

Install:

```bash
npm install prisma @prisma/client
npm install bcrypt jsonwebtoken
npm install zod
```

Initialize Prisma:

```bash
npx prisma init
```

Then create your schema.

Conceptually:

```prisma
model User {
  id           String    @id @default(cuid())
  name         String
  email        String    @unique
  passwordHash String
  bookings     Booking[]
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
}

model Booking {
  id         String   @id @default(cuid())
  userId     String
  movieId    String
  seatNumber String
  createdAt  DateTime @default(now())

  user User @relation(fields: [userId], references: [id])

  @@unique([movieId, seatNumber])
}
```

Then:

```bash
npx prisma migrate dev --name init
```

---

# Phase 6 — Authentication Flow

You need four authentication APIs.

## Register

```http
POST /api/auth/register
```

Request:

```json
{
  "name": "Dhanraj",
  "email": "dhanraj@example.com",
  "password": "password123"
}
```

Flow:

```text
Request
   ↓
Validate input
   ↓
Check existing email
   ↓
Hash password with bcrypt
   ↓
Create User
   ↓
Return success
```

Never store:

```text
password
```

Store:

```text
passwordHash
```

---

# Phase 7 — Login

```http
POST /api/auth/login
```

Flow:

```text
Email + Password
       ↓
Find User
       ↓
Compare bcrypt hash
       ↓
Generate JWT
       ↓
Set HttpOnly Cookie
       ↓
Return user information
```

Cookie example:

```text
auth_token=JWT
```

with:

```text
HttpOnly
Secure
SameSite
```

For production, use secure cookie configuration.

---

# Phase 8 — Current User Endpoint

Add:

```http
GET /api/auth/me
```

Flow:

```text
Cookie
  ↓
JWT verification
  ↓
userId
  ↓
Database lookup
  ↓
User response
```

Example:

```json
{
  "id": "user_123",
  "name": "Dhanraj",
  "email": "dhanraj@example.com"
}
```

This is useful for both frontend and debugging.

---

# Phase 9 — Authentication Middleware

This is one of the most important parts of the assignment.

Create something conceptually like:

```text
middleware/auth.ts
```

The middleware should:

```text
Request
   ↓
Read auth cookie
   ↓
Verify JWT
   ↓
Invalid?
 ├── Yes → 401 Unauthorized
 └── No
       ↓
    Extract userId
       ↓
    Continue request
```

Don't trust:

```json
{
  "userId": "someone-else"
}
```

coming from the client.

The authenticated user must come from the verified authentication token.

---

# Phase 10 — Protect Booking API

Your booking endpoint should become:

```http
POST /api/bookings
```

and require authentication.

Request:

```json
{
  "movieId": "movie_1",
  "seatNumber": "A10"
}
```

Notice that the client should **not** send:

```json
{
  "userId": "123"
}
```

The server determines:

```text
userId = authenticated user
```

from the JWT/session.

This gives:

```text
Logged-in User
      ↓
POST /api/bookings
      ↓
Auth Middleware
      ↓
Extract userId
      ↓
Validate movie
      ↓
Check seat availability
      ↓
Create booking
```

---

# Phase 11 — Prevent Duplicate Seat Bookings

This is one area where you should rely on both:

### Application-level check

```text
Does movieId + seatNumber already exist?
```

and:

### Database-level protection

```prisma
@@unique([movieId, seatNumber])
```

Why both?

Imagine this:

```text
User A ── booking A10 ──┐
                        ├── simultaneously
User B ── booking A10 ──┘
```

Both requests might pass an initial availability check.

The database constraint guarantees that only one can succeed.

Your API should catch the unique constraint error and return something like:

```http
409 Conflict
```

```json
{
  "message": "Seat A10 is already booked"
}
```

---

# Phase 12 — Booking GET API

Add:

```http
GET /api/bookings
```

This should also require authentication.

Return only the logged-in user's bookings:

```text
JWT userId
   ↓
WHERE booking.userId = userId
```

Example:

```json
[
  {
    "id": "booking_1",
    "movieId": "movie_1",
    "seatNumber": "A10"
  },
  {
    "id": "booking_2",
    "movieId": "movie_2",
    "seatNumber": "B12"
  }
]
```

A user should never be able to retrieve another user's booking history.

---

# Phase 13 — Movie API

Keep movie data mocked as requested.

For example:

```text
GET /api/movies
```

returns:

```json
[
  {
    "id": "movie_1",
    "title": "Avengers",
    "duration": 150
  },
  {
    "id": "movie_2",
    "title": "Interstellar",
    "duration": 169
  }
]
```

You don't need to over-engineer the movie system.

The assignment's real focus is:

```text
Authentication
       +
Authorization
       +
Booking consistency
```

---

# Phase 14 — Recommended API Structure

Your final backend should roughly have:

```text
/api
│
├── auth
│   ├── register
│   ├── login
│   ├── logout
│   └── me
│
├── movies
│
└── bookings
    ├── GET
    └── POST
```

### Authentication

```text
POST /api/auth/register
POST /api/auth/login
POST /api/auth/logout
GET  /api/auth/me
```

### Movies

```text
GET /api/movies
```

### Bookings

```text
GET  /api/bookings
POST /api/bookings
```

---

# Phase 15 — Error Handling

Keep your responses consistent.

### Validation

```http
400 Bad Request
```

```json
{
  "message": "Invalid input"
}
```

### Unauthorized

```http
401 Unauthorized
```

```json
{
  "message": "Authentication required"
}
```

### Forbidden

```http
403 Forbidden
```

when a user is authenticated but lacks permission.

### Duplicate seat

```http
409 Conflict
```

```json
{
  "message": "Seat already booked"
}
```

### Server error

```http
500 Internal Server Error
```

---

# Phase 16 — Frontend

Frontend is optional, so don't spend too much time here.

A simple UI is enough:

```text
Login
  ↓
Movies
  ↓
Select Movie
  ↓
Seat Grid
  ↓
Select Seat
  ↓
Book Seat
  ↓
My Bookings
```

Example seat layout:

```text
        SCREEN

A1  A2  A3  A4  A5

B1  B2  B3  B4  B5

C1  C2  C3  C4  C5
```

Booked seats:

```text
A2 → unavailable
B4 → unavailable
```

Available seats can be selected.

---

# Phase 17 — Testing Strategy

This project will be evaluated heavily through APIs, so test them systematically.

### Test 1 — Register

```text
POST /auth/register
```

Expected:

```text
201 Created
```

### Test 2 — Duplicate registration

Use same email.

Expected:

```text
409 Conflict
```

### Test 3 — Login

```text
POST /auth/login
```

Expected:

```text
200 OK
Set-Cookie
```

### Test 4 — Protected booking without login

```text
POST /bookings
```

without cookie.

Expected:

```text
401 Unauthorized
```

### Test 5 — Booking after login

Expected:

```text
201 Created
```

### Test 6 — Duplicate seat

Book same:

```text
movie_1 + A10
```

again.

Expected:

```text
409 Conflict
```

### Test 7 — My bookings

```text
GET /bookings
```

Expected:

```text
Only authenticated user's bookings
```

---

# Phase 18 — Repository Structure

A clean Next.js structure could look like:

```text
book-my-ticket/
│
├── app/
│   ├── api/
│   │   ├── auth/
│   │   │   ├── register/
│   │   │   ├── login/
│   │   │   ├── logout/
│   │   │   └── me/
│   │   │
│   │   ├── movies/
│   │   └── bookings/
│   │
│   ├── login/
│   ├── register/
│   ├── movies/
│   └── bookings/
│
├── lib/
│   ├── prisma.ts
│   ├── auth.ts
│   ├── jwt.ts
│   └── validation.ts
│
├── prisma/
│   └── schema.prisma
│
├── middleware.ts
│
├── .env
├── .env.example
├── package.json
└── README.md
```

The exact placement should follow the starter repository where practical so you **extend rather than unnecessarily restructure** it.

---

# Phase 19 — README

Your README should clearly document:

```text
# Book My Ticket

## Features
- User registration
- User login
- JWT authentication
- Protected booking APIs
- Seat booking
- Duplicate seat prevention
- User-specific booking history

## Tech Stack
- Next.js
- PostgreSQL
- Neon
- Prisma
- bcrypt
- JWT

## Setup

git clone ...
npm install

Create .env

npm run dev
```

Then document API endpoints:

```text
POST /api/auth/register
POST /api/auth/login
POST /api/auth/logout
GET  /api/auth/me
GET  /api/movies
GET  /api/bookings
POST /api/bookings
```

Also document the booking flow:

```text
Register
   ↓
Login
   ↓
JWT stored in HttpOnly cookie
   ↓
Access protected endpoint
   ↓
Server verifies user
   ↓
Create booking
   ↓
Database enforces unique seat
```

---

# Your Development Order

I strongly recommend doing it in this exact sequence:

```text
1. Clone starter repository
        ↓
2. Run existing project
        ↓
3. Understand existing endpoints
        ↓
4. Connect Neon
        ↓
5. Add Prisma
        ↓
6. Create User + Booking schema
        ↓
7. Implement Register
        ↓
8. Implement Login
        ↓
9. Implement JWT/Auth utility
        ↓
10. Implement middleware
        ↓
11. Protect booking endpoint
        ↓
12. Associate booking with logged-in user
        ↓
13. Add unique(movieId, seatNumber)
        ↓
14. Add GET /bookings
        ↓
15. Test all edge cases
        ↓
16. Optional frontend
        ↓
17. Write README
        ↓
18. Deploy Neon + Vercel
        ↓
19. Push final GitHub repository
```

## Most Important Evaluation Points

Make these airtight:

```text
✅ Starter endpoints still work
✅ Register works
✅ Login works
✅ Passwords are hashed
✅ Auth is actually verified server-side
✅ Booking requires authentication
✅ userId comes from authenticated identity
✅ Users cannot access other users' bookings
✅ Duplicate seats are impossible
✅ Proper HTTP status codes
✅ Neon database is actually used
✅ README explains setup and architecture
```

### One architectural recommendation

For this particular assignment, I would **not introduce NextAuth/Auth.js unless the starter project already uses it**. A small custom JWT + HttpOnly cookie layer is easier to understand, demonstrate, and debug during a backend-focused hackathon. The key is to implement it securely rather than merely checking for the presence of a cookie.


                    ┌───────────────────┐
                    │     Next.js       │
                    │     Frontend      │
                    └─────────┬─────────┘
                              │
                              │ HTTP
                              ▼
                    ┌───────────────────┐
                    │      Express      │
                    │      Backend      │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
         Auth API        Seat API        Booking API
              │               │                │
              └───────────────┼────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  Neon PostgreSQL  │
                    │                   │
                    │ users             │
                    │ seats             │
                    └───────────────────┘

                    book-my-ticket/
│
├── backend/
│   ├── index.mjs
│   ├── db.js
│   ├── auth.js
│   ├── middleware/
│   │   └── authMiddleware.js
│   ├── routes/
│   │   ├── auth.js
│   │   ├── seats.js
│   │   └── bookings.js
│   └── ...
│
├── frontend/
│   └── Next.js
│
├── README.md
└── package.json