# fastapi-blog

A small blog application built with [FastAPI](https://fastapi.tiangolo.com/) — a learning
project for working through the framework's routing, Jinja2 templating, static files,
database access, and automatic API docs.

Posts and users are stored in a local SQLite database, reached asynchronously through
SQLAlchemy's async engine and `aiosqlite`. Every route is an `async def` coroutine. The
database file is created on startup and is not tracked in git.

Writing, editing, and deleting posts happens in the browser: Bootstrap modals collect the
input and `fetch` calls talk to the same JSON API the docs expose, so no page does a
classic form POST.

## Features

- Server-rendered pages with Jinja2 template inheritance (`layout.html` → `home.html`)
- Responsive layout styled with [Bootstrap 5](https://getbootstrap.com/) plus a custom stylesheet
- Light / dark / auto theme switcher, with the choice persisted in `localStorage`
- SQLite persistence through [SQLAlchemy](https://www.sqlalchemy.org/) ORM models, with an
  `AsyncSession` injected into routes as a dependency
- Fully asynchronous request handling: `async def` routes awaiting non-blocking queries, so
  a slow database call does not tie up the event loop
- Users and posts as related tables, so a post knows its author and a user knows its posts
- Static file serving mounted at `/static`, uploaded media at `/media`
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
- Content-aware error handling: `/api/*` paths return JSON, page routes render an error template
- Pydantic schemas validating request bodies and shaping API responses

## Data model

Two tables, defined in `models.py`:

| Model  | Fields                                                  |
| ------ | ------------------------------------------------------- |
| `User` | `id`, `username` (unique), `email` (unique), `password_hash`, `image_file` |
| `Post` | `id`, `title`, `content`, `user_id` → `users.id`, `date_posted` |

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

### JSON API

| Method   | Path                         | Auth       | Description                              |
| -------- | ---------------------------- | ---------- | ---------------------------------------- |
| `GET`    | `/api/posts`                 | public     | All posts, newest first                  |
| `POST`   | `/api/posts`                 | token      | Create a post as yourself, returns `201` |
| `GET`    | `/api/posts/{post_id}`       | public     | Single post                              |
| `PUT`    | `/api/posts/{post_id}`       | **owner**  | Replace a post, every field required     |
| `PATCH`  | `/api/posts/{post_id}`       | **owner**  | Update only the fields that are sent     |
| `DELETE` | `/api/posts/{post_id}`       | **owner**  | Delete a post, returns `204`             |
| `POST`   | `/api/users`                 | public     | Register a user, returns `201`           |
| `POST`   | `/api/users/token`           | public     | Log in with form data, returns a JWT     |
| `GET`    | `/api/users/me`              | token      | The authenticated user                   |
| `GET`    | `/api/users/{user_id}`       | public     | Single user                              |
| `GET`    | `/api/users/{user_id}/posts` | public     | Posts written by one user                |
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

`database.py` builds a `create_async_engine` over the `sqlite+aiosqlite` driver and an
`async_sessionmaker`, and `get_db` yields an `AsyncSession`. Routes are `async def` and
`await` every query, commit, and refresh.

Two consequences worth knowing about, since both raise `MissingGreenlet` if ignored:

- A relationship can no longer lazy-load during response serialization or template
  rendering, so any query whose result exposes `post.author` selects it up front with
  `selectinload(models.Post.author)`. After a write, `db.refresh(post, attribute_names=["author"])`
  does the same job.
- The session maker sets `expire_on_commit=False`, so attributes stay readable after
  `await db.commit()` without another round trip.

Table creation moved out of import time into a `lifespan` context manager, which runs
`create_all` through `run_sync` on startup and disposes the engine on shutdown.

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
`ALGORITHM`, and `ACCESS_TOKEN_EXPIRE_MINUTES` from `.env` via pydantic-settings. Copy
`.env.example` to `.env` and set `SECRET_KEY` before starting the app — it has no default,
so `Settings()` fails without it. Changing it invalidates every token already issued.

In the browser, the token goes into `localStorage`. `static/js/auth.js` owns that: it caches
the `/api/users/me` lookup, de-duplicates concurrent calls, drops a token the API rejects,
and provides `logout`. `layout.html` uses it to swap the navbar between the logged-out
(Login / Register) and logged-in (New Post / Logout) states on every page.

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
Avatar upload and password change are stubbed out as disabled fields for later.

### Error handling

Exception handlers are registered for `HTTPException` and `RequestValidationError`, and
they branch on the request path:

- Requests under `/api` get a JSON body, e.g. `{"detail": "Post not found"}`
- Everything else renders `error.html` inside the normal site layout

An unknown id returns `404`; a non-integer id fails path validation and returns `422`.

## Requirements

- Python 3.13 (see `.python-version`)
- [uv](https://docs.astral.sh/uv/) for dependency management

Jinja2 comes in via the `fastapi[standard]` extra, so no separate install is needed.

## Getting started

Install dependencies:

```bash
uv sync
```

Create your `.env` from the template and give it a secret key:

```bash
cp .env.example .env
python -c "import secrets; print(secrets.token_hex(32))"   # paste into SECRET_KEY
```

Run the development server:

```bash
uv run fastapi dev main.py
```

`blog.db` is created automatically on first start, with empty tables. Register, log in to get
a token, then post as that user:

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

The app is then available at:

- http://127.0.0.1:8000 — home page
- http://127.0.0.1:8000/api/posts — JSON API
- http://127.0.0.1:8000/docs — interactive Swagger UI
- http://127.0.0.1:8000/redoc — ReDoc

## Project structure

```
.
├── main.py                  # FastAPI app: lifespan, page routes, mounts, exception handlers
├── routers/
│   ├── posts.py             # APIRouter for /api/posts
│   └── users.py             # APIRouter for /api/users
├── auth.py                  # Hashing, JWT helpers, and the CurrentUser dependency
├── config.py                # Settings loaded from .env by pydantic-settings
├── database.py              # Async engine, session factory, Base, get_db dependency
├── models.py                # SQLAlchemy ORM models: User and Post
├── schemas.py               # Pydantic models for request and response bodies
├── templates/
│   ├── layout.html          # Base template: navbar, sidebar, footer, theme toggle, create/result modals
│   ├── home.html            # Post list, extends layout.html
│   ├── post.html            # Single post page, with edit/delete modals and their scripts
│   ├── user_posts.html      # Posts belonging to one user
│   ├── login.html           # Login form
│   ├── register.html        # Registration form
│   ├── account.html         # Account settings, profile update, delete account
│   └── error.html           # Error page used by the exception handlers
├── static/
│   ├── css/main.css         # Custom styles on top of Bootstrap
│   ├── js/utils.js          # ES module: error message + modal helpers
│   ├── js/auth.js           # ES module: token storage and current-user cache
│   ├── icons/               # Favicons, touch icons, PWA icons
│   ├── profile_pics/        # Default avatar
│   └── site.webmanifest     # PWA manifest
├── media/profile_pics/      # Uploaded avatars, ignored by git
├── .env.example             # Template for the .env file, safe to commit
├── pyproject.toml           # Project metadata and dependencies
└── uv.lock                  # Pinned dependency versions
```

`blog.db` sits in the project root at runtime and is gitignored, so each clone starts with
its own empty database.

## Roadmap

Ideas to build on as the project grows:

- [x] Render real templates with Jinja2 instead of inline HTML
- [x] Add a detail route for a single post (`/posts/{id}`)
- [x] Move posts into a database (SQLite via SQLAlchemy)
- [x] Add Pydantic models for request/response validation
- [x] Support creating, updating, and deleting posts
- [x] Convert the app to async, with an async database driver
- [x] Split the API endpoints into routers
- [x] Add a browser UI for writing posts, rather than JSON-only endpoints
- [x] User accounts with real authentication, so the Login and Register buttons work
- [x] Take `user_id` from the logged-in user instead of a value in the request body
- [x] Authorization on the write endpoints, so only a post's author can edit or delete it
- [ ] Avatar uploads writing into `media/profile_pics/`
- [ ] Database migrations with Alembic, instead of `create_all` on startup
- [ ] Add tests with `pytest` and `httpx`
