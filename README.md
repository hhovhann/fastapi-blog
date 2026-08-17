# fastapi-blog

A small blog application built with [FastAPI](https://fastapi.tiangolo.com/) — a learning
project for working through the framework's routing, response types, and automatic API docs.

Posts currently live in an in-memory list in `main.py`; there is no database yet.

## Endpoints

| Method | Path         | Response  | Description                                  |
| ------ | ------------ | --------- | -------------------------------------------- |
| `GET`  | `/`          | HTML      | Home page                                    |
| `GET`  | `/posts`     | HTML      | Same handler as `/`                          |
| `GET`  | `/api/posts` | JSON      | All posts                                    |

The HTML routes are excluded from the OpenAPI schema, so `/docs` shows only `/api/posts`.

## Requirements

- Python 3.13 (see `.python-version`)
- [uv](https://docs.astral.sh/uv/) for dependency management

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
├── main.py          # FastAPI app: routes and in-memory post data
├── snippets.txt     # Scratch notes kept while following along
├── pyproject.toml   # Project metadata and dependencies
└── uv.lock          # Pinned dependency versions
```

## Roadmap

Ideas to build on as the project grows:

- [ ] Render real templates with Jinja2 instead of inline HTML
- [ ] Add a detail route for a single post (`/posts/{id}`)
- [ ] Move posts into a database (SQLite via SQLModel)
- [ ] Add Pydantic models for request/response validation
- [ ] Support creating, updating, and deleting posts
- [ ] Add tests with `pytest` and `httpx`
