# 10,000 Hours Backend API

Express REST API backed by PostgreSQL. It manages accounts, learner profiles, study posts, friendships, progress statistics, and image uploads.

**Hosted API:** `https://peanutbutterandjelly-backend-production.up.railway.app`

## Requirements

- Node.js and npm.
- PostgreSQL database.
- Supabase project with a public Storage bucket named `post-images` for uploaded post and profile pictures.

## Setup

Install dependencies:

```sh
npm install
```

Create a `.env` file in the project root:

```env
PORT=3000
DATABASE_URL=your-postgres-connection-string
JWT_SECRET=your-jwt-secret
SUPABASE_URL=your-supabase-project-url
SUPABASE_ANON_KEY=your-supabase-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-supabase-service-role-key
```

Keep `.env` and the Supabase service-role key private. Start the API with:

```sh
npm run dev
```

The server listens on port `3000` by default.

## Database

The API expects these tables and fields:

| Table | Fields used by the API |
| --- | --- |
| `users` | `user_id`, `user_email`, `password`, `role` |
| `profiles` | `user_id`, `username`, `age`, `bio`, `city`, `avatar_url` |
| `posts` | `post_id`, `user_id`, `visibility`, `title`, `content`, `image_url`, `skill`, `hours`, `created_at` |
| `friendships` | `id`, `user_id_1`, `user_id_2`, `status`, `requested_by` |

For an existing database, add the study-tracking and profile-picture columns if they are not present:

```sql
ALTER TABLE profiles ADD COLUMN IF NOT EXISTS avatar_url TEXT;
ALTER TABLE posts ADD COLUMN IF NOT EXISTS skill TEXT;
ALTER TABLE posts ADD COLUMN IF NOT EXISTS hours NUMERIC(4, 2);
```

Study hours are validated by the API to be from `0.25` through `24`. A study post must include both `skill` and `hours`, or neither. Supported skills are Piano, Guitar, Singing, Japanese, Korean, Spanish, Python, JavaScript, Drawing, Mathematics, and Other.

Friendship rows store the smaller user ID in `user_id_1` and the larger ID in `user_id_2`.

## Authentication

`POST /signup` and `POST /login` return a JSON Web Token. Send it to protected endpoints using:

```http
Authorization: Bearer <token>
```

Tokens expire after one hour. Missing tokens return `401`; invalid or expired tokens return `403`.

## Endpoints

`Auth` indicates whether an endpoint requires a bearer token.

| Method | Endpoint | Auth | Description |
| --- | --- | --- | --- |
| GET | `/` | No | API welcome message |
| GET | `/db-check` | No | Check database connectivity |
| POST | `/signup` | No | Create an account |
| POST | `/login` | No | Log in and receive a token |
| POST | `/auth/google` | No | Authenticate a Supabase Google user |
| GET | `/profiles/:user_id` | Yes | Read a profile, including `avatar_url` |
| POST | `/profiles` | Yes | Create your profile (`username` required; optional `age`, `bio`, `city`, `avatar_url`) |
| PATCH | `/profiles` | Yes | Update your profile fields |
| GET | `/friends` | Yes | List accepted friends |
| GET | `/friend/requests` | Yes | List incoming friend requests |
| POST | `/friend` | Yes | Send a request (`friend_id`) |
| PATCH | `/friend/accept` | Yes | Accept a request (`friend_id`) |
| DELETE | `/friend/:friend_id` | Yes | Remove, reject, or cancel a friendship request |
| GET | `/skills` | Yes | List supported study skills |
| GET | `/stats/me` | Yes | Get personal study totals, weekly hours, streak, and hours by skill |
| GET | `/leaderboard?skill=Piano` | Yes | Get the top public study-hour totals; `skill` is optional |
| GET | `/posts` | Yes | List posts visible to the signed-in user |
| GET | `/posts/:id` | Yes | Read a post the signed-in user may access |
| POST | `/posts` | Yes | Create a post; optional study post fields are `skill` and `hours` |
| PATCH | `/posts/:id` | Yes | Update your post fields |
| DELETE | `/posts/:id` | Yes | Delete your post |
| POST | `/upload` | Yes | Upload an image and receive its public `image_url` |

### Study statistics

`GET /stats/me` returns `total_hours`, `study_count`, `week_hours`, `streak_days`, and `by_skill`. The streak is measured in the `Asia/Kuala_Lumpur` time zone. A streak can end today or yesterday and counts consecutive days with at least one study post.

The leaderboard counts only posts marked `Public` that include hours. It is ordered by total hours and returns up to 10 learners.

### Image uploads

Send multipart form data with the field name `image` to `/upload`. JPG, PNG, WebP, GIF, and AVIF files up to 5 MB are accepted. Uploads use the `post-images` Supabase Storage bucket and are limited to 20 uploads per user per 15 minutes.

### Post visibility

| Value | Who can see the post |
| --- | --- |
| `Public` | All signed-in users |
| `Friends-Only` | The author and accepted friends |
| `Private` | The author |

## Response and errors

Errors use JSON with an `error` message. Common HTTP statuses are `400` for invalid input, `401` for missing credentials, `403` for invalid credentials or denied access, `404` for missing records, and `500` for server errors.
