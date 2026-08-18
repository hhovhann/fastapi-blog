# fastapi-blog

A small blog application built with [FastAPI](https://fastapi.tiangolo.com/) — a learning
project for working through the framework's routing, Jinja2 templating, static files,
database access, and automatic API docs.

Posts and users are stored in a local SQLite database via SQLAlchemy. The database file is
created on startup and is not tracked in git.

## Features

- Server-rendered pages with Jinja2 template inheritance (`layout.html` → `home.html`)
- Responsive layout styled with [Bootstrap 5](https://getbootstrap.com/) plus a custom stylesheet
- Light / dark / auto theme switcher, with the choice persisted in `localStorage`
- SQLite persistence through [SQLAlchemy](https://www.sqlalchemy.org/) ORM models, with the
  session injected into routes as a dependency
- Users and posts as related tables, so a post knows its author and a user knows its posts
- Static file serving mounted at `/static`, uploaded media at `/media`
- PWA basics: web app manifest, favicons, touch icons, and theme color
- JSON API alongside the HTML pages, with interactive docs
- Content-aware error handling: `/api/*` paths return JSON, page routes render an error template
- Pydantic schemas validating request bodies and shaping API responses

## Data model

Two tables, defined in `models.py`:

| Model  | Fields                                                  |
| ------ | ------------------------------------------------------- |
| `User` | `id`, `username` (unique), `email` (unique), `image_file` |
| `Post` | `id`, `title`, `content`, `user_id` → `users.id`, `date_posted` |

`Post.author` and `User.posts` are the two sides of the relationship. `User.image_path`
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

### JSON API

| Method | Path                   | Description                                     |
| ------ | ---------------------- | ----------------------------------------------- |
| `GET`  | `/api/posts`           | All posts                                       |
| `POST` | `/api/posts`           | Create a post, returns `201`                    |
| `GET`  | `/api/posts/{post_id}` | Single post                                     |
| `POST` | `/api/users`           | Create a user, returns `201`                    |
| `GET`  | `/api/users/{user_id}` | Single user                                     |

The page routes are excluded from the OpenAPI schema, so `/docs` shows only the `/api` routes.

Creating a user rejects a duplicate `username` or `email` with `400`. Creating a post
requires a `user_id` that exists, and returns `404` when it does not.

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

Run the development server:

```bash
uv run fastapi dev main.py
```

`blog.db` is created automatically on first start, with empty tables. Create a user before
posting, since a post needs an existing `user_id`:

```bash
curl -X POST http://127.0.0.1:8000/api/users \
  -H 'Content-Type: application/json' \
  -d '{"username": "alice", "email": "alice@example.com"}'

curl -X POST http://127.0.0.1:8000/api/posts \
  -H 'Content-Type: application/json' \
  -d '{"title": "Hello", "content": "First post.", "user_id": 1}'
```

The app is then available at:

- http://127.0.0.1:8000 — home page
- http://127.0.0.1:8000/api/posts — JSON API
- http://127.0.0.1:8000/docs — interactive Swagger UI
- http://127.0.0.1:8000/redoc — ReDoc

## Project structure

```
.
├── main.py                  # FastAPI app: routes, mounts, exception handlers
├── database.py              # Engine, session factory, Base, get_db dependency
├── models.py                # SQLAlchemy ORM models: User and Post
├── schemas.py               # Pydantic models for request and response bodies
├── templates/
│   ├── layout.html          # Base template: head, navbar, sidebar, footer, theme toggle
│   ├── home.html            # Post list, extends layout.html
│   ├── post.html            # Single post page, with edit/delete actions
│   ├── user_posts.html      # Posts belonging to one user
│   └── error.html           # Error page used by the exception handlers
├── static/
│   ├── css/main.css         # Custom styles on top of Bootstrap
│   ├── js/utils.js          # Placeholder for shared scripts
│   ├── icons/               # Favicons, touch icons, PWA icons
│   ├── profile_pics/        # Default avatar
│   └── site.webmanifest     # PWA manifest
├── media/profile_pics/      # Uploaded avatars, ignored by git
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
- [ ] Support creating, updating, and deleting posts
- [ ] Add HTML forms for writing posts, rather than JSON-only endpoints
- [ ] User accounts with real authentication, so the Login and Register buttons work,
      and `user_id` comes from the session instead of the request body
- [ ] Avatar uploads writing into `media/profile_pics/`
- [ ] Database migrations with Alembic, instead of `create_all` on startup
- [ ] Add tests with `pytest` and `httpx`
