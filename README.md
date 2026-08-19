# fastapi-blog

A small blog application built with [FastAPI](https://fastapi.tiangolo.com/) — a learning
project for working through the framework's routing, Jinja2 templating, static files,
database access, and automatic API docs.

Posts and users are stored in PostgreSQL, reached asynchronously through SQLAlchemy's async
engine and `psycopg`. Every route is an `async def` coroutine. The schema is owned by Alembic
migrations rather than created on startup.

Writing, editing, and deleting posts happens in the browser: Bootstrap modals collect the
input and `fetch` calls talk to the same JSON API the docs expose, so no page does a
classic form POST.

## Features

- Server-rendered pages with Jinja2 template inheritance (`layout.html` → `home.html`)
- Responsive layout styled with [Bootstrap 5](https://getbootstrap.com/) plus a custom stylesheet
- Light / dark / auto theme switcher, with the choice persisted in `localStorage`
- PostgreSQL persistence through [SQLAlchemy](https://www.sqlalchemy.org/) ORM models, with an
  `AsyncSession` injected into routes as a dependency
- Schema managed by [Alembic](https://alembic.sqlalchemy.org/) migrations, so the database is
  versioned and upgrades are repeatable rather than implicit
- Fully asynchronous request handling: `async def` routes awaiting non-blocking queries, so
  a slow database call does not tie up the event loop
- Users and posts as related tables, so a post knows its author and a user knows its posts
- Static file serving mounted at `/static`; user uploads live in S3, not on the app server
- PWA basics: web app manifest, favicons, touch icons, and theme color
- JSON API alongside the HTML pages, with interactive docs, covering full CRUD for posts and users
- API endpoints split into `APIRouter` modules under `routers/`, mounted with a path prefix
  and a tag, so `main.py` keeps only the page routes and app setup
- Post creating, editing, and deleting driven from the page with Bootstrap modals and
  `fetch` against the JSON API, with success and error modals for the result
- Shared front-end helpers in an ES module (`static/js/utils.js`), imported by the page
  scripts with `<script type="module">`
- Posts listed newest first everywhere, on the pages and in the API
- Registration and login with password hashing (argon2 via [pwdlib](https://pypi.org/project/pwdlib/))
  and JWT access tokens, with settings read from a `.env` file by pydantic-settings
- Ownership-based authorization: reads are public, writes need a bearer token, and editing or
  deleting something you do not own returns `403`
- Async test suite on `pytest` and `httpx`, with each test rolled back in its own transaction
  and S3 faked by `moto`, so nothing reaches AWS
- Two deployment paths: a VPS behind Nginx under systemd (`vps_setup.txt`), or a multi-stage
  Docker image for a serverless container platform
- Security headers on every response — frame and MIME-sniffing protection, a referrer policy,
  and HSTS once it is served from a real domain
- Avatar uploads processed with [Pillow](https://python-pillow.org/): cropped to a square
  300x300 JPEG, re-encoded, and stored in an [Amazon S3](https://aws.amazon.com/s3/) bucket
  with `boto3` under a random key
- Offset/limit pagination on the post listings, with a Load More button that appends the next
  page without a reload
- Password reset by emailed single-use token, sent with `aiosmtplib` from a FastAPI background
  task, plus an authenticated change-password endpoint
- Content-aware error handling: `/api/*` paths return JSON, page routes render an error template
- Pydantic schemas validating request bodies and shaping API responses

## Data model

Two tables, defined in `models.py`:

| Model  | Fields                                                  |
| ------ | ------------------------------------------------------- |
| `User` | `id`, `username` (unique), `email` (unique), `password_hash`, `image_file` |
| `Post` | `id`, `title`, `content`, `user_id` → `users.id`, `date_posted` |
| `PasswordResetToken` | `id`, `user_id` → `users.id`, `token_hash` (unique), `expires_at`, `created_at` |

`Post.author` and `User.posts` are the two sides of the relationship, configured with
`cascade="all, delete-orphan"` so deleting a user also deletes their posts. `User.image_path`
returns the user's uploaded avatar when one is set, and falls back to the default picture
in `static/profile_pics/` otherwise.

## Endpoints

### Pages

| Method | Path                    | Description                          |
| ------ | ----------------------- | ------------------------------------ |
| `GET`  | `/`                     | Home page, lists all posts           |
| `GET`  | `/posts`                | Same handler as `/`                  |
| `GET`  | `/posts/{post_id}`      | Single post page                     |
| `GET`  | `/users/{user_id}/posts`| Posts written by one user            |
| `GET`  | `/login`                | Login form                           |
| `GET`  | `/register`             | Registration form                    |
| `GET`  | `/account`              | Account settings for the logged-in user |
| `GET`  | `/forgot-password`      | Form to request a reset link, linked from `/login` |
| `GET`  | `/reset-password`       | Form to set a new password, reached from the email |

### JSON API

| Method   | Path                         | Auth       | Description                              |
| -------- | ---------------------------- | ---------- | ---------------------------------------- |
| `GET`    | `/api/posts`                 | public     | Page of posts, newest first              |
| `POST`   | `/api/posts`                 | token      | Create a post as yourself, returns `201` |
| `GET`    | `/api/posts/{post_id}`       | public     | Single post                              |
| `PUT`    | `/api/posts/{post_id}`       | **owner**  | Replace a post, every field required     |
| `PATCH`  | `/api/posts/{post_id}`       | **owner**  | Update only the fields that are sent     |
| `DELETE` | `/api/posts/{post_id}`       | **owner**  | Delete a post, returns `204`             |
| `POST`   | `/api/users`                 | public     | Register a user, returns `201`           |
| `POST`   | `/api/users/token`           | public     | Log in with form data, returns a JWT     |
| `GET`    | `/api/users/me`              | token      | The authenticated user                   |
| `GET`    | `/health`                    | public     | Liveness probe, `503` if the DB is down  |
| `PATCH`  | `/api/users/me/password`     | token      | Change password, needs the current one   |
| `POST`   | `/api/users/forgot-password` | public     | Request a reset link, always `202`       |
| `POST`   | `/api/users/reset-password`  | public     | Set a new password using a reset token   |
| `GET`    | `/api/users/{user_id}`       | public     | Single user                              |
| `GET`    | `/api/users/{user_id}/posts` | public     | Page of posts by one user                |
| `PATCH`  | `/api/users/{user_id}/picture` | **owner** | Upload an avatar (multipart)           |
| `DELETE` | `/api/users/{user_id}/picture` | **owner** | Remove the avatar, back to the default |
| `PATCH`  | `/api/users/{user_id}`       | **owner**  | Update only the fields that are sent     |
| `DELETE` | `/api/users/{user_id}`       | **owner**  | Delete a user and their posts, `204`     |

`public` needs nothing, `token` needs any valid bearer token, and **owner** additionally
requires that the token belongs to the post's author or to that same account.

The page routes are excluded from the OpenAPI schema, so `/docs` shows only the `/api` routes,
grouped into `users` and `posts` sections by the tag each router is included with.

The paths above are the full URLs. Inside the router modules the paths are written relative
to the prefix given in `main.py` — `@router.get("/{post_id}")` in `routers/posts.py` becomes
`/api/posts/{post_id}` because the router is included with `prefix="/api/posts"`.

Creating a user rejects a duplicate `username` or `email` with `400`. Creating a post no
longer takes a `user_id` — the author is the token's owner, so a `user_id` sent in the body
is simply ignored rather than honoured.

`PUT` is a full replacement: `title` and `content` both have to be sent. It cannot move a
post to a different author, since the author is fixed at creation. `PATCH` takes a partial body — `PostUpdate` and
`UserUpdate` make every field optional, and only the fields present in the request are
written. Updating a user to a `username` or `email` that another user already has returns
`400`. Both deletes return `204` with an empty body, and `404` for an unknown id.

### Async and eager loading

`database.py` builds a `create_async_engine` from `settings.database_url` — a
`postgresql+psycopg://` URL — and an `async_sessionmaker`, and `get_db` yields an
`AsyncSession`. Routes are `async def` and
`await` every query, commit, and refresh.

Two consequences worth knowing about, since both raise `MissingGreenlet` if ignored:

- A relationship can no longer lazy-load during response serialization or template
  rendering, so any query whose result exposes `post.author` selects it up front with
  `selectinload(models.Post.author)`. After a write, `db.refresh(post, attribute_names=["author"])`
  does the same job.
- The session maker sets `expire_on_commit=False`, so attributes stay readable after
  `await db.commit()` without another round trip.

The `lifespan` context manager no longer creates tables; it only disposes the engine on
shutdown. Alembic owns the schema now — see below.

### Authentication

Registration takes a `password` (8-50 characters), hashes it with argon2 through `pwdlib`,
and stores only the hash in `User.password_hash`. Usernames and emails are compared
case-insensitively when checking for duplicates, and emails are stored lowercased.

`POST /api/users/token` is the login endpoint. It takes `OAuth2PasswordRequestForm`, so the
body is form-encoded rather than JSON, and the form's `username` field carries the email.
A wrong password and an unknown email both return the same `401` with
`"Incorrect email or password"`, so the response does not reveal which accounts exist.
On success it returns a JWT whose `sub` is the user id.

`GET /api/users/me` reads the `Authorization: Bearer <token>` header and returns the token's
user. An invalid signature, an expired token, a `sub` that is not an integer, and a `sub`
pointing at a deleted user all return `401`.

`auth.py` holds the hashing and token helpers, and `config.py` reads `SECRET_KEY`,
`ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES`, `MAX_UPLOAD_SIZE_BYTES`, `POSTS_PER_PAGE`,
`RESET_TOKEN_EXPIRE_MINUTES`, the `MAIL_*` and `S3_*` groups, and `FRONTEND_URL` from `.env`
via pydantic-settings. `DATABASE_URL`, `SECRET_KEY`, and `S3_BUCKET_NAME` have no defaults, so
the app will not start until all three are set. Copy
`.env.example` to `.env` and set `SECRET_KEY` before starting the app — it has no default,
so `Settings()` fails without it. Changing it invalidates every token already issued.

In the browser, the token goes into `localStorage`. `static/js/auth.js` owns that: it caches
the `/api/users/me` lookup, de-duplicates concurrent calls, drops a token the API rejects,
and provides `logout`. `layout.html` uses it to swap the navbar between the logged-out
(Login / Register) and logged-in (New Post / Logout) states on every page.

### Migrations

The app used to call `Base.metadata.create_all` on startup, which is fine for a single
developer and wrong for anything else: it creates tables that do not exist but never alters
one that has changed, so a new column on an existing database silently never appears.

Alembic replaces it. `alembic/versions/` holds the ordered revisions, each with an `upgrade`
and a `downgrade`, and the database records which revision it is on in an `alembic_version`
table. `alembic/env.py` is wired for the async engine and pulls the URL from
`settings.database_url`, so migrations and the app always agree on which database they mean.

```bash
uv run alembic upgrade head             # apply everything outstanding
uv run alembic current                  # which revision this database is on
uv run alembic history                  # the full chain
uv run alembic downgrade -1             # step back one revision
uv run alembic revision --autogenerate -m "add something"
```

`--autogenerate` diffs `models.py` against the live database and writes the difference into a
new revision. Read what it produces before applying it — it detects added and dropped columns
well, but it cannot see a rename, which it emits as a drop plus an add, and that loses data.

The `likes` column on `Post` is the worked example: it exists as both a model field and a
revision, added with `server_default="0"` so the column can be `NOT NULL` on a table that
already has rows.

### Pagination

The two listing endpoints take `skip` and `limit` query parameters, validated by FastAPI —
`skip` must be `>= 0` and `limit` between `1` and `100`, so a bad value returns `422` rather
than a huge query. Both default to `limit=10`.

They no longer return a bare list. The response is an object, which is what makes a Load More
button possible — the client needs to know whether to keep going:

```json
{ "posts": [...], "total": 26, "skip": 0, "limit": 10, "has_more": true }
```

`total` comes from a separate `COUNT(*)`, and `has_more` is `skip + len(posts) < total`.

The page routes render the first `settings.posts_per_page` posts server-side and pass `limit`
and `has_more` into the template. The button only renders when `has_more` is true, and the
script picks up from there, appending each page and hiding the button once `has_more` comes
back false. Rows added by JavaScript are built with `escapeHtml` from `utils.js`, since a
post title goes into a template literal rather than through Jinja's autoescaping.

### Password reset

`POST /api/users/forgot-password` always returns `202` with the same message whether or not
the address belongs to an account, so the endpoint cannot be used to discover which emails are
registered. The work only happens when a user actually matches.

The token itself is a `secrets.token_urlsafe(32)` value. Only its SHA-256 hash goes into
`password_reset_tokens` — the raw token exists in the email and nowhere else, so a leaked
database does not hand over working reset links. Requesting a new link deletes any previous
token for that user, so only the newest one works.

Sending happens through `BackgroundTasks`, so the response returns immediately instead of
waiting on the SMTP conversation. `email_utils.py` builds a multipart message with both a
plain-text and an HTML part (rendered from `templates/email/password_reset.html`) and sends it
with `aiosmtplib`.

`POST /api/users/reset-password` hashes the submitted token, looks it up, and rejects an
unknown or expired one with the same `400` either way. Expired rows are deleted when found.
On success the password is rehashed and **every** reset token for that user is deleted, so a
token cannot be replayed.

`PATCH /api/users/me/password` is the logged-in path. It verifies the current password before
accepting a new one, and also clears any outstanding reset tokens — so if someone requested a
reset link and then remembered their password, the emailed link stops working.

On the page side, `/forgot-password` collects the address and `/reset-password` reads the
token out of its own query string, so the token never goes into a form field or a template
variable. That page also sends `Referrer-Policy: no-referrer`, which matters because the token
sits in the URL — without it the browser would leak the full address to the CDNs the layout
loads Bootstrap and fonts from. Opening it without a token disables the submit button, and
the confirm field is checked before anything is sent.

The mail settings (`MAIL_SERVER`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM`,
`MAIL_USE_TLS`) and `FRONTEND_URL`, which builds the link in the email, all come from `.env`.
The defaults point at `localhost:587`, so for local development run a debug SMTP server such
as [MailHog](https://github.com/mailhog/MailHog) or Python's `aiosmtpd` and watch the mail
arrive there.

### File uploads

> **Requires an S3 bucket.** `S3_BUCKET_NAME` has no default, so the app will not start until
> it is set. See [Setting up S3](#setting-up-s3) below for the bucket, IAM user, and policies
> you need to create in AWS first.

`PATCH /api/users/{id}/picture` takes a multipart upload and hands it to `image_utils.py`,
which does the processing in a thread pool so the event loop is not blocked by Pillow:

- `ImageOps.exif_transpose` applies the EXIF orientation, so phone photos are not sideways
- `ImageOps.fit` crops and scales to a square 300x300 with LANCZOS resampling
- `RGBA`, `LA`, and `P` images are converted to `RGB` so they can be saved as JPEG
- The result is encoded to JPEG at quality 85 **in memory** and uploaded to
  `s3://<bucket>/profile_pics/{uuid4}.jpg`

Nothing is written to the app's own disk. That is the point of the move: the server keeps no
local state, so it can be redeployed or run behind several instances without avatars living
on whichever machine happened to receive the upload.

`boto3` is synchronous, so the upload and delete calls are wrapped in `run_in_threadpool` the
same way the Pillow work is, keeping the event loop free.

Storing a random key rather than the uploaded filename means the original name never reaches
S3, so there is no path traversal to worry about, and every upload gets a unique URL that
sidesteps caching of a replaced avatar. Uploads over `max_upload_size_bytes` (5MB by default)
are rejected with `400`, and the size and format checks both run before anything is sent to
S3. A failed upload is reported as a `500` with a plain message rather than a stack trace, and
leaves the existing avatar in place.

The old object is deleted after a successful replace, on `DELETE .../picture`, and when the
account itself is deleted, so orphans do not accumulate.

`User.image_path` returns the public object URL,
`https://<bucket>.s3.<region>.amazonaws.com/profile_pics/<key>`, which is why the bucket needs
a policy allowing public reads on that prefix. Users without an avatar still fall back to
`/static/profile_pics/default.jpg`, which is served by the app.

### Setting up S3

Do this in AWS **before** running the app — there is no default bucket and nothing is created
for you.

1. **Create the bucket.** Any name and region; the name goes in `S3_BUCKET_NAME` and the
   region in `S3_REGION`. Uploads are stored under a `profile_pics/` prefix.

2. **Allow public reads on that prefix.** Avatars are loaded straight from S3 by the browser.
   Turn off "Block all public access" for the bucket, then apply `aws_bucket_policy.json`
   (Permissions → Bucket policy), replacing `fastapi-blog-uploads` with your bucket name:

   ```json
   {
     "Effect": "Allow",
     "Principal": "*",
     "Action": "s3:GetObject",
     "Resource": "arn:aws:s3:::fastapi-blog-uploads/profile_pics/*"
   }
   ```

   Note this grants read access to anyone with the URL. Keys are random UUIDs, so they are not
   guessable, but they are not private either — fine for avatars, not for anything sensitive.

3. **Create an IAM user for the app** and attach `aws_iam_policy.json`, which grants only
   `s3:PutObject` and `s3:DeleteObject`, and only under `profile_pics/`. The app never needs
   to read objects back or touch the rest of the bucket, so it is not granted either.

4. **Give the app credentials.** Either put the IAM user's access key in `S3_ACCESS_KEY_ID`
   and `S3_SECRET_ACCESS_KEY`, or leave both unset and let boto3 pick up the standard sources
   — `~/.aws/credentials`, environment variables, or an instance role in production, which is
   the better option there since it avoids long-lived keys entirely.

Then check the wiring before starting the app:

```bash
uv run python check_s3.py
```

It uploads a small test object and deletes it again, printing what failed if anything did.
`S3_ENDPOINT_URL` exists for pointing at an S3-compatible service such as MinIO or LocalStack
instead of AWS; leave it unset for real S3.

### Authorization

`auth.py` exposes `get_current_user` as a dependency, aliased to `CurrentUser`, which
resolves the bearer token to a `User` row or raises `401`. Any endpoint that declares a
`current_user: CurrentUser` parameter is authenticated by that alone — FastAPI resolves the
dependency before the handler body runs.

Ownership is then checked in the handler:

```python
if post.user_id != current_user.id:
    raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, ...)
```

The two failures are distinct on purpose: `401` means no usable token, `403` means a valid
token for the wrong account. Because the dependency runs first, an unauthenticated request
for a post that does not exist returns `401`, not `404` — the request never reaches the
lookup.

Reads stayed public. Only the write endpoints gained `CurrentUser`, so the home page, single
posts, and user profiles still work signed out.

`PostCreate` no longer carries `user_id`; `create_post` takes it from `current_user.id`. That
removes the spoofing question entirely rather than validating against it — a `user_id` in the
request body is not a field on the model, so it is discarded before the handler sees it.

### Front end

The pages are server-rendered, but the write operations are done from the browser against
the JSON API rather than through form POSTs and redirects:

| Action         | Where it lives                          | Request                     |
| -------------- | --------------------------------------- | --------------------------- |
| New post       | Navbar button, modal in `layout.html`   | `POST /api/posts`           |
| Edit post      | Buttons on the post page, `post.html`   | `PATCH /api/posts/{id}`     |
| Delete post    | Buttons on the post page, `post.html`   | `DELETE /api/posts/{id}`    |
| Update profile | `account.html`                          | `PATCH /api/users/{id}`     |
| Delete account | Danger zone in `account.html`           | `DELETE /api/users/{id}`    |

Every one of those attaches `Authorization: Bearer <token>` from `localStorage`. A `401`
response sends the browser to `/login`; a `403` shows the error modal.

The create modal lives in `layout.html`, so the New Post button works from any page.
`post.html` adds its own edit and delete modals and fills a `{% block scripts %}` that
`layout.html` renders at the end of `<body>`.

`static/js/utils.js` holds the three helpers both scripts share: `getErrorMessage` unpacks
either shape the API returns — a plain `{"detail": "Post not found"}` string or the list of
objects a `422` produces — and `showModal` / `hideModal` wrap the Bootstrap modal instances.
It is a real ES module, so the scripts that use it are `<script type="module">`.

On success the page swaps the form modal for the success modal and reloads once it is
dismissed; a delete redirects home instead. A failed request shows the message in the error
modal, and a network failure falls back to a generic one.

`post.html` renders the edit and delete buttons hidden, and `checkOwnership()` reveals them
only when `getCurrentUser()` comes back as the post's author. That is presentation only — the
`403` from the API is the actual protection, and it holds whether or not the buttons are on
screen.

`/account` is the logged-in user's own page: it fills the form from `/api/users/me`, sends a
`PATCH` on save, clears the cached user so the navbar picks up a rename, and holds the logout
button plus a delete-account danger zone. It redirects to `/login` when there is no token.

### Error handling

Exception handlers are registered for `HTTPException` and `RequestValidationError`, and
they branch on the request path:

- Requests under `/api` get a JSON body, e.g. `{"detail": "Post not found"}`
- Everything else renders `error.html` inside the normal site layout

An unknown id returns `404`; a non-integer id fails path validation and returns `422`.

## Requirements

- Python 3.13 (see `.python-version`)
- [uv](https://docs.astral.sh/uv/) for dependency management
- PostgreSQL, running and reachable at whatever `DATABASE_URL` points to
- An AWS S3 bucket and credentials that can write to it — see [Setting up S3](#setting-up-s3)

Jinja2 comes in via the `fastapi[standard]` extra, so no separate install is needed.

## Running it locally

This section is the **development** setup: the app served by `fastapi dev` on your own
machine, with settings in a local `.env`. For putting it on a server, see
[Deployment](#deployment).

Install dependencies:

```bash
uv sync
```

Create the database:

```bash
createdb blog
```

Set up the S3 bucket, IAM user, and policies in AWS — the app has no default bucket and will
not start without one. The steps are in [Setting up S3](#setting-up-s3).

Create your `.env` from the template, then fill in the three settings that have no default —
`DATABASE_URL`, `SECRET_KEY`, and `S3_BUCKET_NAME`:

```bash
cp .env.example .env
python -c "import secrets; print(secrets.token_hex(32))"   # paste into SECRET_KEY
```

Confirm the bucket is reachable before going further:

```bash
uv run python check_s3.py
```

Apply the migrations. Nothing creates tables at runtime any more, so this step is required
before the first start:

```bash
uv run alembic upgrade head
```

Run the development server:

```bash
uv run fastapi dev main.py
```

The tables start empty. Register, log in to get a token, then post as that user:

```bash
curl -X POST http://127.0.0.1:8000/api/users \
  -H 'Content-Type: application/json' \
  -d '{"username": "alice", "email": "alice@example.com", "password": "supersecret1"}'

TOKEN=$(curl -s -X POST http://127.0.0.1:8000/api/users/token \
  -d 'username=alice@example.com&password=supersecret1' | python -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')

curl -X POST http://127.0.0.1:8000/api/posts \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title": "Hello", "content": "First post."}'
```

The login endpoint takes form-encoded data, not JSON, and its `username` field is your email.
Or just use `/docs` — the Authorize button drives the same flow.

To get a database with enough content to see pagination working, run the seed script instead:

```bash
uv run python populate_db.py
```

It creates demo users with avatars from `populate_images/` and enough posts to fill several
pages, driving the real API so the avatars go through the same processing and S3 upload path
as a normal request.

It **wipes first**: every post, user, and reset token is deleted, along with the avatar
objects those users pointed at in S3. That makes it safe to re-run, and unsafe to point at
anything you care about — check `DATABASE_URL` and `S3_BUCKET_NAME` before running it.

The app is then available at:

- http://127.0.0.1:8000 — home page
- http://127.0.0.1:8000/api/posts — JSON API
- http://127.0.0.1:8000/docs — interactive Swagger UI
- http://127.0.0.1:8000/redoc — ReDoc

## Deployment

Two supported paths. A VPS gives you one machine you administer; a container gives you an
artefact a platform runs for you, scaled to zero when idle. The application code is the same
either way — everything that differs is configuration.

### On a VPS

`vps_setup.txt` is the full runbook for putting this on an Ubuntu 24.04 VPS — every command
in order, from a fresh server to a live HTTPS site. In outline:

| Stage | What it covers |
| --- | --- |
| Server hardening | Non-root user, SSH keys, password auth disabled, UFW, Fail2Ban, unattended security upgrades |
| Nginx and TLS | Nginx as a reverse proxy, DNS pointed at the box, a Let's Encrypt certificate with automatic renewal |
| The application | Python and uv, PostgreSQL, the repository, `.env` at `chmod 600`, `alembic upgrade head` |
| Process supervision | A systemd unit running `fastapi run` bound to `127.0.0.1`, so only Nginx can reach it |

A few points where the deployment shapes the application rather than the other way round:

- **Uvicorn binds to `127.0.0.1`, not `0.0.0.0`.** Nginx is the only thing that talks to it,
  and the firewall never needs port 8000 open.
- **`--proxy-headers`** tells Uvicorn to trust the `X-Forwarded-*` headers Nginx sets, so the
  app sees the real client IP and knows the original request was HTTPS.
- **`FRONTEND_URL` must become the real `https://` domain.** It builds the link inside the
  password reset email, so a stale `localhost` value sends users somewhere that does not exist.
- **Settings come from a `.env` file that is created on the server**, never copied from a
  development machine and never committed. `.env.example` remains the list of what to fill in.
- **Migrations are a deploy step.** Nothing creates tables at runtime, so
  `alembic upgrade head` runs on the server after each pull.

### In a container

The `Dockerfile` is a two-stage build. The first stage installs dependencies with `uv` and
the second copies only the result, so the toolchain never ships in the running image:

```bash
docker build -t fastapi-blog .
docker run --rm -p 8080:8080 --env-file .env fastapi-blog
```

What each part is doing, and why it matters on a serverless platform:

- **Dependencies install before the source is copied**, so a code change reuses the cached
  dependency layer instead of resolving the lockfile again.
- **`uv sync --locked --no-dev`** installs exactly what `uv.lock` pins and leaves `pytest`
  and `moto` out of the image.
- **The container runs as `appuser`, not root**, which is a requirement on several platforms
  and good practice everywhere else.
- **The port is read from `$PORT`** rather than hardcoded. Serverless platforms assign a port
  and expect the process to listen on it; `8080` is only the fallback.
- **`--proxy-headers --forwarded-allow-ips '*'`** does the same job the Nginx setup needs,
  since the platform's load balancer terminates TLS and forwards over plain HTTP.
- **`exec` in the `CMD`** replaces the shell, so `SIGTERM` reaches the app and shutdown is
  clean rather than a ten-second kill.

Configuration arrives as environment variables set on the platform, not from a `.env` file
in the image. `.env.example` is still the list of what to set. Migrations do not run at
startup, so `alembic upgrade head` needs to happen as a release step or a one-off job before
new code serves traffic.

`.dockerignore` is what keeps that honest. `COPY . ./` would otherwise copy the whole project
directory into the image — `.env` included — so a published image would carry the secret key,
database URL, and S3 credentials to whoever can pull it. It also drops `.venv`, `.git`,
`tests/`, and `populate_images/`, which takes the build context from roughly 200MB to under
1MB. Two entries are less obvious than the rest:

- **`.python-version` is ignored on purpose.** It pins 3.13, but the base image chooses the
  interpreter and `UV_PYTHON_DOWNLOADS=0` means uv cannot fetch a different one.
- **`README.md` is deliberately kept.** `pyproject.toml` declares `readme = "README.md"`, so
  the project build reads it and fails without it.

### Security headers

A middleware sets `X-Frame-Options`, `X-Content-Type-Options`, and a default
`Referrer-Policy` on every response, including static files and error pages.
`Strict-Transport-Security` is added too, but only when the request host is not `localhost`
or `127.0.0.1` — so local development over plain HTTP is not pinned to HTTPS by a stale
header. The stricter `no-referrer` that `/reset-password` sets for itself is left alone.

### Health check

`GET /health` runs `SELECT 1` and returns `{"status": "healthy"}`, or `503` if the database
does not answer. It is a liveness probe for a load balancer, uptime monitor, container
platform, or systemd watchdog — a plain `GET /` would also touch the database, but it renders
a full page to do it.

## Tests

```bash
uv sync --all-groups     # pytest and moto live in the dev group
uv run pytest
```

The suite needs **one thing set up: a PostgreSQL database for tests.** `tests/conftest.py`
connects to `postgresql+psycopg://bloguser:blogpass@localhost/test_blog`, so create that role
and database first:

```bash
psql -d postgres -c "CREATE ROLE bloguser LOGIN PASSWORD 'blogpass';"
createdb -O bloguser test_blog
```

It does **not** need AWS. `moto` intercepts the S3 calls and serves an in-memory bucket, and
the fixtures set dummy `AWS_*` credentials so a misconfigured test cannot reach a real
account. The same goes for email: `send_password_reset_email` is patched with an `AsyncMock`
and the test asserts on the arguments it was called with rather than sending anything.

It also does not need a `.env`. `conftest.py` sets every required setting in `os.environ`
before `config` is imported, so a fresh clone can run the suite without configuring the app
first.

How the fixtures fit together:

| Fixture | Scope | What it does |
| --- | --- | --- |
| `test_engine` | session | One async engine on the test database, `NullPool` so connections are not reused across the loop |
| `setup_database` | session | `create_all` before the run, `drop_all` after, leaving the database empty |
| `db_session` | function | Opens a connection and an outer transaction, binds the session to it with `join_transaction_mode="create_savepoint"`, and **rolls back afterwards** |
| `mocked_aws` | function | Starts `mock_aws` and creates the bucket, yielding the client so tests can assert on its contents |
| `client` | function | An `AsyncClient` over `ASGITransport`, with `get_db` overridden to hand out `db_session` |

The rollback in `db_session` is what keeps the tests independent: the app commits normally
inside its handlers, but those commits land in a savepoint that disappears when the outer
transaction rolls back. Tests can therefore assume an empty database without truncating
tables between runs, and the order they run in does not matter.

`conftest.py` also exposes `create_test_user`, `login_user`, and `auth_header`, so a test that
needs an authenticated client is three lines rather than a repeated registration and login.

## Project structure

```
.
├── main.py                  # FastAPI app: lifespan, page routes, mounts, exception handlers
├── routers/
│   ├── posts.py             # APIRouter for /api/posts
│   └── users.py             # APIRouter for /api/users
├── auth.py                  # Hashing, JWT helpers, and the CurrentUser dependency
├── email_utils.py           # aiosmtplib sending and the reset-email builder
├── image_utils.py           # Pillow processing plus S3 upload and delete
├── populate_db.py           # Wipes and reseeds demo users, posts, and avatars
├── config.py                # Settings loaded from .env by pydantic-settings
├── database.py              # Async engine, session factory, Base, get_db dependency
├── models.py                # SQLAlchemy ORM models: User and Post
├── schemas.py               # Pydantic models for request and response bodies
├── alembic.ini              # Alembic configuration
├── alembic/
│   ├── env.py               # Migration environment, async engine, URL from settings
│   ├── script.py.mako       # Template for generated revisions
│   └── versions/            # The ordered migration revisions
├── templates/
│   ├── layout.html          # Base template: navbar, sidebar, footer, theme toggle, create/result modals
│   ├── home.html            # Post list, extends layout.html
│   ├── post.html            # Single post page, with edit/delete modals and their scripts
│   ├── user_posts.html      # Posts belonging to one user
│   ├── login.html           # Login form
│   ├── register.html        # Registration form
│   ├── account.html         # Account settings, profile update, change password, delete account
│   ├── forgot_password.html # Request a reset link
│   ├── reset_password.html  # Set a new password, opened from the emailed link
│   ├── error.html           # Error page used by the exception handlers
│   └── email/
│       └── password_reset.html  # HTML body of the reset email
├── static/
│   ├── css/main.css         # Custom styles on top of Bootstrap
│   ├── js/utils.js          # ES module: modals, error messages, escapeHtml, formatDate
│   ├── js/auth.js           # ES module: token storage and current-user cache
│   ├── icons/               # Favicons, touch icons, PWA icons
│   ├── profile_pics/        # Default avatar
│   └── site.webmanifest     # PWA manifest
├── aws_bucket_policy.json   # Public-read policy for the profile_pics/ prefix
├── aws_iam_policy.json      # Least-privilege policy for the app's IAM user
├── check_s3.py              # Verifies the S3 credentials and bucket work
├── vps_setup.txt            # Step-by-step VPS deployment runbook
├── Dockerfile               # Two-stage build for container deployment
├── tests/
│   ├── conftest.py          # Fixtures: test engine, rolled-back session, moto S3, client
│   ├── test_users.py        # Registration, validation, avatar upload, password reset
│   ├── test_posts.py        # CRUD, authorization, pagination
│   └── test_image.jpg       # Small fixture image for the upload test
├── populate_images/         # Source images used by populate_db.py
├── .env.example             # Template for the .env file, safe to commit
├── pyproject.toml           # Project metadata and dependencies
└── uv.lock                  # Pinned dependency versions
```

The database lives in PostgreSQL rather than in the project directory, so a clone starts with
no schema at all until `alembic upgrade head` builds it.

## Roadmap

### Next

- [ ] Extract structured data from uploaded documents with the Claude API

<details>
<summary><strong>Covered so far</strong> — 18 items, from Jinja2 templates to VPS and container deploys</summary>

- [x] Render real templates with Jinja2 instead of inline HTML
- [x] Add a detail route for a single post (`/posts/{id}`)
- [x] Move posts into a database (SQLAlchemy ORM, later moved from SQLite to PostgreSQL)
- [x] Add Pydantic models for request/response validation
- [x] Support creating, updating, and deleting posts
- [x] Convert the app to async, with an async database driver
- [x] Split the API endpoints into routers
- [x] Add a browser UI for writing posts, rather than JSON-only endpoints
- [x] User accounts with real authentication, so the Login and Register buttons work
- [x] Take `user_id` from the logged-in user instead of a value in the request body
- [x] Authorization on the write endpoints, so only a post's author can edit or delete it
- [x] Avatar uploads, first onto local disk and then into an S3 bucket
- [x] Paginate the post listings instead of returning every row
- [x] Password reset over email, with single-use expiring tokens
- [x] Database migrations with Alembic, instead of `create_all` on startup
- [x] Add tests with `pytest` and `httpx`
- [x] Deploy to a VPS: Nginx as a reverse proxy, HTTPS via Let's Encrypt, a custom
      domain, and basic server hardening
- [x] Containerise with Docker and deploy to a serverless container platform, with
      the same custom domain

</details>
