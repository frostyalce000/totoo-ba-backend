# AI RAG Product Checker

AI RAG Product Checker is a FastAPI-based service for verifying FDA Philippines-regulated products using database lookups and AI-powered image analysis. The application supports drugs, food products, medical devices, cosmetics, and establishments.

## Features

- **ID-based verification**: Check products by registration numbers, license numbers, or tracking numbers.
- **Hybrid image verification**: Upload product images for AI extraction and database matching using Groq vision models.
- **Fuzzy matching**: Similarity scoring with PostgreSQL full-text search and trigram matching.
- **Multi-category support**: Drugs, food, medical devices, cosmetics, and establishments.
- **RESTful API**: FastAPI endpoints with Pydantic validation and automatic OpenAPI documentation.
- **High performance**: uvloop, httptools, asyncpg connection pooling, and orjson response serialization.

## Performance Optimizations

The application is tuned for high-throughput async workloads:

- **uvloop**: Drop-in asyncio event loop replacement (2–4× throughput improvement in production).
- **httptools**: Fast HTTP request/response parsing.
- **asyncpg**: Async PostgreSQL driver with connection pooling.
- **orjson**: Fast JSON serialization via `ORJSONResponse`.
- **GZip middleware**: Compresses responses larger than 1 KB.

In development mode, hot reload runs without uvloop/httptools to avoid conflicts. Production mode enables both with multiple workers.

## Project Structure

```
├── app/
│   ├── api/
│   │   ├── deps.py
│   │   ├── products.py
│   │   └── repository/
│   │       ├── database_repository.py
│   │       └── products_repository.py
│   ├── core/
│   │   ├── config.py
│   │   ├── database.py
│   │   └── logging.py
│   ├── models/
│   ├── schemas/
│   ├── services/
│   │   ├── vision_service.py
│   │   ├── product_verification_service.py
│   │   ├── extractor.py
│   │   └── upsert_extractors/
│   ├── utils/
│   └── main.py
├── bruno_api_testing/
├── migrations/
├── tests/
├── .env.example
├── pyproject.toml
├── requirements.txt
├── run_production.py
└── README.md
```

- **app/api**: API routes and dependency injection.
- **app/core**: Settings, database setup, and Loguru logging configuration.
- **app/models**: SQLAlchemy database models.
- **app/schemas**: Pydantic schemas for request/response validation.
- **app/services**: Product verification, vision processing, and data extractors.
- **app/utils**: Shared helper functions.
- **bruno_api_testing**: Bruno API test collection.
- **migrations**: SQL scripts for FTS indexes and PostgreSQL extensions.
- **tests**: Pytest test suite.

## Prerequisites

- Python 3.9 or higher
- PostgreSQL
- pip

## Installation

1. Clone the repository and enter the project directory.

2. Create and activate a virtual environment:

   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. Install dependencies:

   ```bash
   pip install -r requirements.txt
   ```

   For development tools (Ruff, pytest, coverage):

   ```bash
   pip install -e ".[dev]"
   ```

4. Copy the example environment file:

   ```bash
   cp .env.example .env
   ```

## Configuration

Update `.env` with your local settings. Key variables:

```
# Environment
ENVIRONMENT=development
DEBUG=true

# Database
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/product_checker_dev
DATABASE_ECHO=true
DATABASE_POOL_SIZE=10

# Security
SECRET_KEY=your-dev-secret-key-change-in-production

# API
API_PREFIX=/api/v1
DOCS_URL=/docs

# AI
GROQ_API_KEY=
CEREBRAS_API_KEY=

# CORS
CORS_ORIGINS=http://localhost:3000,http://localhost:8000,http://127.0.0.1:3000

# FDA Scraper
FDA_BASE_URL=https://verification.fda.gov.ph
FDA_TIMEOUT=30
FDA_MAX_PAGES_PER_RUN=10
FDA_RATE_LIMIT_DELAY=1.0

# Fuzzy Matching
FUZZY_MATCH_THRESHOLD=80
FUZZY_MATCH_LIMIT=5

# Caching
CACHE_ENABLED=true
CACHE_TTL_MINUTES=30
CACHE_MAX_SIZE=1000

# Logging (Loguru)
LOG_LEVEL=DEBUG
LOG_FILE=logs/app.log
LOG_ROTATION=500 MB
LOG_RETENTION=10 days
```

Generate a secure secret key:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

### Database Setup

1. Ensure PostgreSQL is running.
2. Create the database named in `DATABASE_URL`.
3. Tables are created automatically on startup.
4. Apply optional migration scripts in `migrations/` for full-text search indexes and the `pg_trgm` extension.

## Usage

### Running the Application

The recommended entry point is `run_production.py`, which selects development or production settings based on `ENVIRONMENT`:

```bash
python run_production.py
```

- **Development** (`ENVIRONMENT=development`): Single worker with hot reload on port **6769**.
- **Production** (`ENVIRONMENT=production`): Four workers with uvloop and httptools on port **6769**.

Alternatively, run uvicorn directly:

```bash
# Development
uvicorn app.main:app --reload --host 0.0.0.0 --port 6769

# Production
uvicorn app.main:app --host 0.0.0.0 --port 6769 --loop uvloop --http httptools --workers 4
```

Once running, interactive API docs are available at `/docs` and `/redoc`.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Application name, version, and status |
| GET | `/health` | Health check with environment and database status |
| GET | `/api/v1/products/verify/{product_id}` | Verify a product by registration, license, or tracking number |
| POST | `/api/v1/products/new-verify-image` | Verify a product from an uploaded image (hybrid Groq vision pipeline) |

The image verification endpoint accepts JPEG, PNG, GIF, or WebP files up to 5 MB. It runs a three-layer pipeline: vision extraction, structured field parsing, and fuzzy database matching.

## Development

### Code Quality

This project uses Ruff for linting and formatting. Configuration lives in `pyproject.toml`.

```bash
ruff check . --fix
ruff format .
```

See `LINTER.md` for details.

### Running Tests

```bash
pytest
```

Tests use pytest with asyncio support and coverage reporting for the `app` package.

### Development Environment

Recommended `.env` values for local work:

```
ENVIRONMENT=development
DEBUG=true
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/product_checker_dev
LOG_LEVEL=DEBUG
```

## Production Deployment

Before deploying:

- Set `ENVIRONMENT=production`
- Set `DEBUG=false`
- Use a strong, unique `SECRET_KEY`
- Set `LOG_DIAGNOSE=false`
- Configure file logging if needed (`LOG_FILE`)
- Run migration scripts against the production database

Start the server:

```bash
python run_production.py
```

## Troubleshooting

**Database connection errors** — Confirm PostgreSQL is running and `DATABASE_URL` uses the `postgresql+asyncpg://` driver prefix.

**Missing dependencies** — Activate the virtual environment and reinstall from `requirements.txt`.

**AI verification failures** — Verify `GROQ_API_KEY` is set and the service can reach Groq APIs.

**Production restart loop** — Do not combine hot reload with uvloop/httptools. Use `run_production.py`, which applies the correct settings per environment.

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 400 | Invalid request data |
| 404 | Product not found |
| 422 | Validation error |
| 500 | Internal server error |

## Contributing

See `CONTRIBUTING.md` for commit conventions, issue templates, and pull request guidelines.

## License

This project is licensed under the MIT License. See `LICENSE` for the full text.

## Acknowledgments

- FastAPI for the web framework
- SQLAlchemy and asyncpg for database access
- Groq vision and language models for AI capabilities
- Contributors to the open-source libraries used in this project
