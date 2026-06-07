# Around

A location-based social media REST API that lets authenticated users share posts (images/videos) at geographic coordinates and discover nearby content.

## Features

- **Post creation** — Upload images or videos with a location, stored in Google Cloud Storage
- **Geo-spatial search** — Find posts within a configurable radius of any coordinate
- **Face clustering** — Query posts by face-detection confidence score via Google Cloud Vision
- **JWT authentication** — Stateless auth on all protected routes

## Architecture

```
Client → Gorilla Mux Router → JWT Middleware (protected routes)
                           ↓
                      Handlers
                     /   |    \
          Google    ES   GCS   Google
          Vision API     ↑     Storage
                    Elasticsearch
```

| Component | Technology |
|---|---|
| Language | Go |
| HTTP router | Gorilla Mux |
| Database / search | Elasticsearch |
| Media storage | Google Cloud Storage |
| Image analysis | Google Cloud Vision API |
| Auth | JWT (HS256) via Auth0 middleware |
| Container | Docker |
| CI/CD | CircleCI |

## Prerequisites

- Go 1.13+
- A running Elasticsearch instance
- A Google Cloud project with:
  - Cloud Storage bucket
  - Cloud Vision API enabled
  - Application Default Credentials configured (`GOOGLE_APPLICATION_CREDENTIALS`)

## Configuration

All configuration is in `main.go` as constants:

| Constant | Default | Description |
|---|---|---|
| `ES_URL` | `http://10.128.0.2:9200` | Elasticsearch URL |
| `BUCKET_NAME` | `ryia-bucket` | GCS bucket for media uploads |
| `POST_INDEX` | `post` | Elasticsearch index for posts |
| `DISTANCE` | `200km` | Default geo-search radius |

The JWT signing key is set in `user.go`:

```go
var mySigningKey = []byte("secret")
```

> **Note:** Replace this with a strong, environment-sourced secret before deploying.

## Running Locally

```bash
# 1. Set up Elasticsearch indices (one-time)
go run index/index.go

# 2. Start the API server
go run main.go handler.go user.go vision.go
```

The server listens on `:8080`.

## Running with Docker

```bash
docker build -t around .
docker run -p 8080:8080 \
  -e GOOGLE_APPLICATION_CREDENTIALS=/credentials.json \
  -v /path/to/credentials.json:/credentials.json \
  around
```

## API Reference

### Authentication

Protected routes require a `Bearer` token in the `Authorization` header:

```
Authorization: Bearer <token>
```

---

### POST /signup

Register a new user.

**Request body (JSON):**
```json
{
  "username": "alice",
  "password": "secret123",
  "age": 25,
  "gender": "female"
}
```

**Constraints:**
- `username` must be non-empty and contain only lowercase letters and digits (`[a-z0-9]+`)
- `password` must be non-empty

**Responses:**

| Status | Meaning |
|---|---|
| 200 | User created successfully |
| 400 | Invalid input or username already exists |
| 500 | Elasticsearch error |

---

### POST /login

Authenticate and receive a JWT token.

**Request body (JSON):**
```json
{
  "username": "alice",
  "password": "secret123"
}
```

**Response (200):** Plain-text JWT token valid for 24 hours.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

| Status | Meaning |
|---|---|
| 200 | Token returned |
| 401 | Bad credentials |
| 500 | Internal error |

---

### POST /post *(protected)*

Create a new post with a media file.

**Content-Type:** `multipart/form-data`

| Field | Type | Description |
|---|---|---|
| `lat` | float | Latitude |
| `lon` | float | Longitude |
| `message` | string | Post text |
| `image` | file | Image or video (`.jpg`, `.jpeg`, `.png`, `.gif`, `.mp4`, `.mov`, `.avi`, `.flv`, `.wmv`) |

**Responses:**

| Status | Meaning |
|---|---|
| 200 | Post saved |
| 400 | Image missing or invalid |
| 500 | GCS, Vision API, or Elasticsearch error |

---

### GET /search *(protected)*

Find posts near a location.

**Query parameters:**

| Parameter | Required | Description |
|---|---|---|
| `lat` | Yes | Latitude |
| `lon` | Yes | Longitude |
| `range` | No | Radius in km (default: `200`) |

**Response (200, JSON):**
```json
[
  {
    "user": "alice",
    "message": "Beautiful sunset!",
    "location": { "lat": 37.7749, "lon": -122.4194 },
    "url": "https://storage.googleapis.com/...",
    "type": "image",
    "face": 0.98
  }
]
```

---

### GET /cluster *(protected)*

Retrieve posts filtered by a numeric field threshold (≥ 0.9).

**Query parameters:**

| Parameter | Description |
|---|---|
| `term` | Elasticsearch field name to range-query (e.g., `face`) |

**Response (200, JSON):** Array of matching `Post` objects (same schema as `/search`).

---

## Data Models

### Post

```go
type Post struct {
    User     string   `json:"user"`
    Message  string   `json:"message"`
    Location Location `json:"location"`
    Url      string   `json:"url"`
    Type     string   `json:"type"` // "image" | "video" | "unknown"
    Face     float32  `json:"face"` // face detection confidence [0.0, 1.0]
}
```

### User

```go
type User struct {
    Username string `json:"username"`
    Password string `json:"password"`
    Age      int64  `json:"age"`
    Gender   string `json:"gender"`
}
```

## Elasticsearch Index Setup

Before first use, run the index initializer to create the required mappings:

```bash
go run index/index.go
```

This creates two indices:
- **`post`** — with a `geo_point` mapping on `location`
- **`user`** — with `keyword` mapping on `username` and `password`

## CI/CD

CircleCI builds and pushes a Docker image to DockerHub on every push. Required environment variables in CircleCI:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_PASSWORD`

## Project Structure

```
Around/
├── main.go         # Entry point, router and JWT middleware setup
├── handler.go      # HTTP handlers and GCS/Elasticsearch helpers
├── user.go         # User auth handlers and JWT token generation
├── vision.go       # Google Cloud Vision face detection wrapper
├── index/
│   └── index.go    # Elasticsearch index initializer (standalone)
├── Dockerfile
└── .circleci/
    └── config.yml
```

## Known Limitations

- Passwords are stored in plain text in Elasticsearch; production deployments should hash passwords (e.g., bcrypt).
- The JWT signing key is hardcoded; use an environment variable in production.
- Media files are set to public-read ACL on GCS upload.
- No file size limit is enforced on uploads.
