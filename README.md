# fastapi-blog

A small blog application built with [FastAPI](https://fastapi.tiangolo.com/) — a learning
project for working through the framework's routing, Jinja2 templating, static files, and
automatic API docs.

Posts currently live in an in-memory list in `main.py`; there is no database yet.

## Features

- Server-rendered pages with Jinja2 template inheritance (`layout.html` → `home.html`)
- Responsive layout styled with [Bootstrap 5](https://getbootstrap.com/) plus a custom stylesheet
- Light / dark / auto theme switcher, with the choice persisted in `localStorage`
- Static file serving mounted at `/static`
- PWA basics: web app manifest, favicons, touch icons, and theme color
- JSON API alongside the HTML pages, with interactive docs

## Endpoints

| Method | Path         | Response | Description                          |
| ------ | ------------ | -------- | ------------------------------------ |
| `GET`  | `/`          | HTML     | Home page, renders `home.html`       |
| `GET`  | `/posts`     | HTML     | Same handler as `/`                  |
| `GET`  | `/api/posts` | JSON     | All posts                            |

The HTML routes are excluded from the OpenAPI schema, so `/docs` shows only `/api/posts`.

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

The app is then available at:

- http://127.0.0.1:8000 — home page
- http://127.0.0.1:8000/api/posts — JSON API
- http://127.0.0.1:8000/docs — interactive Swagger UI
- http://127.0.0.1:8000/redoc — ReDoc

## Project structure

```
.
├── main.py                  # FastAPI app: routes, template/static config, post data
├── templates/
│   ├── layout.html          # Base template: head, navbar, sidebar, footer, theme toggle
│   └── home.html            # Post list, extends layout.html
├── static/
│   ├── css/main.css         # Custom styles on top of Bootstrap
│   ├── js/utils.js          # Placeholder for shared scripts
│   ├── icons/               # Favicons, touch icons, PWA icons
│   ├── profile_pics/        # Author avatars
│   └── site.webmanifest     # PWA manifest
├── pyproject.toml           # Project metadata and dependencies
└── uv.lock                  # Pinned dependency versions
```

## Roadmap

Ideas to build on as the project grows:

- [x] Render real templates with Jinja2 instead of inline HTML
- [ ] Add a detail route for a single post (`/posts/{id}`)
- [ ] Move posts into a database (SQLite via SQLModel)
- [ ] Add Pydantic models for request/response validation
- [ ] Support creating, updating, and deleting posts
- [ ] User accounts, so the Login and Register buttons do something
- [ ] Add tests with `pytest` and `httpx`
