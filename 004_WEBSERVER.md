

# Python Webserver Fundamentals (FastAPI) — updated Oct 5, 2025

Beginner‑friendly guide to build and run a modern Python web API using **FastAPI** and **Uvicorn**. Covers routing, parameters, status codes, models, error handling, middleware, and testing. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13.  
- Examples run inside an activated virtual environment.  
- Commands show input only. Output is shown without prompts.

---

## Table of Contents
- [0) Why FastAPI](#0-why-fastapi)
- [1) Create project and install deps](#1-create-project-and-install-deps)
- [2) Hello API and server run](#2-hello-api-and-server-run)
- [3) Routing: path and query params](#3-routing-path-and-query-params)
- [4) Request bodies and Pydantic models](#4-request-bodies-and-pydantic-models)
- [5) Responses, status codes, and headers](#5-responses-status-codes-and-headers)
- [6) Validation, defaults, and enums](#6-validation-defaults-and-enums)
- [7) Error handling](#7-error-handling)
- [8) Middleware (logging, CORS)](#8-middleware-logging-cors)
- [9) Dependencies and configuration via .env](#9-dependencies-and-configuration-via-env)
- [10) Static files and templates (optional)](#10-static-files-and-templates-optional)
- [11) Testing with pytest + httpx](#11-testing-with-pytest--httpx)
- [12) Production run options](#12-production-run-options)
- [13) Docker (optional)](#13-docker-optional)
- [14) Troubleshooting](#14-troubleshooting)
- [15) Recap](#15-recap)

---

## 0) Why FastAPI
**What**: Modern, async‑first web framework.  
**Why**: Type hints → automatic validation and OpenAPI docs. High performance with Uvicorn/ASGI. Clean developer experience.

Alternatives: Flask (simple WSGI), Django (full‑stack), Starlette (ASGI micro). In 2025, FastAPI is a safe default for APIs.

---

## 1) Create project and install deps
**What**: Create folders and install FastAPI + Uvicorn.  
**Why**: Minimal reproducible setup.
```sh
mkdir -p ~/Code_Stuff/fastapi-demo/src && cd ~/Code_Stuff/fastapi-demo
python3 -m venv .venv && source .venv/bin/activate
pip install --upgrade pip setuptools wheel
pip install fastapi uvicorn[standard] pydantic-settings python-dotenv
pip freeze > requirements.txt
```
Project shape:
```
fastapi-demo/
├── .venv/
├── .env
├── requirements.txt
├── src/
│   └── main.py
└── tests/
```

---

## 2) Hello API and server run
**What**: Minimal app and dev server.  
**Why**: Verify stack is working.

`src/main.py`:
```python
from fastapi import FastAPI

app = FastAPI(title="Demo API", version="0.1.0")

@app.get("/")
async def root():
    return {"message": "Hello, World!"}
```
Run the server:
```sh
uvicorn src.main:app --reload --port 8000
```
Open docs at:
```
http://127.0.0.1:8000/docs
```
The interactive Swagger UI is generated from type hints and route metadata.

---

## 3) Routing: path and query params
**What**: Define endpoints with dynamic segments and optional filters.  
**Why**: Core of HTTP APIs.
```python
from fastapi import FastAPI
from typing import Optional

app = FastAPI()

@app.get("/items/{item_id}")
async def get_item(item_id: int, q: Optional[str] = None):
    return {"item_id": item_id, "query": q}
```
- `item_id` is a **path parameter** converted to `int`.
- `q` is a **query parameter** with default `None`.

---

## 4) Request bodies and Pydantic models
**What**: Validate JSON bodies with Pydantic v2 models.  
**Why**: Type safety and auto‑docs.
```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float
    tags: list[str] = []

@app.post("/items", status_code=201)
async def create_item(item: Item):
    return {"ok": True, "item": item}
```
FastAPI coerces and validates input, returns a 422 on invalid body.

---

## 5) Responses, status codes, and headers
**What**: Control status, shape, and metadata of responses.  
**Why**: Correct API semantics.
```python
from fastapi import FastAPI, Response, status
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: int
    name: str

@app.get("/users/{uid}", response_model=User)
async def read_user(uid: int, response: Response):
    # pretend lookup
    if uid == 1:
        response.headers["X-From"] = "demo"
        return User(id=1, name="Ned")
    response.status_code = status.HTTP_404_NOT_FOUND
    return {"detail": "not found"}
```
`response_model` enforces output schema and filters extra fields.

---

## 6) Validation, defaults, and enums
**What**: Express constraints using type hints.  
**Why**: Early failure, better docs.
```python
from enum import Enum
from fastapi import FastAPI, Query, Path

app = FastAPI()

class Color(str, Enum):
    red = "red"
    green = "green"
    blue = "blue"

@app.get("/paint/{amount}")
async def paint(
    amount: int = Path(ge=1, le=100),
    color: Color = Query(default=Color.red),
):
    return {"amount": amount, "color": color}
```
Constraints return 422 with clear error details.

---

## 7) Error handling
**What**: Raise HTTP errors with details.  
**Why**: Consistent failure modes.
```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/fail")
async def fail():
    raise HTTPException(status_code=400, detail={"error": "bad input"})
```
Custom handlers:
```python
from fastapi.responses import JSONResponse
from starlette.requests import Request

@app.exception_handler(KeyError)
async def key_error_handler(_: Request, exc: KeyError):
    return JSONResponse(status_code=500, content={"detail": str(exc)})
```

---

## 8) Middleware (logging, CORS)
**What**: Run logic around every request.  
**Why**: Cross‑cutting concerns.
```python
import time
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

@app.middleware("http")
async def timing(request, call_next):
    t0 = time.perf_counter()
    resp = await call_next(request)
    resp.headers["X-Elapsed-ms"] = str(round((time.perf_counter()-t0)*1000, 2))
    return resp

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], allow_methods=["*"], allow_headers=["*"]
)
```

---

## 9) Dependencies and configuration via .env
**What**: Centralize config and resource wiring.  
**Why**: Testability and environment parity.
```python
from fastapi import Depends, FastAPI
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "Demo API"
    debug: bool = False
    database_url: str | None = None

    class Config:
        env_file = ".env"

settings = Settings()
app = FastAPI(title=settings.app_name, debug=settings.debug)

def get_settings():
    return settings

@app.get("/config")
async def get_config(cfg: Settings = Depends(get_settings)):
    return {"app_name": cfg.app_name, "debug": cfg.debug}
```
`.env` example:
```
APP_NAME=Prod API
DEBUG=true
DATABASE_URL=sqlite:///./local.db
```

---

## 10) Static files and templates (optional)
**What**: Serve assets or HTML pages.  
**Why**: Lightweight UI or health pages.
```sh
pip install jinja2 python-multipart
```
```python
from fastapi import FastAPI, Request, UploadFile, File
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

app = FastAPI()
app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

@app.get("/health")
async def health():
    return {"ok": True}

@app.get("/hello")
async def hello(request: Request):
    return templates.TemplateResponse("hello.html", {"request": request, "name": "Ned"})

@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    return {"filename": file.filename, "content_type": file.content_type}
```

---

## 11) Testing with pytest + httpx
**What**: Verify endpoints programmatically.  
**Why**: Prevent regressions.
```sh
pip install pytest httpx[http2]
```
`tests/test_api.py`:
```python
import pytest
from httpx import AsyncClient
from src.main import app

@pytest.mark.asyncio
async def test_root():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        r = await ac.get("/")
        assert r.status_code == 200
        assert r.json()["message"] == "Hello, World!"
```
Run:
```sh
pytest -q
```

---

## 12) Production run options
**What**: Start the ASGI server for real traffic.  
**Why**: Performance and reliability.
```sh
# dev hot‑reload
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000

# production (one worker per CPU core typical)
pip install gunicorn
gunicorn -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8000 src.main:app
```
Notes:
- Put `--workers` around number of CPU cores.
- Set `LOG_LEVEL=info` or higher in prod.
- Terminate TLS at a reverse proxy (nginx, Traefik, ALB) or use a PaaS.

---

## 13) Docker (optional)
**What**: Containerize the app for consistent deploys.
```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY src ./src
ENV PORT=8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
Build and run:
```sh
docker build -t fastapi-demo .
docker run -p 8000:8000 --env-file .env fastapi-demo
```

---

## 14) Troubleshooting
- **Cannot import `src.main`**: run from project root or set `PYTHONPATH=.`
- **422 Unprocessable Entity**: request body or params fail validation; check types and required fields.
- **CORS blocked by browser**: add `CORSMiddleware` with allowed origins.
- **Hot reload not working**: ensure `--reload` and run from project root.
- **Port in use**: change `--port` or stop old process.

---

## 15) Recap
```plaintext
Install deps → Create app → Define routes → Validate with models → Handle errors → Add middleware → Configure via .env → Test → Run in prod
```

**Next**: Add auth (JWT/OAuth2), database (SQLModel/SQLAlchemy), and background tasks (Celery/RQ) as needed. Keep endpoints small and typed.